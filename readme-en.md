Of course. This document describes a new TypeScript transpiler with a strong focus on optimization, achieved through a "contract" between the developer and the compiler. The localization requires translating the technical concepts and the tool's philosophy into natural, idiomatic English for a developer audience.

Here is the localized English version:

---

# Strict-Ts

Strict-Ts is a TypeScript transpiler focused on generating highly secure and compact JavaScript. It is designed not only as a standalone compiler but also to integrate seamlessly with modern build tool ecosystems like Webpack and Vite.

## Core Philosophy: Mutual Guarantees

The power of Strict-Ts's optimizations is built upon a set of "mutual guarantees"—a contract between the developer and the compiler. The core of this philosophy is that developers express their intent clearly by adhering to specific coding principles. In return, Strict-Ts leverages TypeScript's type information to perform exhaustive analysis and aggressive optimization on the output JavaScript.

## The Developer's Contract

To activate Strict-Ts's optimization engine, developers must adhere to the following coding contract.

### Strict Semantic Distinction between `any` and `unknown`

Strict-Ts differentiates the semantics of `any` and `unknown` to enhance the effectiveness of static analysis.

#### `any`: Intended for Use After Runtime Checks

The `any` type is interpreted as a declaration by the developer that "its value will be type-checked at runtime." Based on this assumption, Strict-Ts will infer a more precise type for the variable within type guard blocks.

To enforce this contract, Strict-Ts, in its strict mode, will report a compile-time error for any `any`-typed variable used directly without a type guard, thereby reducing potential runtime type errors.

```ts
let value: any = someFn();

// The contract is fulfilled by performing a type guard.
if (value instanceof Foo) {
  // Within this block, Strict-Ts safely infers `value` as type `Foo`.
}
```

#### `unknown`: Indifference to Its Specific Type

The `unknown` type signifies that the value's specific type is irrelevant to the current logic. This type is suitable for scenarios where a value is merely passed through or its return value is ignored.

```ts
function someFn(callback: (val: number) => unknown) {
  // In this function, we don't care what the callback returns,
  // only that it is a callable function.
}
```

### Type Assertions

It is recommended to use the `satisfies` operator for type validation. It verifies that a variable conforms to a specific structure without altering its original inferred type. In contrast, `as` directly overrides the compiler's type inference.

If you must use `as` for an assertion, Strict-Ts will adopt different strategies based on the direction of the assertion:

#### Up-casting (Abstraction)

When a more specific type is asserted to its parent type or an interface, the operation is considered type-safe. Strict-Ts will continue to apply optimizations based on this.

```ts
interface Foo { /* ... */ }
interface Bar extends Foo { /* ... */ }

let myBar: Bar = new Bar();
let myFoo: Foo = myBar as Foo; // Safe up-cast, optimizations are preserved.
```

#### Down-casting (Specialization)

When a more abstract type is asserted to one of its subtypes, there is a risk of a type mismatch. Strict-Ts will adopt a more conservative optimization strategy for its scope and related dependencies.

```ts
let myFoo: Foo = someFn();
let myBar: Bar = myFoo as Bar; // Unsafe down-cast, optimizations will be conservative.
```

Developers can use the `/*#__UNSAFE_TYPE_TRANSFORM__*/` annotation to mark a cast that is known to be safe. This annotation instructs the compiler to trust the developer's assertion and continue applying its standard optimization strategies.

```ts
// Object.keys(foo) returns string[] by default, which cannot be directly assigned to keyof typeof foo.
// for (const key of Object.keys(foo) as keyof typeof foo) {
//   // Aggressive, type-based optimizations cannot be applied here.
// }

for (const key of /*#__UNSAFE_TYPE_TRANSFORM__*/ Object.keys(foo) as keyof typeof foo) {
  // Here, the developer's assertion is fully trusted, and type information
  // will be used for aggressive optimizations.
}
```

### Type Assertion Functions

Assertion functions that return `value is T` can assist Strict-Ts's type inference. Inside the assertion function, Strict-Ts will rigorously check if sufficient evidence is provided; otherwise, it will report a compile-time error to reduce potential runtime type errors.

If the function's input type is a finite set of types, such as a union type, a parent class, or an interface, the assertion is relatively safe and can be made with less evidence.

```ts
function isFoo(value: Foo | Bar): value is Foo {
  return "prop_foo" in value; // Differentiating `Foo` from `Foo | Bar`.
}

let a: Foo | Bar = someFn();
if (isFoo(a)) {
  // Here, `a` will be treated as a `Foo` object.
} else {
  // Here, `a` will be treated as a `Bar` object.
}
```

When the input type is `any`, the assertion carries a higher risk. The developer is responsible for providing sufficient runtime checks to support the assertion.

