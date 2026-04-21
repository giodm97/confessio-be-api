---
repo: giodm97/confessio-be-api
title: "Sprint 2 — Confession API: DTOs, service, controller, tests"
labels: enhancement
status: pending
base_branch: develop
---

## Overview

Implement POST /confession: receives confession text + optional uuid, calls Groq AI,
parses JSON response, optionally persists sin categories, returns absolution.
Confession text is NEVER stored or logged.

## Changes required

### 1. `src/main/java/com/confessio/api/dto/request/ConfessionRequest.java`

```java
public record ConfessionRequest(
    String uuid,
    @NotBlank String confessionText,
    List<String> sinCategories
) {}
```

### 2. `src/main/java/com/confessio/api/dto/response/PrayersResponse.java`

```java
public record PrayersResponse(
    int hailMary,
    int ourFather,
    int gloryBe
) {}
```

### 3. `src/main/java/com/confessio/api/dto/response/AbsolutionResponse.java`

```java
public record AbsolutionResponse(
    String absolution,
    PrayersResponse prayers,
    int gravityScore
) {}
```

### 4. `src/main/java/com/confessio/api/ai/AiAbsolutionResponse.java`

Internal record used to parse the AI JSON response. Not exposed to the client.

```java
@JsonIgnoreProperties(ignoreUnknown = true)
public record AiAbsolutionResponse(
    String absolution,
    Prayers prayers,
    int gravityScore,
    List<String> sinCategories
) {
    @JsonIgnoreProperties(ignoreUnknown = true)
    public record Prayers(int hailMary, int ourFather, int gloryBe) {}
}
```

### 5. `src/main/java/com/confessio/api/service/IConfessionService.java`

```java
public interface IConfessionService {
    AbsolutionResponse confess(ConfessionRequest request);
}
```

### 6. `src/main/java/com/confessio/api/service/ConfessionService.java`

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class ConfessionService implements IConfessionService {

    private final AiClient aiClient;
    private final UserProfileRepository profileRepository;
    private final SinEntryRepository sinEntryRepository;
    private final ObjectMapper objectMapper;

    @Override
    @Transactional
    public AbsolutionResponse confess(ConfessionRequest request) {
        String rawJson = aiClient.complete(
            AiConstants.SYSTEM_PROMPT, request.confessionText());

        AiAbsolutionResponse aiResponse = parseAiResponse(rawJson);

        if (request.uuid() != null) {
            profileRepository.findById(request.uuid())
                    .ifPresent(profile ->
                        persistSinCategories(profile, aiResponse));
        }

        return new AbsolutionResponse(
                aiResponse.absolution(),
                new PrayersResponse(
                    aiResponse.prayers().hailMary(),
                    aiResponse.prayers().ourFather(),
                    aiResponse.prayers().gloryBe()),
                aiResponse.gravityScore());
    }

    private AiAbsolutionResponse parseAiResponse(String rawJson) {
        try {
            String json = rawJson.replaceAll("(?s)^[^\\{]*(\\{.*\\})[^\\}]*$", "$1");
            return objectMapper.readValue(json, AiAbsolutionResponse.class);
        } catch (Exception e) {
            throw new RuntimeException("Failed to parse AI absolution response", e);
        }
    }

    private void persistSinCategories(
            UserProfile profile, AiAbsolutionResponse aiResponse) {
        int weight = Math.max(1, Math.min(5, aiResponse.gravityScore()));
        List<SinEntry> entries = aiResponse.sinCategories().stream()
                .map(cat -> SinEntry.builder()
                        .userProfile(profile)
                        .category(cat.toLowerCase())
                        .weight(weight)
                        .build())
                .toList();
        sinEntryRepository.saveAll(entries);
    }
}
```

### 7. `src/main/java/com/confessio/api/controller/ConfessionController.java`

```java
@RestController
@RequestMapping(ControllerConstants.CONFESSION_BASE)
@RequiredArgsConstructor
@Tag(name = "Confession", description = "AI confession endpoint")
public class ConfessionController {

    private final IConfessionService confessionService;

    @PostMapping
    @Operation(summary = "Submit confession and receive absolution")
    public ResponseEntity<AbsolutionResponse> confess(
            @Valid @RequestBody ConfessionRequest request) {
        return ResponseEntity.ok(confessionService.confess(request));
    }
}
```

### 8. `src/test/java/com/confessio/api/service/ConfessionServiceTest.java`

`@ExtendWith(MockitoExtension.class)`. Test scenarios:
- `confess_returnsAbsolution_withoutPersisting_whenNoUuid`: uuid is null,
  `sinEntryRepository` must not be called.
- `confess_persistsSinCategories_whenUuidPresent`: uuid present, profile found,
  verify `sinEntryRepository.saveAll(...)` is called.
- `confess_throwsRuntimeException_whenAiResponseMalformed`: `AiClient` returns
  invalid JSON, expect `RuntimeException`.

### 9. `src/test/java/com/confessio/api/controller/ConfessionControllerTest.java`

`@WebMvcTest(ConfessionController.class)` + `@Import(SecurityConfig.class)`.
Test: POST with valid body returns 200 with absolution. POST with missing
`confessionText` returns 400.

## Acceptance criteria

- [ ] `./mvnw verify` passes (including all tests)
- [ ] `POST /api/confession` returns absolution JSON with `prayers` and `gravityScore`
- [ ] `confessionText` does not appear in any log statement in `ConfessionService`
- [ ] When `uuid` is null, no DB writes occur (verified by test)
- [ ] `parseAiResponse` handles JSON wrapped in extra text (regex strip)
- [ ] All service and controller tests pass
