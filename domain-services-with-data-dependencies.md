# Domain Services mit Datenabhängigkeiten

> **Kontext:** Domain-Centric Architecture · Pricing Bounded Context
> **Zielgruppe:** Entwickler, die Domain Services mit externem Datenzugriff implementieren

---

## Problemstellung

Domain Services sind per Definition **stateless** und gehören zum Domain Layer — dem innersten Ring der Architektur. Sie haben **keine Abhängigkeiten nach außen**. Aber was passiert, wenn ein Domain Service zusätzliche Daten braucht, um seine Berechnung durchzuführen?

**Konkretes Beispiel:** Ein `BundleDiscountService` soll einen Rabatt berechnen, der von der Kategorie des Produkts abhängt. Die Kategorie-Preis-Zuordnung liegt aber nicht im aktuellen Aggregate — sie muss nachgeladen werden.

```java
// Das Problem: Der Domain Service braucht Daten, die er nicht hat
public final class BundleDiscountService implements DomainService {

    public Price calculateBundleDiscount(ProductId productId, Price basePrice) {
        // Woher kommt die Kategorie-Information?
        // Der Domain Layer darf keine Repositories oder Adapter kennen!
        CategoryDiscount discount = ???;
        return applyDiscount(basePrice, discount);
    }
}
```

Die Dependency Rule verbietet es dem Domain Layer, auf den Application Layer oder Adapter zuzugreifen. Trotzdem muss der Domain Service an die Daten kommen.

---

## Default-Regel: Pure Domain Services (90% der Fälle)

In den meisten Fällen ist die richtige Lösung: **Der Application Service (Use Case) orchestriert.** Er lädt alle benötigten Daten über Output Ports und übergibt sie dem Domain Service als Parameter.

```java
// Application Layer — Use Case orchestriert
public class CalculateBundleDiscountUseCase implements CalculateBundleDiscountInputPort {

    private final ProductPriceRepository productPriceRepository;
    private final BundleDiscountService bundleDiscountService;

    @Override
    public DiscountResult execute(DiscountCommand command) {
        ProductPrice productPrice = productPriceRepository
            .findByProductId(command.productId())
            .orElseThrow();

        // Domain Service erhält alle Daten als Parameter — pure, testbar, einfach
        Price discountedPrice = bundleDiscountService.calculateBundleDiscount(
            productPrice.price(),
            productPrice.category(),
            command.bundleSize()
        );

        return new DiscountResult(discountedPrice);
    }
}
```

```java
// Domain Layer — Pure Domain Service, keine Abhängigkeiten
public final class BundleDiscountService implements DomainService {

    public Price calculateBundleDiscount(Price basePrice, Category category, int bundleSize) {
        int discountPercentage = category.bundleDiscountFor(bundleSize);
        return basePrice.applyDiscount(discountPercentage);
    }
}
```

**Das ist der bevorzugte Ansatz.** Er hält den Domain Service pure und testbar. Erst wenn die Orchestrierung im Use Case zu komplex wird oder die Domänenlogik selbst entscheiden muss, welche Daten sie braucht, kommen die folgenden Alternativen ins Spiel.

---

## Ansatz 1: DomainGateway Pattern

### Konzept

Ein **DomainGateway** ist ein schmales, read-only Interface im Domain Layer, das in der **Ubiquitous Language** formuliert ist. Es erlaubt dem Domain Service, gezielt Daten nachzuladen, ohne die Dependency Rule zu verletzen.

**Wichtige Abgrenzung:**
- Ein DomainGateway ist ein **taktisches DDD-Pattern** — es gehört in den Domain Layer
- Es ist **kein OutputPort** — OutputPorts gehören zum Application Layer (Hexagonal Architecture)
- Es ist **kein Repository** — Repositories verwalten Aggregate Roots mit vollem Lifecycle (CRUD)
- Ein DomainGateway ist **read-only** und liefert nur die Daten, die der Domain Service für seine Berechnung braucht

### Marker-Interface