```ts
interface Foo {
  prop: string;
}

function isFoo(value: any): value is Foo {
  // Provides "evidence" for type safety through a full property and type check.
  return value && typeof value === 'object' &&
    "prop" in value && typeof value.prop === "string";
}
```

Only when you are unable or, for performance reasons, unwilling to perform a complete runtime check, but still want the compiler to trust your assertion, should you use the `UNSAFE_ASSERT_` prefix to explicitly mark the risk. This tells the compiler: "Trust me, I know this is safe in this specific context. Please optimize based on this assumption."

```ts
// Unsafe implementation: The check is incomplete, but the developer is confident
// about the type based on the context.
function UNSAFE_ASSERT_isFoo(value: any): value is Foo {
  // This only checks for property existence, not its type, which could cause a runtime error.
  // Using `UNSAFE_ASSERT_` acknowledges and flags this risk.
  return "prop" in value;
}
```

### Pure Operation Declarations

To enable more thorough Tree Shaking, developers can mark certain function calls as "pure." If the return value of a pure call is not used, the call expression can be removed in production builds.

#### Call-site Annotation: `/*#__PURE__*/`

This is an industry-standard annotation used to mark a single function call or object instantiation as pure. By using this annotation, the developer signals to the compiler: "Removing this specific call will not cause unintended side effects."

```ts
// A side-effectful ID generator
function generate_id(): number {
  // This generator might skip already-used numbers, for example.
  let id: number = -1;
  while (id < 0 || mem.has(id))
    id = Math.round(Math.random() * 0x1000);

  mem.add(id);
  return id;
}

// This call can be removed in production mode, but only if `unused_id` is unused.
const unused_id = /*#__PURE__*/ generate_id();

// This call will be retained because it lacks the PURE annotation.
generate_id();
```

This annotation can also be used with module imports.

```ts
// This import can be removed in production mode if its members are not used.
/*#__PURE__*/ import * as ImportName from "module";
/*#__PURE__*/ import { someFn, constant } from "module";
```

#### Definition-site Annotation: `/*#__HINT_PURE__*/`

`/*#__HINT_PURE__*/` is a Strict-Ts-specific annotation used to mark all calls to a function as pure by default. This simplifies the code, but this behavior is only effective within the Strict-Ts toolchain. For projects requiring consistent behavior across toolchains (e.g., Babel, SWC), the standard `/*#__PURE__*/` annotation is recommended.

It is recommended to place this annotation between the `function` keyword and the function name.

```ts
// Example 1: A convenient shorthand for pure calls
function /*#__HINT_PURE__*/ hint_generate_id(): number {
  // ...
}

// In Strict-Ts, the following call is automatically treated as pure,
// equivalent to adding a /*#__PURE__*/ annotation to a regular function call.
const my_id = hint_generate_id();

// Equivalent to:
// const my_id = /*#__PURE__*/ generate_id();
```

```ts
// Example 2: Removing a development-only function
// The goal is for this function to exist only in development and disappear in production.
function /*#__HINT_PURE__*/ debug_report(message: string): void {
  // In development mode, this will print a log.
  console.log(message);
}

// In production mode, the line below will be completely removed,
// because the function returns void, its return value is always considered "unused."
debug_report("Some debug message...");
```

### Unsafe Declarations

For code regions where type safety cannot be guaranteed, you can use the `/*#__UNSAFE__*/` or `/*#__UNSAFE_START__*/` and `/*#__UNSAFE_END__*/` annotations. Strict-Ts will completely trust the type information provided by the developer within these regions and optimize accordingly.

## The Compiler's Payoff

When developers follow the coding contract, Strict-Ts can perform the following optimizations:

### Enhanced Dead Code Elimination

Through global type-flow analysis, Strict-Ts can identify and remove code branches that are unreachable based on type logic.

```ts
// Source Code
interface Foo { /* ... */ }
class A implements Foo { /* ... */ }
class B implements Foo { /* ... */ }
class C implements Foo { /* ... */ }

function branch(value: Foo) {
  if (value instanceof A) return doSthA(value);
  if (value instanceof B) return doSthB(value);
  if (value instanceof C) return doSthC(value);
  return doDefault(value);
}

// ...

// Actual calls
const a = new A();
const b = new B();
branch(a);
branch(b);
```

Strict-Ts traces all inputs, identifies potentially unexecuted code paths, and removes them.

```js
// Strict-Ts Dead Code Elimination Example
class A { /* ... */ }
class B { /* ... */ }

function branch(value) {
  if (value instanceof A) return doSthA(value);
  return doSthB(value); // The last two branches were removed
}

// ...

const a = new A();
const b = new B();
branch(a);
branch(b);
```

