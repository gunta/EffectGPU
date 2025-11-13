# TypeGPU Research Summary

## Overview

TypeGPU is a type-safe toolkit for WebGPU that allows writing shaders in TypeScript. It provides:
- Type-safe WebGPU abstractions
- TypeScript-to-WGSL compilation
- Resource management (buffers, textures, pipelines)
- Shader function definitions

## Core API Surface

### Initialization

```typescript
// Request GPU device and create root
const root = await tgpu.init(options?: InitOptions)

// Or use existing device
const root = tgpu.initFromDevice({ device: GPUDevice })
```

### Resource Creation

```typescript
// Buffers
const buffer = root.createBuffer(typeSchema, initialData?)
const uniform = root.createUniform(typeSchema, initialData?)
const mutable = root.createMutable(typeSchema, initialData?)
const readonly = root.createReadonly(typeSchema, initialData?)

// Textures
const texture = root.createTexture({
  size: [width, height, depth?],
  format: GPUTextureFormat,
  usage: GPUTextureUsageFlags,
  // ... other options
})

// Samplers
const sampler = root.createSampler(props)
const comparisonSampler = root.createComparisonSampler(props)
```

### Shader Functions

```typescript
// Generic function (can be called in JS or used in shaders)
const fn = tgpu.fn([inputTypes], outputType)(() => {
  'use gpu'
  // WGSL code
})

// Compute shader
const computeFn = tgpu.computeFn({
  workgroupSize: [x, y, z],
  in: { /* inputs */ }
}) /* wgsl */`{
  // WGSL code
}`

// Vertex shader
const vertexFn = tgpu['~unstable'].vertexFn({
  in: { /* inputs */ },
  out: { /* outputs */ }
}) /* wgsl */`{
  // WGSL code
}`

// Fragment shader
const fragmentFn = tgpu['~unstable'].fragmentFn({
  in: { /* inputs */ },
  out: outputType
}) /* wgsl */`{
  // WGSL code
}`
```

### Pipeline Creation

```typescript
// Compute pipeline
const computePipeline = root
  .withCompute(computeFn)
  .createPipeline()

// Render pipeline
const renderPipeline = root['~unstable']
  .withVertex(vertexFn, vertexAttribs)
  .withFragment(fragmentFn, targets)
  .createPipeline()
```

### Pipeline Execution

```typescript
// Compute dispatch
computePipeline.dispatchWorkgroups(x, y, z)

// Guarded compute (handles thread bounds)
const guarded = root['~unstable']
  .createGuardedComputePipeline((x, y, z) => {
    'use gpu'
    // ...
  })
guarded.dispatchThreads(x, y, z)

// Render pass
root.beginRenderPass(descriptor, (pass) => {
  pass.setPipeline(renderPipeline)
  pass.draw(vertexCount, instanceCount?, firstVertex?, firstInstance?)
})
```

### Bind Groups

```typescript
// Create layout
const layout = tgpu.bindGroupLayout({
  uniform: tgpu.uniform(d.vec3f),
  texture: tgpu.texture('2d', 'float'),
  sampler: tgpu.sampler()
})

// Create bind group
const bindGroup = root.createBindGroup(layout, {
  uniform: uniformBuffer,
  texture: texture,
  sampler: sampler
})

// Use in pipeline
root
  .with(bindGroup)
  .withCompute(computeFn)
  .createPipeline()
```

### Shader Resolution

```typescript
// Generate WGSL code
const wgsl = tgpu.resolve({
  externals: { fn1, fn2, ... }
})
```

## Key TypeGPU Concepts

### Data Types

TypeGPU uses a data type system for type-safe GPU data:

```typescript
import * as d from 'typegpu/data'

// Primitives
d.f32, d.i32, d.u32, d.bool

// Vectors
d.vec2f, d.vec3f, d.vec4f
d.vec2i, d.vec3i, d.vec4i
d.vec2u, d.vec3u, d.vec4u

// Matrices
d.mat2x2f, d.mat3x3f, d.mat4x4f

// Arrays
d.arrayOf(d.f32, length)

// Structs
d.struct({
  position: d.vec3f,
  color: d.vec4f
})

// Builtins
d.builtin.position
d.builtin.vertexIndex
d.builtin.globalInvocationId
```

