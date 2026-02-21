# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is **tree-sitter-cpp**, a C++ grammar for [tree-sitter](https://tree-sitter.github.io/tree-sitter/), the incremental parsing framework. It generates a C parser from a grammar DSL and ships multi-language bindings (Node.js, Rust, Python, Go, Swift, C).

The grammar extends `tree-sitter-c` and adds C++-specific constructs including C++20 modules/concepts and C++26 reflection, annotations, and expansion statements.

## Key Commands

### Grammar Development

```bash
# After editing grammar.js, regenerate the parser
tree-sitter generate src/grammar.json

# Run all corpus tests
tree-sitter test

# Run tests matching a pattern
tree-sitter test -f "pattern"

# Interactive playground (builds WASM first)
npm run prestart && npm start
```

### Running Tests

```bash
# Corpus tests (primary test suite)
tree-sitter test

# Node.js binding tests
npm test

# Rust binding tests
cargo test

# Parse a specific file to verify output
tree-sitter parse path/to/file.cpp
```

### Linting

```bash
npm run lint    # ESLint on grammar.js
```

### Building

```bash
npm install            # Build Node.js native module
make                   # Build static/shared C library
cmake -B build && cmake --build build   # CMake build
```

## Architecture

### Code Generation Pipeline

```
grammar.js  (hand-written)
    ↓  tree-sitter generate src/grammar.json
src/grammar.json  (intermediate, committed)
    ↓  tree-sitter generate
src/parser.c + src/node-types.json  (auto-generated, committed)
```

**Never edit `src/parser.c` or `src/node-types.json` directly** — they are regenerated from `grammar.js`. After any grammar change, run `tree-sitter generate src/grammar.json` and commit all generated files (the project uses a `chore: generate` commit convention).

### Key Files

- **`grammar.js`** — The grammar definition (~1600 lines). Extends `tree-sitter-c`'s grammar via `module.exports` and adds C++-specific rules. Precedence constants are defined in a `PREC` object that extends the base C `PREC`.
- **`src/scanner.c`** — Hand-written external scanner (~150 lines). Handles raw string literals (`R"delim(...)delim"`) which require stateful, non-context-free parsing.
- **`queries/highlights.scm`** — Syntax highlighting rules (extends tree-sitter-c highlights).
- **`queries/injections.scm`** — Language injection rules.
- **`queries/tags.scm`** — Symbol definitions for code navigation.

### Test Corpus

Tests live in `test/corpus/*.txt` in tree-sitter's corpus format:

```
===============================
Test Name
===============================
source code here
---
(expected_ast_node
  (child_node))
```

Relevant test files by topic: `declarations.txt`, `expressions.txt`, `statements.txt`, `concepts.txt`, `modules.txt`, `reflection.txt`, `types.txt`, `definitions.txt`, `ambiguities.txt`, `microsoft.txt`.

### Bindings

Each subdirectory of `bindings/` is a self-contained binding for that language. The parser itself (`parser.c` + `scanner.c`) is shared across all of them via the build system.

## Grammar Conventions

- C++-specific precedence levels extend the base `C.PREC` object.
- Grammar conflicts are resolved with explicit `conflicts:` declarations and `prec`/`prec.left`/`prec.right` annotations in rules.
- Microsoft-specific extensions (e.g., `__declspec`, `__cdecl`) are handled separately and tested in `test/corpus/microsoft.txt`.
- New C++ standard features reference their proposal numbers in commit messages (e.g., P2996 for reflection, P1306 for expansion statements, P3394 for annotations).
