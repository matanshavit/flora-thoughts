---
date: 2025-10-16T01:43:40-04:00
researcher: matanshavit
git_commit: a9cd08eed7d02f35d3820dd9b5e40defa6014db2
branch: dev-webgl
repository: flora-frontend
topic: "Codebase Coupling Analysis and Decoupling Opportunities"
tags:
  [
    research,
    codebase,
    architecture,
    coupling,
    decoupling,
    refactoring,
    state-management,
    node-system,
  ]
status: complete
last_updated: 2025-10-16
last_updated_by: matanshavit
---

# Research: Codebase Coupling Analysis and Decoupling Opportunities

**Date**: 2025-10-16T01:43:40-04:00
**Researcher**: matanshavit
**Git Commit**: a9cd08eed7d02f35d3820dd9b5e40defa6014db2
**Branch**: dev-webgl
**Repository**: flora-frontend

## Research Question

I notice some tickets require changes in a lot of disparate files, while others do not. Examine how tightly coupled or not the codebase is and find opportunities for decoupling that would improve the code.

## Summary

The Flora frontend exhibits a **hub-and-spoke architecture** centered around `ProjectStore` (1,938 lines) that 122+ files depend on directly. This creates a situation where many features require changes across multiple files due to:

1. **Triplicated logic** - Mode determination and parent node selection implemented 3+ times
2. **Direct store access** - 171 instances of `.getState()` calls bypass reactive patterns
3. **Schema-driven coupling** - Database structure flows directly to UI without DTOs
4. **Missing abstraction layers** - No services for node selection, mode determination, or endpoint filtering
5. **Configuration sprawl** - 7,382-line endpoints-config.ts mixes multiple concerns

While this centralized approach enables real-time collaboration, it creates widespread coupling that makes feature changes require modifications to 5-20+ files on average.

## Current Architecture Analysis

### 1. Central Hub Pattern - ProjectStore

The codebase is organized around a central `ProjectStore` at `src/lib/projects/context.tsx` that acts as the hub for all canvas and node operations:

**File**: `src/lib/projects/context.tsx:1-1938`

- **Dependencies**: 122 files import and use this store
- **State Properties**: 50+ properties including nodes, edges, inputs, outputs, history
- **Methods**: 40+ methods for node CRUD, history management, viewport control
- **Liveblocks Integration**: Real-time sync for multiplayer

This creates a **critical coupling point** where:

- Any change to ProjectState structure affects 122+ files
- Components directly access store internals via hooks
- Services require full ProjectStore reference to function
- No abstraction layer between store structure and consumers

### 2. Triplicated Node Logic Pattern

The same "determine node generation mode" logic exists in three places:

**Location 1**: `src/components/r3f/video-block.tsx:86-194`

```typescript
const memoizedSelector = createEvenMoreExpensiveSelector();
// Count parent nodes by output type
// Determine mode based on counts
// Select endpoints based on mode
```

**Location 2**: `src/lib/generation/generation-service.ts:400-557`

```typescript
private getNodeGenerationProps(nodeId: string) {
  // Count parent nodes by output type (duplicate logic)
  // Determine mode based on counts (duplicate logic)
  // Select endpoints based on mode (duplicate logic)
}
```

**Location 3**: `src/components/nodes/models/block-modes.ts:1-94`

```typescript
export function determineBlockMode(...) {
  // Central function exists but not used everywhere
}
```

**Impact**: Adding a new node mode requires updating all three locations with identical logic.

### 3. Direct Database-to-UI Coupling

The database schema at `src/db/schema.ts` directly shapes the UI state:

```typescript
// Database
projects table: {
  nodeInputsMap: JSONB
  nodeOutputsMap: JSONB
  flow: JSON
}

// Mirrors exactly in ProjectState
ProjectState: {
  nodeInputsMap: Record<string, unknown>
  nodeOutputsMap: Record<string, FloraNodeOutput>
  nodes: Node[]
}
```

**Impact**:

- No transformation layer between DB and UI
- Schema changes require updates to 128+ files
- Components expect exact database structure

### 4. Endpoint Configuration Monolith

The file `src/lib/schema/endpoints-config.ts` (7,382 lines) combines:

- Endpoint definitions
- System prompts
- User prompts
- Model parameters
- Request transformers
- Response mappers
- Grouping logic

**Impact**: 40+ files import this massive configuration object for different purposes.

### 5. State Access Anti-Patterns

Found 171 instances of direct state access:

```typescript
// Pattern throughout codebase
projectStore.getState().nodes;
selectionStore.getState().clearSelection();
onboardingStore.getState().forcedOnboardingActive;
```

**Impact**:

- Bypasses React's reactive subscriptions
- Makes testing difficult
- Creates hidden dependencies

## Identified Coupling Patterns

