# TypeScript Best Practices Checklist

## Type Safety & Declarations

### Typing Variables & Constants

- **`any` usage**: Defeats type checking; use `unknown` with refinement instead
  ```typescript
  // Bad: any accepts anything
  const value: any = getThing()
  value.someMethod() // No error, but might crash
  
  // Good: unknown requires type guard
  const value: unknown = getThing()
  if (typeof value === 'object' && value !== null && 'someMethod' in value) {
    value.someMethod()
  }
  ```
- **Implicit `any` from loose tsconfig**: `noImplicitAny: true` should be enabled
- **Over-typing primitives**: `const x: string = "hello"` (let TS infer)
- **Not using `as const`** for literal types that shouldn't widen
  ```typescript
  // Bad: Type is 'string'
  const status = "active"
  
  // Good: Type is literal 'active'
  const status = "active" as const
  ```
- **Union type instead of overload**: Complex overloads better than wide unions
- **Missing `readonly` on immutable data**: Allows accidental mutation
  ```typescript
  // Good: Immutable intent clear
  type Point = { readonly x: number; readonly y: number }
  interface Config { readonly timeout: number }
  ```

### Null & Undefined Handling

- **`null` vs `undefined` inconsistency**: No clear convention throughout codebase
- **Not using strict null checks**: `strictNullChecks: true` required
- **Optional chaining misuse**: `a?.b?.c` shouldn't silently hide structural errors
  ```typescript
  // Better: Explicit checks to catch bugs early
  if (a === null || a === undefined) throw new Error('a is null')
  if (a.b === null || a.b === undefined) return default
  const c = a.b.c // Now safe
  ```
- **Assertion without validation**: Using `as` to force type without runtime check
  ```typescript
  // Bad: Assertion is a lie
  const user = response.data as User
  
  // Good: Validate shape
  const parseUser = (data: unknown): User => {
    if (!isUser(data)) throw new Error('Invalid user')
    return data
  }
  ```

### Questions to Ask
- "Is `any` used? Why not use `unknown`?"
- "Are null/undefined properly handled throughout?"
- "Could a type be more specific?"

---

## Functions & Overloads

### Function Types

- **No explicit return type**: Return type should be annotated for public functions
  ```typescript
  // Bad: Return type inferred, might change unexpectedly
  export function process(data) {
    return data.length > 0 ? data[0] : null
  }
  
  // Good: Intent clear
  export function process(data: string[]): string | null {
    return data.length > 0 ? data[0] : null
  }
  ```
- **Parameters without types**: Function behavior unclear
- **Using `Function` type**: Meaningless signature; use specific function type
  ```typescript
  // Bad
  type Callback = Function
  
  // Good
  type Callback = (error: Error | null, result: Data) => void
  ```
- **Implicit `any` from callback parameters**: Requires explicit typing
- **Not using `Pick` for callback options**: Bad DX when passing options

### Function Overloads

- **Unnecessary overloads**: Multiple overloads when union types would work
  ```typescript
  // Overload-heavy (avoid if possible)
  function get(key: string): string
  function get(key: number): number
  function get(key: string | number): string | number { ... }
  
  // Better: Generic with constraint
  function get<T extends string | number>(key: T): T { ... }
  ```
- **Implementation signature missing or uncovered**: Overload signatures must be compatible with implementation
- **Overload order matters but unclear**: Most specific should come first

### Questions to Ask
- "Is the return type explicit and correct?"
- "Could this use generics instead of multiple overloads?"
- "Are all function parameters typed?"

---

## Generics & Constraints

### Generic Design

- **Unused generic parameters**: Type parameter not affecting any output
  ```typescript
  // Bad: T unused
  interface Container<T> { getValue(): string }
  
  // Good: T affects output
  interface Container<T> { getValue(): T }
  ```
- **No constraints on generics**: `T` too permissive, missing bounds
  ```typescript
  // Risky: T could be anything
  function print<T>(value: T): void { console.log(value.toString()) }
  
  // Better: Constrain T
  function print<T extends { toString(): string }>(value: T): void { console.log(value.toString()) }
  ```
