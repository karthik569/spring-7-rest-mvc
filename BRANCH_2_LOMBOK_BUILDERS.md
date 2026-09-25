# Branch 2: `2-lombok-builders` - Code Explanation

## Overview
The second branch (`origin/2-lombok-builders`) builds upon the basic POJO setup from branch 1 by introducing:
1. **Lombok's `@Builder` Annotation**: Implementing the Builder Design Pattern declaratively on model classes.
2. **Service Layer Abstraction**: Introducing a service interface (`BeerService`) and its concrete implementation (`BeerServiceImpl`).
3. **Mock Data Instantiation**: Using the generated builder pattern to construct domain objects with clear, fluent syntax.

---

## Code Breakdown

### 1. `Beer.java` (`guru.springframework.spring7restmvc.model.Beer`)
```java
package guru.springframework.spring7restmvc.model;

import lombok.Builder;
import lombok.Data;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

/**
 * Created by jt, Spring Framework Guru.
 */
@Builder
@Data
public class Beer {
    private UUID id;
    private Integer version;
    private String beerName;
    private BeerStyle beerStyle;
    private String upc;
    private Integer quantityOnHand;
    private BigDecimal price;
    private LocalDateTime createdDate;
    private LocalDateTime updateDate;
}
```

#### Key Changes:
- **`@Builder` Annotation**:
  - Automatically generates a static builder class (`Beer.BeerBuilder`) and a `Beer.builder()` entry point.
  - Allows fluent object instantiation (e.g. `Beer.builder().beerName("Galaxy Cat").build()`).
  - Eliminates the need for telescoping constructors with numerous parameters.

---

### 2. `BeerService.java` (`guru.springframework.spring7restmvc.services.BeerService`)
```java
package guru.springframework.spring7restmvc.services;

import guru.springframework.spring7restmvc.model.Beer;

import java.util.UUID;

/**
 * Created by jt, Spring Framework Guru.
 */
public interface BeerService {

    Beer getBeerById(UUID id);
}
```

#### Key Concepts:
- **Interface-Driven Design**: Decouples business contracts from implementation details.
- Defines core operations expected from the service layer (`getBeerById(UUID id)`).

---

### 3. `BeerServiceImpl.java` (`guru.springframework.spring7restmvc.services.BeerServiceImpl`)
```java
package guru.springframework.spring7restmvc.services;

import guru.springframework.spring7restmvc.model.Beer;
import guru.springframework.spring7restmvc.model.BeerStyle;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

/**
 * Created by jt, Spring Framework Guru.
 */
public class BeerServiceImpl implements BeerService {
    @Override
    public Beer getBeerById(UUID id) {
        return Beer.builder()
                .id(id)
                .version(1)
                .beerName("Galaxy Cat")
                .beerStyle(BeerStyle.PALE_ALE)
                .upc("12356")
                .price(new BigDecimal("12.99"))
                .quantityOnHand(122)
                .createdDate(LocalDateTime.now())
                .updateDate(LocalDateTime.now())
                .build();
    }
}
```

#### Key Concepts:
- **Demonstration of Builder Pattern**: Instantiates a sample `Beer` object cleanly without invoking a massive constructor or chaining numerous setter calls.
- **Mock Data**: Serves as a stub / mock implementation before hooking up repositories or database persistence.

---

### 4. Maven Compiler Plugin Configuration (`pom.xml`)
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <annotationProcessorPaths>
            <path>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
            </path>
        </annotationProcessorPaths>
    </configuration>
</plugin>
```
- Ensures Lombok's annotation processor generates the builder classes during the build step.

---

## Summary of Architectural Benefits
1. **Readability & Maintainability**: The builder pattern provides named parameters at creation time, preventing errors where parameters of the same type might be swapped.
2. **Decoupling**: Having `BeerService` as an interface allows Spring's Inversion of Control (IoC) and dependency injection mechanisms to swap out implementations (e.g., JPA-backed service, Mock service for testing) seamlessly later on.