```java
package de.sample.aiarchitecture.sharedkernel.marker.tactical;

/**
 * Marker interface for Domain Gateways.
 *
 * <p>A Domain Gateway is a narrow, read-only interface defined in the domain layer
 * that allows Domain Services to retrieve data needed for domain logic.
 * Unlike Repositories (which manage aggregate lifecycle via OutputPort),
 * Domain Gateways are tactical DDD patterns focused purely on data lookup.
 *
 * <p><b>Characteristics:</b>
 * <ul>
 *   <li>Read-only — no mutations, no save/delete operations
 *   <li>Narrow — only the data the domain logic needs, not entire aggregates
 *   <li>Named in Ubiquitous Language — e.g., CategoryPriceLookup, TaxRateResolver
 *   <li>Defined in domain layer, implemented by adapters
 *   <li>Should NOT have Spring annotations in the interface
 * </ul>
 *
 * <p><b>Naming conventions:</b> {@code *Lookup}, {@code *Resolver}, {@code *Provider}
 *
 * @see DomainService
 */
public interface DomainGateway {}
```

**Einordnung im Shared Kernel:**

```
sharedkernel/marker/tactical/
├── DomainService.java
├── DomainGateway.java          ← NEU
├── AggregateRoot.java
├── Entity.java
├── Value.java
└── ...
```

### Naming-Konventionen

| Suffix       | Verwendung                                           | Beispiel                    |
|--------------|------------------------------------------------------|-----------------------------|
| `*Lookup`    | Einfache Datenabfrage (Key → Value)                  | `CategoryPriceLookup`       |
| `*Resolver`  | Auflösung mit Logik (z.B. Fallback, Hierarchie)     | `TaxRateResolver`           |
| `*Provider`  | Bereitstellung von Kontextdaten                      | `ExchangeRateProvider`      |

### Vollständiges Code-Beispiel

**1. DomainGateway Interface (Domain Layer)**

```java
package de.sample.aiarchitecture.pricing.domain.gateway;

import de.sample.aiarchitecture.pricing.domain.model.CategoryDiscount;
import de.sample.aiarchitecture.sharedkernel.domain.model.ProductId;
import de.sample.aiarchitecture.sharedkernel.marker.tactical.DomainGateway;
import java.util.Optional;

/**
 * Looks up category-based discount rates for products.
 */
public interface CategoryPriceLookup extends DomainGateway {

    Optional<CategoryDiscount> discountForProduct(ProductId productId);
}
```

**2. Domain Value Object (Domain Layer)**

```java
package de.sample.aiarchitecture.pricing.domain.model;

import de.sample.aiarchitecture.sharedkernel.marker.tactical.Value;

public record CategoryDiscount(String categoryName, int discountPercentage) implements Value {

    public CategoryDiscount {
        if (discountPercentage < 0 || discountPercentage > 100) {
            throw new IllegalArgumentException(
                "Discount percentage must be between 0 and 100");
        }
    }
}
```

**3. Domain Service mit DomainGateway (Domain Layer)**

```java
package de.sample.aiarchitecture.pricing.domain.service;

import de.sample.aiarchitecture.pricing.domain.gateway.CategoryPriceLookup;
import de.sample.aiarchitecture.pricing.domain.model.CategoryDiscount;
import de.sample.aiarchitecture.sharedkernel.domain.model.Price;
import de.sample.aiarchitecture.sharedkernel.domain.model.ProductId;
import de.sample.aiarchitecture.sharedkernel.marker.tactical.DomainService;

public final class BundleDiscountService implements DomainService {

    private final CategoryPriceLookup categoryPriceLookup;

    public BundleDiscountService(CategoryPriceLookup categoryPriceLookup) {
        this.categoryPriceLookup = categoryPriceLookup;
    }

    public Price calculateBundleDiscount(ProductId productId, Price basePrice, int bundleSize) {
        int discountPercentage = categoryPriceLookup.discountForProduct(productId)
            .map(CategoryDiscount::discountPercentage)
            .map(base -> base + bonusForBundleSize(bundleSize))
            .orElse(bonusForBundleSize(bundleSize));

        return basePrice.applyDiscount(discountPercentage);
    }

    private int bonusForBundleSize(int bundleSize) {
        if (bundleSize >= 10) return 15;
        if (bundleSize >= 5) return 10;
        if (bundleSize >= 3) return 5;
        return 0;
    }
}
```