### Pattern Analysis Table

| Pattern                  | Files Affected | Change Impact          | Risk Level |
| ------------------------ | -------------- | ---------------------- | ---------- |
| ProjectStore dependency  | 122            | Any state shape change | CRITICAL   |
| Schema type imports      | 128            | Database changes       | HIGH       |
| Constants imports        | 74             | Node type changes      | HIGH       |
| Endpoint config usage    | 40+            | Model changes          | MEDIUM     |
| Logic triplication       | 3x3 node types | Mode/selection changes | MEDIUM     |
| Direct .getState() calls | 171 instances  | Hidden dependencies    | MEDIUM     |

### File Change Requirements by Feature Type

**Adding a new node type requires changes to:**

1. `src/app/app/constants.ts` - Add to NodeTypes enum and nodesConfig
2. `src/components/nodes/types.ts` - Add data types
3. `src/components/nodes/models/block-modes.ts` - Add mode determination
4. `src/components/r3f/[new-node]-block.tsx` - Create component with duplicated logic
5. `src/lib/generation/generation-service.ts` - Add switch case
6. `src/lib/schema/endpoints-config.ts` - Add endpoint arrays
7. `src/lib/generation/processing/` - Add processing logic
8. Multiple test files

**Total: 8-12 files minimum**

## Decoupling Opportunities

### Opportunity 1: Extract Node Selection Service

**Current State**: Parent node selection logic duplicated in every block component and generation service.

**Proposed Solution**:

```typescript
// New file: src/lib/nodes/node-selection-service.ts
export class NodeSelectionService {
  getParentNodes(nodeId: string, edges: Edge[], nodes: Node[]): ParentNodeData;
  getConnectedCounts(nodeId: string, state: ProjectState): OutputTypeCounts;
  determineMode(nodeType: NodeTypes, counts: OutputTypeCounts): BlockMode;
}
```

**Benefits**:

- Single source of truth for parent traversal
- Reduces 3 implementations to 1
- Testable in isolation
- Changes localized to one file

**Implementation Effort**: Medium (2-3 days)

### Opportunity 2: Create DTO Layer for Database

**Current State**: Database schema directly used in UI components.

**Proposed Solution**:

```typescript
// New file: src/lib/dto/project-dto.ts
export class ProjectDTO {
  static fromDatabase(dbProject: DBProject): ProjectState;
  static toDatabase(state: ProjectState): DBProject;
  static migrateSchema(data: unknown): ProjectState;
}
```

**Benefits**:

- Isolates UI from database changes
- Enables schema evolution without breaking UI
- Centralizes data transformation
- Supports versioning and migration

**Implementation Effort**: High (1 week)

### Opportunity 3: Split Endpoint Configuration

**Current State**: Single 7,382-line file with mixed concerns.

**Proposed Solution**:

```
src/lib/endpoints/
  ├── definitions/
  │   ├── text-to-image.ts
  │   ├── image-to-image.ts
  │   └── ...
  ├── prompts/
  │   ├── system-prompts.ts
  │   └── improvement-prompts.ts
  ├── transformers/
  │   ├── request-mappers.ts
  │   └── response-mappers.ts
  └── index.ts (exports aggregated config)
```

**Benefits**:

- Modular configuration
- Parallel development possible
- Easier to find and modify specific endpoints
- Reduces merge conflicts

**Implementation Effort**: Low (1 day)

### Opportunity 4: Create Generation Context Interface

**Current State**: GenerationService directly coupled to ProjectStore structure.

**Proposed Solution**:

```typescript
// New interface
interface GenerationContext {
  getNode(id: string): Node
  getParentNodes(id: string): Node[]
  getNodeInput(id: string): unknown
  updateNodeOutput(id: string, output: FloraNodeOutput): void
}

// ProjectStore implements interface
class ProjectStore implements GenerationContext { ... }

// GenerationService depends on interface
class GenerationService {
  constructor(private context: GenerationContext) { ... }
}
```

**Benefits**:

- Decouples service from store implementation
- Enables testing with mock contexts
- Allows alternative implementations
- Clear contract between layers

**Implementation Effort**: Medium (2-3 days)

### Opportunity 5: Centralize Mode and Endpoint Selection

**Current State**: Each block component implements own endpoint filtering and selection.

**Proposed Solution**:

```typescript
// New file: src/lib/endpoints/endpoint-selector.ts
export class EndpointSelector {
  getEndpointsForMode(mode: BlockMode): FloraEndpoint[];
  filterByConstraints(endpoints: FloraEndpoint[], constraints: Constraints): FloraEndpoint[];
  selectDefault(endpoints: FloraEndpoint[], userPrefs: UserPreferences): FloraEndpoint;
}
```

**Benefits**:

