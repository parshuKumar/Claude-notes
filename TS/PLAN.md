# TypeScript Learning Plan

## Status Legend
- [ ] Not started
- [x] Completed

---

## PHASE 1 — What TypeScript is and setup

- [x] 01. What is TypeScript — superset of JS, static typing, compiled language, why companies use it, what problems it solves that JS cannot
- [x] 02. Setting up TypeScript — installing typescript, tsc, ts-node, nodemon with ts-node, tsconfig.json basics, compiling your first .ts file, the build folder
- [x] 03. tsconfig.json in depth — target, module, outDir, rootDir, strict, esModuleInterop, moduleResolution, include, exclude, paths — what each option does and why
- [x] 04. TypeScript with Node.js project setup — full project setup from scratch: package.json, tsconfig, folder structure, scripts (dev, build, start), @types/node

## PHASE 2 — Type basics

- [x] 05. Type annotations — annotating variables, function parameters, return types, the basic syntax
- [x] 06. Primitive types — string, number, boolean, null, undefined, symbol, bigint in TypeScript — how they differ from JS
- [x] 07. Special types — any, unknown, never, void — what each means, when to use, when NOT to use, why any defeats the purpose of TS
- [x] 08. Type inference — how TypeScript figures out the type automatically, when to annotate and when to let TS infer, let vs const inference
- [x] 09. Union types — string | number, handling unions with type guards, discriminated unions
- [x] 10. Intersection types — combining types with &, when to use over union, real use cases
- [x] 11. Literal types — exact value types ("GET" | "POST"), const assertions, template literal types
- [x] 12. Type aliases — the type keyword, creating reusable type definitions, naming conventions
- [x] 13. Arrays and tuples — string[], Array<T>, tuple types [string, number], readonly arrays, when to use tuples vs arrays

## PHASE 3 — Interfaces and objects

- [x] 14. Interfaces — the interface keyword, defining object shapes, optional properties (?), readonly properties, vs type aliases
- [x] 15. type vs interface — the real differences, when to use each, extending both, declaration merging with interface
- [x] 16. Nested object types — typing deeply nested objects, inline types vs named types, keeping it readable
- [x] 17. Index signatures — [key: string]: value, dynamic object keys, when you need them, combining with fixed properties
- [x] 18. Extending interfaces — extends keyword, inheriting and adding properties, extending multiple interfaces
- [ ] 19. Implementing interfaces in classes — implements keyword, enforcing a contract on a class ← current

## PHASE 4 — Functions in TypeScript

- [ ] 20. Function types — typing parameters and return values, void return type, never return type
- [ ] 21. Optional and default parameters — optional params with ?, default values in TS, how they differ from JS
- [ ] 22. Rest parameters and spread in TS — typing ...args, rest param types, spread type safety
- [ ] 23. Function overloads — defining multiple call signatures, implementation signature, when to use
- [ ] 24. Higher-order function types — typing callbacks, typing functions that return functions, generic function types
- [ ] 25. this in TypeScript — the this parameter, typing this explicitly, avoiding this bugs

## PHASE 5 — Generics

- [ ] 26. What are generics — the problem they solve, the T placeholder, basic generic function
- [ ] 27. Generic functions — writing reusable functions with type parameters, multiple type params, constraints
- [ ] 28. Generic interfaces — parameterized interfaces, ApiResponse<T> pattern, real backend examples
- [ ] 29. Generic classes — typed classes, generic class methods, real use cases
- [ ] 30. Generic constraints — extends keyword in generics, keyof constraint, only accepting certain shapes
- [ ] 31. Default type parameters — T = string, making generics optional, when this helps
- [ ] 32. Utility types — Partial, Required, Readonly, Pick, Omit, Record, Exclude, Extract, NonNullable, ReturnType, Parameters — all of them with real examples

## PHASE 6 — Classes in TypeScript

- [ ] 33. Classes in TypeScript — class syntax, typed properties, constructor typing, how TS classes extend JS classes
- [ ] 34. Access modifiers — public, private, protected — what each allows, real encapsulation vs JS # private fields
- [ ] 35. Readonly class properties — readonly keyword, set once in constructor, difference from private
- [ ] 36. Parameter properties — constructor shorthand (private name: string in constructor params), cleaning up boilerplate
- [ ] 37. Abstract classes — abstract keyword, abstract methods, why they exist, when over interface
- [ ] 38. Class implementing interface — enforcing shape, multiple interfaces, abstract class vs interface decision
- [ ] 39. Static members in TS — static typed properties and methods, singleton pattern in TS

