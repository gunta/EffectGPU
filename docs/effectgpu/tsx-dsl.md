# EffectGPU TSX DSL

A declarative, component-based syntax for building GPU programs using JSX/TSX, inspired by [React Three Fiber](https://r3f.docs.pmnd.rs/getting-started/introduction) but for Effect-native GPU computation.

## Design Philosophy

Inspired by React Three Fiber's approach to Three.js, EffectGPU TSX provides:
- **Declarative syntax**: Describe what you want, not how to do it
- **Component composition**: Build complex GPU programs from simple components
- **Type safety**: Full TypeScript support with branded types
- **Effect integration**: All components are Effect programs that compose naturally
- **Visual structure**: JSX makes GPU program structure visible
- **No overhead**: Components compile to Effect programs, no runtime cost
- **Full compatibility**: Everything that works in EffectGPU works in TSX

Just like R3F expresses Three.js in JSX (`<mesh />` → `new THREE.Mesh()`), EffectGPU TSX expresses GPU operations in JSX (`<ComputePipeline />` → Effect program).

## Basic Concepts

### Components as GPU Operations

Each JSX element represents a GPU operation or resource, similar to how R3F expresses Three.js objects:

```tsx
import { Canvas } from '@effectgpu/react'
import { useFrame } from '@effectgpu/react'
import * as d from 'effectgpu/data'

function ComputeExample() {
  return (
    <Canvas>
      <ComputePipeline workgroupSize={[64, 1, 1]}>
        <BindGroup>
          <Binding slot="input" resource={inputBuffer} />
          <Binding slot="output" resource={outputBuffer} />
        </BindGroup>
        <ComputeShader>
          {() => {
            'use gpu'
            const id = builtin.globalInvocationId.x
            output[id] = input[id] * 2.0
          }}
        </ComputeShader>
        <Dispatch threads={[1024, 1, 1]} />
      </ComputePipeline>
    </Canvas>
  )
}
```

### Resources as Components

GPU resources are created declaratively, just like R3F's geometry and materials:

```tsx
function ParticleSystem() {
  return (
    <Canvas>
      <Buffer type={d.arrayOf(d.vec3f, 1000)}>
        {(buffer) => (
          <ComputePipeline workgroupSize={[64, 1, 1]}>
            <BindGroup>
              <Binding slot="particles" resource={buffer} />
            </BindGroup>
            <ComputeShader>
              {() => {
                'use gpu'
                // Update particles
              }}
            </ComputeShader>
            <Dispatch threads={[100, 1, 1]} />
          </ComputePipeline>
        )}
      </Buffer>
    </Canvas>
  )
}
```

### Effect Hooks

Like R3F's `useFrame`, EffectGPU provides hooks for Effect programs:

```tsx
import { useEffect } from '@effectgpu/react'

function AnimatedBuffer() {
  const bufferRef = useRef<EffectGPUBuffer>()
  
  useEffect(function* () {
    const buffer = yield* EffectGPU.createBuffer(d.vec3f)
    bufferRef.current = buffer
    
    // Animate in render loop
    yield* Effect.forever(
      Effect.gen(function* () {
        yield* d.writeData(buffer, d.vec3f, [Math.random(), Math.random(), Math.random()])
        yield* Effect.sleep("100 millis")
      })
    )
  })
  
  return (
    <Buffer ref={bufferRef}>
      {(buffer) => (
        <ComputePipeline>
          {/* Use buffer */}
        </ComputePipeline>
      )}
    </Buffer>
  )
}
```

## Core Components

### Resource Components

#### `<Buffer>`
Creates a GPU buffer with automatic lifecycle management.

```tsx
import * as d from 'effectgpu/data'

<Buffer type={d.vec3f} initial={d.vec3f(1, 2, 3)}>
  {(buffer) => (
    // Use buffer
  )}
</Buffer>
```

#### `<DataIO>`
Reads or writes data to/from buffers.

```tsx
<DataIO.Write buffer={buffer} schema={d.vec3f} value={d.vec3f(1, 2, 3)}>
  {() => (
    <DataIO.Read buffer={buffer} schema={d.vec3f}>
      {(value) => (
        // Use value
      )}
    </DataIO.Read>
  )}
</DataIO.Write>
```

#### `<Texture>`
Creates a GPU texture.

```tsx
<Texture
  size={[512, 512]}
  format="rgba8unorm"
  usage={['render', 'storage']}
>
  {(texture) => (
    // Use texture
  )}
</Texture>
```

#### `<Uniform>`
Creates a uniform buffer.

```tsx
<Uniform type={d.struct({ time: d.f32, scale: d.f32 })}>
  {(uniform) => (
    <Uniform.write value={{ time: 0.5, scale: 2.0 }}>
      {/* Use uniform */}
    </Uniform.write>
  )}
</Uniform>
```

### Pipeline Components

#### `<ComputePipeline>`
Creates and executes a compute pipeline.

```tsx
<ComputePipeline workgroupSize={[64, 1, 1]}>
  <BindGroup>
    <Binding slot="input" resource={inputBuffer} />
    <Binding slot="output" resource={outputBuffer} />
  </BindGroup>
  <ComputeShader>
    {() => {
      'use gpu'
      const id = builtin.globalInvocationId.x
      output[id] = input[id] * 2.0
    }}
  </ComputeShader>
  <Dispatch threads={[1024, 1, 1]} />
</ComputePipeline>
```

#### `<RenderPipeline>`
Creates and executes a render pipeline.

```tsx
<RenderPipeline>
  <VertexShader>
    {() => {
      'use gpu'
      // Vertex shader code
    }}
  </VertexShader>
  <FragmentShader>
    {() => {
      'use gpu'
      // Fragment shader code
    }}
  </FragmentShader>
  <RenderPass target={canvas}>
    <Draw vertexCount={3} />
  </RenderPass>
</RenderPipeline>
```

### Binding Components

#### `<BindGroup>`
Groups resource bindings together.

```tsx
<BindGroup>
  <Binding slot="uniforms" resource={uniformBuffer} />
  <Binding slot="texture" resource={texture} />
  <Binding slot="sampler" resource={sampler} />
</BindGroup>
```

#### `<Binding>`
Binds a resource to a slot.

```tsx
<Binding slot="data" resource={buffer} />
```

### Execution Components

#### `<Dispatch>`
Dispatches a compute shader.

```tsx
<Dispatch threads={[1024, 1, 1]} />
```

#### `<Draw>`
Draws primitives in a render pass.

```tsx
<Draw vertexCount={3} instanceCount={1} />
```

## Advanced Patterns

### Conditional Rendering

```tsx
<Effect.gen>
  {function* () {
    const useGPU = yield* Config.get("useGPU")
    
    return useGPU ? (
      <ComputePipeline>
        <ComputeShader>{/* GPU path */}</ComputeShader>
        <Dispatch threads={[1024, 1, 1]} />
      </ComputePipeline>
    ) : (
      <CPUFallback />
    )
  }}
</Effect.gen>
```

### Loops and Iteration

```tsx
<Effect.forEach([buffer1, buffer2, buffer3], (buffer) => (
  <ComputePipeline>
    <BindGroup>
      <Binding slot="data" resource={buffer} />
    </BindGroup>
    <ComputeShader>{/* ... */}</ComputeShader>
    <Dispatch threads={[100, 1, 1]} />
  </ComputePipeline>
))>
```

### Error Handling

```tsx
<Effect.catchAll(
  <ComputePipeline>
    <ComputeShader>{/* ... */}</ComputeShader>
    <Dispatch threads={[1024, 1, 1]} />
  </ComputePipeline>,
  (error) => (
    <ErrorFallback error={error} />
  )
)>
```

### Resource Sharing

```tsx
<Scope>
  {(scope) => (
    <Effect.gen>
      {function* () {
        const buffer = yield* pipe(
          EffectGPU.createBuffer(d.vec3f),
          Scope.extend(scope)
        )
        
        return (
          <>
            <ComputePipeline>
              <BindGroup>
                <Binding slot="input" resource={buffer} />
              </BindGroup>
              <ComputeShader>{/* ... */}</ComputeShader>
              <Dispatch threads={[100, 1, 1]} />
            </ComputePipeline>
            <ComputePipeline>
              <BindGroup>
                <Binding slot="input" resource={buffer} />
              </BindGroup>
              <ComputeShader>{/* ... */}</ComputeShader>
              <Dispatch threads={[100, 1, 1]} />
            </ComputePipeline>
          </>
        )
      }}
    </Effect.gen>
  )}
</Scope>
```

## Component Library

### Custom Components

Build reusable GPU components, just like R3F components:

```tsx
interface BlurPassProps {
  input: EffectGPUTexture
  output: EffectGPUTexture
  radius: number
}

function BlurPass({ input, output, radius }: BlurPassProps) {
  const radiusBuffer = useBuffer(d.f32)
  
  useEffect(function* () {
    yield* d.writeData(radiusBuffer, d.f32, radius)
  }, [radius])
  
  return (
    <ComputePipeline workgroupSize={[16, 16, 1]}>
      <BindGroup>
        <Binding slot="input" resource={input} />
        <Binding slot="output" resource={output} />
        <Binding slot="radius" resource={radiusBuffer} />
      </BindGroup>
      <ComputeShader>
        {() => {
          'use gpu'
          // Blur implementation
        }}
      </ComputeShader>
      <Dispatch threads={[512, 512, 1]} />
    </ComputePipeline>
  )
}

// Usage - self-contained, reusable
<BlurPass input={sourceTexture} output={blurredTexture} radius={5} />
```

### Higher-Order Components

Compose GPU operations:

```tsx
const withTiming = <T,>(component: Component<T>) => (
  <Effect.gen>
    {function* () {
      const start = yield* Clock.currentTimeMillis
      const result = yield* component
      const end = yield* Clock.currentTimeMillis
      yield* Console.log(`Took ${end - start}ms`)
      return result
    }}
  </Effect.gen>
)

<withTiming>
  <ComputePipeline>
    <ComputeShader>{/* ... */}</ComputeShader>
    <Dispatch threads={[1024, 1, 1]} />
  </ComputePipeline>
</withTiming>
```

## React Integration

Like R3F, EffectGPU TSX integrates seamlessly with React:

### Canvas Component

The `<Canvas>` component sets up the EffectGPU runtime and provides context:

```tsx
import { Canvas } from '@effectgpu/react'

function App() {
  return (
    <Canvas>
      <ComputePipeline>
        {/* GPU operations */}
      </ComputePipeline>
    </Canvas>
  )
}
```

### Hooks API

Similar to R3F's hooks, EffectGPU provides React hooks:

```tsx
import { 
  useEffectGPU,      // Run Effect programs
  useFrame,          // Render loop integration
  useResource,       // Resource management
  useBuffer,         // Buffer creation
  useTexture,        // Texture creation
  usePipeline        // Pipeline creation
} from '@effectgpu/react'

function MyComponent() {
  const buffer = useBuffer(d.vec3f)
  const pipeline = usePipeline({ compute: computeFn })
  
  useFrame(function* (delta) {
    // Update every frame
    yield* d.writeData(buffer, d.vec3f, [delta, delta, delta])
    yield* EffectGPU.dispatchWorkgroups(pipeline, [100, 1, 1])
  })
  
  return (
    <ComputePipeline pipeline={pipeline}>
      <BindGroup>
        <Binding slot="data" resource={buffer} />
      </BindGroup>
      <Dispatch threads={[100, 1, 1]} />
    </ComputePipeline>
  )
}
```

## Transformation Pipeline

The TSX DSL is transformed into EffectGPU operations, similar to how R3F transforms JSX to Three.js:

1. **Parse**: JSX is parsed into an AST
2. **Resolve**: Components are resolved to EffectGPU functions
3. **Optimize**: Resource usage is optimized
4. **Generate**: Effect programs are generated
5. **Execute**: Programs are run with Effect runtime

### Example Transformation

**Input (TSX):**
```tsx
<Canvas>
  <ComputePipeline workgroupSize={[64, 1, 1]}>
    <BindGroup>
      <Binding slot="data" resource={buffer} />
    </BindGroup>
    <ComputeShader>
      {() => {
        'use gpu'
        const id = builtin.globalInvocationId.x
        data[id] = data[id] * 2.0
      }}
    </ComputeShader>
    <Dispatch threads={[1024, 1, 1]} />
  </ComputePipeline>
</Canvas>
```

**Output (EffectGPU):**
```typescript
Effect.gen(function* () {
  const root = yield* EffectGPU.RootService
  const computeFn = EffectGPU.computeFn({
    workgroupSize: [64, 1, 1],
    in: { id: d.builtin.globalInvocationId }
  }) /* wgsl */`{
    data[id.x] = data[id.x] * 2.0
  }`
  
  const pipeline = yield* EffectGPU.createComputePipeline({
    compute: computeFn,
    bindings: {
      data: buffer
    }
  })
  
  yield* EffectGPU.dispatchWorkgroups(pipeline, [1024, 1, 1])
})
```

Just like R3F's `<mesh />` becomes `new THREE.Mesh()`, our `<ComputePipeline />` becomes an Effect program.

## Benefits

Inspired by R3F's benefits for Three.js:

1. **Visual Structure**: JSX makes GPU program structure clear
2. **Composability**: Components can be composed and reused
3. **Type Safety**: Full TypeScript support with branded types
4. **Effect Integration**: All components are Effects that compose naturally
5. **Declarative**: Describe what, not how
6. **Familiar**: JSX syntax is familiar to many developers
7. **No Overhead**: Components compile to Effect programs, no runtime cost
8. **Full Compatibility**: Everything that works in EffectGPU works in TSX
9. **React Ecosystem**: Integrates with React hooks, state management, etc.
10. **Self-Contained**: Components are reusable and self-contained

## Complete Example

Inspired by R3F's Box example, here's a complete EffectGPU TSX example:

```tsx
import { Canvas, useFrame, useBuffer } from '@effectgpu/react'
import { EffectGPU } from 'effectgpu'
import * as d from 'effectgpu/data'
import { useRef, useState } from 'react'

function ParticleSystem() {
  const particlesRef = useRef<EffectGPUBuffer>()
  const [active, setActive] = useState(false)
  
  const particles = useBuffer(d.arrayOf(d.struct({
    position: d.vec3f,
    velocity: d.vec3f
  }), 1000))
  
  useFrame(function* (delta) {
    if (particlesRef.current) {
      // Update particles every frame
      yield* EffectGPU.dispatchWorkgroups(updatePipeline, [100, 1, 1])
    }
  })
  
  return (
    <ComputePipeline
      ref={particlesRef}
      workgroupSize={[64, 1, 1]}
      active={active}
      onClick={() => setActive(!active)}>
      <BindGroup>
        <Binding slot="particles" resource={particles} />
        <Binding slot="delta" resource={deltaBuffer} />
      </BindGroup>
      <ComputeShader>
        {() => {
          'use gpu'
          const id = builtin.globalInvocationId.x
          particles[id].position += particles[id].velocity * delta
        }}
      </ComputeShader>
      <Dispatch threads={[100, 1, 1]} />
    </ComputePipeline>
  )
}

function App() {
  return (
    <Canvas>
      <ParticleSystem />
    </Canvas>
  )
}
```

## Ecosystem

Like R3F has an ecosystem of libraries, EffectGPU TSX can integrate with:

- **@effectgpu/react**: React integration and hooks
- **@effectgpu/drei**: Useful helpers and utilities
- **@effectgpu/postprocessing**: Post-processing effects
- **@effectgpu/test-renderer**: Testing utilities
- **Effect ecosystem**: All Effect libraries work seamlessly

## Future Enhancements

- **Visual Editor**: Drag-and-drop GPU program builder
- **Hot Reload**: Live updates during development (like R3F)
- **Performance Profiling**: Built-in performance monitoring
- **Shader Debugging**: Visual shader debugging tools
- **Code Generation**: Generate optimized GPU code
- **DevTools**: React DevTools integration for GPU components
- **Suspense Support**: Async resource loading with React Suspense
