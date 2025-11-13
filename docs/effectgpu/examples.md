# EffectGPU Examples

This document demonstrates EffectGPU usage patterns and compares them with plain TypeGPU.

## Example 1: Simple Triangle Rendering

### Plain TypeGPU

```typescript
import tgpu from 'typegpu';
import * as d from 'typegpu/data';

const mainVertex = tgpu['~unstable'].vertexFn({
  in: { vertexIndex: d.builtin.vertexIndex },
  out: { outPos: d.builtin.position, uv: d.vec2f },
}) /* wgsl */`{
  var pos = array<vec2f, 3>(
    vec2(0.0, 0.5),
    vec2(-0.5, -0.5),
    vec2(0.5, -0.5)
  );
  return Out(vec4f(pos[in.vertexIndex], 0.0, 1.0), vec2f(0.5));
}`;

const mainFragment = tgpu['~unstable'].fragmentFn({
  in: { uv: d.vec2f },
  out: d.vec4f,
}) /* wgsl */`{
  return vec4f(1.0, 0.0, 1.0, 1.0);
}`;

// Issues:
// - No error handling
// - Resource cleanup is manual
// - Can't be tested without GPU
// - Throws exceptions on failure
const root = await tgpu.init();

const pipeline = root['~unstable']
  .withVertex(mainVertex, {})
  .withFragment(mainFragment, { format: 'bgra8unorm' })
  .createPipeline();

// Manual cleanup required
root.destroy();
```

### EffectGPU

```typescript
import { Effect } from "effect"
import { EffectGPU } from "effectgpu"
import * as d from "effectgpu/data"

const mainVertex = EffectGPU.vertexFn({
  in: { vertexIndex: d.builtin.vertexIndex },
  out: { outPos: d.builtin.position, uv: d.vec2f },
}) /* wgsl */`{
  var pos = array<vec2f, 3>(
    vec2(0.0, 0.5),
    vec2(-0.5, -0.5),
    vec2(0.5, -0.5)
  );
  return Out(vec4f(pos[in.vertexIndex], 0.0, 1.0), vec2f(0.5));
}`

const mainFragment = EffectGPU.fragmentFn({
  in: { uv: d.vec2f },
  out: d.vec4f,
}) /* wgsl */`{
  return vec4f(1.0, 0.0, 1.0, 1.0);
}`

// Benefits:
// - Error handling in type system
// - Automatic resource cleanup
// - Testable without GPU
// - Composable with other Effects
const program = Effect.gen(function* () {
  const root = yield* EffectGPU.RootService
  
  const pipeline = yield* EffectGPU.createRenderPipeline({
    vertex: mainVertex,
    fragment: mainFragment,
    format: 'bgra8unorm'
  })
  
  return pipeline
})

// Automatic cleanup, error handling
const result = await Effect.runPromise(
  Effect.scoped(program)
)
```

## Example 2: Compute Shader with Error Handling

### Plain TypeGPU

```typescript
import tgpu from 'typegpu';
import * as d from 'typegpu/data';

const root = await tgpu.init(); // Can throw, no error type

const buffer = root.createMutable(d.arrayOf(d.f32, 1000));
// No error handling - what if creation fails?

const computeFn = tgpu.computeFn({
  workgroupSize: [64, 1, 1],
  in: { id: d.builtin.globalInvocationId },
}) /* wgsl */`{
  // Compute logic
}`;

const pipeline = root
  .withCompute(computeFn)
  .createPipeline(); // Can throw

pipeline.dispatchWorkgroups(16, 1, 1); // Can fail at runtime
```

### EffectGPU

```typescript
import { Effect, pipe } from "effect"
import { EffectGPU } from "effectgpu"
import * as d from "effectgpu/data"

const program = Effect.gen(function* () {
  const root = yield* EffectGPU.RootService
  
  // Error handling in type system
  const buffer = yield* EffectGPU.createMutable(
    d.arrayOf(d.f32, 1000)
  )
  
  const computeFn = EffectGPU.computeFn({
    workgroupSize: [64, 1, 1],
    in: { id: d.builtin.globalInvocationId },
  }) /* wgsl */`{
    // Compute logic
  }`
  
  const pipeline = yield* EffectGPU.createComputePipeline({
    compute: computeFn
  })
  
  // Dispatch returns Effect, can handle errors
  yield* EffectGPU.dispatchWorkgroups(pipeline, [16, 1, 1])
  
  return buffer
})

// All errors are typed and handled
const result = await pipe(
  Effect.scoped(program),
  Effect.catchAll((error) => {
    // Handle GPUInitializationError, ResourceCreationError, etc.
    return Effect.sync(() => console.error("GPU error:", error))
  }),
  Effect.runPromise
)
```

## Example 3: Resource Management and Cleanup

### Plain TypeGPU