- **Overly broad constraints**: Constraint defeats purpose of generic
  ```typescript
  // Might as well not be generic
  function process<T extends unknown>(value: T): T { return value }
  ```
- **Generic defaults missing**: `<T = string>` when sensible default exists
- **Leaking internal types in generics**: Generic type parameters expose implementation

### Generic Inference

- **Not letting TS infer generics**: Overly explicit when inference works
  ```typescript
  // Verbose
  const result: string[] = map<string>(data, x => x.toUpperCase())
  
  // Better: Let TS infer
  const result = map(data, x => x.toUpperCase())
  ```
- **Generic inference too broad**: Type becomes wider than intended
  ```typescript
  // Bad: result is (string | number)[]
  const values = [1, "two"]
  
  // Good: Explicit type
  const values: (string | number)[] = [1, "two"]
  ```

### Conditional Types

- **Overly complex conditional types**: Harder to understand than equivalent
  ```typescript
  // Hard to follow
  type Flatten<T> = T extends Array<infer U> ? Flatten<U> : T
  
  // Could be simplified or well-documented
  ```
- **Conditional types in wrong place**: Helper type library instead of inline

### Questions to Ask
- "Is this generic parameter actually used?"
- "Can TS infer this generic, or do I need to specify it?"
- "Is the constraint appropriate?"

---

## Interfaces & Types

### Interface vs Type

- **Over-using `type` for everything**: `interface` better for object contracts
  ```typescript
  // Reasonable use of type (union, tuple)
  type Color = 'red' | 'blue' | 'green'
  type Coordinates = [number, number]
  
  // Use interface for objects
  interface User {
    id: string
    name: string
  }
  ```
- **Not using `interface` declaration merging**: When library users extend types
- **Type aliases for simple primitives**: Unnecessary abstraction
  ```typescript
  // Questionable: Just use string
  type UserId = string
  
  // Better: Nominal type
  type UserId = string & { readonly __brand: 'UserId' }
  const userId = (id: string): UserId => id as UserId
  ```

### Interface Design

- **Over-large interfaces**: Wide interfaces hard to implement and test
  ```typescript
  // Too broad
  interface Service {
    fetch: () => Promise<Data>
    save: (data: Data) => Promise<void>
    delete: (id: string) => Promise<void>
    validate: (data: unknown) => boolean
    format: (data: Data) => string
  }
  
  // Better: Separate concerns
  interface Fetcher { fetch(): Promise<Data> }
  interface Storage { save(data: Data): Promise<void> }
  interface Validator { validate(data: unknown): boolean }
  ```
- **Optional properties overused**: Weakens contract and complicates implementation
  ```typescript
  // Too permissive
  interface Config {
    host?: string
    port?: number
    timeout?: number
  }
  
  // Better: Make intent clear or split types
  interface RequiredConfig {
    host: string
    timeout: number
  }
  interface OptionalConfig {
    port?: number
  }
  ```
- **Missing extends for contract**: Base interface not enforced

### Indexing & Mapped Types

- **Index signature that's too broad**: `[key: string]` allows any property
  ```typescript
  // Risky: Allows any key
  interface Options { [key: string]: any }
  
  // Better: Enumerate or restrict
  type OptionKey = 'timeout' | 'retries' | 'cache'
  interface Options { [K in OptionKey]?: number }
  ```
- **Mapped types misunderstood**: `in keyof` works only on known keys
- **Index signature with incompatible property**: `[key: string]` + specific keys may conflict

### Questions to Ask
- "Should this be an `interface` or `type`?"
- "Is this interface doing too much?"
- "Could mapped types simplify this?"

---

## Classes & OOP

### Class Design

- **Not using `readonly` on class properties**: Allows accidental mutation
  ```typescript
  // Bad: Can mutate
  class Point { x: number; y: number }
  
  // Good: Intent clear
  class Point {
    readonly x: number
    readonly y: number
    constructor(x: number, y: number) {
      this.x = x
      this.y = y
    }
  }
  ```
