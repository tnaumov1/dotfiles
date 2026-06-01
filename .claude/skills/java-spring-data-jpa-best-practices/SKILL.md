---
name: java-spring-data-jpa-best-practices
description: Spring Data JPA/Hibernate best practices. Use when writing or modifying @Entity classes, repositories, or JPQL/Criteria queries in Spring projects.
---

# Java JPA/Hibernate Best Practices

## Entity

### Always use plain class

Use a plain Java class — not a `record` — for `@Entity` types. Hibernate requires:
- A no-arg constructor (at least `protected`) so the proxy/enhancement layer can instantiate the entity.
- Non-final class and non-final fields so Hibernate can subclass for lazy proxies and apply bytecode enhancement.
- Mutable state so the persistence context can manage dirty checking.

Records are final, immutable, and expose only canonical constructors — they break all of the above. Use records for domain models and DTOs, never for entities.

```java
@Entity
@Table(name = "clients")
public class ClientEntity {
    // no-arg constructor required by JPA
    protected ClientEntity() {}

    public ClientEntity(String name) {
        this.name = name;
    }
    // ...
}
```

### ID field

#### Simple id column

Prefer a database-generated identity for simple numeric keys.

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

#### UUID id column

Prefer time-ordered UUIDs (UUIDv7) over random v4 to keep B-tree index inserts sequential and avoid index fragmentation. Hibernate 6.2+ supports this natively:

```java
@Id
@GeneratedValue
@UuidGenerator(style = UuidGenerator.Style.TIME) // UUIDv7-style time-ordered
@Column(columnDefinition = "uuid", updatable = false, nullable = false)
private UUID id;
```

Map to the native PostgreSQL `uuid` type — never `VARCHAR(36)`, which doubles storage and disables efficient indexing.

#### Embedded id column

For composite keys use `@EmbeddedId` over `@IdClass` — it keeps the key cohesive and the surrounding entity readable. The embedded class must implement `equals`/`hashCode` and `Serializable`.

```java
@Embeddable
public class BookingId implements Serializable {
    private UUID clientId;
    private LocalDate date;

    protected BookingId() {}
    // constructor, equals/hashCode based on all fields
}

@Entity
public class BookingEntity {
    @EmbeddedId
    private BookingId id;
}
```

Avoid composite keys when possible — prefer a surrogate `id` plus a `UNIQUE` constraint on the natural key columns.

### Relations

#### Lazy Fetch type is LAZY

Always declare `fetch = FetchType.LAZY` explicitly on every association — including `@ManyToOne` and `@OneToOne`, which default to `EAGER`. Eager fetching causes N+1 queries, fetches data nobody asked for, and breaks pagination of parent entities.

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "client_id")
private ClientEntity client;

@OneToMany(mappedBy = "client", fetch = FetchType.LAZY)
private Set<BookingEntity> bookings = new HashSet<>();
```

To load associations on demand, use `@EntityGraph` on the repository method or a `JOIN FETCH` in JPQL — never re-enable eager loading globally.

#### Use Set collection for @ManyToMany and @OneToMany relations

`Set` (typically `HashSet`) avoids the `MultipleBagFetchException` that Hibernate throws when two `List`-typed bag collections are fetched in the same query. `Set` semantics also match the relational model: a row either is or isn't in the join table.

```java
@OneToMany(mappedBy = "client", cascade = CascadeType.ALL, orphanRemoval = true)
private Set<BookingEntity> bookings = new HashSet<>();

@ManyToMany
@JoinTable(
    name = "client_tag",
    joinColumns = @JoinColumn(name = "client_id"),
    inverseJoinColumns = @JoinColumn(name = "tag_id"))
private Set<TagEntity> tags = new HashSet<>();
```

Initialize the collection at field declaration so it is never `null`. Use `List` only when ordering matters and you map it with `@OrderColumn`.

### Equals/Hashcode methods

Entity equality is one of the easiest things to get wrong. Rules:

- Never use the auto-generated DB id alone in `hashCode` — the id is `null` before flush, so a transient entity placed in a `HashSet` and later persisted becomes unfindable.
- Never use Lombok's `@Data` or `@EqualsAndHashCode` without exclusions — it pulls in collections and triggers lazy loads.
- Implement `equals` based on the id when present, but make `hashCode` return a constant (or a hash of an immutable business key).

```java
@Override
public final boolean equals(Object o) {
    if (this == o) return true;
    if (o == null) return false;
    Class<?> oEffectiveClass = o instanceof HibernateProxy ? ((HibernateProxy) o).getHibernateLazyInitializer().getPersistentClass() : o.getClass();
    Class<?> thisEffectiveClass = this instanceof HibernateProxy ? ((HibernateProxy) this).getHibernateLazyInitializer().getPersistentClass() : this.getClass();
    if (thisEffectiveClass != oEffectiveClass) return false;
    NewEntity newEntity = (NewEntity) o;
    return getId() != null && Objects.equals(getId(), newEntity.getId());
}