```typescript
const root = await tgpu.init();
const buffer1 = root.createBuffer(d.vec3f);
const buffer2 = root.createBuffer(d.vec3f);
const texture = root.createTexture({ /* ... */ });

// Manual cleanup - easy to forget
try {
  // Use resources
} finally {
  root.destroy(); // Destroys everything
}
```

### EffectGPU

```typescript
import { Effect, Scope } from "effect"
import { EffectGPU } from "effectgpu"

const program = Effect.gen(function* () {
  const scope = yield* Scope.make()
  const root = yield* EffectGPU.RootService
  
  // Resources automatically cleaned up when scope closes
  const buffer1 = yield* pipe(
    EffectGPU.createBuffer(d.vec3f),
    Scope.extend(scope)
  )
  
  const buffer2 = yield* pipe(
    EffectGPU.createBuffer(d.vec3f),
    Scope.extend(scope)
  )
  
  const texture = yield* pipe(
    EffectGPU.createTexture({ /* ... */ }),
    Scope.extend(scope)
  )
  
  // Use resources
  yield* EffectGPU.useResources({ buffer1, buffer2, texture })
  
  // Automatic cleanup when scope closes
  yield* Scope.close(scope, Exit.succeed(undefined))
})

// Resources automatically cleaned up
await Effect.runPromise(Effect.scoped(program))
```

## Example 4: Dependency Injection and Testing

### Plain TypeGPU

```typescript
// Hard to test - requires actual GPU
async function processData(data: Float32Array) {
  const root = await tgpu.init(); // Can't mock
  const buffer = root.createBuffer(d.arrayOf(d.f32, data.length));
  // ... processing
  root.destroy();
}
```

### EffectGPU

```typescript
import { Effect, Context, Layer } from "effect"
import { EffectGPU } from "effectgpu"

// Service-based - easy to test
const processData = (data: Float32Array) =>
  Effect.gen(function* () {
    const root = yield* EffectGPU.RootService
    const buffer = yield* EffectGPU.createBuffer(
      d.arrayOf(d.f32, data.length)
    )
    // ... processing
    return result
  })

// Production: Use real GPU
const LiveProgram = processData(myData).pipe(
  Effect.provide(EffectGPU.Live)
)

// Testing: Use mock GPU
const MockGPU = Layer.succeed(
  EffectGPU.RootService,
  createMockRoot()
)

const TestProgram = processData(myData).pipe(
  Effect.provide(MockGPU)
)
```

## Example 5: Composing GPU Operations

### Plain TypeGPU

```typescript
// Sequential, imperative
const root = await tgpu.init();
const buffer1 = root.createBuffer(d.vec3f);
const buffer2 = root.createBuffer(d.vec3f);

const pipeline1 = root.withCompute(compute1).createPipeline();
pipeline1.dispatchWorkgroups(10, 1, 1);

const pipeline2 = root.withCompute(compute2).createPipeline();
pipeline2.dispatchWorkgroups(10, 1, 1);
```

### EffectGPU

```typescript
import { Effect, pipe } from "effect"
import { EffectGPU } from "effectgpu"

// Composable, functional
const program = Effect.gen(function* () {
  const root = yield* EffectGPU.RootService
  
  const [buffer1, buffer2] = yield* Effect.all([
    EffectGPU.createBuffer(d.vec3f),
    EffectGPU.createBuffer(d.vec3f)
  ])
  
  const pipeline1 = yield* EffectGPU.createComputePipeline({
    compute: compute1
  })
  
  const pipeline2 = yield* EffectGPU.createComputePipeline({
    compute: compute2
  })
  
  // Compose operations
  yield* pipe(
    EffectGPU.dispatchWorkgroups(pipeline1, [10, 1, 1]),
    Effect.andThen(() => 
      EffectGPU.dispatchWorkgroups(pipeline2, [10, 1, 1])
    )
  )
  
  return { buffer1, buffer2 }
})

await Effect.runPromise(Effect.scoped(program))
```

## Example 6: Data I/O Operations

### Plain TypeGPU

```typescript
import { writeData, readData } from 'typegpu/data'
import tgpu from 'typegpu'
import * as d from 'typegpu/data' // Plain TypeGPU still uses typegpu/data

const root = await tgpu.init()
const buffer = root.createBuffer(d.vec3f)

// Direct I/O - can throw, no error handling
writeData(buffer, d.vec3f, [1, 2, 3])
const value = readData(buffer, d.vec3f) // Can throw
```

### EffectGPU

```typescript
import { Effect } from "effect"
import { EffectGPU } from "effectgpu"
import * as d from "effectgpu/data"

const program = Effect.gen(function* () {
  const buffer = yield* EffectGPU.createBuffer(d.vec3f)
  
  // I/O operations from effectgpu/data are Effects with error handling
  yield* d.writeData(buffer, d.vec3f, [1, 2, 3])
  
  const value = yield* d.readData(buffer, d.vec3f)
  
  return value
})

// All errors are typed and handled
const result = await pipe(
  Effect.scoped(program),
  Effect.catchAll((error) => {
    // Handle DataIOError, InvalidSchemaError, etc.
    return Effect.sync(() => console.error("Data I/O error:", error))
  }),
  Effect.runPromise
)
```