- **Access modifiers not explicit**: Properties are public by default (confusing)
  ```typescript
  // Implicit public (unclear intent)
  class Service { apiKey: string }
  
  // Explicit
  class Service {
    private readonly apiKey: string
    public getData(): Promise<Data> { ... }
  }
  ```
- **No distinction between `private` and `protected`**: Leads to accidental misuse
- **`private` fields with `#` vs `private` keyword**: `#` is truly private (not visible in inheritance)
  ```typescript
  // Can be accessed in subclass
  class Base { private secret: string }
  class Child extends Base { getSecret() { return this.secret } } // Error
  
  // Actually private
  class Base { #secret: string }
  class Child extends Base { getSecret() { return this.#secret } } // Error
  ```
- **Not using `abstract` for partial implementations**: Forces subclasses to override
- **Static vs instance methods confusion**: Should be static? (no `this` access needed)

### Inheritance Issues

- **Deep inheritance hierarchies**: More than 2-3 levels hard to reason about
- **Not using composition**: Inheritance when composition would be cleaner
  ```typescript
  // Over-inheritance
  class Animal { move() { ... } }
  class Mammal extends Animal { warm() { ... } }
  class Dog extends Mammal { bark() { ... } }
  
  // Better: Composition
  class Dog {
    private movement = new Movement()
    private sound = new Sound()
    move() { return this.movement.move() }
    bark() { return this.sound.bark() }
  }
  ```
- **Liskov Substitution Principle violations**: Subclass can't be substituted for base

### Questions to Ask
- "Should this property be `private` or `protected`?"
- "Is this class doing too much?"
- "Would composition be clearer than inheritance?"

---

## Type Aliases & Unions

### Union Types

- **Over-broad unions**: Union of many dissimilar types hard to handle
  ```typescript
  // Hard to use safely
  type Result = Success | Error | Pending | Cancelled | Timeout
  
  // Better: Discriminated union
  type Result = 
    | { status: 'success'; data: Data }
    | { status: 'error'; error: Error }
    | { status: 'pending' }
  ```
- **Discriminated union not used**: Discriminator property makes narrowing clear
- **String union without `as const`**: Type string instead of literal

### Intersection Types

- **Intersection instead of type composition**: Hard to read
  ```typescript
  // Unreadable
  type Props = ChildProps & ParentProps & ThemeProps & ValidationProps
  
  // Better: Interface composition
  interface Props extends ChildProps, ParentProps, ThemeProps, ValidationProps {}
  ```
- **Conflicting intersections**: `string & number` creates `never` type
- **Deep object merging with intersection**: Confusing semantics

### Utility Types

- **`Partial<T>` overused**: Should be explicit about which fields are optional
  ```typescript
  // Too vague
  function updateUser(partial: Partial<User>): void { ... }
  
  // Better: Explicit
  function updateUser(changes: Partial<Pick<User, 'name' | 'email'>>): void { ... }
  ```
- **`Record<string, any>`**: Too permissive; use more specific object types
- **`ReturnType<T>` for complex types**: Might be unreadable; consider explicit return
- **Not using `Omit`, `Pick`, `Exclude`**: Repeating type logic instead

### Questions to Ask
- "Is this union type getting too broad?"
- "Should this use a discriminated union for type narrowing?"
- "Could a utility type simplify this?"

---

## Enums & Constants

### Enum Usage

- **String enums for flexible values**: Don't provide type safety benefit
  ```typescript
  // Weak: Allows any string
  enum Status { Active = 'active', Inactive = 'inactive' }
  const status: Status = 'anything' // Error, good
  
  // Better: Union type or const
  type Status = 'active' | 'inactive'
  const Status = { Active: 'active', Inactive: 'inactive' } as const
  type Status = typeof Status[keyof typeof Status]
  ```
