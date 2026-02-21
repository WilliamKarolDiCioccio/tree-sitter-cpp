# Extending tree-sitter-cpp

Reference procedure for adding or modifying grammar nodes. Based on commits `cacfb40` (expansion statements), `916c7dc` (typeid), and `b86280a` (pack-expansion fix).

---

## Files involved

| File                     | Role                                             | Edit?           |
| ------------------------ | ------------------------------------------------ | --------------- |
| `grammar.js`             | Source of truth — hand-written grammar rules     | YES             |
| `queries/highlights.scm` | Syntax highlighting; add new keywords here       | If new keywords |
| `test/corpus/*.txt`      | Corpus tests                                     | YES — always    |
| `src/grammar.json`       | Generated intermediate; **never edit directly**  | NO              |
| `src/node-types.json`    | Generated type metadata; **never edit directly** | NO              |
| `src/parser.c`           | Generated C parser; **never edit directly**      | NO              |

---

## Procedure

### 1. Identify placement in `grammar.js`

- New **expression** → add to `_expression_not_binary` (or a narrower parent if placement would cause GLR ambiguity — see §Pitfalls).
- New **statement** → add to `statement` and any secondary statement-lists (e.g. `_block_item`).
- New **type** or **declaration** → add to the appropriate `_type_specifier` / `declaration` choice.
- Reuse existing `PREC` constants for precedence; extend the `PREC` object only when genuinely needed.

### 2. Write the rule in `grammar.js`

Minimal shape for a new node:

```js
my_node: $ => prec(PREC.SOME_LEVEL, seq(
  'keyword',
  '(',
  choice(
    field('type', $.type_descriptor),
    field('value', $.expression),
  ),
  ')',
)),
```

- Use `field(name, rule)` for named children that consumers (codex, queries) will key off.
- Use `prec.left` / `prec.right` when associativity matters.
- Override an inherited rule with the `($, original) => choice(original, ...)` pattern.

### 3. Update `queries/highlights.scm` (if adding keywords)

Add the literal keyword string to the `@keyword` capture group:

```scheme
[
  ; ... existing keywords ...
  "mynewkeyword"
] @keyword
```

### 4. Write corpus tests

Pick the most relevant file under `test/corpus/` (e.g. `expressions.txt`, `statements.txt`, `declarations.txt`) and append a block:

```
================================================================================
My new node — basic cases
================================================================================

mynewkeyword(int);
mynewkeyword(*ptr);

--------------------------------------------------------------------------------

(translation_unit
  (expression_statement
    (my_node
      type: (type_descriptor
        type: (primitive_type))))
  (expression_statement
    (my_node
      value: (pointer_expression
        argument: (identifier)))))
```

Rules for corpus tests:

- Cover the **primary** usage (keyword + type arg, keyword + expr arg, etc.).
- Cover at least one **edge case** (pointer operand, nested in binary expression, etc.).
- When fixing a bug, add a **regression test** that would have failed before the fix.
- Field names must match those declared in the rule (`field('type', ...)` → `type:` in the AST).

### 5. Regenerate the parser

```bash
tree-sitter generate grammar.js
```

This rewrites `src/grammar.json`, `src/node-types.json`, and `src/parser.c`.

### 6. Run tests

```bash
# Full corpus suite
tree-sitter test

# Targeted (faster during iteration)
tree-sitter test -f "My new node"
```

All tests must pass before committing.

### 7. Commit

Use **two commits**:

```
feat: <concise description>          ← grammar.js + queries + test corpus
chore: generate src/grammar.json, src/node-types.json, src/parser.c
```

For bug fixes use `fix:` instead of `feat:`. The fix commit message body should describe _why_ the original rule was wrong (see `b86280a` for a good example). Reference the C++ proposal number in the message when relevant (e.g. `P1306`, `P2996`).

---

## Modifying existing nodes

Same procedure as above but instead of adding a new rule:

- Adjust the existing rule body in `grammar.js`.
- If the change shifts which parent rules reference the node, update those too.
- Update any affected corpus test expected ASTs.
- Add a regression test for any bug being fixed.
- Regenerate and commit using the two-commit convention.

---

## Pitfalls

**GLR ambiguity from placement in `_expression_not_binary`**

Placing a recursive construct (e.g. `parameter_pack_expansion`, which wraps `expression`) inside `_expression_not_binary` creates the cycle `expression → _expression_not_binary → <node> → expression`. Under GLR merging this causes spurious wrapper nodes to appear in the parse tree. Fix: remove from the broad expression alternative and add explicitly only at legal call-sites (`argument_list`, `initializer_list`, etc.). See `b86280a`.

**Precedence conflicts with `sizeof` / `typeid` / `alignof` family**

These keyword-expressions sit at `PREC.SIZEOF`. New keyword-expressions in the same syntactic class should use the same level.

**Corpus indentation is load-bearing**

The tree-sitter test runner is whitespace-sensitive. Child nodes must be indented exactly two spaces per nesting level relative to their parent. Misaligned expected output causes false failures.

**Never commit generated files in the same commit as grammar.js**

Keep hand-written changes and generated output in separate commits so `git bisect` and code review remain useful.