**4. Adapter-Implementierung (Adapter Layer)**

```java
package de.sample.aiarchitecture.pricing.adapter.outgoing.categorylookup;

import de.sample.aiarchitecture.pricing.domain.gateway.CategoryPriceLookup;
import de.sample.aiarchitecture.pricing.domain.model.CategoryDiscount;
import de.sample.aiarchitecture.sharedkernel.domain.model.ProductId;
import java.util.Map;
import java.util.Optional;
import org.springframework.stereotype.Component;

@Component
class InMemoryCategoryPriceLookup implements CategoryPriceLookup {

    private final Map<ProductId, CategoryDiscount> categoryDiscounts;

    InMemoryCategoryPriceLookup(Map<ProductId, CategoryDiscount> categoryDiscounts) {
        this.categoryDiscounts = categoryDiscounts;
    }

    @Override
    public Optional<CategoryDiscount> discountForProduct(ProductId productId) {
        return Optional.ofNullable(categoryDiscounts.get(productId));
    }
}
```

**5. Wiring im Use Case (Application Layer)**

```java
package de.sample.aiarchitecture.pricing.application.calculatebundlediscount;

import de.sample.aiarchitecture.pricing.domain.gateway.CategoryPriceLookup;
import de.sample.aiarchitecture.pricing.domain.service.BundleDiscountService;
import de.sample.aiarchitecture.sharedkernel.domain.model.Price;

public class CalculateBundleDiscountUseCase implements CalculateBundleDiscountInputPort {

    private final BundleDiscountService bundleDiscountService;

    public CalculateBundleDiscountUseCase(CategoryPriceLookup categoryPriceLookup) {
        this.bundleDiscountService = new BundleDiscountService(categoryPriceLookup);
    }

    @Override
    public DiscountResult execute(DiscountCommand command) {
        Price discountedPrice = bundleDiscountService.calculateBundleDiscount(
            command.productId(),
            command.basePrice(),
            command.bundleSize()
        );
        return new DiscountResult(discountedPrice);
    }
}
```

### Datenfluss

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ Adapter Layer                                                               │
│                                                                             │
│  InMemoryCategoryPriceLookup ──implements──▶ CategoryPriceLookup (Domain)  │
│                                                                             │
└──────────────────────────────────────────────────┬──────────────────────────┘
                                                   │
┌──────────────────────────────────────────────────┼──────────────────────────┐
│ Application Layer                                │                          │
│                                                  │                          │
│  CalculateBundleDiscountUseCase                  │ injects                  │
│      │                                           │                          │
│      │ creates with gateway ─────────────────────┘                          │
│      ▼                                                                      │
└──────┼──────────────────────────────────────────────────────────────────────┘
       │
┌──────┼──────────────────────────────────────────────────────────────────────┐
│ Domain Layer                                                                │
│      ▼                                                                      │
│  BundleDiscountService                                                      │
│      │                                                                      │
│      │──▶ CategoryPriceLookup.discountForProduct(productId)                │
│      │                          │                                           │
│      │◀── CategoryDiscount ◀────┘                                           │
│      │                                                                      │
│      └──▶ Price (berechnet)                                                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Abgrenzung: DomainGateway vs. Repository

| Aspekt              | Repository                              | DomainGateway                            |
|---------------------|-----------------------------------------|------------------------------------------|
| **Marker**          | `extends OutputPort`                    | `extends DomainGateway`                  |
| **Layer**           | Application Layer (Output Port)         | Domain Layer (taktisches Pattern)        |
| **Verantwortung**   | Aggregate Lifecycle (CRUD)              | Read-only Datenabfrage                   |
| **Scope**           | Ganzes Aggregate Root                   | Schmaler Datenausschnitt                 |
| **Mutationen**      | `save()`, `deleteById()`               | Keine                                    |
| **Benutzt von**     | Use Cases (Application Layer)           | Domain Services (Domain Layer)           |
| **Implementiert von** | Outgoing Adapter                      | Outgoing Adapter                         |

