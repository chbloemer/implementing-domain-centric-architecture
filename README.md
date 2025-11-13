# Domain-Centric Architecture
*Synthesis of Domain-Driven Design, Hexagonal Architecture, and Clean Architecture*

## Table of Contents

- [Key Points](#key-points)
- [Related Documentation](#related-documentation)
- [ELEMENTS](#elements)
  - [Domain Layer (Enterprise Business Rules)](#domain-layer-enterprise-business-rules)
  - [Application Layer (Use Cases / Application Business Rules)](#application-layer-use-cases--application-business-rules)
  - [Adapter Layer (Interface Adapters)](#adapter-layer-interface-adapters)
  - [Infrastructure Layer (Frameworks & Drivers)](#infrastructure-layer-frameworks--drivers)
  - [Strategic Architecture](#strategic-architecture)
- [RULES](#rules)
  - [The Fundamental Dependency Rule](#the-fundamental-dependency-rule)
  - [Domain Layer Rules](#domain-layer-rules)
  - [Application Layer Rules](#application-layer-rules)
  - [Adapter Layer Rules](#adapter-layer-rules)
  - [Infrastructure Layer Rules](#infrastructure-layer-rules)
  - [Strategic Design Rules](#strategic-design-rules)
  - [Boundary Crossing Rules](#boundary-crossing-rules)
  - [Testing Rules](#testing-rules)
  - [Packaging Rules](#packaging-rules)
- [JAVA PACKAGE STRUCTURE](#java-package-structure)
  - [Standard Structure (Fully Elaborated)](#standard-structure-fully-elaborated)
  - [Structure Evolution Example: From Startup to Maturity](#structure-evolution-example-from-startup-to-maturity)
- [DEPENDENCY STRUCTURE](#dependency-structure)
  - [Layer Dependency Flow](#layer-dependency-flow)
  - [Request Flow with Dependency Inversion](#request-flow-with-dependency-inversion)
  - [Cross-Bounded Context Communication](#cross-bounded-context-communication)
- [INTEGRATION PATTERNS](#integration-patterns)
  - [Same Bounded Context](#same-bounded-context)
  - [Different Bounded Contexts](#different-bounded-contexts)
- [GENERAL PRINCIPLES](#general-principles)
- [ADDITIONAL TOPICS](#additional-topics)
- [REFERENCES & FURTHER READING](#references--further-reading)

## Key Points

**Domain-Centric Architecture** is an architectural approach that puts **domain logic at the center** and protects it from infrastructure concerns. It synthesizes proven patterns from Domain-Driven Design, Hexagonal Architecture, and Clean Architecture.

### Core Principles

1. **Dependencies Point Inward** - All code dependencies point toward the domain layer. The domain has zero outward dependencies.

2. **Domain is King** - Business logic lives in a rich domain model using tactical DDD patterns (Entities, Value Objects, Aggregates, Domain Services, Domain Events).

3. **Ports & Adapters** - Application layer defines interfaces (ports), infrastructure implements them (adapters). This inverts dependencies.

4. **Bounded Contexts** - Large systems are partitioned into bounded contexts, each with its own ubiquitous language and model.

5. **Event-Driven Integration** - Bounded contexts communicate asynchronously via domain events (internal) and integration events (external).

6. **Progressive Complexity** - Start simple with flat structures, add complexity only when needed based on actual pain points.

### Four Layers

```
Infrastructure  ─→  Frameworks, Database, Message Broker
     ↓ depends on
Adapters       ─→  Controllers, Repositories, API Clients
     ↓ depends on
Application    ─→  Use Cases, Input Ports, Output Ports
     ↓ depends on
Domain         ─→  Entities, Value Objects, Aggregates, Events
(ZERO DEPENDENCIES)
```

### Key Benefits

- ✅ **Business Logic Protection** - Domain isolated from technical concerns
- ✅ **Testability** - Domain and application layers testable without infrastructure
- ✅ **Flexibility** - Easy to swap frameworks, databases, or external services
- ✅ **Team Scaling** - Bounded contexts enable independent teams
- ✅ **Evolution** - Clear path from monolith to microservices
- ✅ **Maintainability** - Clear separation of concerns and explicit boundaries

### When to Use

**Ideal for:**
- Complex business domains with rich logic
- Systems that will evolve and scale over time
- Multiple teams working on different parts
- Event-driven or microservices architectures

**Consider alternatives for:**
- Simple CRUD applications
- Very small systems with minimal business logic
- Short-lived projects or prototypes

### Quick Start

1. **Identify bounded contexts** - Partition your domain by language and model boundaries
2. **Start with 4 layers** - domain, application, adapter, infrastructure (keep it flat initially)
3. **Apply tactical DDD** - Use Entities, Value Objects, Aggregates in domain layer
4. **Define ports** - Input Ports for use cases, Output Ports for infrastructure needs
5. **Implement adapters** - Web controllers, persistence, messaging as adapters
6. **Add complexity progressively** - Subdivide packages only when you feel pain

For detailed patterns and rules, continue reading below. For specific topics, see [Related Documentation](#related-documentation).

## Related Documentation

This document describes the core Domain-Centric Architecture patterns and principles. For specific topics, see:

- **[Clean Architecture Comparison](./clean-architecture-comparison.md)** - Differences from Clean Architecture and when to use each
- **[Deployment Patterns](./deployment-patterns.md)** - Self-Contained Systems, service decomposition, and deployment strategies
- **[Spring Modulith Implementation](./spring-modulith.md)** - Practical implementation using Spring Modulith framework
- **[Team Topologies Integration](./team-topologies.md)** - Organizational patterns and team structure alignment

## ELEMENTS

### Domain Layer (Enterprise Business Rules)

#### Tactical Building Blocks
- **Entity** - Object with identity and lifecycle
- **Value Object** - Immutable object without identity
- **Aggregate** - Transactional consistency boundary
- **Aggregate Root** - Entry point entity for aggregate access
- **Domain Service** - Stateless operation on domain objects
- **Domain Event** - Immutable record of domain occurrence (internal to bounded context)
- **Specification** - Encapsulated business rule
- **Factory** - Complex object creation logic

#### Strategic Building Blocks
- **Bounded Context** - Explicit boundary for unified model
- **Ubiquitous Language** - Shared vocabulary in code and conversation
- **Subdomain** - Logical domain partition (Core, Supporting, Generic)

### Application Layer (Use Cases / Application Business Rules)

#### Use Case Pattern with Input Ports

The application layer organizes business operations using a structured **Use Case pattern** where each use case is isolated in its own folder with dedicated Input/Output models:

**Pattern Structure:**
- **Use Case** - Implements business operation (orchestrates domain objects)
- **Input Port** - Interface defining use case contract (`extends InputPort<INPUT, OUTPUT>`)
- **Command/Query** - Input model (Command for writes, Query for reads)
- **Response** - Output model (standardized return type)
- **Output Port** - Interface for infrastructure needs (repositories, gateways, publishers)

**Organization:**
```
application/
├── {usecasename}/          # e.g., createorder, findorder, cancelorder
│   ├── *InputPort.java     # Interface: public interface CreateOrderInputPort extends InputPort<CreateOrderCommand, CreateOrderResponse>
│   ├── *UseCase.java       # Implementation: @Service public class CreateOrderUseCase implements CreateOrderInputPort
│   ├── *Command.java       # Input model (for write operations)
│   │   OR *Query.java      # Input model (for read operations)
│   └── *Response.java      # Output model
└── shared/                 # Shared output ports
    └── *Repository.java    # Repository interfaces, DomainEventPublisher, etc.
```

**Benefits:**
- ✅ **Single Responsibility** - One use case class per business operation
- ✅ **Explicit Contracts** - Clear input/output via InputPort interface
- ✅ **Self-Contained** - All related files grouped together
- ✅ **Interface Segregation** - Adapters inject only the specific ports they need
- ✅ **Better Hexagonal Alignment** - Input Ports define the application's external API

**Example:**
```java
// Input Port Interface (defines contract)
public interface CreateOrderInputPort extends InputPort<CreateOrderCommand, CreateOrderResponse> {
    CreateOrderResponse execute(CreateOrderCommand command);
}

// Use Case Implementation (orchestrates domain)
@Service
public class CreateOrderUseCase implements CreateOrderInputPort {

    private final OrderRepository orderRepository;  // Output Port
    private final DomainEventPublisher eventPublisher;  // Output Port

    @Override
    public CreateOrderResponse execute(CreateOrderCommand command) {
        // 1. Convert DTO to domain
        Order order = Order.create(command.customerId(), command.items());

        // 2. Business logic (in domain)
        order.validate();

        // 3. Persist via output port
        orderRepository.save(order);

        // 4. Publish events via output port
        eventPublisher.publish(order.getDomainEvents());

        // 5. Convert domain to DTO
        return CreateOrderResponse.from(order);
    }
}

// Input Model (Command)
public record CreateOrderCommand(CustomerId customerId, List<OrderItemDto> items) {}

// Output Model (Response)
public record CreateOrderResponse(OrderId orderId, Money total, OrderStatus status) {
    public static CreateOrderResponse from(Order order) {
        return new CreateOrderResponse(order.getId(), order.getTotal(), order.getStatus());
    }
}
```

#### Application Layer Components

- **Use Case / Application Service** - Orchestrates business operations
- **Input Port** - Interface defining use case entry point
- **Output Port** - Interface for infrastructure needs
- **Command** - Request to change state
- **Query** - Request to retrieve data
- **Input Data / DTO** - Data structure for use case input
- **Output Data / DTO** - Data structure for use case output
- **Repository Interface** - Aggregate persistence abstraction
- **Domain Event Publisher Interface** - Event dispatching abstraction

### Adapter Layer (Interface Adapters)

#### Input Adapters (Driving/Primary)
- **Controller** - Handles HTTP/framework requests
- **Event Consumer** - Handles external events/messages
- **CLI Handler** - Command-line interface
- **GraphQL Resolver** - GraphQL query/mutation handler
- **Scheduled Job** - Time-triggered operations

#### Output Adapters (Driven/Secondary)
- **Repository Adapter** - Persistence implementation
- **Event Publisher Adapter** - Event publishing implementation
- **External API Client** - Third-party service integration
- **Presenter** - Formats use case output for external world
- **Gateway** - Database/external system translation

#### Adapter Components
- **Mapper** - Translation between layers
- **DTO** - Data transfer across boundaries
- **View Model** - Presentation data structure
- **Database Entity** - ORM/persistence model
- **Integration Event** - DTO representing domain event for cross-context communication
- **Event Mapper** - Converts domain events to/from integration events
- **Anti-Corruption Layer (ACL)** - Protects domain from external event formats

### Infrastructure Layer (Frameworks & Drivers)

- **Web Framework** - Spring, Quarkus, etc.
- **Persistence Framework** - JPA, Hibernate, etc.
- **Message Broker** - Kafka, RabbitMQ, etc.
- **Configuration** - Dependency injection, framework setup
- **Cross-Cutting Concerns** - Logging, monitoring, security

### Strategic Architecture

- **Context Map** - Relationships between bounded contexts
- **Anti-Corruption Layer** - Protection from external models
- **Shared Kernel** - Shared code between contexts (see detailed structure below)
- **Open Host Service** - Published integration API
- **Published Language** - Well-documented shared protocol

#### Shared Kernel Pattern (Strategic DDD)

The **Shared Kernel** contains code shared across ALL bounded contexts within your application. It should be kept minimal and requires coordination between teams.

**Structure:**
```
sharedkernel/
├── domain/
│   ├── marker/                    # DDD marker interfaces (define patterns)
│   │   ├── AggregateRoot.java     # Interface for aggregate roots
│   │   ├── Entity.java            # Interface for entities
│   │   ├── Value.java             # Marker for value objects
│   │   ├── DomainEvent.java       # Interface for domain events
│   │   ├── DomainService.java     # Marker for domain services
│   │   ├── Factory.java           # Marker for factories
│   │   └── Specification.java     # Interface for specifications
│   ├── common/                    # Shared value objects (used by multiple contexts)
│   │   ├── Money.java             # Universal money type
│   │   ├── Address.java           # Common address value object
│   │   └── EmailAddress.java      # Common email value object
│   └── exception/                 # Base domain exceptions
│       ├── DomainException.java
│       └── BusinessRuleViolationException.java
└── application/
    └── port/                      # Shared port interfaces (base contracts)
        ├── InputPort.java         # Base interface: InputPort<INPUT, OUTPUT>
        ├── OutputPort.java        # Marker for output ports
        ├── Repository.java        # Base repository interface
        └── DomainEventPublisher.java  # Base event publisher interface
```

**What Belongs in Shared Kernel:**

✅ **Include:**
- **Universal value objects** used by multiple contexts (Money, common IDs if truly shared)
- **DDD marker interfaces** that define your architectural patterns
- **Base port interfaces** (`InputPort<INPUT, OUTPUT>`, `OutputPort`)
- **Base domain exceptions** (DomainException, BusinessRuleViolationException)
- **Cross-cutting domain concepts** that have identical meaning everywhere

❌ **Exclude:**
- **Aggregates** - These belong to specific bounded contexts
- **Business logic** - Should live in context-specific domain layers
- **Context-specific value objects** - Only truly universal ones belong here
- **Use case implementations** - Belong to specific contexts
- **Adapters** - Never shared between contexts

**Guidelines:**
- Keep the Shared Kernel **as small as possible**
- Changes to Shared Kernel affect all contexts - coordinate carefully
- Only include code that has **identical meaning** across all contexts
- When in doubt, duplicate rather than share
- Use versioning if Shared Kernel becomes a separate module

**Example - Marker Interface:**
```java
// sharedkernel/domain/marker/AggregateRoot.java
public interface AggregateRoot<ID> extends Entity<ID> {
    // Marker interface - identifies aggregate roots for all contexts
}

// sharedkernel/domain/marker/Entity.java
public interface Entity<ID> {
    ID getId();
    default boolean isSameAs(Entity<ID> other) {
        return this.getId().equals(other.getId());
    }
}
```

**Example - Shared Value Object:**
```java
// sharedkernel/domain/common/Money.java
public record Money(BigDecimal amount, Currency currency) implements Value {

    public Money {
        Objects.requireNonNull(amount);
        Objects.requireNonNull(currency);
        if (amount.scale() > 2) {
            throw new IllegalArgumentException("Money cannot have more than 2 decimal places");
        }
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Cannot add money with different currencies");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }
}
```

**Example - Base InputPort Interface:**
```java
// sharedkernel/application/port/InputPort.java
public interface InputPort<INPUT, OUTPUT> {
    OUTPUT execute(INPUT input);
}

// Usage in a bounded context:
// order/application/createorder/CreateOrderInputPort.java
public interface CreateOrderInputPort extends InputPort<CreateOrderCommand, CreateOrderResponse> {
    // Inherits execute() method with specific types
}
```

## RULES

### THE FUNDAMENTAL DEPENDENCY RULE

- **All dependencies point inward toward domain**
- Domain has zero outward dependencies
- Domain knows nothing about outer layers
- Application depends only on domain
- Adapters depend on application and domain (through interfaces)
- Infrastructure depends on adapters
- Outer layers know inner layers, never reverse
- Inner layers define interfaces, outer layers implement them

### DOMAIN LAYER RULES

#### Entity Rules
- Entity has unique identity
- Identity remains constant throughout lifecycle
- Entities compared by identity only
- Entity can change attributes while keeping identity
- Entity equality based on ID only
- Entity validates its own invariants

#### Value Object Rules
- Value Objects are immutable
- Value Objects have no identity
- Value Objects compared by all attributes
- Replace entire Value Object instead of modifying
- Value Objects can be shared freely
- Value Objects validate themselves
- Side-effect-free methods only

#### Aggregate Rules
- Access external objects only through Aggregate Root
- Aggregate Root is an Entity
- Aggregate defines transactional boundary
- Aggregate maintains invariants at all times
- Small aggregates preferred
- Reference other aggregates by ID only, not object reference
- One transaction modifies one aggregate only
- Eventual consistency between aggregates
- Delete aggregate deletes all contained entities

#### Domain Service Rules
- Domain Service is stateless
- Use when operation doesn't belong to Entity or Value Object
- Use when operation involves multiple domain objects
- Domain Service operates in domain language
- Domain Service is part of domain layer
- No dependencies on application or outer layers

#### Domain Event Rules (Internal to Bounded Context)
- Domain Events are immutable
- Domain Events use past tense naming (e.g., OrderCreated, not CreateOrder)
- Domain Events represent something that happened in the domain
- Domain Events are defined in `{context}/domain/event/` package
- Domain Events enable eventual consistency within bounded context
- Domain Events emitted by aggregates during state changes
- Domain Events published by use cases via DomainEventPublisher (Output Port)
- Domain Events handled asynchronously when crossing aggregates
- Domain Events must not contain behavior, only data
- Domain Events belong to the domain layer, not adapters

#### Integration Event Rules (Cross-Bounded Context)
- Integration Events are DTOs representing domain events for external systems
- Integration Events defined in `{context}/adapter/outgoing/messaging/event/` package
- Integration Events use past tense + "Event" suffix (e.g., OrderCreatedEvent)
- Integration Events must be serializable (JSON, Protobuf, Avro)
- Integration Events include: event ID, timestamp, version, correlation ID
- Integration Events created by Event Mappers in outgoing adapters
- Domain events never cross bounded context boundaries directly
- Event Mapper converts domain event → integration event DTO
- Integration Events must be backward compatible (add fields, don't remove)
- Integration Events contain only primitives and value types, no domain objects

#### Event Publishing Rules
- Use cases call DomainEventPublisher (Output Port) to publish events
- DomainEventPublisher interface in application/port/out
- DomainEventPublisherAdapter in adapter/outgoing/messaging
- Adapter converts domain events to integration events via mapper
- Message broker (Kafka, RabbitMQ) used for async delivery
- One topic per bounded context or per event type
- Events published after successful transaction commit

#### Event Consumption Rules
- Event Consumer in adapter/incoming/messaging receives integration events
- Anti-Corruption Layer (ACL) protects domain from external formats
- ACL in adapter/incoming/messaging/acl converts events to domain commands
- Event Consumer calls Input Port, never domain directly
- Consuming bounded context maintains its own model
- Eventual consistency between bounded contexts via events

#### General Domain Rules
- Domain contains business logic only
- Domain is framework-agnostic
- Domain is persistence-agnostic
- Domain is UI-agnostic
- Domain uses pure language features
- Domain can be tested without infrastructure
- Domain reflects business, not database structure
- Ubiquitous Language used throughout domain code

### APPLICATION LAYER RULES

#### Use Case / Application Service Rules
- One use case class per business operation
- Use case implements Input Port interface
- Use case orchestrates domain objects
- Use case is thin, delegates to domain
- Use case calls Output Ports for infrastructure
- Use case handles transaction boundaries
- Use case transforms DTOs to domain objects
- Use case transforms domain objects to DTOs
- No business logic in use cases
- Use case tested with port mocks
- Use case knows nothing about presentation
- Use case knows nothing about persistence details

#### Input Port Rules
- Input Port defines use case interface
- Input Port represents business operation
- One Input Port per use case
- Input Port uses domain language
- Input Port accepts Input Data/DTOs
- Input Port has no framework dependencies
- Input Port belongs to application layer

#### Output Port Rules
- Output Port defines infrastructure need
- Output Port uses domain language and types
- Output Port implemented by adapters
- Output Port has no framework dependencies
- Output Port belongs to application layer
- Repository interfaces are Output Ports
- Event Publisher interfaces are Output Ports

#### Repository Interface Rules
- Repository interface in application layer
- Repository returns Aggregate Roots only
- One repository interface per Aggregate Root
- Repository provides collection-like interface
- Repository uses domain types, not DTOs
- Repository manages object lifecycle
- Repository implementation in adapter layer

#### Transaction Rules
- One transaction per use case execution
- Transaction boundaries managed by use case
- Transaction spans single aggregate modification
- Cross-aggregate changes use eventual consistency

### ADAPTER LAYER RULES

#### Input Adapter Rules
- Input Adapter calls Input Port
- Controller extracts data from HTTP request
- Controller creates Input Data/DTO
- Controller delegates to use case
- Controller is thin, no business logic
- Controller handles framework-specific concerns
- One controller method per use case (preferred)

#### Output Adapter Rules
- Output Adapter implements Output Port
- Repository Adapter implements Repository Interface
- Adapter translates between domain and external world
- Adapter contains framework-specific code
- Adapter handles data transformation
- Adapter protects domain from external changes
- Multiple adapters can implement same port
- Adapters are replaceable

#### Presenter Rules
- Presenter implements Output Port (in some variants)
- Presenter formats use case output
- Presenter creates View Models
- Presenter knows about UI needs
- Presenter has no business logic
- Use case doesn't know about presenter implementation

#### Mapper Rules
- Mapper translates between domain and persistence
- Mapper translates between domain and DTOs
- Mapper in adapter layer, not domain
- One mapper per aggregate (typical)

#### General Adapter Rules
- Adapters depend on ports (interfaces)
- Adapters never depend on other adapters
- Adapters can be tested with integration tests
- Adapters handle technical concerns
- Domain types don't leak to external world
- External types don't leak to domain

### INFRASTRUCTURE LAYER RULES

- Framework code stays in infrastructure
- Infrastructure is a detail
- Database is a detail
- Web framework is a detail
- Infrastructure decisions can be delayed
- Infrastructure can be swapped
- No business logic in infrastructure

### STRATEGIC DESIGN RULES

#### Bounded Context Rules
- Each Bounded Context has own Ubiquitous Language
- Each Bounded Context has own model
- Same term can mean different things in different contexts
- Context boundaries are explicit
- Models not unified across contexts
- Teams own Bounded Contexts
- One Bounded Context per deployment unit (preferred)

> **Note:** For deployment variations including multi-service bounded contexts, see [Deployment Patterns](./deployment-patterns.md)

#### Context Integration Rules
- Make all context relationships explicit
- Use Context Map to document relationships
- Protect domain with Anti-Corruption Layer
- Shared Kernel requires team coordination
- Keep Shared Kernel small
- Upstream contexts influence downstream
- Define integration patterns clearly

#### Subdomain Rules
- Focus most effort on Core Domain
- Core Domain provides competitive advantage
- Supporting Subdomains support core
- Generic Subdomains can be outsourced
- Align Bounded Contexts with Subdomains

### BOUNDARY CROSSING RULES

- Data crosses boundaries as simple DTOs
- DTOs have no business logic
- DTOs have no dependencies
- Never pass entities across boundaries
- Never pass value objects across boundaries (convert to DTOs)
- Domain events can cross boundaries (as DTOs)
- Dependencies point inward at boundaries
- Control flow can go any direction
- Use Dependency Inversion when control flow goes outward

### TESTING RULES

- Domain tested in isolation (unit tests)
- Domain tests require no infrastructure
- Domain tests require no frameworks
- Use cases tested with port mocks
- Use cases tested in isolation
- Adapters tested with integration tests
- Full system tested with acceptance tests
- Test pyramid: many unit, fewer integration, few E2E

### PACKAGING RULES

- Package by feature/bounded context preferred
- Layer separation enforced by module structure
- Domain module has zero external dependencies
- Application module depends only on domain
- Adapter modules depend on application
- Infrastructure module depends on adapters
- Modules can be independently deployed

> **Note:** For Spring Modulith module organization, see [Spring Modulith Implementation](./spring-modulith.md)

## JAVA PACKAGE STRUCTURE

### Standard Structure (Fully Elaborated)

#### High-Level Structure Overview

```
com.company.project/
│
├── {boundedcontext}/
│   │
│   ├── domain/              [CORE LAYER]
│   │                        Pure business logic with zero dependencies
│   │                        Entities, Value Objects, Aggregates, Domain Services, Domain Events
│   │
│   ├── application/         [USE CASE LAYER]
│   │                        Orchestrates domain objects and defines boundaries
│   │                        Input Ports, Output Ports, Use Cases, Commands, Queries, DTOs
│   │
│   ├── adapter/             [INFRASTRUCTURE INTERFACE LAYER]
│   │                        Connects application to external world
│   │   ├── incoming/        Controllers, Event Consumers, CLI (call Input Ports)
│   │   └── outgoing/        Repository Impl, API Clients, Publishers (implement Output Ports)
│   │
│   └── infrastructure/      [FRAMEWORKS & DRIVERS LAYER]
│                            Framework-specific configuration and cross-cutting concerns
│                            Spring, JPA, Kafka config, Logging, Security
│
├── sharedkernel/            [SHARED ACROSS ALL CONTEXTS - Keep Minimal]
│   ├── domain/              DDD marker interfaces (AggregateRoot, Entity, Value, etc.)
│   │                        Universal value objects (Money, Address, etc.)
│   │                        Base domain exceptions
│   └── application/port/    Base port interfaces (InputPort<INPUT, OUTPUT>, OutputPort)
│                            Shared repository and publisher interfaces
│
└── infrastructure/          [GLOBAL INFRASTRUCTURE]
                             Application-wide configuration and setup
```

#### Progressive Complexity Principle

This structure shows **ALL possible subdivisions** for a fully-featured bounded context with significant complexity. However, you should **START SIMPLE** and add structure incrementally:

**🎯 Start Minimal**
- Begin with the 4 core layers (domain, application, adapter, infrastructure) without deep nesting
- Place files directly in layer directories until organization becomes difficult
- A simple bounded context may only need 5-10 files total

**📈 Add When Needed**
- Introduce subdirectories only when you have enough files that organization provides clear benefit
- Typically: >10 files in a directory suggests subdividing
- Let pain points guide structure, not theoretical perfection

**🔄 Refactor Later**
- It's easier to add structure later than to maintain unnecessary complexity early
- Moving files into new subdirectories is a simple refactoring
- IDEs handle this automatically with refactoring tools

**The structure below is a REFERENCE showing all options, not a prescription to use everything from day one.**

---

#### Detailed Structure with All Subdivisions

```
com.company.project
│
├── order (bounded context)
│   ├── domain
│   │   ├── model
│   │   │   ├── Order.java (Aggregate Root)
│   │   │   ├── OrderId.java (Value Object)
│   │   │   ├── OrderLine.java (Entity)
│   │   │   ├── Money.java (Value Object)
│   │   │   └── OrderStatus.java (Value Object/Enum)
│   │   ├── service
│   │   │   └── PricingService.java (Domain Service)
│   │   ├── event
│   │   │   ├── OrderCreated.java (Domain Event)
│   │   │   └── OrderCancelled.java (Domain Event)
│   │   └── exception
│   │       ├── OrderNotFoundException.java
│   │       └── InvalidOrderStateException.java
│   │
│   ├── application (use-case focused - each use case self-contained)
│   │   ├── createorder (use case folder - contains ALL related files)
│   │   │   ├── CreateOrderInputPort.java
│   │   │   │   interface CreateOrderInputPort extends InputPort<CreateOrderCommand, CreateOrderResponse> {}
│   │   │   ├── CreateOrderUseCase.java
│   │   │   │   @Service class CreateOrderUseCase implements CreateOrderInputPort { }
│   │   │   ├── CreateOrderCommand.java
│   │   │   └── CreateOrderResponse.java
│   │   │
│   │   ├── findorder (use case folder - contains ALL related files)
│   │   │   ├── FindOrderInputPort.java
│   │   │   │   interface FindOrderInputPort extends InputPort<OrderQuery, OrderResponse> {}
│   │   │   ├── FindOrderUseCase.java
│   │   │   │   @Service class FindOrderUseCase implements FindOrderInputPort { }
│   │   │   ├── OrderQuery.java
│   │   │   └── OrderResponse.java
│   │   │
│   │   ├── cancelorder (use case folder - contains ALL related files)
│   │   │   ├── CancelOrderInputPort.java
│   │   │   │   interface CancelOrderInputPort extends InputPort<CancelOrderCommand, CancelOrderResponse> {}
│   │   │   ├── CancelOrderUseCase.java
│   │   │   │   @Service class CancelOrderUseCase implements CancelOrderInputPort { }
│   │   │   ├── CancelOrderCommand.java
│   │   │   └── CancelOrderResponse.java
│   │   │
│   │   ├── updateorder (use case folder - contains ALL related files)
│   │   │   ├── UpdateOrderInputPort.java
│   │   │   │   interface UpdateOrderInputPort extends InputPort<UpdateOrderCommand, UpdateOrderResponse> {}
│   │   │   ├── UpdateOrderUseCase.java
│   │   │   │   @Service class UpdateOrderUseCase implements UpdateOrderInputPort { }
│   │   │   ├── UpdateOrderCommand.java
│   │   │   └── UpdateOrderResponse.java
│   │   │
│   │   └── shared (SHARED OUTPUT PORTS - infrastructure dependencies)
│   │       ├── OrderRepository.java (Output Port)
│   │       ├── PaymentGateway.java (Output Port)
│   │       ├── InventoryService.java (Output Port)
│   │       └── DomainEventPublisher.java (Output Port)
│   │
│   └── adapter
│       ├── incoming (INCOMING ADAPTERS - call input ports)
│       │   ├── web
│       │   │   ├── OrderController.java
│       │   │   ├── dto
│       │   │   │   ├── CreateOrderWebRequest.java
│       │   │   │   └── OrderWebResponse.java
│       │   │   └── mapper
│       │   │       └── OrderWebMapper.java
│       │   ├── messaging
│       │   │   ├── OrderEventConsumer.java
│       │   │   ├── dto
│       │   │   │   └── ExternalOrderEvent.java
│       │   │   └── acl
│       │   │       └── ExternalEventToCommandMapper.java
│       │   └── cli
│       │       └── OrderCommandHandler.java
│       │
│       └── outgoing (OUTGOING ADAPTERS - implement output ports)
│           ├── persistence
│           │   ├── OrderRepositoryAdapter.java (implements OrderRepository)
│           │   ├── entity
│           │   │   ├── OrderJpaEntity.java
│           │   │   └── OrderLineJpaEntity.java
│           │   ├── mapper
│           │   │   └── OrderPersistenceMapper.java
│           │   └── jpa
│           │       └── SpringDataOrderRepository.java
│           ├── payment
│           │   ├── PaymentGatewayAdapter.java (implements PaymentGateway)
│           │   └── dto
│           │       └── PaymentRequest.java
│           ├── inventory
│           │   └── InventoryServiceAdapter.java (implements InventoryService)
│           └── messaging
│               ├── DomainEventPublisherAdapter.java (implements DomainEventPublisher)
│               ├── event
│               │   ├── OrderCreatedEvent.java (Integration Event DTO)
│               │   └── OrderCancelledEvent.java (Integration Event DTO)
│               └── mapper
│                   └── OrderEventMapper.java
│
├── customer (bounded context)
│   ├── domain
│   ├── application
│   │   ├── registercustomer
│   │   │   ├── RegisterCustomerInputPort.java
│   │   │   ├── RegisterCustomerUseCase.java
│   │   │   ├── RegisterCustomerCommand.java
│   │   │   └── RegisterCustomerResponse.java
│   │   ├── updatecustomer
│   │   │   ├── UpdateCustomerInputPort.java
│   │   │   ├── UpdateCustomerUseCase.java
│   │   │   ├── UpdateCustomerCommand.java
│   │   │   └── UpdateCustomerResponse.java
│   │   ├── findcustomer
│   │   │   ├── FindCustomerInputPort.java
│   │   │   ├── FindCustomerUseCase.java
│   │   │   ├── CustomerQuery.java
│   │   │   └── CustomerResponse.java
│   │   └── shared
│   │       ├── CustomerRepository.java
│   │       └── EmailService.java
│   └── adapter
│       ├── incoming
│       └── outgoing
│
├── inventory (bounded context)
│   ├── domain
│   ├── application
│   │   ├── reservestock
│   │   │   ├── ReserveStockInputPort.java
│   │   │   ├── ReserveStockUseCase.java
│   │   │   ├── ReserveStockCommand.java
│   │   │   └── ReserveStockResponse.java
│   │   ├── releasestock
│   │   │   ├── ReleaseStockInputPort.java
│   │   │   ├── ReleaseStockUseCase.java
│   │   │   ├── ReleaseStockCommand.java
│   │   │   └── ReleaseStockResponse.java
│   │   ├── checkavailability
│   │   │   ├── CheckAvailabilityInputPort.java
│   │   │   ├── CheckAvailabilityUseCase.java
│   │   │   ├── AvailabilityQuery.java
│   │   │   └── AvailabilityResponse.java
│   │   └── shared
│   │       └── StockRepository.java
│   └── adapter
│       ├── incoming
│       └── outgoing
│
├── sharedkernel (Shared across ALL bounded contexts - keep minimal)
│   ├── domain
│   │   ├── marker (DDD pattern interfaces)
│   │   │   ├── AggregateRoot.java
│   │   │   │   public interface AggregateRoot<ID> extends Entity<ID> {}
│   │   │   ├── Entity.java
│   │   │   │   public interface Entity<ID> { ID getId(); }
│   │   │   ├── Value.java
│   │   │   │   public interface Value {}  // Marker for value objects
│   │   │   ├── DomainEvent.java
│   │   │   │   public interface DomainEvent { Instant occurredOn(); }
│   │   │   ├── DomainService.java
│   │   │   │   public interface DomainService {}
│   │   │   ├── Factory.java
│   │   │   │   public interface Factory<T> {}
│   │   │   └── Specification.java
│   │   │       public interface Specification<T> { boolean isSatisfiedBy(T t); }
│   │   ├── common (Universal value objects)
│   │   │   ├── Money.java
│   │   │   ├── Address.java
│   │   │   └── EmailAddress.java
│   │   └── exception (Base domain exceptions)
│   │       ├── DomainException.java
│   │       └── BusinessRuleViolationException.java
│   └── application
│       └── port (Base port interfaces)
│           ├── InputPort.java
│           │   public interface InputPort<INPUT, OUTPUT> {
│           │     OUTPUT execute(INPUT input);
│           │   }
│           ├── OutputPort.java
│           │   public interface OutputPort {}  // Marker interface
│           ├── Repository.java
│           │   public interface Repository<T, ID> extends OutputPort {}
│           └── DomainEventPublisher.java
│               public interface DomainEventPublisher extends OutputPort {
│                 void publish(DomainEvent event);
│               }
│
└── infrastructure (cross-cutting concerns)
    ├── configuration
    │   ├── SpringBootApplication.java
    │   ├── DependencyInjectionConfig.java
    │   ├── WebConfig.java
    │   ├── SecurityConfig.java
    │   └── JpaConfig.java
    ├── persistence
    │   └── DatabaseMigration.java
    ├── messaging
    │   └── KafkaConfig.java
    └── monitoring
        ├── LoggingConfig.java
        └── MetricsConfig.java
```

---

### Structure Evolution Example: From Startup to Maturity

This example shows how a bounded context's structure naturally evolves as complexity grows. For detailed progressive complexity guidelines, see the original structure above.

---

**Use Case Organization - Self-Contained Pattern:**

```
APPLICATION LAYER
├── createorder/                   (USE CASE - All related files together)
│   ├── CreateOrderInputPort.java      ← Input Port Interface
│   │   interface CreateOrderInputPort extends InputPort<CreateOrderCommand, CreateOrderResponse>
│   ├── CreateOrderUseCase.java        ← Use Case Implementation
│   │   @Service class CreateOrderUseCase implements CreateOrderInputPort
│   ├── CreateOrderCommand.java        ← Input Model (Command for writes)
│   └── CreateOrderResponse.java       ← Output Model
│
├── findorder/                     (USE CASE - All related files together)
│   ├── FindOrderInputPort.java        ← Input Port Interface
│   ├── FindOrderUseCase.java          ← Use Case Implementation
│   ├── OrderQuery.java                ← Input Model (Query for reads)
│   └── OrderResponse.java             ← Output Model
│
├── cancelorder/                   (USE CASE - All related files together)
│   ├── CancelOrderInputPort.java      ← Input Port Interface
│   ├── CancelOrderUseCase.java        ← Use Case Implementation
│   ├── CancelOrderCommand.java        ← Input Model
│   └── CancelOrderResponse.java       ← Output Model
│
└── shared/                        (SHARED OUTPUT PORTS)
    ├── OrderRepository.java           ← Output Port (used by multiple use cases)
    ├── PaymentGateway.java            ← Output Port
    ├── InventoryService.java          ← Output Port
    └── DomainEventPublisher.java      ← Output Port
```

**Key Principles:**

1. **Each use case is self-contained** in its own folder with ALL related files
2. **InputPort interface** lives WITH the use case, not in a separate port/in/ directory
3. **UseCase implementation** lives WITH the InputPort in the same folder
4. **Command/Query and Response** models live WITH the use case
5. **Output Ports** (repositories, gateways) are shared across use cases in `shared/` directory

**Naming Convention:**
- Input Ports: `*InputPort extends InputPort<INPUT, OUTPUT>` (e.g., `CreateOrderInputPort extends InputPort<CreateOrderCommand, CreateOrderResponse>`)
- Output Ports: Domain-specific names (e.g., `OrderRepository`, `PaymentGateway`, `DomainEventPublisher`)
- Use Case Implementation: `*UseCase implements *InputPort` (e.g., `CreateOrderUseCase implements CreateOrderInputPort`)
- Commands: `*Command` (e.g., `CreateOrderCommand`)
- Queries: `*Query` (e.g., `OrderQuery`)
- Responses: `*Response` (e.g., `CreateOrderResponse`)
- Adapters: `*Adapter` (e.g., `OrderRepositoryAdapter`, `PaymentGatewayAdapter`)

**Benefits:**
- ✅ **High Cohesion** - All files for one use case are together
- ✅ **Single Responsibility** - One folder = one business operation
- ✅ **Easy Navigation** - Find everything related to a use case in one place
- ✅ **Better Scalability** - Structure grows linearly with use cases
- ✅ **Minimal Coupling** - Use cases are independent, share only via output ports
- ✅ **Clear Dependencies** - Use case depends on domain + shared output ports only
- ✅ **Adapters clearly separated** - `adapter/incoming` and `adapter/outgoing`
- ✅ **Self-documenting** - Folder name = business operation name
- ✅ **Team-friendly** - Different developers can work on different use cases independently


## DEPENDENCY STRUCTURE

### Layer Dependency Flow

```
┌─────────────────────────────────────────────────────┐
│  INFRASTRUCTURE                                     │
│  - Spring Boot, JPA, Kafka, Configuration           │
│  - Glue code only, no business logic                │
└────────────────────┬────────────────────────────────┘
                     │ depends on
                     ↓
┌─────────────────────────────────────────────────────┐
│  ADAPTER                                            │
│                                                     │
│  Input Adapters          Output Adapters            │
│  - Controllers           - Repository Impl          │
│  - Event Consumers       - API Clients              │
│  - CLI Handlers          - Event Publishers         │
│                          - Presenters               │
└────────────────────┬────────────────────────────────┘
                     │ depends on
                     ↓
┌─────────────────────────────────────────────────────┐
│  APPLICATION                                        │
│                                                     │
│  Input Ports ← Use Cases → Output Ports             │
│  (interfaces)  (implementations)  (interfaces)      │
│                                                     │
│  - CreateOrderInputPort  - OrderRepository          │
│  - CreateOrderUseCase    - PaymentGateway           │
│  - DTOs                  - EventPublisher           │
└────────────────────┬────────────────────────────────┘
                     │ depends on
                     ↓
┌─────────────────────────────────────────────────────┐
│  DOMAIN                                             │
│                                                     │
│  - Entities (Order, OrderLine)                      │
│  - Value Objects (Money, OrderId)                   │
│  - Aggregates (Order = Aggregate Root)              │
│  - Domain Services (PricingService)                 │
│  - Domain Events (OrderCreated)                     │
│  - Specifications                                   │
│                                                     │
│  ZERO DEPENDENCIES                                  │
└─────────────────────────────────────────────────────┘
```

### Request Flow with Dependency Inversion

```
HTTP Request
    ↓
┌──────────────────────────────────────────┐
│ Spring Controller (infrastructure)       │
└──────────────────────────────────────────┘
    ↓ delegates to
┌──────────────────────────────────────────┐
│ OrderController (adapter)                │
│ - validates HTTP input                   │
│ - creates CreateOrderCommand (DTO)       │
└──────────────────────────────────────────┘
    ↓ calls (depends on interface)
┌──────────────────────────────────────────┐
│ CreateOrderInputPort (application)       │ ← Interface
└──────────────────────────────────────────┘
    ↑ implemented by
┌──────────────────────────────────────────┐
│ CreateOrderUseCase (application)         │
│ - converts DTO → Domain                  │
│ - calls Order.create()                   │
│ - validates business rules               │
│ - calls repository.save()                │
│ - publishes domain events                │
│ - converts Domain → DTO                  │
└──────────────────────────────────────────┘
    │                           │
    │ uses                      │ calls
    ↓                           ↓
┌─────────────────┐    ┌──────────────────┐
│ Order           │    │ OrderRepository  │ ← Interface (application)
│ (Aggregate)     │    │ (Output Port)    │
│ (domain)        │    └──────────────────┘
│                 │            ↑ implemented by
│ - OrderLine     │    ┌──────────────────────────┐
│ - Money         │    │ OrderRepositoryAdapter   │
│ - OrderId       │    │ (adapter)                │
└─────────────────┘    │ - maps Domain ↔ JPA      │
                       │ - uses Spring Data       │
                       └──────────────────────────┘
                               ↓ uses
                       ┌──────────────────────────┐
                       │ OrderJpaEntity           │
                       │ (adapter)                │
                       └──────────────────────────┘
                               ↓
                       ┌──────────────────────────┐
                       │ Spring Data JPA          │
                       │ (infrastructure)         │
                       └──────────────────────────┘
                               ↓
                           Database
```

### Cross-Bounded Context Communication

```
Order Context                                     Inventory Context
┌───────────────────────┐                        ┌─────────────────────┐
│ CreateOrderUseCase    │                        │ ReserveStockUseCase │
│ (application)         │                        │ (application)       │
└───────────┬───────────┘                        └──────────┬──────────┘
            │ publishes                                     ↑ calls
            ↓                                               │
┌───────────────────────┐                                   │
│ OrderCreated          │                                   │
│ (domain event)        │                                   │
└───────────┬───────────┘                                   │
            │ via DomainEventPublisher                      │
            ↓                                               │
┌───────────────────────┐                                   │
│ OrderEventMapper      │                                   │
│ (adapter/outgoing)    │                                   │
└───────────┬───────────┘                                   │
            │ converts to DTO                               │
            ↓                                               │
┌───────────────────────┐                                   │
│ OrderCreatedEvent     │                                   │
│ (integration event)   │                                   │
└───────────┬───────────┘                                   │
            │                                               │
            └────────→ Message Broker ─────────┐            │
                      (Kafka/RabbitMQ)         │            │
                                               │            │
                                               ↓            │
                          ┌─────────────────────────────────┐
                          │ OrderEventConsumer              │
                          │ (adapter/incoming)              │
                          └──────────┬──────────────────────┘
                                     │ via ACL
                                     ↓
                          ┌─────────────────────────────────┐
                          │ ExternalEventToCommandMapper    │
                          │ (Anti-Corruption Layer)         │
                          │ converts to ReserveStockCommand │
                          └─────────────────────────────────┘
                                     │
                          ┌──────────┘
                          │
                          ↓
                    ┌────────────────────┐
                    │ ReserveStockInput  │
                    │ Port (application) │
                    └────────────────────┘
```

### Complete Cross-Context Event Flow

(Event flow diagram included - see original document for full details)

### Allowed Dependencies

- ✅ Infrastructure → Adapter
- ✅ Adapter → Application
- ✅ Application → Domain
- ✅ Adapter → Port (interface)
- ✅ Use Case → Domain
- ✅ Use Case → Output Port (interface)
- ✅ Controller → Input Port (interface)
- ✅ Outer → Inner (always)

### Forbidden Dependencies

- ❌ Domain → Application
- ❌ Domain → Adapter
- ❌ Domain → Infrastructure
- ❌ Application → Adapter
- ❌ Application → Infrastructure
- ❌ Adapter → Infrastructure
- ❌ Use Case → Controller
- ❌ Use Case → Repository Implementation
- ❌ Port → Adapter (implementation)
- ❌ Inner → Outer (never)

## INTEGRATION PATTERNS

### Same Bounded Context
- Direct method calls within aggregate
- Domain events for cross-aggregate communication
- Use case coordinates multiple aggregates
- Eventual consistency between aggregates

### Different Bounded Contexts
- Anti-Corruption Layer for external models
- Published Language / Open Host Service
- Domain events via message broker
- REST API with DTOs
- Shared Kernel (minimal, coordinated)

> **Note:** For multi-service integration patterns, see [Deployment Patterns](./deployment-patterns.md)

## GENERAL PRINCIPLES

- **Domain First** - Domain is the heart, everything serves it
- **Dependency Inversion** - Depend on abstractions, not concretions
- **Separation of Concerns** - Each layer has single responsibility
- **Framework Independence** - Domain and application know no framework
- **Database Independence** - Domain knows no persistence
- **UI Independence** - Domain knows no presentation
- **Testability** - Test domain without infrastructure
- **Replaceability** - Swap adapters without changing domain
- **Screaming Architecture** - Use cases are visible
- **Ubiquitous Language** - Domain terms everywhere
- **Bounded Contexts** - Explicit boundaries
- **Small Aggregates** - Minimal transactional boundaries
- **Eventual Consistency** - Between aggregates and contexts
- **Interface Segregation** - Small, focused ports
- **Immutability** - Value objects and events are immutable
- **Protection** - Protect domain from external changes
- **Delay Decisions** - Defer infrastructure choices
- **Business Focus** - Code reflects business, not technology

## ADDITIONAL TOPICS

### Deployment Strategies
For information about deployment patterns including Self-Contained Systems (SCS), service decomposition, and multi-service architectures, see [Deployment Patterns](./deployment-patterns.md).

### Implementation with Spring Modulith
For practical implementation using Spring Modulith including event publication, module boundaries, and testing, see [Spring Modulith Implementation](./spring-modulith.md).

### Team Organization and Conway's Law
For guidance on aligning teams with bounded contexts using Team Topologies principles, see [Team Topologies Integration](./team-topologies.md).

## References & Further Reading

Domain-Centric Architecture synthesizes ideas from multiple foundational works and thought leaders. Below are the key sources that have influenced this architectural approach.

### Domain-Driven Design

**Books:**
- **[Domain-Driven Design: Tackling Complexity in the Heart of Software](https://www.domainlanguage.com/ddd/)** by Eric Evans (2003)
  - The seminal work that introduced DDD concepts
  - Defines tactical patterns: Entities, Value Objects, Aggregates, Domain Services
  - Defines strategic patterns: Bounded Contexts, Ubiquitous Language, Context Mapping
  - ISBN: 978-0321125217

- **[Implementing Domain-Driven Design](https://vaughnvernon.com/implementing-domain-driven-design/)** by Vaughn Vernon (2013)
  - Practical guide to implementing DDD patterns
  - Deep dive into Aggregates and bounded contexts
  - Event Sourcing and CQRS patterns
  - ISBN: 978-0321834577

- **[Domain-Driven Design Distilled](https://vaughnvernon.com/domain-driven-design-distilled/)** by Vaughn Vernon (2016)
  - Concise introduction to DDD core concepts
  - Great starting point for learning DDD
  - ISBN: 978-0134434421

**Online Resources:**
- [Domain Language - Eric Evans](https://www.domainlanguage.com/) - Official DDD resources
- [DDD Community](https://github.com/ddd-crew) - Tools, patterns, and community resources

### Hexagonal Architecture (Ports & Adapters)

**Articles:**
- **[Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/)** by Alistair Cockburn (2005)
  - Original article introducing Ports & Adapters pattern
  - Foundation for dependency inversion in Domain-Centric Architecture

**Books:**
- **[Get Your Hands Dirty on Clean Architecture](https://thombergs.gumroad.com/l/gyhdoca)** by Tom Hombergs (2019)
  - Practical implementation of Hexagonal Architecture
  - Detailed package structures and code examples
  - Spring Boot implementation patterns
  - ISBN: 978-1839211966

### Clean Architecture

**Books:**
- **[Clean Architecture: A Craftsman's Guide to Software Structure and Design](https://www.informit.com/store/clean-architecture-a-craftsmans-guide-to-software-structure-9780134494166)** by Robert C. Martin (2017)
  - Defines the dependency rule and layer structure
  - Framework independence principles
  - ISBN: 978-0134494166

**Articles:**
- **[The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)** by Robert C. Martin (2012)
  - Original blog post introducing Clean Architecture circles

### Team Topologies

**Books:**
- **[Team Topologies: Organizing Business and Technology Teams for Fast Flow](https://teamtopologies.com/book)** by Matthew Skelton & Manuel Pais (2019)
  - Four fundamental team types
  - Team interaction modes
  - Conway's Law and organizational design
  - ISBN: 978-1942788812

**Online Resources:**
- [Team Topologies Website](https://teamtopologies.com/) - Official resources and tools
- [Team Topologies Academy](https://teamtopologies.com/academy) - Training and workshops

### Spring Modulith

**Official Resources:**
- **[Spring Modulith Reference Documentation](https://docs.spring.io/spring-modulith/reference/)** - Official documentation
- **[Spring Modulith GitHub](https://github.com/spring-projects/spring-modulith)** - Source code and examples
- **[Spring Blog - Introducing Spring Modulith](https://spring.io/blog/2022/10/21/introducing-spring-modulith)** - Announcement and overview

**Presentations:**
- **[Spring Modulith – Spring for the Architecturally Curious Developer](https://www.youtube.com/watch?v=QX6lP-h-u8I)** by Oliver Drotbohm - SpringOne 2023

### Self-Contained Systems (SCS)

**Online Resources:**
- **[SCS Architecture](https://scs-architecture.org/)** - Official SCS website
  - Principles and characteristics
  - Comparison with microservices
  - Implementation examples

### Microservices & Distributed Systems

**Books:**
- **[Building Microservices: Designing Fine-Grained Systems](https://www.oreilly.com/library/view/building-microservices-2nd/9781492034018/)** by Sam Newman (2nd Edition, 2021)
  - Microservices patterns and practices
  - Service decomposition strategies
  - ISBN: 978-1492034025

- **[Monolith to Microservices](https://www.oreilly.com/library/view/monolith-to-microservices/9781492047834/)** by Sam Newman (2019)
  - Migration patterns from monolith to microservices
  - ISBN: 978-1492047841

### Event-Driven Architecture

**Books:**
- **[Designing Event-Driven Systems](https://www.confluent.io/designing-event-driven-systems/)** by Ben Stopford (2018)
  - Event-driven patterns with Apache Kafka
  - Free ebook from Confluent

**Articles:**
- **[Domain Events vs. Integration Events](https://www.kamilgrzybek.com/blog/posts/domain-events-vs-integration-events)** by Kamil Grzybek
  - Clear explanation of event types

### Software Architecture Patterns

**Books:**
- **[Patterns of Enterprise Application Architecture](https://www.martinfowler.com/books/eaa.html)** by Martin Fowler (2002)
  - Foundational enterprise patterns
  - Repository, Unit of Work, and more
  - ISBN: 978-0321127420

- **[Software Architecture: The Hard Parts](https://www.oreilly.com/library/view/software-architecture-the/9781492086888/)** by Neal Ford, Mark Richards, Pramod Sadalage, Zhamak Dehghani (2021)
  - Modern architectural decision-making
  - Trade-off analysis
  - ISBN: 978-1492086895

### Influential Blogs & Communities

**Blogs:**
- **[Martin Fowler's Blog](https://martinfowler.com/)** - Software architecture patterns and practices
- **[Vaughn Vernon's Blog](https://vaughnvernon.com/)** - DDD patterns and implementations
- **[Udi Dahan's Blog](https://udidahan.com/)** - SOA and DDD insights
- **[Tom Hombergs' Blog (Reflectoring)](https://reflectoring.io/)** - Clean/Hexagonal Architecture tutorials

**Communities:**
- **[DDD/CQRS Google Group](https://groups.google.com/g/dddcqrs)** - Active DDD community
- **[Software Architecture Slack](https://softwarearchitecture.slack.com/)** - Architecture discussions
- **[Virtual DDD](https://virtualddd.com/)** - Online DDD meetups and resources

### Related Patterns & Practices

**CQRS (Command Query Responsibility Segregation):**
- **[CQRS Pattern](https://martinfowler.com/bliki/CQRS.html)** by Martin Fowler
- **[CQRS Journey](https://docs.microsoft.com/en-us/previous-versions/msp-n-p/jj554200(v=pandp.10))** by Microsoft patterns & practices

**Event Sourcing:**
- **[Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)** by Martin Fowler
- **[Event Sourcing Basics](https://eventstore.com/blog/what-is-event-sourcing/)** by Event Store

**Aggregates:**
- **[Effective Aggregate Design](https://www.dddcommunity.org/library/vernon_2011/)** by Vaughn Vernon (3-part series)

### Acknowledgments

Domain-Centric Architecture stands on the shoulders of giants. Special recognition to:

- **Eric Evans** - For Domain-Driven Design and the concept of Bounded Contexts
- **Alistair Cockburn** - For Hexagonal Architecture and the Ports & Adapters pattern
- **Robert C. Martin (Uncle Bob)** - For Clean Architecture and the Dependency Rule
- **Vaughn Vernon** - For practical DDD implementation guidance
- **Matthew Skelton & Manuel Pais** - For Team Topologies and organizational patterns
- **Tom Hombergs** - For practical Hexagonal Architecture implementation examples
- **Oliver Drotbohm** - For Spring Modulith framework
- **The DDD Community** - For continuous evolution of patterns and practices

### Contributing to This Documentation

This documentation is a living resource. Contributions, corrections, and improvements are welcome. The patterns and practices described here continue to evolve based on real-world experience and community feedback.
