---
name: occams-razor
description: >
  Prefer the simplest design and implementation that fully satisfies current,
  demonstrated requirements. Use when designing, implementing, refactoring, or
  reviewing code to avoid speculative abstractions and unnecessary complexity.
---

# Occam's Razor

- Prefer the simplest design and implementation that fully satisfies the
  current, demonstrated requirements.
- Do not add abstractions, layers, configuration, extension points, dependencies,
  or infrastructure for hypothetical future needs. Introduce them only when
  concrete requirements or repeated patterns justify their cost.
- Before adding code, consider whether the goal can be met by deleting,
  consolidating, or reusing existing code.
- When multiple approaches are correct, choose the one with fewer concepts,
  moving parts, and maintenance obligations, unless evidence shows that a more
  complex approach is necessary.
- Treat patterns and principles as tools, not goals. Do not apply SOLID, design
  patterns, or architectural boundaries in ways that make a small solution more
  complicated than the problem requires.
