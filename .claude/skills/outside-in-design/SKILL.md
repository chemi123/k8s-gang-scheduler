---
name: outside-in-design
description: Structured system design workflow that starts from interfaces and works inward. Use when designing new systems, features, or components that need CRD/API/schema definitions, internal behavior decisions, and edge case handling.
---

# Outside-In Design Workflow

Structured design process that starts from the outermost layer (interfaces) and works inward (internal behavior, edge cases). Decisions are documented incrementally in REQUIREMENTS.md and committed at each milestone.

## When to Activate

- Designing a new system or component from scratch
- Defining CRDs, APIs, or schemas before implementation
- Refactoring a system with interface changes
- Any design work where "what comes first?" is unclear

## Core Principle

**Always work from the outside in.** The outermost layer is the one users or other systems interact with (interfaces, schemas, APIs). Inner layers are implementation decisions that depend on the outer ones.

## Design Layers (in order)

### Layer 1: Interface / Schema Definition

Define what users and other components see first:
- CRD specs, API schemas, data models
- Resource relationships and ownership
- Scoping (cluster-scoped vs namespaced)
- Validation rules

At this stage, focus on **spec fields** (what users provide). Status fields come later.

For each resource/interface:
1. Draft the schema as a concrete YAML/JSON example
2. List each field with a one-line description
3. Identify relationships between resources
4. Confirm with the user before moving on

### Layer 2: Flow / Lifecycle

Define how resources move through the system:
- State transitions (e.g., Pending -> Running -> Succeeded/Failed)
- Which component is responsible for each transition
- The end-to-end flow from user action to completion

Present as a numbered sequence or ASCII diagram. Verify the flow is consistent with the schemas from Layer 1.

### Layer 3: Edge Cases / Error Handling

Identify and resolve abnormal scenarios:
- Failure mid-operation (partial state, rollback)
- Crash recovery
- Resource cleanup
- External system failures
- Concurrency / race conditions

For each edge case:
1. State the scenario in one sentence
2. Propose a resolution
3. Confirm with the user
4. Document in one or two sentences

### Layer 4: Operational Concerns

Address deployment and runtime decisions:
- Monitoring and observability
- Performance characteristics
- Scaling approach
- TTL / cleanup policies

## Documentation Format

Use REQUIREMENTS.md as the single source of truth. Structure:

```markdown
# Project Name

## Background
## Architecture
## Core Rules / Invariants
## Lifecycle
## Flow
## Scope Exclusions
## Edge Cases / Error Handling
### Topic 1
### Topic 2
...
## Schemas
### Resource A
### Resource B
...
## Open Items
- [x] Resolved item -> decision
- [ ] Pending item
  - [ ] Sub-item
```

### Writing Guidelines

- **Section names must be self-explanatory to a first-time reader.** "Edge Cases / Error Handling" not "Internal Design".
- **Each decision is one or two sentences.** Not paragraphs.
- **Schemas use concrete YAML examples** with a field description list below.
- **Open items use checkboxes** with nesting for sub-items. Mark resolved items with the decision.

## Workflow Rules

### Commit Cadence
- Commit and push after each design decision or group of related decisions
- Never accumulate large uncommitted changes
- Commit messages: `docs: <what was decided>`

### Contradiction Detection
After each decision, check for contradictions with previous decisions:
- Does this break an ownership chain?
- Does this conflict with a lifecycle rule?
- Does this make a previously decided flow impossible?

If a contradiction is found, raise it immediately before documenting.

### Staying at the Right Layer
- Don't jump to implementation details while defining interfaces
- Don't define internal behavior before the interface is stable
- If a question arises from a deeper layer, note it in Open Items and continue at the current layer

### Confirming with the User
- Present a concrete proposal (schema, decision, flow)
- Ask a specific yes/no or choice question
- Don't present open-ended options when a recommendation is possible
- Incorporate domain expertise the user shares from their current environment

## Anti-Patterns

- **Starting with implementation** before interfaces are defined
- **Vague section names** that require context to understand
- **Long unstructured discussions** without documenting decisions
- **Accumulating design debt** by skipping edge cases "for later"
- **Over-engineering** by adding fields or behaviors not yet needed
- **Deleting resolved items** instead of marking them with checkmarks