- **Numeric enums with gaps**: Sparse enum values confusing
- **Enums in public APIs**: Causes versioning issues; prefer discriminated unions
- **Two-way mapping enums**: Bloats bundle, only use if truly needed
  ```typescript
  // Bad: Bloated
  enum Color { Red = 0, Green = 1, Blue = 2 }
  // Generates: Color[0] = 'Red', Color['Red'] = 0, etc.
  
  // Better: Use const
  const Colors = { Red: 0, Green: 1, Blue: 2 } as const
  ```

### Constants & Literal Types

- **Magic values instead of named constants**: Unclear intent
  ```typescript
  // Bad: What's 1000?
  setTimeout(() => doThing(), 1000)
  
  // Good: Named constant
  const DEBOUNCE_MS = 1000
  setTimeout(() => doThing(), DEBOUNCE_MS)
  ```
- **`const` assertion not used for object literals**: Type widens unnecessarily

### Questions to Ask
- "Could this enum be a union type instead?"
- "Is this constant name descriptive?"

---

## Type Refinement & Narrowing

### Type Guards

- **`typeof` checks wrong for objects**: `typeof {} === 'object'` includes null
  ```typescript
  // Bad: typeof check incomplete
  if (typeof value === 'object') {
    value.prop // value could be null
  }
  
  // Good: Proper check
  if (value !== null && typeof value === 'object') {
    value.prop
  }
  ```
- **`instanceof` used on primitive types**: Doesn't work; use `typeof`
- **Type predicate functions not written correctly**: Wrong return type
  ```typescript
  // Bad: Should be 'is User'
  function isUser(value: unknown): boolean {
    return 'id' in value && 'name' in value
  }
  
  // Good: Type predicate
  function isUser(value: unknown): value is User {
    return value instanceof Object && 'id' in value && 'name' in value
  }
  ```
- **Not using `satisfies` operator**: Validate shape without losing type
  ```typescript
  // TypeScript 4.9+
  const config = {
    host: 'localhost',
    port: 3000
  } satisfies Record<string, string | number>
  ```

### Type Assertion

- **Excessive `as` usage**: Should be last resort after type guards
- **`as const` used inappropriately**: Creates immutable types when not needed
- **Double assertion to bypass checks**: `as unknown as Type` defeats type system

### Questions to Ask
- "Is there a better way to narrow this type without `as`?"
- "Are type predicate functions correct?"

---

## Module & Declaration

### Import/Export

- **Barrel exports hiding internals**: `index.ts` re-exporting everything confuses tree-shaking
  ```typescript
  // Bad: Barrel exports too much
  // index.ts
  export * from './internal'
  export * from './utils'
  export * from './types'
  
  // Good: Explicit exports
  export { PublicAPI } from './api'
  export type { User } from './types'
  ```