### Slots

Slots are placeholders for values that will be provided later:

```typescript
const mySlot = tgpu.slot(d.vec3f)
const value = tgpu.slot(d.f32)

// Use in shader
const fn = tgpu.fn([], d.vec3f)(() => {
  'use gpu'
  return mySlot * value
})

// Provide values when resolving
tgpu.resolve({
  externals: { fn },
  slots: {
    mySlot: [1, 2, 3],
    value: 2.0
  }
})
```

### Resource Unwrapping

TypeGPU resources can be unwrapped to get underlying WebGPU resources:

```typescript
const gpuBuffer = root.unwrap(buffer) // GPUBuffer
const gpuTexture = root.unwrap(texture) // GPUTexture
const gpuPipeline = root.unwrap(pipeline) // GPUComputePipeline | GPURenderPipeline
```

## Error Handling

TypeGPU throws exceptions for:
- GPU initialization failures
- Resource creation failures
- Pipeline creation failures
- Missing bind groups/vertex buffers
- Invalid operations

## Resource Lifecycle

- Resources are created through the root
- Root must be manually destroyed: `root.destroy()`
- Destroying root destroys all resources
- No automatic cleanup

## What Needs Effect Wrapping

1. **Initialization**: `tgpu.init()` - async, can fail
2. **Resource Creation**: All `create*` methods - can fail
3. **Pipeline Creation**: `createPipeline()` - can fail
4. **Pipeline Execution**: `dispatch*`, `draw*` - can fail at runtime
5. **Resource Cleanup**: `root.destroy()` - should be automatic
6. **Error Handling**: Currently throws - should be in error channel
7. **Resource Sharing**: No built-in sharing mechanism

## TypeGPU Packages

- `typegpu`: Core library
- `@typegpu/sdf`: Signed distance field functions
- `@typegpu/color`: Color manipulation functions
- `@typegpu/noise`: Noise generation functions
- `unplugin-typegpu`: Build tool integration

## Example Patterns

### Simple Compute

```typescript
const root = await tgpu.init()
const buffer = root.createMutable(d.arrayOf(d.f32, 1000))

const compute = tgpu.computeFn({
  workgroupSize: [64, 1, 1],
  in: { id: d.builtin.globalInvocationId }
}) /* wgsl */`{
  buffer[id.x] = buffer[id.x] * 2.0
}`

const pipeline = root
  .with(buffer)
  .withCompute(compute)
  .createPipeline()

pipeline.dispatchWorkgroups(16, 1, 1)
```

### Render Pipeline

```typescript
const root = await tgpu.init()

const vertex = tgpu['~unstable'].vertexFn({
  in: { vertexIndex: d.builtin.vertexIndex },
  out: { position: d.builtin.position }
}) /* wgsl */`{
  // vertex code
}`

const fragment = tgpu['~unstable'].fragmentFn({
  in: {},
  out: d.vec4f
}) /* wgsl */`{
  return vec4f(1.0, 0.0, 0.0, 1.0)
}`

const pipeline = root['~unstable']
  .withVertex(vertex, {})
  .withFragment(fragment, { format: 'bgra8unorm' })
  .createPipeline()

root.beginRenderPass({
  colorAttachments: [{
    view: textureView,
    clearValue: [0, 0, 0, 1],
    loadOp: 'clear',
    storeOp: 'store'
  }]
}, (pass) => {
  pass.setPipeline(pipeline)
  pass.draw(3)
})
```

## typegpu/data Module

The `typegpu/data` module provides data type schemas and constructors:

### Data Type Schemas

```typescript
import * as d from 'typegpu/data'

// Primitives
d.f32, d.i32, d.u32, d.bool, d.f16, d.u16

// Vectors
d.vec2f, d.vec3f, d.vec4f
d.vec2i, d.vec3i, d.vec4i
d.vec2u, d.vec3u, d.vec4u
d.vec2h, d.vec3h, d.vec4h
d.vec2b, d.vec3b, d.vec4b

// Matrices
d.mat2x2f, d.mat3x3f, d.mat4x4f

// Arrays
d.arrayOf(schema, length)

// Structs
d.struct({ field1: d.f32, field2: d.vec3f })

// Disarrays (for vertex buffers)
d.disarrayOf(schema, length)

// Unstructs (for vertex buffers)
d.unstruct({ field1: d.f32 })

// Textures
d.texture2d('float'), d.texture3d('float'), etc.

// Samplers
d.sampler(), d.comparisonSampler()

// Builtins
d.builtin.position
d.builtin.vertexIndex
d.builtin.globalInvocationId
```

