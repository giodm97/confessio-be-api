---
repo: giodm97/confessio-be-api
title: "Sprint 1 — SecurityConfig, exception handling, constants"
labels: enhancement
status: pending
base_branch: develop
---

## Overview

Add the infrastructure layer: permit-all security config (Sprint 1), global exception
handler, ResourceNotFoundException, and all constants classes.

## Changes required

### 1. `src/main/java/com/confessio/api/config/SecurityConfig.java`

```java
@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()
                .anyRequest().permitAll()
            );
        return http.build();
    }
}
```

### 2. `src/main/java/com/confessio/api/exception/ResourceNotFoundException.java`

```java
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}
```

### 3. `src/main/java/com/confessio/api/exception/GlobalExceptionHandler.java`

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<Map<String, Object>> handleNotFound(
            ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(errorBody(ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, Object>> handleValidation(
            MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
                .map(e -> e.getField() + ": " + e.getDefaultMessage())
                .collect(Collectors.joining(", "));
        return ResponseEntity.badRequest().body(errorBody(message));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<Map<String, Object>> handleGeneric(Exception ex) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(errorBody("Internal server error"));
    }

    private Map<String, Object> errorBody(String message) {
        return Map.of("timestamp", Instant.now().toString(), "error", message);
    }
}
```

### 4. `src/main/java/com/confessio/api/constants/ControllerConstants.java`

```java
public final class ControllerConstants {

    private ControllerConstants() {}

    public static final String PROFILE_BASE    = "/profile";
    public static final String CONFESSION_BASE = "/confession";
    public static final String SESSION_PATH    = "/{uuid}/session";
}
```

### 5. `src/main/java/com/confessio/api/constants/ServiceConstants.java`

```java
public final class ServiceConstants {

    private ServiceConstants() {}

    public static final String PROFILE_NOT_FOUND = "Profile not found: ";
}
```

### 6. `src/main/java/com/confessio/api/constants/SecurityConstants.java`

```java
public final class SecurityConstants {

    private SecurityConstants() {}

    public static final String AUTHORIZATION_HEADER  = "Authorization";
    public static final String BEARER_PREFIX         = "Bearer ";
    public static final String CONTENT_TYPE_HEADER   = "Content-Type";
    public static final String APPLICATION_JSON      = "application/json";
}
```

## Acceptance criteria

- [ ] `./mvnw verify -DskipTests` completes without errors
- [ ] `SecurityConfig` disables CSRF and permits all requests
- [ ] `GlobalExceptionHandler` handles `ResourceNotFoundException`, validation,
      and generic exceptions
- [ ] All three constants classes compile without errors
