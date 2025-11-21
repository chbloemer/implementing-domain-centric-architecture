# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a **documentation repository** focused on Domain-Centric Architecture - a synthesis of Domain-Driven Design (DDD), Hexagonal Architecture (Ports & Adapters), and Clean Architecture principles. It contains no source code, only comprehensive architectural guidance and implementation patterns.

**Purpose:** Reference documentation for teams implementing domain-centric architectures in Java/Spring Boot projects.

**Reference Implementation:** [ai-architecture-sample](https://github.com/chbloemer/ai-architecture-sample) - Complete working implementation demonstrating all patterns described in this documentation.

## Documentation Structure

The repository contains interconnected markdown documents:

### Core Documentation
- **README.md** - Main architectural reference guide covering:
  - Four-layer architecture (Domain, Application, Adapter, Infrastructure)
  - Tactical DDD patterns (Entities, Value Objects, Aggregates, Domain Events)
  - Strategic DDD patterns (Bounded Contexts, Shared Kernel, Context Maps)
  - Java package structures and naming conventions
  - Comprehensive architectural rules and principles
  - Integration patterns and dependency flows

### Supplementary Guides
- **spring-modulith.md** - Practical implementation using Spring Modulith framework
- **archunit-governance.md** - Automated architecture testing with ArchUnit
- **clean-architecture-comparison.md** - Comparison with Clean Architecture and when to use each
- **deployment-patterns.md** - Self-Contained Systems, service decomposition, deployment strategies
- **team-topologies.md** - Organizational patterns and team structure alignment

### Templates
- **adr-template.md** - Architecture Decision Record template

## Key Architectural Patterns

### Layer Structure
```
Infrastructure → Adapters → Application → Domain
                                          (zero dependencies)
```

### Use Case Organization (Application Layer)
Each use case is self-contained in its own folder:
```
application/
├── {usecasename}/
│   ├── *InputPort.java      # Interface: extends InputPort<INPUT, OUTPUT>
│   ├── *UseCase.java         # Implementation
│   ├── *Command.java         # Input (writes) or *Query.java (reads)
│   └── *Response.java        # Output
└── shared/
    └── *Repository.java      # Shared Output Ports
```

### Bounded Context Pattern
```
com.company.project/
├── {boundedcontext}/
│   ├── domain/               # Pure business logic (zero dependencies)
│   ├── application/          # Use cases, ports, orchestration
│   ├── adapter/
│   │   ├── incoming/         # Controllers, event consumers
│   │   └── outgoing/         # Repository impls, API clients
│   └── infrastructure/       # Framework configuration
└── sharedkernel/             # Keep minimal - DDD markers, universal value objects
```

### Event Patterns
- **Domain Events** - Internal to bounded context (in `domain/event/`)
- **Integration Events** - Cross-context DTOs (in `adapter/outgoing/messaging/event/`)
- **Event Mappers** - Convert domain events to integration events

## Naming Conventions

### Application Layer
- **Use Case Folders**: lowercase (e.g., `createorder`, `additemtocart`, `getproductbyid`)
- **Input Ports**: `*InputPort extends UseCase<Command, Response>` (extends `UseCase`, not `InputPort`)
- **Output Ports**: Domain names (e.g., `OrderRepository`, `PaymentGateway`, `DomainEventPublisher`)
- **Use Cases**: `*UseCase implements *InputPort`
- **Commands**: `*Command` (writes)
- **Queries**: `*Query` (reads)
- **Responses**: `*Response`

### Adapter Layer
- **Incoming Adapters**:
  - Web: `*PageController` (e.g., `ProductPageController`)
  - API: `*Resource` (e.g., `ProductResource`, `OrderResource`)
  - Event: `*EventConsumer` (e.g., `ProductEventConsumer`, `CartEventConsumer`)
  - MCP: `*McpToolProvider` (e.g., `ProductCatalogMcpToolProvider`)
- **Outgoing Adapters**: Implementation-specific (e.g., `InMemoryProductRepository`, `OrderRepositoryAdapter`)

### Domain Layer
- **Domain Events**: Past tense (e.g., `OrderCreated`, `OrderCancelled`)
- **Integration Events**: Past tense + "Event" suffix (e.g., `OrderCreatedEvent`)

## Key Implementation Details from Reference

The [ai-architecture-sample](https://github.com/chbloemer/ai-architecture-sample) demonstrates these specific choices:

### Shared Kernel Structure
- **Base Interface**: `UseCase<INPUT, OUTPUT>` (not `InputPort<INPUT, OUTPUT>`)
- **Marker Interfaces**: Includes `Id.java`, `IntegrationEvent.java`, `BaseAggregateRoot.java`
- **Specification Pattern**: Fully implemented in `sharedkernel/domain/spec/` with Composite, And, Or, Not specifications
- **Common Value Objects**: `Money.java`, `Price.java`, `ProductId.java`

### Domain Layer
- **Structure**: Only `model/`, `event/`, `service/` subdirectories
- **No separate**: `exception/` or `specification/` packages at domain level

### Adapter Layer
- **Incoming**: `web/`, `api/`, `event/`, `mcp/` (Model Context Protocol for AI integration)
- **Outgoing**: Simplified `persistence/` with `InMemoryRepository` (no complex JPA entity structure in simple version)

### Technology Choices
- Java 21+
- Spring Boot 3.x
- ArchUnit for architecture testing
- In-memory storage for simplicity (ConcurrentHashMap)
- Model Context Protocol (MCP) server for AI tooling

## Progressive Complexity Principle

**Critical:** The documentation shows ALL possible subdivisions for reference. In practice:

1. **Start simple** - Begin with 4 core layers without deep nesting
2. **Add when needed** - Introduce subdirectories only when >10 files in a directory
3. **Refactor later** - Easier to add structure than maintain unnecessary complexity early

The detailed package structures are **reference templates**, not prescriptions to use everything from day one.

## When Editing Documentation

### Making Changes
When updating this documentation, maintain:
- **Cross-references** - Keep document links accurate (use relative paths)
- **Consistency** - Ensure examples align across documents
- **Completeness** - Update all affected sections when changing patterns
- **Clarity** - Use examples to illustrate complex concepts

### Key Sections to Update Together
If changing architectural patterns, update these sections across documents:
1. Main README.md rules and package structures
2. Spring Modulith implementation examples
3. ArchUnit test examples
4. Deployment pattern implications

### Code Examples
All code examples should:
- Use consistent package naming: `com.company.project.{boundedcontext}`
- Follow established naming conventions (see above)
- Show complete context (package declarations, imports where relevant)
- Include comments explaining the pattern being demonstrated

### Cross-Document Dependencies
- Spring Modulith guide assumes knowledge from main README.md
- ArchUnit guide references rules from main README.md
- Deployment patterns extend concepts from main README.md and Team Topologies
- All guides reference back to core concepts in README.md

## Git Workflow

This is a documentation-focused repository:
- Changes should maintain consistency across all documents
- Use ADR template for significant architectural decisions
- Commit messages should reference which pattern/section is being updated

## Java Version

Examples assume Java 21+ with:
- Records for Value Objects and DTOs
- Modern Java features (var, text blocks where appropriate)
- Spring Boot 3.x conventions