@Override
public final int hashCode() {
    return this instanceof HibernateProxy ? ((HibernateProxy) this).getHibernateLazyInitializer().getPersistentClass().hashCode() : getClass().hashCode();
}
```

If a stable, immutable business key exists (e.g., email, ISBN), prefer it for both `equals` and `hashCode` — that's the most correct choice.

### Auditable

Centralize audit columns with a `@MappedSuperclass` and Spring Data JPA's auditing support. Enable it once on a `@Configuration` with `@EnableJpaAuditing`.

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class Auditable {
    @CreatedDate
    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    @Column(nullable = false)
    private Instant updatedAt;

    @CreatedBy
    @Column(updatable = false)
    private String createdBy;

    @LastModifiedBy
    private String updatedBy;

    @Version
    private long version; // optimistic locking
}
```

Provide an `AuditorAware<String>` bean that pulls the current user from the security context.

## Repositories

### Use Interface

Always use the `interface` keyword to define the repository file. Spring Data generates the implementation at runtime. Never `@Autowired` `EntityManager` directly into a service when a repository will do.

```java
public interface ClientRepository extends JpaRepository<ClientEntity, UUID> {}
```

### CRUD repositories

- Extend `JpaRepository<T, ID>` for full CRUD + paging + sorting
- Prefer derived query methods for simple lookups: `findByEmail`, `existsByEmailIgnoreCase`, `countByStatus`.
- Use `@Query` with JPQL (not native SQL) for anything more complex; native SQL only when you need PostgreSQL-specific features (e.g., `JSONB`, window functions, `INSERT ... ON CONFLICT`).
- Always return `Optional<T>` for single-result lookups — never `null`.
- Return `Slice<T>` instead of `Page<T>` when you don't need a total count — `Page` issues a second `COUNT(*)` query.
- Use named parameters (`:email`), never positional ones — they survive query refactors.
- Annotate modifying queries with `@Modifying(clearAutomatically = true, flushAutomatically = true)` so the persistence context doesn't serve stale cached entities.

```java
public interface ClientRepository extends JpaRepository<ClientEntity, UUID> {

    Optional<ClientEntity> findByEmailIgnoreCase(String email);

    @EntityGraph(attributePaths = "bookings")
    Optional<ClientEntity> findWithBookingsById(UUID id);

    @Query("select c from ClientEntity c where c.status = :status")
    Slice<ClientEntity> findActive(@Param("status") Status status, Pageable pageable);

    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query("update ClientEntity c set c.status = :status where c.id = :id")
    int updateStatus(@Param("id") UUID id, @Param("status") Status status);
}
```

For dynamic queries (filters, sorts that vary at runtime) use JPA Criteria API via Spring Data Specifications over string concatenation:

```java
public interface ClientRepository extends JpaRepository<ClientEntity, UUID>, JpaSpecificationExecutor<ClientEntity> {
    
}
```

### Custom repositories

For repository methods that need `EntityManager` directly, follow the `Custom` fragment pattern:

```java
public interface ClientRepositoryCustom {
    List<ClientProjection> searchProjected(SearchCriteria criteria);
}

public class ClientRepositoryImpl implements ClientRepositoryCustom {
    @PersistenceContext
    private EntityManager em;
    // ...
}

public interface ClientRepository
    extends JpaRepository<ClientEntity, UUID>, ClientRepositoryCustom {}
```

For read-heavy queries that don't need full entities, return DTO projections — they skip the persistence context and avoid loading lazy associations.

## Transactions

- Apply `@Transactional` at the service layer, not the controller and not the repository (Spring Data already wraps individual repo methods).
- Default to `@Transactional(readOnly = true)` at the class level on read-oriented services; override per-method with `@Transactional` for writes. `readOnly = true` lets Hibernate skip dirty checking and lets the JDBC driver hint read-only routing.
- Keep transactions short. Never call external HTTP APIs, send Kafka messages, or block on user input inside a transactional method — long-held connections starve the pool.
- Use `org.springframework.transaction.annotation.Transactional`, not `jakarta.transaction.Transactional` — the Spring annotation supports `readOnly`, `propagation`, `isolation`, and rollback rules; the Jakarta one does not.
- Spring rolls back on unchecked exceptions only. To roll back on a checked exception use `@Transactional(rollbackFor = SomeCheckedException.class)`.
- Be deliberate about propagation. `REQUIRES_NEW` opens a second physical connection — easy way to deadlock against the calling transaction.
- For optimistic concurrency, add `@Version` to the entity and catch `OptimisticLockingFailureException` at the service boundary.

