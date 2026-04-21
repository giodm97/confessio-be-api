---
repo: giodm97/confessio-be-api
title: "Sprint 2 — AI client: AiConfig, AiClient, AiResponse, AiConstants"
labels: enhancement
status: pending
---

## Overview

Implement the Groq AI client layer following the same pattern as peekly-be-api.
Includes config bean, RestClient-based client, response record, and constants
with the Father Confessio system prompt.

## Changes required

### 1. `src/main/java/com/confessio/api/constants/AiConstants.java`

```java
public final class AiConstants {

    private AiConstants() {}

    public static final String CHAT_COMPLETIONS_URI = "/chat/completions";

    public static final String KEY_MODEL      = "model";
    public static final String KEY_MAX_TOKENS = "max_tokens";
    public static final String KEY_MESSAGES   = "messages";
    public static final String KEY_ROLE       = "role";
    public static final String KEY_CONTENT    = "content";
    public static final String ROLE_SYSTEM    = "system";
    public static final String ROLE_USER      = "user";

    public static final String SYSTEM_PROMPT = """
            You are Father Confessio, a compassionate and non-judgmental AI priest.\
            Your role is to receive confessions and offer absolution with penance.\

            Rules:\
            - Always respond in the same language the user writes in.\
            - Never judge harshly. Be warm, forgiving, and pastoral.\
            - Assign prayers proportional to the gravity of the sins confessed.\
            - Always respond ONLY with valid JSON — no extra text before or after.\

            Response format (strict JSON):\
            {\
              "absolution": "<2-4 sentence absolution text>",\
              "prayers": {\
                "hailMary": <int>,\
                "ourFather": <int>,\
                "gloryBe": <int>\
              },\
              "gravityScore": <int 1-5>,\
              "sinCategories": ["<category1>", "<category2>"]\
            }\

            gravityScore: 1 = venial/light, 5 = very serious.\
            sinCategories: normalized lowercase labels\
              (e.g. ira, lussuria, superbia, avarizia, menzogna, invidia, gola, accidia).\
            prayers: at least 1 total. Max: hailMary=10, ourFather=5, gloryBe=5.\
            """;

    public static final String LOG_AI_RETRY = "AI call failed, retrying once. Reason: {}";
}
```

### 2. `src/main/java/com/confessio/api/config/AiConfig.java`

```java
@Configuration
public class AiConfig {

    @Value("${ai.api-key}")
    private String apiKey;

    @Value("${ai.base-url}")
    private String baseUrl;

    @Bean
    public RestClient aiRestClient() {
        return RestClient.builder()
                .baseUrl(baseUrl)
                .defaultHeader(
                    SecurityConstants.AUTHORIZATION_HEADER,
                    SecurityConstants.BEARER_PREFIX + apiKey)
                .defaultHeader(
                    SecurityConstants.CONTENT_TYPE_HEADER,
                    SecurityConstants.APPLICATION_JSON)
                .build();
    }
}
```

### 3. `src/main/java/com/confessio/api/ai/AiResponse.java`

```java
@JsonIgnoreProperties(ignoreUnknown = true)
public record AiResponse(List<Choice> choices) {

    @JsonIgnoreProperties(ignoreUnknown = true)
    public record Choice(Message message) {}

    @JsonIgnoreProperties(ignoreUnknown = true)
    public record Message(String content) {}
}
```

### 4. `src/main/java/com/confessio/api/ai/AiClient.java`

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class AiClient {

    private final RestClient aiRestClient;

    @Value("${ai.model}")
    private String model;

    @Value("${ai.max-tokens:400}")
    private int maxTokens;

    public String complete(String systemPrompt, String userPrompt) {
        Map<String, Object> request = Map.of(
                AiConstants.KEY_MODEL, model,
                AiConstants.KEY_MAX_TOKENS, maxTokens,
                AiConstants.KEY_MESSAGES, List.of(
                        Map.of(AiConstants.KEY_ROLE, AiConstants.ROLE_SYSTEM,
                               AiConstants.KEY_CONTENT, systemPrompt),
                        Map.of(AiConstants.KEY_ROLE, AiConstants.ROLE_USER,
                               AiConstants.KEY_CONTENT, userPrompt)
                )
        );
        try {
            return extractContent(aiRestClient.post()
                    .uri(AiConstants.CHAT_COMPLETIONS_URI)
                    .body(request)
                    .retrieve()
                    .body(AiResponse.class));
        } catch (Exception e) {
            log.warn(AiConstants.LOG_AI_RETRY, e.getMessage());
            return extractContent(aiRestClient.post()
                    .uri(AiConstants.CHAT_COMPLETIONS_URI)
                    .body(request)
                    .retrieve()
                    .body(AiResponse.class));
        }
    }

    private String extractContent(AiResponse response) {
        return response.choices().get(0).message().content();
    }
}
```

## Acceptance criteria

- [ ] `./mvnw verify -DskipTests` completes without errors
- [ ] `AiClient` compiles and injects `RestClient` from `AiConfig`
- [ ] `SYSTEM_PROMPT` in `AiConstants` contains the Father Confessio persona
- [ ] `ai.api-key` property has no default (fail-fast if not set)
- [ ] `ai.base-url` and `ai.model` have defaults so app starts locally without env vars
