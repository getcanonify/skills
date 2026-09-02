# The boolean wire-encoding contract

Canonify stores boolean-valued columns as SQLite INTEGER 0/1. This contract defines which values the platform accepts as "boolean" at each layer, and how to resolve type mismatches when you retype a numeric column to `boolean`.

## Values accepted at each boundary

### Handler `set`/`values` steps (type-check time)

The handler DSL type-checker accepts these values when writing to a boolean-mapped column:

- **Literal 0 or 1**: `set: { flag: 0 }` or `set: { flag: 1 }`
  - These are the canonical false/true encoding—the same 0/1 SQLite stores.
  - Always safe. Use this pattern for handler-level flags.
- **$ref resolving to boolean**: `set: { flag: $input.is_approved }`
  - The ref must resolve to a property typed `boolean`. A ref resolving to `number` is rejected with a type mismatch, even if you know the value is 0/1 at runtime.
  - This strict rule prevents silent coercion of unexpected values (e.g., a column holding 2, -1, or any number > 1).
- **Null (for nullable columns)**: `set: { flag: null }`

**Rejects:** Any other `$ref` to a numeric column, numeric expressions (e.g., `$x.count + 1`), or string values.

### Auto-CRUD action inputs

When you invoke an action via the REST, CLI, or MCP surfaces, the input schemas generated for auto-CRUD actions expect boolean values to be one of:

- `true` / `false` (boolean literals)
- `'true'` / `'false'` (string forms)
- `1` / `0` (numeric forms, once ticket kmp lands)

All are coerced to the native boolean type before validation.

**Rejects:** strings like `"yes"`, `"1"`, numeric values outside [0,1], or `null` (for non-nullable fields).

### Custom action inputs (declarative)

Custom action input schemas you author follow the same rules as auto-CRUD:

```json
{
  "at_risk": { "type": "boolean" }
}
```

Accepts `true`, `false`, `'true'`, `'false'`, and (post-kmp) `0`, `1`.

### Catalog bundle structure

When authoring a bundle (JSON or programmatic):

```json
{
  "object_types": [
    {
      "name": "customer",
      "properties": {
        "at_risk": { "type": "boolean", "column": "at_risk" }
      }
    }
  ],
  "actions": [
    {
      "name": "customer.import",
      "input": { "at_risk": { "type": "boolean" } }
    }
  ]
}
```

## When you retype a property from number to boolean

The platform nudges you to retype numeric columns with boolean-looking names—`is_`, `has_`, `can_`, `at_risk`, `flag`, `active`—toward the `boolean` type (ticket as7 elegance rule). This improves UI rendering (showing "Yes/No" instead of "0/1") and clarity in read results.

**If you follow that nudge:**

1. **In the ObjectType:** Change `"type": "number"` to `"type": "boolean"`
   ```json
   // Before
   "at_risk": { "type": "number", "column": "at_risk" }
   // After
   "at_risk": { "type": "boolean", "column": "at_risk" }
   ```

2. **In any action input schema referencing that property:**  
   Also change `"type": "number"` to `"type": "boolean"`:
   ```json
   // Before
   "customer.import": { "input": { "at_risk": { "type": "number" } } }
   // After
   "customer.import": { "input": { "at_risk": { "type": "boolean" } } }
   ```

   If you forget this step, the next `catalog apply` will fail with:
   ```
   handler type-check failed: … $input.at_risk resolves to 'number', but customer.at_risk is declared 'boolean' …
   ```

   **Why?** Handler type-checking is strict: a property typed boolean in storage must be fed a boolean or boolean-literal from the action input. A `$ref` to a numeric column is not allowed, because 0/1 literals inside your handler are safe (they're 0/1), but a $ref to a field that *might hold* 2 or -1 is not.

3. **In handler `set`/`values` clauses:**  
   If you were writing literal 0/1 directly (e.g., `set: { flag: 0 }`), no change needed—literals stay accepted.
   If you were referencing the property from prior steps (`set: { flag: $customer.at_risk }`), you now *must* ensure it's been retyped to boolean. The old rule (numeric $ref) no longer applies.

## Troubleshooting: "resolves to 'number', but X is declared 'boolean'"

This error means an action's input schema declares a field as one type, but the corresponding ObjectType property is declared as a different type.

**Common case:** You retyped a property from number to boolean, but forgot to update an action that references it.

**Fix:** Find all actions whose input schemas mention this property name, and update them to match the property's new type.

**To find affected actions:**

```bash
# List all actions that mention the property (e.g., "at_risk")
canon actions list --filter 'input.*at_risk'

# For each action, check its input schema
canon actions describe customer.import
```

Then update the action's input schema in your bundle and re-apply.

## Storage: how 0/1 round-trip

When you declare a column `type: 'boolean'`:

- **On write:** Canonify accepts 0, 1, true, false, '0', '1', 'true', 'false' as input and stores INTEGER 0 or 1.
- **On read:** Reads return the native JavaScript boolean (`true` or `false`), not the stored integer.
- **In the UI:** Detail and list views render as "Yes" / "No" or a toggle, not "1" / "0".

The storage is durable integers; the wire protocol is typed booleans.

## See also

- [Declaring actions](./declaring-actions.md) — action input schemas and type matching
- [Declaring ObjectTypes](./declaring-objecttypes.md) — property types and mapping
- Handler DSL type-checking: `packages/canon/src/handler-dsl/typecheck.ts`