- **Circular dependencies**: A imports B, B imports A (TS can't resolve)
- **Default exports for classes/functions**: Named exports generally better for refactoring
- **Not using `export type` for types**: Re-export as values causes bundle bloat
- **Common JS interop issues**: `require()` and `export default` mixing

### Declaration Files

- **Hand-written `.d.ts` missing or incomplete**: Missing types for JS dependencies
- **`@ts-ignore` comments without explanation**: Defeats type checking
- **`declare` used in wrong places**: Global declarations pollute namespace
  ```typescript
  // Bad: Pollutes global scope
  declare const globalThing: string
  
  // Better: Module augmentation
  declare global {
    interface Window { globalThing: string }
  }
  ```
- **DefinitelyTyped dependencies not installed**: `@types/pkg` missing
- **Not updating types when library updates**: Type stubs become incorrect

### Questions to Ask
- "Should this be `export type` or `export`?"
- "Are there circular dependencies?"
- "Are type definitions up to date with library?"

---

## Advanced Patterns

### Type Inference

- **`infer` keyword misunderstood**: Used to capture types in conditional types
  ```typescript
  // Extract type from Promise
  type Awaited<T> = T extends Promise<infer U> ? U : T
  type Result = Awaited<Promise<string>> // string
  ```
- **Inference in wrong place**: Let TS infer from usage, not declaration

### Variance Issues

- **Covariance/contravariance not understood**: Function parameters are contravariant
  ```typescript
  // This is allowed (contravariance in parameters)
  type Base = (x: Animal) => void
  type Derived = (x: Dog) => void
  const fn: Base = (x: Dog) => {} // OK
  
  // But not the other way around
  ```

### Brands & Opaque Types

- **Primitive obsession without branded types**: `string` could be userId, email, etc.
  ```typescript
  // Bad: Can mix up string types
  type UserId = string
  type Email = string
  const user: UserId = email // No error!
  
  // Good: Branded types
  type UserId = string & { readonly __brand: 'UserId' }
  type Email = string & { readonly __brand: 'Email' }
  function makeUserId(id: string): UserId { return id as UserId }
  ```
- **Nominal typing in structural TypeScript**: Impossible; use brands or enums instead

### Questions to Ask
- "Could this use a branded type for better type safety?"
- "Is this using variance correctly?"

---

## Configuration & Strictness

### TypeScript Compiler Options

- **Strict mode not fully enabled**: `strict: true` includes multiple flags; verify all are on
  ```json
  {
    "compilerOptions": {
      "strict": true,
      "noImplicitAny": true,
      "strictNullChecks": true,
      "strictFunctionTypes": true,
      "strictBindCallApply": true,
      "strictPropertyInitialization": true,
      "noImplicitThis": true,
      "noUnusedLocals": true,
      "noUnusedParameters": true,
      "noImplicitReturns": true,
      "noFallthroughCasesInSwitch": true
    }
  }
  ```
- **`skipLibCheck` enabling silent errors**: Library types ignored
- **`esModuleInterop` and `allowSyntheticDefaultImports` misunderstood**: Different behaviors
- **Target version too old**: `ES2020` or newer recommended for modern features

### Type Checking in JS

- **Not using `checkJs` for JavaScript files**: JavaScript files have no type checking
- **JSDoc comments sufficient without TS**: Verbose; migrate to TypeScript
- **`.d.ts` files in `dist` not separated**: Publishing types along with runtime

### Questions to Ask
- "Is `strict: true` enabled?"
- "Are compiler error counts growing or shrinking?"
- "Is the TypeScript version up to date?"

---

## Testing & Type Safety

### Testing Types

- **Not testing types**: Runtime tests miss type errors
- **No test for complex generic**: Generic implementation hard to verify
  ```typescript
  // Good: Compile-time type test
  const _: TypeEqual<Awaited<Promise<string>>, string> = true
  ```
- **`as const` not in test data**: Test data types too broad, missing precision
- **Type-only imports not used in tests**: Tests import runtime code

### Runtime Validation

- **Trusting types at runtime**: Types are erased; validate inputs
  ```typescript
  // Bad: Assumes API returns User type
  const user = await fetch('/user').then(r => r.json() as User)
  
  // Good: Validate schema
  const user = parseUser(await fetch('/user').then(r => r.json()))
  function parseUser(data: unknown): User {
    if (!isUser(data)) throw new Error('Invalid user')
    return data
  }
  ```
- **No input validation for user data**: TypeScript doesn't validate at runtime
- **Missing zod/yup for API responses**: No schema validation

### Questions to Ask
- "Is type correctness tested?"
- "Are external inputs validated against expected types?"
- "Are types generating correct runtime behavior?"

---

## Performance & Bundle Size

### Type-Only Constructs

- **Not using `type` for import/export**: Unused types might be bundled
  ```typescript
  // Risk of being bundled
  import { User } from './types'
  export { Status } from './enums'
  
  // Safe: Only types, never bundled
  import type { User } from './types'
  export type { Status } from './enums'
  ```
- **Conditional types in public APIs**: Complex types hard to infer, slow compilation

### Build Performance

- **`skipLibCheck` enabled for speed**: Hides errors, bad for correctness
- **Project references not used**: Monorepo compilation slow
- **Type-heavy dependencies**: Dependencies with complex types slow compilation

### Questions to Ask
- "Are imports marked as `type` when appropriate?"
- "Does this type slow down build/IDE?"
- "Could project references improve build speed?"
