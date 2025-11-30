# Sigil – Language Goals (v0)

## Purpose

Sigil is a domain-focused DSL designed to be written by LLMs and read by humans.
It uses a restricted, English-like syntax and compiles to a target language
(TypeScript in v0).

Sigil is not intended to replace general-purpose languages. It is a layer
between natural-language prompts and production code, with the goal of:

* Making generated code easier to review and reason about.
* Keeping domain logic explicit and centralized.
* Enforcing simple, functional semantics (immutability, pure vs effectful functions).

## Non-goals (v0)

* General-purpose systems programming.
* UI rendering or complex front-end code.
* High-performance numerical computing.
* Multi-backend compilation (we only target TypeScript in v0).

## v0 Pilot Domain

For v0, Sigil will focus on a small backend domain:

**OrderService**:

* Entities: Customer, Order, Payment, Task.
* Operations: creating orders, updating order status, charging payments,
  sending notifications.

This domain is rich enough to exercise:

* pure business logic,
* persistence and external APIs (effects),
* simple data modelling,
* and end-to-end compilation from Sigil to TypeScript.

---

# Sigil-Core Syntax (v0)

Sigil-Core is the minimal subset of the language used to express computation
independent of any specific domain. It is intentionally small and regular so
that LLMs can emit it reliably and the compiler can analyze it easily.

All statements end with a period (`.`) unless otherwise specified.

## 1. Bindings

### 1.1 Immutable binding

Canonical form:

```sigil
Let <name> be <expression>.
```

Example:

```sigil
Let total be 0.
Let message be "Hello".
Let order_count be length of orders.
```

Bindings are immutable. Once bound, `<name>` cannot be re-assigned within the
same scope.

## 2. Functions

### 2.1 Pure function

Canonical form:

```sigil
Define function <name> taking <param1> as <Type1>, <param2> as <Type2> returning <ReturnType>:
  <statements>
End function.
```

Example:

```sigil
Define function compute_total taking prices as List of Money returning Money:
  Let total be 0.
  For each price in prices:
    Let total be total + price.
  Return total.
End function.
```

Pure functions may only call other pure functions.

### 2.2 Effectful function

Canonical form:

```sigil
Define effectful function <name> taking <params> returning <ReturnType>:
  <statements>
End function.
```

Effectful functions may call both pure and effectful functions.

Example (using domain operations):

```sigil
Define effectful function place_order taking request as CreateOrderRequest returning Order:
  Let order be validate_request of request.
  Call save_order with order.
  Call send_confirmation_email with order.
  Return order.
End function.
```

## 3. Conditionals

Canonical form:

```sigil
If <expression> then:
  <statements>
Otherwise:
  <statements>
End if.
```

The `Otherwise` block is optional.

Example:

```sigil
If order_status is "Pending" then:
  Return true.
Otherwise:
  Return false.
End if.
```

## 4. Loops

### 4.1 For each loop

Canonical form:

```sigil
For each <item> in <expression>:
  <statements>
End for.
```

Example:

```sigil
For each order in orders:
  Call archive_order with order.
End for.
```

## 5. Function calls

### 5.1 Call as statement

Canonical form:

```sigil
Call <name> with <arg1>, <arg2>.
```

Example:

```sigil
Call log_info with "Starting batch".
Call send_notification with customer, "Order shipped".
```

### 5.2 Call as expression

Canonical form:

```sigil
<name> of <arg1>, <arg2>
```

Example:

```sigil
Let total be compute_total of prices.
```

## 6. Return

Canonical form:

```sigil
Return <expression>.
```

Example:

```sigil
Return total.
```

---

# Type System and Effects (v0)

## 1. Types

Sigil uses a simple static type system in v0.

### 1.1 Base types

* `Boolean`
* `Number`
* `String`
* `DateTime`

### 1.2 Composite types

* `List of <Type>`: ordered collection of elements of `<Type>`.

### 1.3 Domain types

Domain-specific types are defined in domain modules, for example:

* `CustomerId`
* `OrderId`
* `Order`
* `Money`
* `CreateOrderRequest`

These are treated as nominal types at the Sigil level. Conversions between
domain types and base types must be explicit via domain functions.

## 2. Functions and purity

### 2.1 Pure functions

* Declared with `Define function`.
* May not perform side effects (IO, DB, HTTP, logging).
* May only call other pure functions.

