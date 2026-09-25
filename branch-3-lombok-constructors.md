# Branch 3 — `3-lombok-constructors`

This branch builds on branch `2-lombok-builders` and introduces a Spring MVC
`BeerController`, demonstrating Lombok's `@AllArgsConstructor` for
constructor-based dependency injection.

## What changed from branch 2

Branch 2 ended with the `Beer` POJO and the `BeerService` / `BeerServiceImpl`
in place, but no controller. Branch 3 adds the controller layer and uses a
Lombok-generated constructor to wire in the `BeerService` dependency.

Single new file:

- `src/main/java/guru/springframework/spring7restmvc/controller/BeerController.java`

Full diff vs. `origin/2-lombok-builders`:

```diff
 .../spring7restmvc/controller/BeerController.java   | 15 +++++++++++++++
 1 file changed, 15 insertions(+)
```

## The new code

`src/main/java/guru/springframework/spring7restmvc/controller/BeerController.java`:

```java
package guru.springframework.spring7restmvc.controller;

import guru.springframework.spring7restmvc.services.BeerService;
import lombok.AllArgsConstructor;
import org.springframework.stereotype.Controller;

/**
 * Created by jt, Spring Framework Guru.
 */
@AllArgsConstructor
@Controller
public class BeerController {
    private final BeerService beerService;

}
```

## Line-by-line explanation

- `package guru.springframework.spring7restmvc.controller;`
  Namespace for the web/controller layer of the application.

- `import guru.springframework.spring7restmvc.services.BeerService;`
  Brings in the service abstraction defined in branch 1/2.

- `import lombok.AllArgsConstructor;`
  Lombok annotation which, at compile time, generates a constructor
  accepting every field in the class as a parameter.

- `import org.springframework.stereotype.Controller;`
  Marks the class as a Spring MVC controller so it is registered as a
  bean during component scanning and can handle web requests.

- `@AllArgsConstructor`
  Instructs Lombok to generate:
  ```java
  public BeerController(BeerService beerService) {
      this.beerService = beerService;
  }
  ```
  This is the classic (preferred) way to do constructor-based dependency
  injection in Spring. The generated constructor makes the dependency
  explicit, the field can be `final`, and the class is easy to unit-test.

- `@Controller`
  Stereotype annotation that registers the class as a Spring-managed
  component. In Spring MVC, `@Controller` (combined with handler-method
  annotations like `@GetMapping`) is what makes a class capable of
  serving HTTP requests. In Spring WebFlux the equivalent would be
  `@RestController` for REST endpoints; here the class is a stub that
  will grow into a `@RestController` in later branches.

- `public class BeerController { ... }`
  The controller class. At this stage it is intentionally empty of
  behavior — the goal of the branch is purely to demonstrate the
  Lombok-generated constructor pattern.

- `private final BeerService beerService;`
  The single dependency. It is `final` because it is supplied via the
  constructor and never reassigned. `final` fields are a strong signal
  that the dependency is required and immutable for the lifetime of
  the bean.

- Empty body `{}`
  No request-handling methods yet. Later branches (e.g.
  `5-list-beer`, `6-get-by-id`, `9-http-post`) will add handler methods
  that call `beerService` to serve REST endpoints.

## Why `@AllArgsConstructor` here

- Avoids boilerplate: no need to hand-write the constructor.
- Promotes constructor injection over field injection — easier to test,
  fields can be `final`, and required dependencies are obvious.
- Keeps the controller minimal; subsequent branches only need to add
  handler methods, not plumbing.

## Prerequisites carried over from previous branches

For this branch to compile, the following must already be in place
(from branches 1 and 2):

- Lombok on the classpath/annotation processor path in `pom.xml`
  (Lombok is already configured for Spring Boot 4 / Java 25 in `pom.xml`).
- `BeerService` interface with a `getBeerById(UUID)` method.
- `BeerServiceImpl` returning a `Beer` built via Lombok's `@Builder`.
- `Beer` POJO annotated with `@Builder` and `@Data`.

## Build / verify

```bash
./mvnw clean compile
```

After compilation, `BeerController` will have a synthetic constructor
`BeerController(BeerService beerService)`, and Spring will autowire
the `BeerServiceImpl` bean into it when the application context starts.

## Next branch

Branch `4-lombok-logging` adds Lombok's `@Slf4j` (or equivalent) to give
the controller (and other classes) a logger field without manual
declaration.