### Constructor Functions

All data type schemas are also constructor functions that work in both JS and GPU code:

```typescript
// In JavaScript
const vec = d.vec3f(1, 2, 3) // [1, 2, 3]

// In GPU code
const vec = d.vec3f(1, 2, 3) // vec3f(1, 2, 3) in WGSL
```

### Data I/O Operations

TypeGPU provides data I/O operations for reading/writing buffers:

```typescript
import { writeData, readData } from 'typegpu/data'

// Write data to buffer
writeData(output, schema, value)

// Read data from buffer
const value = readData(input, schema)
```

### Type Utilities

```typescript
import { sizeOf, alignmentOf, deepEqual } from 'typegpu/data'

const size = sizeOf(d.vec3f) // 12 bytes
const align = alignmentOf(d.vec3f) // 16 bytes
const equal = deepEqual(value1, value2, schema)
```

## Effectification of typegpu/data

EffectGPU provides its own `effectgpu/data` module that wraps `typegpu/data` with Effect integration:

### Pure Operations (Re-exported)

All pure operations from `typegpu/data` are re-exported unchanged:
- **Type schemas**: Pure constructors, no side effects
- **Constructor functions**: Pure, work in both JS and GPU
- **Type checking**: Pure validation functions

### Effect-Wrapped Operations

Operations that can fail are wrapped in Effect:
- **Data I/O operations**: `writeData`, `readData` - return Effects
- **Validation operations**: Schema validation - returns Effect
- **Size/alignment calculations**: Can fail for invalid schemas - return Effects

### EffectGPU Data API

```typescript
import { Effect } from 'effect'
import * as d from 'effectgpu/data'

// Pure operations work the same way
const vec = d.vec3f(1, 2, 3)
const schema = d.struct({ pos: d.vec3f, color: d.vec4f })

// I/O operations are Effects
const writeProgram = Effect.gen(function* () {
  const buffer = yield* EffectGPU.createBuffer(d.vec3f)
  yield* d.writeData(buffer, d.vec3f, [1, 2, 3]) // Returns Effect
  return buffer
})

const readProgram = Effect.gen(function* () {
  const buffer = yield* EffectGPU.createBuffer(d.vec3f)
  const value = yield* d.readData(buffer, d.vec3f) // Returns Effect
  return value
})

// Validation is an Effect
const validateProgram = Effect.gen(function* () {
  const isValid = yield* d.validateSchema(schema) // Returns Effect
  return isValid
})
```

### Module Structure

```typescript
// effectgpu/data/index.ts

// Re-export all pure operations from typegpu/data
export * from 'typegpu/data'

// Effect-wrapped I/O operations
export function writeData<TData extends BaseData>(
  buffer: EffectGPUBuffer<TData>,
  schema: TData,
  value: Infer<TData>
): Effect<void, DataIOError>

export function readData<TData extends BaseData>(
  buffer: EffectGPUBuffer<TData>,
  schema: TData
): Effect<Infer<TData>, DataIOError>

// Effect-wrapped validation
export function validateSchema(
  schema: AnyData
): Effect<boolean, InvalidSchemaError>

// Effect-wrapped utilities
export function sizeOf(
  schema: AnyData
): Effect<number, InvalidSchemaError>

export function alignmentOf(
  schema: AnyData
): Effect<number, InvalidSchemaError>
```

## Key Takeaways for EffectGPU

1. **All operations are synchronous or async promises** - need Effect wrapping
2. **Errors are thrown** - need error channel
3. **Manual resource management** - need Scope-based cleanup
4. **No dependency injection** - need service layer
5. **Imperative API** - can be made more functional
6. **Type-safe but not branded** - can add branding for extra safety
7. **typegpu/data mostly pure** - but I/O operations need Effect wrapping
