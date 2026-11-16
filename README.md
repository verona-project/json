# Verona JSON

A UTF-8 JSON event parser and string serializer written in Verona, using Base
and Foundation byte/UTF-8 utilities. Parsing allocates no memory.

## Build and test

From this package directory, using the repository toolchain:

```sh
../../build/verona build json ../../build/formats
../../build/verona build json-tests ../../build/formats
../../build/formats/json-tests
```

Consumers add `src`, `../base/src`, and `../foundation/src` to their module
paths and import `json`. The static archive is a build artifact; Verona
consumers use source modules. Tests also use `../testing/src`.

## Shared contracts

- `result` contains `status: error:status` and `offset: u64`. Success is `(ok)`
  with offset equal to the input byte length. Syntax errors are `(invalid-input)`
  with a zero-based byte offset (EOF may equal length). Invalid UTF-8 is detected
  before parsing and reports offset zero; INI's NUL precheck also reports zero.
- Input is a borrowed `data:view`, not a NUL-terminated string. The caller keeps
  its bytes alive and unchanged throughout parsing and callbacks.
- `visitor` carries an opaque `(pointer void)` context and a typed callback.
  There is no global parser state. Callbacks return `unit`; cancellation is not
  supported. Events already delivered are not rolled back on a later error.
- Writers append to an initialized `data:data`, using its chosen allocator.
  They validate input and reserve a checked worst-case capacity before appending,
  so validation, overflow, or allocation failure preserves existing output.
  Source bytes must not alias output storage. Check every returned status.
- No implicit file I/O, document ownership, or reference counting. Arena reset
  or destruction invalidates all associated buffers. The algorithms use
  recursion; unbounded input lengths are not a stack-safety guarantee.

## API

| Export | Contract |
| --- | --- |
| `token` | Sum: `null`, `boolean`, `number`, `string`, `key`, `begin-array`, `end-array`, `begin-object`, `end-object` |
| `visitor(context, emit)` | `emit(context, token, data:view) -> unit` |
| `parse(source, visitor)` | Validate exactly one document and emit tokens in source order; returns `result` |
| `write-string(source, out)` | Append a quoted JSON string from unescaped UTF-8 bytes; returns `error:status` |

Token views borrow the original source and remain valid for its lifetime.
String/key tokens include their quotes and preserve escapes. Number tokens
preserve the complete decimal lexeme; no floating-point conversion or numeric
range restriction is imposed. Container events carry the single bracket or brace.
Commas and colons produce no events. Boolean tokens carry `true` or `false`.

The parser checks JSON whitespace, number grammar, escapes, complete consumption,
UTF-8, paired UTF-16 surrogate escapes, and nesting up to 128 containers. Duplicate
object keys are emitted in order. Lone surrogate escapes, comments, trailing
commas, BOM prefixes, NaN and Infinity are rejected.

This initial API is streaming over a complete in-memory view. It does not build
a DOM, decode escaped string tokens, or serialize an arbitrary value tree.
`write-string` quotes/backslash-escapes bytes and encodes controls as `\u00xx`.
Other valid UTF-8 bytes pass through unchanged.

## Example

```lisp
(import base)
(import json)
(import data)

(%function on-token ((context (pointer void)) (kind json:token) (source data:view)) unit
  ; Consume a borrowed token here.
  unit)

(%function inspect-json () json:result
  (json:parse (data:view "[1,true]" 8)
    (json:visitor (cast (pointer void) "") on-token)))
```

The tests cover every token kind, nesting, number syntax, Unicode/surrogates,
malformed input, exact error offsets, arena output, and allocation failure.
