# EffectGPU Vision

EffectGPU is an idiomatic Effect wrapper around TypeGPU that brings functional programming principles, dependency injection, error safety, and modularity to WebGPU programming.

## Core Principles

### 1. Effect-First Architecture

Everything in EffectGPU is an Effect, providing:
- **Composable computations**: All GPU operations are Effect programs that can be composed
- **Explicit error handling**: GPU errors are tracked through Effect's error channel
- **Resource safety**: Automatic cleanup and resource management through Effect's Scope
- **Testability**: All operations can be tested without actual GPU hardware

### 2. Service-Based Dependency Injection

GPU resources are provided through Effect services, enabling:
- **Modularity**: Swap implementations (e.g., mock GPU for testing)
- **Explicit dependencies**: Type-safe dependency tracking
- **Layered architecture**: Services can depend on other services
- **Testability**: Easy to inject test doubles

### 3. Branded Types for Type Safety

All GPU resources use branded types to prevent:
- **Type confusion**: Can't accidentally mix different resource types
- **Invalid operations**: Type system prevents unsafe operations
- **Better IDE support**: Autocomplete and type checking

### 4. Error Safety

All operations that can fail are typed with Effect's error channel:
- **GPU initialization failures**: Tracked as errors
- **Resource creation failures**: Explicit error types
- **Shader compilation errors**: Typed error responses
- **Runtime GPU errors**: Captured and handled

## Architecture Overview

### Service Layer Structure

```typescript
// Core services
- GPUDeviceService: Provides GPUDevice
- GPUAdapterService: Provides GPUAdapter  
- TypeGPURootService: Provides TgpuRoot (wrapped in Effect)
- BufferService: Buffer creation and management
- TextureService: Texture creation and management
- PipelineService: Pipeline creation and execution
- ShaderService: Shader compilation and resolution
- DataIOService: Data I/O operations (read/write buffers)
- DataValidationService: Schema validation and type checking
```

### Layer Composition

Services are composed in layers:
1. **Infrastructure Layer**: GPUDevice, GPUAdapter
2. **Resource Layer**: Buffers, Textures, Samplers
3. **Computation Layer**: Pipelines, Shaders
4. **Application Layer**: High-level GPU operations

Each layer depends only on layers below it, enabling clean separation of concerns.

## Key Design Decisions

### 1. Effect Wrapping Strategy

TypeGPU's imperative API is wrapped in Effect operations:
- **Synchronous operations** → `Effect.sync()`
- **Async operations** → `Effect.promise()` or `Effect.async()`
- **Resource creation** → `Effect.acquireUseRelease()` with Scope
- **Error-prone operations** → `Effect.tryPromise()` or `Effect.try()`
- **Data I/O operations** → `Effect.try()` for read/write operations (in `effectgpu/data`)
- **Pure operations** → Re-exported from `typegpu/data` unchanged (in `effectgpu/data`)

### 2. Resource Lifecycle Management

All GPU resources are managed through Effect's Scope:
- Automatic cleanup when scope closes
- Resource sharing across Effect programs
- Leak prevention through type system

### 3. Error Types

Structured error types for different failure modes:
- `GPUInitializationError`: Device/adapter failures
- `ResourceCreationError`: Buffer/texture creation failures
- `ShaderCompilationError`: Shader resolution failures
- `PipelineExecutionError`: Runtime execution failures

### 4. Branded Types

All resources use branded types:
- `EffectGPUBuffer<T>`: Branded buffer type
- `EffectGPUTexture<T>`: Branded texture type
- `EffectGPUPipeline<T>`: Branded pipeline type
- Prevents accidental type mixing

## Benefits Over Plain TypeGPU

1. **Composability**: All operations are Effects that compose naturally
2. **Error Safety**: Errors are tracked in types, not thrown
3. **Testability**: Mock services enable testing without GPU
4. **Resource Safety**: Automatic cleanup prevents leaks
5. **Modularity**: Services can be swapped and tested independently
6. **Type Safety**: Branded types prevent invalid operations
7. **Functional Style**: Immutable, pure operations where possible

## Example Architecture

```typescript
// Service definitions
export class GPUDeviceService extends Context.Tag("GPUDeviceService")<
  GPUDeviceService,
  GPUDevice
>() {}

export class TypeGPURootService extends Context.Tag("TypeGPURootService")<
  TypeGPURootService,
  TgpuRoot
>() {}

// Layer composition
const LiveGPUDevice = Layer.effect(
  GPUDeviceService,
  Effect.gen(function* () {
    const adapter = yield* GPUAdapterService
    const device = yield* Effect.tryPromise({
      try: () => adapter.requestDevice(),
      catch: (error) => new GPUInitializationError({ cause: error })
    })
    return device
  })
)

const LiveTypeGPURoot = Layer.effect(
  TypeGPURootService,
  Effect.gen(function* () {
    const device = yield* GPUDeviceService
    const root = yield* Effect.sync(() => 
      tgpu.initFromDevice({ device })
    )
    return root
  })
).pipe(Layer.provide(LiveGPUDevice))
```

## effectgpu/data Module

EffectGPU provides its own `effectgpu/data` module that:
- **Re-exports all pure operations** from `typegpu/data` (schemas, constructors)
- **Wraps I/O operations** in Effect (readData, writeData)
- **Adds Effect-wrapped validation** (validateSchema, sizeOf, alignmentOf)
- **Maintains API compatibility** with `typegpu/data` for pure operations

### Usage Pattern

```typescript
import * as d from 'effectgpu/data'

// Pure operations work exactly like typegpu/data
const vec = d.vec3f(1, 2, 3)
const schema = d.struct({ pos: d.vec3f })

// I/O operations are Effects
const program = Effect.gen(function* () {
  const buffer = yield* EffectGPU.createBuffer(schema)
  yield* d.writeData(buffer, schema, { pos: d.vec3f(0, 0, 0) })
  const value = yield* d.readData(buffer, schema)
  return value
})
```

This architecture enables:
- **Dependency injection**: Services provided through layers
- **Testability**: Easy to provide test implementations
- **Modularity**: Each service is independent
- **Type safety**: All dependencies are typed
- **Unified API**: Single import for all data operations (`effectgpu/data`)
- **Effect integration**: I/O operations naturally integrate with Effect programs