- Eliminates duplicate filtering logic
- Consistent endpoint selection
- Centralized business rules
- Easier to modify selection criteria

**Implementation Effort**: Low (1-2 days)

### Opportunity 6: Replace Direct State Access with Facades

**Current State**: 171 instances of `.getState()` bypass React patterns.

**Proposed Solution**:

```typescript
// Add facade methods to stores
class ProjectStoreFacade {
  // Instead of: projectStore.getState().nodes
  getNodes(): Node[];

  // Instead of: projectStore.getState().updateNode(...)
  updateNodeById(id: string, updates: Partial<Node>): void;
}
```

**Benefits**:

- Encapsulates state structure
- Enables refactoring without changing call sites
- Can add validation/logging
- Maintains single access pattern

**Implementation Effort**: Medium (3-4 days)

### Opportunity 7: Extract Node Type Registry

**Current State**: Node types hardcoded in constants with switch statements.

**Proposed Solution**:

```typescript
// New pattern: Registry-based node system
class NodeTypeRegistry {
  private types = new Map<NodeTypes, NodeTypeDefinition>();

  register(type: NodeTypes, definition: NodeTypeDefinition): void;
  getDefinition(type: NodeTypes): NodeTypeDefinition;
  getMode(type: NodeTypes, context: NodeContext): BlockMode;
  getEndpoints(type: NodeTypes, mode: BlockMode): FloraEndpoint[];
}
```

**Benefits**:

- Add new node types without modifying existing code
- Plugin-like architecture
- Open/closed principle
- Reduces switch statement proliferation

**Implementation Effort**: High (1 week)

## Implementation Priority Matrix

| Opportunity                  | Impact    | Effort | Priority   | Files Saved per Feature |
| ---------------------------- | --------- | ------ | ---------- | ----------------------- |
| Split Endpoint Config        | Medium    | Low    | **HIGH**   | 2-3 files               |
| Endpoint Selector Service    | High      | Low    | **HIGH**   | 3-4 files               |
| Node Selection Service       | High      | Medium | **HIGH**   | 4-5 files               |
| Generation Context Interface | High      | Medium | **MEDIUM** | 2-3 files               |
| State Access Facades         | Medium    | Medium | **MEDIUM** | 1-2 files               |
| DTO Layer                    | High      | High   | **LOW**    | 3-4 files               |
| Node Type Registry           | Very High | High   | **LOW**    | 5-8 files               |

## Quick Wins (Can implement immediately)

1. **Use existing `determineBlockMode()` everywhere** - The function exists but isn't used consistently
2. **Create helper for parent node counting** - Extract the duplicated counting logic
3. **Split endpoints-config.ts by provider** - Simple file reorganization
4. **Add facade methods to ProjectStore** - Gradual migration from `.getState()`

## Long-term Architectural Improvements

1. **Migrate to plugin architecture for nodes** - Allow dynamic node type registration
2. **Implement proper DTO layer** - Separate database schema from domain model
3. **Create service layer abstraction** - Decouple UI from business logic
4. **Adopt repository pattern** - Abstract data access behind interfaces
5. **Implement CQRS for state management** - Separate read models from write models

## Metrics for Success

After implementing decoupling:

- **File changes per feature**: Reduce from 8-12 to 3-5 files
- **Test isolation**: Services testable without ProjectStore
- **Merge conflicts**: 50% reduction in configuration file conflicts
- **New developer onboarding**: Find and modify specific functionality faster
- **Build times**: Reduced recompilation scope

## Code References

Key coupling points to address:

- `src/lib/projects/context.tsx:1-1938` - Central ProjectStore
- `src/lib/schema/endpoints-config.ts:1-7382` - Monolithic configuration
- `src/components/r3f/video-block.tsx:86-194` - Duplicated selector logic
- `src/lib/generation/generation-service.ts:400-557` - Duplicated mode determination
- `src/app/app/constants.ts:73-302` - Hardcoded node registry

## Historical Context

Previous developers have noted similar issues:

- `thoughts/shared/research/2025-01-14-ENG-982-workspace-user-membership-creation-refactoring.md` - Documented duplication patterns
- `thoughts/shared/prs/2063_description.md` - Addressed routing duplication with centralized helpers
- Multiple tickets document coupling making features hard to implement

## Related Research

- `thoughts/shared/research/2025-10-14-performance-reliability-features-analysis.md` - Performance impacted by tight coupling
- `thoughts/shared/research/2025-10-13-eng-976-guest-permissions.md` - Permission logic scattered due to coupling

## Open Questions

1. Would migrating to a plugin architecture break existing Liveblocks integration?
2. How much of the triplicated logic is actually intentional for performance?
3. Should we maintain backward compatibility with current ProjectState structure?
4. Is the team willing to invest in a DTO layer given the effort required?