```java
@Service
@Transactional(readOnly = true)
public class ClientService {

    private final ClientRepository repo;

    @Transactional
    public ClientEntity register(NewClient cmd) { /* ... */ }

    public Optional<ClientEntity> findById(UUID id) {
        return repo.findById(id);
    }
}
```

## Logging

- Enable SQL logging only via Hibernate's logger — never `spring.jpa.show-sql=true`, which writes to stdout and can't be filtered.
  ```properties
  logging.level.org.hibernate.SQL=DEBUG
  logging.level.org.hibernate.orm.jdbc.bind=TRACE   # parameter values
  logging.level.org.hibernate.stat=DEBUG            # statistics
  spring.jpa.properties.hibernate.format_sql=true
  spring.jpa.properties.hibernate.generate_statistics=true
  ```
- For a development-time view of N+1 queries and slow queries, use Hibernate statistics (`SessionFactory#getStatistics()`) or a JDBC proxy like datasource-proxy / p6spy that logs query counts per request.
- In production, expose Hibernate stats through Micrometer (`hibernate-micrometer`) and watch `hibernate.query.execution.count` per HTTP request — a sudden spike points to a missing fetch graph.

## Testing

- Use `@DataJpaTest` for repository slice tests — it configures only the JPA layer and runs each test in a rollback transaction.
- Disable the embedded-DB replacement so tests run against the real PostgreSQL via Testcontainers:
  ```java
  @DataJpaTest
  @AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
  @Import(TestcontainersConfiguration.class)
  class ClientRepositoryTest { /* ... */ }
  ```
- Use `TestEntityManager` (not the production repository) to set up fixtures, then exercise the repository under test — this catches mismatches between what the repo writes and what JPQL reads.
- Always call `entityManager.flush()` and `entityManager.clear()` between "arrange" and "act" so the test reads from the database, not the first-level cache. This is how you catch missing `@Column(nullable=false)`, broken cascades, and lazy-init bugs.
- Assert query counts for hot paths to catch N+1 regressions — Hibernate `Statistics#getQueryExecutionCount()` or `datasource-proxy` assertions.
- Don't mock `EntityManager` or repositories in service-layer tests that exercise transactional behavior — use Testcontainers, per project convention.

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `LazyInitializationException: could not initialize proxy — no Session` | Accessing a lazy association outside the transactional boundary (typically in a controller or template). | Load the association inside the service via `@EntityGraph` / `JOIN FETCH`, or map to a DTO before returning. Never enable `open-in-view`. |
| `MultipleBagFetchException: cannot simultaneously fetch multiple bags` | Two `List`-typed `@OneToMany` collections fetched in one query. | Switch the collections to `Set`, or split into two queries (Hibernate will batch via `@BatchSize`). |
| N+1 queries | Lazy association iterated in a loop. | Add `@EntityGraph(attributePaths = ...)` on the repository method, or use `JOIN FETCH` in JPQL. For batch-style access, set `@BatchSize(size = 50)` on the association. |
| Pagination warning `HHH000104: firstResult/maxResults specified with collection fetch; applying in memory` | `JOIN FETCH` of a collection plus `Pageable`. | Two-step fetch: page parent IDs first, then fetch parents+collection by `IN (:ids)`. |
| `OptimisticLockingFailureException` / `StaleObjectStateException` | Two transactions updated the same `@Version`-ed row. | Catch at the service boundary, retry or surface a 409 to the user. |
| `IdentifierGenerationException: ids for this class must be manually assigned` | `@GeneratedValue` missing on a non-trivial id type. | Add `@GeneratedValue` (and `@UuidGenerator` for UUIDs) or assign the id before persisting. |
| `detached entity passed to persist` | Calling `save()` on an entity that already has an id assigned externally. | Use `merge()` semantics (`save()` does this when the id is set and the row exists) or load + mutate inside the transaction. |
| Inserts run one-by-one despite `spring.jpa.properties.hibernate.jdbc.batch_size=50` | Using `GenerationType.IDENTITY` (Hibernate cannot batch inserts that depend on DB-returned ids). | Switch to `SEQUENCE` with a matching `allocationSize` and enable `order_inserts=true`, `order_updates=true`. |
| Connection pool exhausted under load | Long-running `@Transactional` method making external calls. | Move the external call out of the transaction; shorten transactional scope to just the DB work. |
| Rollback didn't happen on a checked exception | Default rollback rule covers unchecked only. | `@Transactional(rollbackFor = MyCheckedException.class)`. |
| Duplicate-key error on retry of a save | Relying on `existsBy...` then `save` without a unique constraint. | Add the DB unique constraint and handle `DataIntegrityViolationException`; or use `INSERT ... ON CONFLICT DO NOTHING` via native query. |