### Vor- und Nachteile

**Vorteile:**
- Domain Service kann eigenständig entscheiden, welche Daten er wann braucht
- Interface ist in Ubiquitous Language formuliert — explizit im Domain Model
- Einfach testbar: Mock des DomainGateway im Unit Test
- Gut geeignet für komplexe Domänenlogik mit bedingten Datenabfragen

**Nachteile:**
- Führt eine Abhängigkeit in den Domain Layer ein (wenn auch abstrakt)
- Kann als "Hintertür" missbraucht werden — Disziplin nötig
- Mehr Klassen: Interface + Implementierung + Marker
- Nicht in allen DDD-Literaturquellen als Pattern etabliert

---

## Ansatz 2: Strategy/Callback Pattern

### Konzept

Der Domain Service erhält die Datenbeschaffung als **funktionalen Parameter** (Strategy). Der Application Service übergibt ein Lambda oder eine Method Reference, die die Daten liefert. Der Domain Layer definiert kein Interface — die Abhängigkeit existiert nur zur Aufrufzeit.

### Vollständiges Code-Beispiel

**1. Domain Service mit funktionalem Parameter (Domain Layer)**

```java
package de.sample.aiarchitecture.pricing.domain.service;

import de.sample.aiarchitecture.pricing.domain.model.CategoryDiscount;
import de.sample.aiarchitecture.sharedkernel.domain.model.Price;
import de.sample.aiarchitecture.sharedkernel.domain.model.ProductId;
import de.sample.aiarchitecture.sharedkernel.marker.tactical.DomainService;
import java.util.Optional;
import java.util.function.Function;

public final class BundleDiscountService implements DomainService {

    public Price calculateBundleDiscount(
            ProductId productId,
            Price basePrice,
            int bundleSize,
            Function<ProductId, Optional<CategoryDiscount>> discountLookup) {

        int discountPercentage = discountLookup.apply(productId)
            .map(CategoryDiscount::discountPercentage)
            .map(base -> base + bonusForBundleSize(bundleSize))
            .orElse(bonusForBundleSize(bundleSize));

        return basePrice.applyDiscount(discountPercentage);
    }

    private int bonusForBundleSize(int bundleSize) {
        if (bundleSize >= 10) return 15;
        if (bundleSize >= 5) return 10;
        if (bundleSize >= 3) return 5;
        return 0;
    }
}
```

**2. Wiring im Use Case (Application Layer)**

```java
package de.sample.aiarchitecture.pricing.application.calculatebundlediscount;

import de.sample.aiarchitecture.pricing.application.shared.ProductPriceRepository;
import de.sample.aiarchitecture.pricing.domain.model.CategoryDiscount;
import de.sample.aiarchitecture.pricing.domain.service.BundleDiscountService;
import de.sample.aiarchitecture.sharedkernel.domain.model.Price;

public class CalculateBundleDiscountUseCase implements CalculateBundleDiscountInputPort {

    private final ProductPriceRepository productPriceRepository;
    private final BundleDiscountService bundleDiscountService = new BundleDiscountService();

    public CalculateBundleDiscountUseCase(ProductPriceRepository productPriceRepository) {
        this.productPriceRepository = productPriceRepository;
    }

    @Override
    public DiscountResult execute(DiscountCommand command) {
        Price discountedPrice = bundleDiscountService.calculateBundleDiscount(
            command.productId(),
            command.basePrice(),
            command.bundleSize(),
            productId -> productPriceRepository.findByProductId(productId)
                .map(pp -> new CategoryDiscount(pp.category(), pp.categoryDiscountPercentage()))
        );
        return new DiscountResult(discountedPrice);
    }
}
```

