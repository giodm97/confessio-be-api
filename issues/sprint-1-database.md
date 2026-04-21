---
repo: giodm97/confessio-be-api
title: "Sprint 1 — Entities, Flyway migration, repositories"
labels: enhancement
status: pending
base_branch: develop
---

## Overview

Create the two JPA entities (`UserProfile`, `SinEntry`), the Flyway V1 migration,
and the Spring Data repositories. No confession text is ever stored — only abstract
sin category labels.

## Changes required

### 1. `src/main/resources/db/migration/V1__init.sql`

```sql
CREATE TABLE user_profiles (
    uuid          VARCHAR(36)  PRIMARY KEY,
    session_count INT          NOT NULL DEFAULT 0,
    created_at    TIMESTAMPTZ  NOT NULL
);

CREATE TABLE sin_entries (
    id          BIGSERIAL    PRIMARY KEY,
    user_uuid   VARCHAR(36)  NOT NULL
                    REFERENCES user_profiles(uuid) ON DELETE CASCADE,
    category    VARCHAR(100) NOT NULL,
    weight      INT          NOT NULL CHECK (weight BETWEEN 1 AND 5),
    created_at  TIMESTAMPTZ  NOT NULL
);
```

### 2. `src/main/java/com/confessio/api/entity/UserProfile.java`

```java
@Entity
@Table(name = "user_profiles")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class UserProfile {

    @Id
    @Column(length = 36)
    private String uuid;

    @Column(name = "session_count", nullable = false)
    private int sessionCount;

    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    @OneToMany(mappedBy = "userProfile",
               cascade = CascadeType.ALL, orphanRemoval = true)
    private List<SinEntry> sinEntries = new ArrayList<>();

    @PrePersist
    private void prePersist() { createdAt = Instant.now(); }
}
```

### 3. `src/main/java/com/confessio/api/entity/SinEntry.java`

```java
@Entity
@Table(name = "sin_entries")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class SinEntry {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_uuid", nullable = false)
    private UserProfile userProfile;

    @Column(nullable = false, length = 100)
    private String category;

    @Column(nullable = false)
    private int weight;

    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    @PrePersist
    private void prePersist() { createdAt = Instant.now(); }
}
```

### 4. `src/main/java/com/confessio/api/repository/UserProfileRepository.java`

```java
public interface UserProfileRepository extends JpaRepository<UserProfile, String> {}
```

### 5. `src/main/java/com/confessio/api/repository/SinEntryRepository.java`

```java
public interface SinEntryRepository extends JpaRepository<SinEntry, Long> {
    List<SinEntry> findByUserProfileUuid(String uuid);
}
```

## Acceptance criteria

- [ ] `./mvnw verify -DskipTests` completes without errors
- [ ] `V1__init.sql` present in `src/main/resources/db/migration/`
- [ ] `UserProfile` and `SinEntry` entities compile with correct JPA annotations
- [ ] `ON DELETE CASCADE` present on `sin_entries.user_uuid` foreign key
- [ ] Both repositories extend `JpaRepository`