## Example 7: Schema Validation

### Plain TypeGPU

```typescript
import * as d from 'typegpu/data' // Plain TypeGPU

// No validation - just type checking at compile time
const schema = d.struct({
  position: d.vec3f,
  color: d.vec4f
})

// Runtime validation not available in plain TypeGPU
```

### EffectGPU

```typescript
import { Effect } from "effect"
import { EffectGPU } from "effectgpu"
import * as d from "effectgpu/data"

const program = Effect.gen(function* () {
  const schema = d.struct({
    position: d.vec3f,
    color: d.vec4f
  })
  
  // Validate schema at runtime using effectgpu/data
  const isValid = yield* d.validateSchema(schema)
  
  if (!isValid) {
    return yield* Effect.fail(new InvalidSchemaError({ schema }))
  }
  
  // Use validated schema
  const buffer = yield* EffectGPU.createBuffer(schema)
  return buffer
})
```

## Example 8: Error Recovery

### Plain TypeGPU

```typescript
// Errors are thrown - hard to recover
try {
  const root = await tgpu.init();
  const buffer = root.createBuffer(d.vec3f);
} catch (error) {
  // Can't distinguish error types
  // Hard to recover gracefully
  console.error(error);
}
```

### EffectGPU

```typescript
import { Effect, Match } from "effect"
import { EffectGPU, GPUInitializationError, ResourceCreationError } from "effectgpu"

const program = Effect.gen(function* () {
  const root = yield* EffectGPU.RootService
  
  const buffer = yield* pipe(
    EffectGPU.createBuffer(d.vec3f),
    Effect.catchTag("ResourceCreationError", (error) => {
      // Specific error handling
      return Effect.succeed(createFallbackBuffer())
    })
  )
  
  return buffer
})

const result = await pipe(
  Effect.scoped(program),
  Effect.match({
    onFailure: (error) => {
      // Pattern match on error types
      return Match.value(error).pipe(
        Match.when(GPUInitializationError, () => "GPU not available"),
        Match.when(ResourceCreationError, () => "Resource creation failed"),
        Match.orElse(() => "Unknown error")
      )
    },
    onSuccess: (value) => value
  }),
  Effect.runPromise
)
```

## Example 9: Using typegpu/data with EffectGPU

### Pure Data Operations

```typescript
import { Effect } from "effect"
import { EffectGPU } from "effectgpu"
import * as d from "effectgpu/data"

// Pure operations remain pure - no Effect needed
const vec = d.vec3f(1, 2, 3) // Works in JS
const schema = d.struct({
  position: d.vec3f,
  velocity: d.vec3f,
  mass: d.f32
})

// Use in Effect programs
const program = Effect.gen(function* () {
  // Create buffer with schema
  const buffer = yield* EffectGPU.createBuffer(schema)
  
  // Write data using pure constructor
  const initialData = schema({
    position: d.vec3f(0, 0, 0),
    velocity: d.vec3f(1, 0, 0),
    mass: 1.0
  })
  
  yield* d.writeData(buffer, schema, initialData)
  
  return buffer
})
```

### Data Type Composition

```typescript
import { Effect } from "effect"
import { EffectGPU } from "effectgpu"
import * as d from "effectgpu/data"

// Compose data types functionally
const Particle = d.struct({
  pos: d.vec3f,
  vel: d.vec3f,
  mass: d.f32
})

const ParticleArray = d.arrayOf(Particle, 1000)

const program = Effect.gen(function* () {
  // Create buffer with composed schema
  const buffer = yield* EffectGPU.createBuffer(ParticleArray)
  
  // Initialize with array of particles
  const particles = Array.from({ length: 1000 }, () =>
    Particle({
      pos: d.vec3f(0, 0, 0),
      vel: d.vec3f(0, 0, 0),
      mass: 1.0
    })
  )
  
  yield* d.writeData(buffer, ParticleArray, particles)
  
  return buffer
})
```

## Key Advantages Summary

1. **Type Safety**: Errors are in the type system, not thrown
2. **Composability**: All operations are Effects that compose
3. **Resource Safety**: Automatic cleanup through Scope
4. **Testability**: Services can be mocked for testing
5. **Error Handling**: Structured error types with recovery
6. **Functional Style**: Immutable, composable operations
7. **Dependency Injection**: Services enable modularity
8. **Data I/O Safety**: Data operations are effectified with error handling
9. **Schema Validation**: Runtime validation of data schemas
10. **Pure Data Operations**: typegpu/data constructors remain pure and composable
