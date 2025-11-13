# EffectGPU

EffectGPU is an idiomatic Effect-native wrapper around [TypeGPU](https://docs.swmansion.com/TypeGPU), bringing functional programming, dependency injection, error safety, and resource-scoped lifecycles to WebGPU development.

## Why EffectGPU?
- Composable Effect programs (no hidden side effects)
- Typed error channels (recoverable failures)
- Scoped resources (automatic cleanup)
- Dependency injection with Layers (testable and modular)
- Branded types (prevent resource misuse)
- Optional TSX DSL inspired by React Three Fiber

## Packages / Modules
- Core (EffectGPU): Effect-wrapped TypeGPU root and operations
- effectgpu/data: Re-exports pure TypeGPU data schemas; adds Effect-wrapped I/O and validation
- @effectgpu/react (concept): Canvas + hooks (useFrame, useBuffer, usePipeline, ...) for TSX usage

## Documentation
- docs/README.md (index)
- docs/effectgpu/vision.md (architecture and services)
- docs/effectgpu/typegpu-research.md (TypeGPU research and mapping)
- docs/effectgpu/examples.md (Effect vs TypeGPU examples)
- docs/effectgpu/tsx-dsl.md (Declarative TSX DSL; inspired by React Three Fiber)

## TSX Inspiration
Our TSX DSL follows the approach of React Three Fiber by expressing GPU operations declaratively and integrating with React hooks. See: https://r3f.docs.pmnd.rs/getting-started/introduction

## Status
Design complete; implementation scaffold next (core services, data I/O, TSX react bindings).