In fact, after dead code elimination, the complexity of the `branch` function is reduced. If `branch` has few call sites or its body becomes simple enough, Strict-Ts will further apply Function Inlining, replacing the call with the function's body and, combined with Tree Shaking, removing the now-unreferenced `branch` function itself. This can result in extremely optimized code like this:

```js
// Final Strict-Ts Output
class A { /* ... */ }
class B { /* ... */ }

// ...

doSthA(new A());
doSthB(new B());
```

### More Aggressive Tree Shaking

Strict-Ts can analyze all instances of a class to determine which members (methods or properties) are never used and remove them from the final output.

```ts
// Source Code
class DataProcessor {
    constructor(private data: string) {}

    encode(): string {
        return btoa(this.data);
    }

    // The `decode` method is never called by any code.
    decode(): string {
        return atob(this.data);
    }

    validate(): boolean {
        return this.data.length > 0;
    }
}

const processor = new DataProcessor("hello");
console.log(processor.encode());
if (processor.validate()) {
    // ...
}
```

```js
// Strict-Ts Output
// The `decode` method has been safely removed.
class DataProcessor {
    constructor(data) {
        this.data = data;
    }
    encode() {
        return btoa(this.data);
    }
    validate() {
        return this.data.length > 0;
    }
}

const processor = new DataProcessor("hello");
console.log(processor.encode());
if (processor.validate()) {
    // ...
}
```

### `enum` Optimization

Standard TypeScript `enum`s generate objects with both forward and reverse lookups, even if the reverse lookup is never used. Strict-Ts performs a global usage analysis on `enum`s.

For better readability and to clarify developer intent, using `const enum` directly is still encouraged.

#### Without Reverse Lookup

If the analysis finds no code using a reverse lookup (e.g., `Status[0]`), it will inline the `enum` members as constant values, similar to `const enum` behavior, and remove the unused lookup object.

```ts
// Source Code
enum Status { sleep, active }

switch (statusA) {
    case Status.sleep: break; // Uses forward lookup
    case Status.active: break; // Uses forward lookup
    default: // Dead code
}
```

```js
// Strict-Ts Output
switch (statusA) {
    case 0 /* Status.sleep */:
        break;
    case 1 /* Status.active */:
        break;
}
```

#### With Reverse Lookup

If reverse lookup usage is detected, a `tsc`-compatible lookup object will be generated.

```ts
// Source Code
enum Status { sleep, active }

function getStatusName(status: Status) {
    return Status[status];
}
```

```js
// Strict-Ts Output, behavior is identical to tsc
"use strict";
var Status;
(function (Status) {
    Status[Status["sleep"] = 0] = "sleep";
    Status[Status["active"] = 1] = "active";
})(Status || (Status = {}));

function getStatusName(status) {
    return Status[status];
}
```

## Optimization Practices

Following these additional conventions can lead to more robust code and even better optimization results.

### Using `Result<T, E>` for Predictable Errors

It is recommended to use a `Result<T, E>` type to handle predictable errors instead of the traditional `throw` statement. This pattern incorporates error handling into the type system, allowing the compiler to statically analyze control flow via type guards. The `throw` statement should be reserved for handling exceptional, unrecoverable situations that break the normal flow of execution.

Similarly, for `Promise`s and `async function`s, it is recommended to `resolve` or `return` results and errors as a `Result<T, E>`. The `reject` and `throw` keywords should be reserved for exceptional, flow-breaking errors.

## Configuration

The behavior of Strict-Ts can be configured via a `"strictTs"` property in your project's `tsconfig.json` file or in a separate `strict-ts.config.json` file. The following is a detailed explanation of each configuration option.

### `"downcastingTransform"`

This option defines how Strict-Ts handles down-casting. It controls the compiler's level of trust in these potentially unsafe operations and directly affects the optimization strategy for related code paths.

-   `"error"`: **Recommended.** The strictest mode. Any down-cast will result in a compile-time error. This mode forces developers to explicitly handle all type uncertainties, achieving the highest level of runtime safety.
-   `"hint"`: **Default & Recommended.** When a down-cast is detected, the compiler adopts a conservative optimization strategy (i.e., it only removes types, without performing aggressive type-based optimizations) and reports a hint or warning in the console, advising the developer to review the type conversion.
-   `"ignore"`: Similar to `"hint"`, the compiler adopts a conservative strategy but does not output any hints. This option is suitable for projects that are migrating gradually and do not want to deal with a large number of warnings at once.
-   `"trust"`: Fully trusts the developer's type assertions. The compiler will apply aggressive optimization strategies based on these assertions. This can lead to significant performance and bundle size gains, but if an assertion proves incorrect at runtime, it may lead to unexpected behavior or errors.
-   `"trust-and-hint"`: Same as `"trust"`, the compiler applies aggressive optimizations but also outputs a hint to the console, informing the developer that a trusted transformation occurred. This helps track potential risk points while still enjoying the benefits of optimization.

