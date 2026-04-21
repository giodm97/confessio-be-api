---
repo: giodm97/confessio-be-api
title: "Sprint 1 — Profile API: DTOs, mapper, service, controller, tests"
labels: enhancement
status: pending
---

## Overview

Implement the full profile CRUD layer: request/response DTOs, MapStruct mapper,
service interface + implementation, REST controller, and unit tests.
Endpoints: POST /profile, GET /profile/{uuid}, PUT /profile/{uuid}/session,
DELETE /profile/{uuid}.

## Changes required

### 1. `src/main/java/com/confessio/api/dto/request/ProfileRequest.java`

```java
public record ProfileRequest(
    @NotBlank String uuid
) {}
```

### 2. `src/main/java/com/confessio/api/dto/response/SinSummaryItem.java`

```java
public record SinSummaryItem(
    String category,
    int totalWeight,
    long occurrences
) {}
```

### 3. `src/main/java/com/confessio/api/dto/response/ProfileResponse.java`

```java
public record ProfileResponse(
    String uuid,
    int sessionCount,
    List<SinSummaryItem> sinSummary,
    Instant createdAt
) {}
```

### 4. `src/main/java/com/confessio/api/dto/response/SessionResponse.java`

```java
public record SessionResponse(
    String uuid,
    int sessionCount
) {}
```

### 5. `src/main/java/com/confessio/api/mapper/ProfileMapper.java`

```java
@Mapper(componentModel = "spring")
public interface ProfileMapper {

    @Mapping(target = "sessionCount", constant = "0")
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "sinEntries", ignore = true)
    UserProfile toEntity(ProfileRequest request);

    @Mapping(target = "sinSummary", ignore = true)
    ProfileResponse toResponse(UserProfile profile);
}
```

### 6. `src/main/java/com/confessio/api/service/IProfileService.java`

```java
public interface IProfileService {
    ProfileResponse createProfile(ProfileRequest request);
    ProfileResponse getProfile(String uuid);
    SessionResponse incrementSession(String uuid);
    void deleteProfile(String uuid);
}
```

### 7. `src/main/java/com/confessio/api/service/ProfileService.java`

```java
@Service
@RequiredArgsConstructor
public class ProfileService implements IProfileService {

    private final UserProfileRepository profileRepository;
    private final SinEntryRepository sinEntryRepository;
    private final ProfileMapper profileMapper;

    @Override
    @Transactional
    public ProfileResponse createProfile(ProfileRequest request) {
        UserProfile profile = profileMapper.toEntity(request);
        return buildResponse(profileRepository.save(profile));
    }

    @Override
    @Transactional(readOnly = true)
    public ProfileResponse getProfile(String uuid) {
        return buildResponse(findOrThrow(uuid));
    }

    @Override
    @Transactional
    public SessionResponse incrementSession(String uuid) {
        UserProfile profile = findOrThrow(uuid);
        profile.setSessionCount(profile.getSessionCount() + 1);
        profileRepository.save(profile);
        return new SessionResponse(profile.getUuid(), profile.getSessionCount());
    }

    @Override
    @Transactional
    public void deleteProfile(String uuid) {
        if (!profileRepository.existsById(uuid)) {
            throw new ResourceNotFoundException(
                ServiceConstants.PROFILE_NOT_FOUND + uuid);
        }
        profileRepository.deleteById(uuid);
    }

    private UserProfile findOrThrow(String uuid) {
        return profileRepository.findById(uuid)
                .orElseThrow(() -> new ResourceNotFoundException(
                    ServiceConstants.PROFILE_NOT_FOUND + uuid));
    }

    private ProfileResponse buildResponse(UserProfile profile) {
        List<SinEntry> entries =
            sinEntryRepository.findByUserProfileUuid(profile.getUuid());

        List<SinSummaryItem> summary = entries.stream()
                .collect(Collectors.groupingBy(SinEntry::getCategory))
                .entrySet().stream()
                .map(e -> new SinSummaryItem(
                        e.getKey(),
                        e.getValue().stream()
                            .mapToInt(SinEntry::getWeight).sum(),
                        e.getValue().size()))
                .sorted(Comparator.comparingInt(
                    SinSummaryItem::totalWeight).reversed())
                .toList();

        ProfileResponse base = profileMapper.toResponse(profile);
        return new ProfileResponse(
            base.uuid(), base.sessionCount(), summary, base.createdAt());
    }
}
```

### 8. `src/main/java/com/confessio/api/controller/ProfileController.java`

```java
@RestController
@RequestMapping(ControllerConstants.PROFILE_BASE)
@RequiredArgsConstructor
@Tag(name = "Profile", description = "Pseudonymous user profile management")
public class ProfileController {

    private final IProfileService profileService;

    @PostMapping
    @Operation(summary = "Create a pseudonymous profile")
    public ResponseEntity<ProfileResponse> createProfile(
            @Valid @RequestBody ProfileRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(profileService.createProfile(request));
    }

    @GetMapping("/{uuid}")
    @Operation(summary = "Get profile with sin summary")
    public ResponseEntity<ProfileResponse> getProfile(
            @PathVariable String uuid) {
        return ResponseEntity.ok(profileService.getProfile(uuid));
    }

    @PutMapping(ControllerConstants.SESSION_PATH)
    @Operation(summary = "Increment session count")
    public ResponseEntity<SessionResponse> incrementSession(
            @PathVariable String uuid) {
        return ResponseEntity.ok(profileService.incrementSession(uuid));
    }

    @DeleteMapping("/{uuid}")
    @Operation(summary = "Delete profile and all data (GDPR Art. 17)")
    public ResponseEntity<Void> deleteProfile(@PathVariable String uuid) {
        profileService.deleteProfile(uuid);
        return ResponseEntity.noContent().build();
    }
}
```

### 9. `src/test/java/com/confessio/api/service/ProfileServiceTest.java`

Test the following scenarios with `@ExtendWith(MockitoExtension.class)`:
- `createProfile_savesAndReturnsProfile`
- `getProfile_throwsWhenNotFound`
- `incrementSession_incrementsCountAndSaves`
- `deleteProfile_throwsWhenNotFound`

### 10. `src/test/java/com/confessio/api/controller/ProfileControllerTest.java`

`@WebMvcTest(ProfileController.class)` + `@Import(SecurityConfig.class)`.
Test: POST returns 201, GET returns 200, GET unknown uuid returns 404, DELETE returns 204.

## Acceptance criteria

- [ ] `./mvnw verify` passes (including tests)
- [ ] `POST /api/profile` returns 201 with profile body
- [ ] `GET /api/profile/{uuid}` returns 200 with `sinSummary: []` for new profiles
- [ ] `DELETE /api/profile/{uuid}` returns 204
- [ ] `GET /api/profile/nonexistent` returns 404 with error body
- [ ] All `ProfileServiceTest` and `ProfileControllerTest` tests pass