### 2.2 Effectful functions

* Declared with `Define effectful function`.
* May perform side effects via domain-provided effectful operations.
* May call both pure and effectful functions.

The compiler enforces that:

* Pure functions do not call effectful functions.
* All effectful operations are explicitly declared in domain modules.

---

# OrderService Domain Module (v0)

The `OrderService` module defines domain types and operations for working with
customers, orders, and payments.

## 1. Types

```sigil
type CustomerId
type OrderId
type Money
type OrderStatus = "Pending" | "Confirmed" | "Cancelled" | "Completed"

record Order:
  id: OrderId
  customer_id: CustomerId
  total: Money
  status: OrderStatus
```

(Exact syntax for type declarations may be refined later; this document defines
the logical shape.)

## 2. Pure operations

Examples:

* `compute_total(prices: List of Money) -> Money`
* `can_cancel(order: Order) -> Boolean`

In Sigil:

```sigil
Define function can_cancel taking order as Order returning Boolean:
  If order.status is "Pending" then:
    Return true.
  Otherwise:
    Return false.
  End if.
End function.
```

## 3. Effectful operations

Examples:

* `save_order(order: Order) -> Order`
* `load_order(id: OrderId) -> Order`
* `send_confirmation_email(order: Order) -> Unit`

In Sigil:

```sigil
Define effectful function save_order taking order as Order returning Order:
  -- effectful logic defined in target language
End function.
```

In v0, effectful operations may be treated as external declarations that map
directly to TypeScript functions provided by the runtime or infrastructure.

---

# Scribe – Transpiler Architecture (v0)

## 1. Pipeline

For v0, Scribe implements a simple pipeline:

1. **Parse** Sigil source into an AST.
2. **Type-check** the AST using base types and domain type definitions.
3. **Effect-check** (ensure pure functions do not call effectful ones).
4. **Emit TypeScript** for Node.js runtime.

## 2. AST (high-level sketch)

The AST distinguishes expressions, statements, and declarations.

* Expressions: variables, literals, binary operations, function calls.
* Statements: bindings, conditionals, loops, returns, expression statements.
* Declarations: functions (pure and effectful), type and record definitions.

(Actual AST definitions will live in code; this is a conceptual outline.)

## 3. TypeScript emission

For each Sigil function:

```sigil
Define function compute_total taking prices as List of Money returning Money:
  ...
End function.
```

Scribe emits:

```ts
export const compute_total = (prices: Money[]): Money => {
  // ...
};
```

For each effectful function:

```sigil
Define effectful function place_order taking request as CreateOrderRequest returning Order:
  ...
End function.
```

Scribe emits:

```ts
export const place_order = async (
  request: CreateOrderRequest
): Promise<Order> => {
  // ...
};
```

(We may choose not to use `async` for all effectful functions initially, but
this is a reasonable default.)

---

# LLM Integration (v0)

## 1. Generation loop

The intended usage pattern is:

1. User provides a natural-language request for behaviour in the pilot domain.
2. LLM generates Sigil code using the Sigil-Core and domain syntax.
3. Scribe parses and compiles the Sigil code to TypeScript.
4. Tests or runtime checks are executed against the generated code.
5. Any compile-time or runtime errors are fed back to the LLM using a
   structured diagnostics format for auto-repair.

## 2. Diagnostics format

Scribe should return machine-readable diagnostics, for example:

```json
{
  "errors": [
    {
      "code": "SIGIL_TYPE_MISMATCH",
      "message": "Function 'create_order' expects 'CustomerId' but got 'String'.",
      "location": {
        "line": 12,
        "column": 5
      },
      "hint": "Wrap the string in 'to_customer_id(...)' or adjust the parameter type."
    }
  ]
}
```

The LLM is expected to:

* Read the diagnostics.
* Update the Sigil source.
* Re-submit the updated code to Scribe.

## 3. Canonical style

The LLM should be trained (or strongly prompted) to emit Sigil in a canonical
form:

* Always use the `Let <name> be <expression>.` binding pattern.
* Always use the `Define function ...` and `Define effectful function ...`
  patterns for declarations.
* Always use the `If ... then: ... Otherwise: ... End if.` pattern for
  conditionals.
* Always use `Call <name> with ...` for effectful calls as statements.

This canonical style reduces ambiguity and makes static analysis and repair
simpler.