### Variante: Eigenes Functional Interface statt `java.util.function.Function`

Wenn die Signatur von `Function<ProductId, Optional<CategoryDiscount>>` zu generisch ist, kann ein eigenes Functional Interface die Lesbarkeit verbessern:

```java
package de.sample.aiarchitecture.pricing.domain.service;

import de.sample.aiarchitecture.pricing.domain.model.CategoryDiscount;
import de.sample.aiarchitecture.sharedkernel.domain.model.ProductId;
import java.util.Optional;

@FunctionalInterface
public interface CategoryDiscountLookup {
    Optional<CategoryDiscount> lookup(ProductId productId);
}
```

Der Domain Service verwendet dann:

```java
public Price calculateBundleDiscount(
        ProductId productId,
        Price basePrice,
        int bundleSize,
        CategoryDiscountLookup discountLookup) {

    int discountPercentage = discountLookup.lookup(productId)
        // ...
}
```

Der Aufruf im Use Case bleibt identisch — Java's Lambda-Kompatibilität sorgt dafür, dass das Lambda automatisch zum Functional Interface passt.

**Empfehlung:** Verwende ein eigenes Functional Interface wenn:
- Die Methode mehr als einmal verwendet wird
- Die generische Signatur `Function<A, B>` die Lesbarkeit verschlechtert
- Du die Methode dokumentieren willst (Javadoc auf dem Interface)

### Datenfluss

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ Application Layer                                                           │
│                                                                             │
│  CalculateBundleDiscountUseCase                                             │
│      │                                                                      │
│      │ ruft auf mit Lambda: productId -> repository.find(...)              │
│      │                                    │                                 │
│      ▼                                    ▼                                 │
└──────┼──────────────────────────────┬─────┼─────────────────────────────────┘
       │                              │     │
┌──────┼──────────────────────────────┼─────┼─────────────────────────────────┐
│ Domain Layer                        │     │                                 │
│      ▼                              │     │                                 │
│  BundleDiscountService              │     │                                 │
│      │                              │     │                                 │
│      │──▶ discountLookup.apply(id) ─┘     │                                 │
│      │         (Lambda-Callback)          │                                 │
│      │                                    │                                 │
│      │◀── CategoryDiscount ◀──────────────┘                                 │
│      │                                                                      │
│      └──▶ Price (berechnet)                                                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Vor- und Nachteile

**Vorteile:**
- **Zero Abhängigkeiten** im Domain Layer — nicht mal ein abstraktes Interface
- Domain Service bleibt ein echtes Pure Object (stateless, no fields)
- Maximale Testbarkeit: Lambda im Test inline definieren
- Kein zusätzliches Marker-Interface nötig
- Leichtgewichtig — keine zusätzlichen Klassen

**Nachteile:**
- Methodensignatur wird länger und komplexer
- Weniger explizit: `Function<ProductId, Optional<CategoryDiscount>>` ist nicht sofort verständlich
- Callback-Logik kann im Use Case unübersichtlich werden
- Kein Platz für Javadoc am Contract (bei `java.util.function.Function`)
- Bei mehreren Datenquellen: Parameter-Explosion

---

## Vergleich: Wann welchen Ansatz nutzen

| Kriterium                        | Pure Domain Service     | DomainGateway              | Strategy/Callback          |
|----------------------------------|-------------------------|----------------------------|----------------------------|
| **Komplexität der Datenabfrage** | Einfach (1-2 Quellen)  | Mittel bis komplex         | Einfach (1 Quelle)        |
| **Abhängigkeiten im Domain**     | Keine                   | Abstraktes Interface       | Keine                      |
| **Testbarkeit**                  | Trivial                 | Mock des Gateway           | Lambda inline              |
| **Lesbarkeit**                   | Sehr gut                | Gut (explizites Interface) | Mäßig (lange Signaturen)  |
| **Wiederverwendbarkeit**         | Hoch                    | Hoch (Interface geteilt)   | Niedrig (pro Aufruf)      |
| **Anzahl Klassen**               | Minimal                 | +2 (Interface + Impl)      | Optional +1 (Func. Interf.)|
| **Domain Model Explizitheit**    | —                       | Hoch (Ubiquitous Language) | Niedrig                    |
| **Empfohlen wenn...**            | Daten vorab ladbar      | Domain entscheidet über Datenbedarf, mehrere Services nutzen gleiche Abfrage | Einzelne, einfache Abfrage bei einem Service |