#### Related Configuration: `"transformFromExternalAny"`

This option specifically handles down-casts from `any` types originating from external modules. Since third-party library typings may use `any`, a separate configuration is provided. For example:

```ts
import { returnAny } from "legacy-module";
// This function returns `any` and is down-cast to `Foo`.
// This operation is normally untrusted in Strict-Ts.
const value = returnAny() as Foo;
```

When set, this option overrides the global `"downcastingTransform"` configuration for this specific case.

-   `"inherit"`: **Default & Recommended.** Follows the `"downcastingTransform"` setting.
-   `"error"`: **Recommended.** The strictest mode. Any down-cast from an external `any` not exempted with an `UNSAFE_` annotation will cause a compile-time error.
-   `"hint"`: **Recommended.** The compiler adopts a conservative strategy and reports a hint in the console.
-   `"ignore"`: The compiler adopts a conservative strategy but does not output any hints.
-   `"trust"`: Fully trusts the developer's assertion and applies aggressive optimizations.
-   `"trust-and-hint"`: Applies aggressive optimizations while also outputting a hint.

You can also configure this on a per-module basis:

```json
"transformFromExternalAny": {
  "react": "trust",                // Fully trust `any` assertions from the 'react' module
  "some-old-library": "ignore",    // Ignore `any` from a legacy library, no errors or hints
  "*": "hint"                      // Provide a hint for all other external modules
}
```

#### `UNSAFE_` Exemption Behavior

When you use the `/*#__UNSAFE_TYPE_TRANSFORM__*/` annotation, Strict-Ts will always adopt an aggressive optimization strategy (equivalent to `"trust"` mode). The `"downcastingTransform"` setting only affects whether an additional hint is displayed. If set to `"error"` or `"hint"`, Strict-Ts will still output a hint informing you that an unsafe exemption occurred; in other modes, it will be handled silently.

### `"unreachableBranch"`

This option controls how Strict-Ts behaves when it discovers an unreachable code branch through type-flow analysis.

-   `"hint"`: **Default & Recommended.** Removes the unreachable branch while displaying a hint to the developer. This helps identify redundant defensive code or potential logic errors.
-   `"error"`: Treats the discovery of an unreachable branch as a compile-time error. This mode is suitable for projects aiming for the highest code quality, where any unreachable code is considered a bug to be fixed.
-   `"ignore"`: Silently removes the unreachable branch without generating any output. This option is for developers who do not care about such hints.

### `"unusedIdentifier"`

This option controls how to handle identifiers (local variables, functions, classes, un-exported top-level declarations, etc.) that are determined to be unused after global analysis.

This feature is similar to `tsc`'s `noUnusedLocals` or ESLint's `no-unused-vars`, but its analysis is based on Strict-Ts's global dependency graph, making it potentially more accurate than traditional single-file linters.

-   `"hint"`: **Default & Recommended.** Outputs a hint to the console for any unused identifier. The associated dead code (like the variable declaration itself) will be removed during the optimization phase.
-   `"error"`: Treats any unused identifier as a compile-time error, forcing developers to maintain clean code.
-   `"ignore"`: Silently removes dead code related to unused identifiers without generating any output.

## Example

### `Result` Structure

Below is a sample implementation of the `Result` type in Strict-Ts:

```ts
// Result is a union type representing success (OK) or failure (ERR).
type Result<T, E> = OK<T> | ERR<E>;

// Using a tuple type [T] facilitates value extraction via destructuring: `const [value] = result`.
// An intersection type `& { ok: boolean }` is used to attach a discriminant property for type guards.
type OK<T> = [T] & { ok: true };
type ERR<T> = [T] & { ok: false };

function ok<T>(value: T): OK<T> {
  return Object.assign<[T], { ok: true }>([value], { ok: true });
}
function err<T>(value: T): ERR<T> {
  return Object.assign<[T], { ok: false }>([value], { ok: false });
}

// ...

let result: Result<string, string> = getMessage();

if (result.ok) {
  // The value can be extracted directly using destructuring.
  const [message] = result;
  // ...
} else {
  const [error] = result;
  // ...
}
```

## Future Directions

### `try-catch` to `Result<T, E>` Transformation

As a potential future direction, Strict-Ts could offer a feature to automatically transform `try-catch` structures that follow specific conventions (e.g., Web APIs) into the `Result<T, E>` type at compile time. This feature would aim to lower the adoption cost of the Result pattern, but its implementation involves complex code analysis and is considered a long-term exploration goal.