## PHASE 7 — Advanced types

- [ ] 40. Type narrowing in depth — typeof, instanceof, in operator, equality narrowing, truthiness narrowing
- [ ] 41. Type guards — custom type guard functions, the is keyword (value is Type), discriminated union guards
- [ ] 42. Discriminated unions — the tag/kind field pattern, exhaustive checks with never, real state machine example
- [ ] 43. Mapped types — transforming every key of a type, Readonly and Partial built with mapped types, custom mapped types
- [ ] 44. Conditional types — T extends U ? X : Y, infer keyword, extracting types conditionally
- [ ] 45. Template literal types — building string types dynamically, combining with unions, real use cases
- [ ] 46. Recursive types — types that reference themselves, JSON type, deeply nested structures
- [ ] 47. keyof and typeof operators — keyof for object keys, typeof for getting types from values, combining them

## PHASE 8 — Modules and declaration files

- [ ] 48. Modules in TypeScript — import/export in TS, esModuleInterop, default vs named imports, module resolution
- [ ] 49. Declaration files — .d.ts files, what they are, how @types packages work, writing a basic .d.ts
- [ ] 50. Ambient declarations — declare keyword, declaring global variables, augmenting existing types
- [ ] 51. Module augmentation — adding to existing module types, extending Express Request with custom properties

## PHASE 9 — TypeScript with Node.js and Express

- [ ] 52. Typing Express routes — Request, Response, NextFunction types, typed route handlers
- [ ] 53. Extending Express Request — adding req.user, req.body types, declaration merging on Express
- [ ] 54. Typed middleware — writing middleware with correct TS types, error middleware types
- [ ] 55. Typing environment variables — process.env is string | undefined problem, typed env with zod or custom solution
- [ ] 56. Typing database models with Mongoose — Document type, Model type, Schema generics, lean() return type
- [ ] 57. Typing database queries with pg (PostgreSQL) — QueryResult type, typed rows, generic query helper
- [ ] 58. Typing API responses — consistent ApiResponse<T> type, error response type, using across the whole app
- [ ] 59. Typing configuration objects — config files, typed settings, environment-specific config

## PHASE 10 — Error handling and advanced patterns in TS

- [ ] 60. Error handling in TypeScript — catch block gives unknown not Error, narrowing errors, custom error classes with types
- [ ] 61. Result type pattern — avoiding thrown errors, Result<T, E> type, functional error handling
- [ ] 62. Strict null checks — what strictNullChecks does, non-null assertion operator (!), optional chaining in TS
- [ ] 63. Assertion functions — asserts keyword, throwing on bad input, making TypeScript trust your checks
- [ ] 64. Satisfies operator — satisfies keyword (ES2022+), checking type without widening, real use cases

## PHASE 11 — TypeScript patterns and best practices

- [ ] 65. Readonly and immutability patterns — Readonly<T>, as const, readonly arrays, enforcing immutability
- [ ] 66. Builder pattern in TypeScript — fluent API with typed chaining, real query builder example
- [ ] 67. Repository pattern in TypeScript — typed repository interface, generic base repository, with Mongoose
- [ ] 68. Dependency injection basics in TS — constructor injection, interface-based injection, why it helps testing
- [ ] 69. Enum vs union types — TypeScript enums (string and numeric), const enums, why many prefer union types over enums
- [ ] 70. Namespaces — the namespace keyword, when they were used, why modules replaced them, legacy code
- [ ] 71. Triple slash directives — /// references, when you still see them, what they do

## PHASE 12 — Tooling and workflow

- [ ] 72. ESLint with TypeScript — @typescript-eslint setup, important rules, integrating with Prettier
- [ ] 73. Prettier with TypeScript — setup, config file, format on save, integrating in CI
- [ ] 74. Path aliases in TypeScript — configuring paths in tsconfig, using @/ imports, making it work with Node at runtime
- [ ] 75. Debugging TypeScript in VS Code — source maps, launch.json config, breakpoints in .ts files
- [ ] 76. Building for production — tsc build, removing source maps in prod, build scripts, ts-node vs compiled JS in production
- [ ] 77. Monorepo basics with TypeScript — project references, composite projects, sharing types across packages

---

## Current Progress

**Current Topic:** 19  
**Completed:** 18 / 77  