### Entscheidungsbaum

```
START: Domain Service braucht Daten, die er nicht hat
   │
   ├─ Kann der Application Service alle Daten vorab laden?
   │     JA → Pure Domain Service (Default)
   │     │
   │     NEIN ↓
   │
   ├─ Entscheidet der Domain Service dynamisch, welche Daten er braucht?
   │     JA → DomainGateway Pattern
   │     │
   │     NEIN ↓
   │
   ├─ Ist es eine einzelne, einfache Datenabfrage?
   │     JA → Strategy/Callback Pattern
   │     │
   │     NEIN ↓
   │
   └─ Brauchen mehrere Domain Services die gleiche Abfrage?
         JA → DomainGateway Pattern (wiederverwendbares Interface)
         NEIN → Strategy/Callback Pattern (leichtgewichtig)
```

---

## ArchUnit Governance

### DomainGateway-Regeln

```java
@ArchTest
static final ArchRule domain_gateways_must_be_read_only =
    classes().that().implement(DomainGateway.class)
        .should().haveOnlyFinalFields()
        .andShould().notHaveModifier(JavaModifier.ABSTRACT)
        .because("DomainGateways are read-only lookup interfaces implemented by adapters");

@ArchTest
static final ArchRule domain_gateways_must_reside_in_domain_layer =
    classes().that().implement(DomainGateway.class)
        .and().areInterfaces()
        .should().resideInAnyPackage("..domain.gateway..")
        .because("DomainGateway interfaces belong to the domain layer");

@ArchTest
static final ArchRule domain_gateway_interfaces_must_not_extend_output_port =
    classes().that().implement(DomainGateway.class)
        .should().notImplement(OutputPort.class)
        .because("DomainGateways are tactical DDD patterns, not hexagonal OutputPorts");
```

### Strategy/Callback-Regeln

Da das Strategy/Callback Pattern kein eigenes Interface im Domain Layer definiert, sind die bestehenden ArchUnit-Regeln bereits ausreichend:

```java
// Bestehende Regel: Domain Layer hat keine Abhängigkeiten nach außen
@ArchTest
static final ArchRule domain_layer_has_no_outward_dependencies =
    classes().that().resideInAnyPackage("..domain..")
        .should().onlyDependOnClassesThat()
        .resideInAnyPackage("..domain..", "..sharedkernel..", "java..")
        .because("Domain layer must not depend on application, adapter, or infrastructure layers");
```

Diese Regel stellt automatisch sicher, dass:
- Kein `Function`-Parameter auf Adapter- oder Application-Klassen verweist
- Die Domain nur `java.util.function.*` verwendet (erlaubt unter `java..`)
- Keine versteckten Abhängigkeiten über Lambdas eingeschleust werden

### Zusätzliche Governance für eigene Functional Interfaces

```java
@ArchTest
static final ArchRule functional_interfaces_in_domain_must_be_annotated =
    classes().that().resideInAnyPackage("..domain..")
        .and().areInterfaces()
        .and().haveSimpleNameNotEndingWith("DomainGateway")
        .and().areAnnotatedWith(FunctionalInterface.class)
        .should().resideInAnyPackage("..domain.service..", "..domain..")
        .because("Domain functional interfaces should be co-located with their Domain Services");
```

---

## Referenzen

- [Domain-Driven Design](https://www.domainlanguage.com/ddd/) — Eric Evans (2003), Chapter 5
- [Implementing Domain-Driven Design](https://www.informit.com/store/implementing-domain-driven-design-9780321834577) — Vaughn Vernon (2013), Chapter 7