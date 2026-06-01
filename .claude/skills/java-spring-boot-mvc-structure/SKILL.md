---
name: java-spring-boot-mvc-architecture
description: DDD package layout and MVC architecture for Spring Boot apps serving both JSON APIs and server-rendered views. Use when adding a new domain entity, controller, or service.
---

# Spring Boot MVC Architecture

## Package Layout — Domain-Driven

- Root package: `dev.tnaumov.<app>` (or the project's configured root).
- One package per domain entity: `…/booking/`, `…/master/`, `…/timeslot/`, etc.
- Each package contains the entity, repository, service, and both controllers for that domain. No shared `controllers/` or `services/` packages.

Example:
```
dev.tnaumov.masseuse.booking/
    Booking.java
    BookingRepository.java
    BookingService.java
    BookingApiController.java   // @RestController
    BookingViewController.java  // @Controller
```

## Dual Controller Layers

Each domain exposes two controllers over the same service:

### `@RestController` — JSON API

- Path prefix: `/api/...` (e.g., `/api/bookings`).
- Returns DTOs serialized as JSON.
- Used by external clients and JSON consumers.

```java
@RestController
@RequestMapping("/api/bookings")
public class BookingApiController {
    private final BookingService service;
    // delegates only
}
```

### `@Controller` — HTMX Views

- Path prefix: `/...` (e.g., `/bookings`).
- Returns JTE template names (or fragments for HTMX swaps).
- Used by the HTMX frontend.

```java
@Controller
@RequestMapping("/bookings")
public class BookingViewController {
    private final BookingService service;

    @GetMapping
    public String list(Model model) {
        model.addAttribute("bookings", service.findAll());
        return "bookings";
    }
}
```

## Service Layer

- All business logic lives in the service (`@Service`).
- Controllers are **thin delegates** — no validation logic, no orchestration, no DB access.
- Both controllers call the same service methods. Don't duplicate logic per controller.

## Repository Layer

- Spring Data JPA repositories (`extends JpaRepository<Entity, Long>`).
- Custom queries via `@Query` (apply `java-jpa-conventions`: `NOT EXISTS`, half-open ranges).
