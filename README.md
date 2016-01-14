# tree-sitter

[![Build Status](https://travis-ci.org/maxbrunsfeld/tree-sitter.png?branch=master)](https://travis-ci.org/maxbrunsfeld/tree-sitter)

Tree-sitter is a C library for incremental parsing, intended to be used via
[bindings](https://github.com/maxbrunsfeld/node-tree-sitter) to higher-level
languages. It can be used to build a concrete syntax tree for a program and
efficiently update the syntax tree as the program is edited. This makes it suitable
for use in text-editing programs.

Tree-sitter uses a sentential-form incremental [LR parsing](https://en.wikipedia.org/wiki/LR_parser)
algorithm, as described in the paper *[Efficient and Flexible Incremental Parsing](http://harmonia.cs.berkeley.edu/papers/twagner-parsing.ps.gz)*
by Tim Wagner. It handles ambiguity at compile-time via [precedence annotations](https://en.wikipedia.org/wiki/Operator-precedence_parser),
and at run-time via the [GLR algorithm](https://en.wikipedia.org/wiki/GLR_parser).
This allows it to generate a fast parser for any context-free grammar.

### Installation

```sh
script/configure.sh  # Generate a Makefile
make                 # Build static libraries for the compiler and runtime
```

### Overview

Tree-sitter consists of two libraries. The first library, `libcompiler`, can be
used to generate a parser for a language by supplying a [context-free grammar](https://en.wikipedia.org/wiki/Context-free_grammar) describing the
language. Once the parser has been generated, `libcompiler` is no longer needed.

The second library, `libruntime`, is used in combination with the parsers
generated by `libcompiler`, to generate syntax trees based on text documents, and keep the
syntax trees up-to-date as changes are made to the documents.


### Writing a grammar

Tree-sitter's grammars are specified as JSON strings. This format allows them
to be easily created and manipulated in high-level languages like [JavaScript](https://github.com/maxbrunsfeld/node-tree-sitter-compiler).
The structure of a grammar is formally specified by [this JSON schema](./doc/grammar-schema.json).
You can generate a parser for a grammar using the `ts_compile_grammar` function
provided by `libcompiler`.

Here's a simple example of using `ts_compile_grammar` to create a parser for basic
arithmetic expressions. It uses C++11 multi-line strings for readability.

```cpp
// arithmetic_grammar.cc

#include <stdio.h>
#include "tree_sitter/compiler.h"

int main() {
  TSCompileResult result = ts_compile_grammar(R"JSON(
    {
      "name": "arithmetic",

      "extras": [
        {"type": "PATTERN", "value": "\\s"},
      ],

      "rules": {
        "expression": {
          "type": "CHOICE",
          "members": [
            {"type": "SYMBOL", "name": "sum"},
            {"type": "SYMBOL", "name": "product"},
            {"type": "SYMBOL", "name": "number"},
            {"type": "SYMBOL", "name": "variable"},
            {
              "type": "SEQ",
              "members": [
                {"type": "STRING", "value": "("},
                {"type": "SYMBOL", "name": "expression"},
                {"type": "STRING", "value": ")"}
              ]
            }
          ]
        },

        "sum": {
          "type": "PREC_LEFT",
          "value": 1,
          "content": {
            "type": "SEQ",
            "members": [
              {"type": "SYMBOL", "name": "expression"},
              {"type": "STRING", "value": "+"},
              {"type": "SYMBOL", "name": "expression"}
            ]
          }
        },

        "product": {
          "type": "PREC_LEFT",
          "value": 2,
          "content": {
            "type": "SEQ",
            "members": [
              {"type": "SYMBOL", "name": "expression"},
              {"type": "STRING", "value": "*"},
              {"type": "SYMBOL", "name": "expression"}
            ]
          }
        },

        "number": {"type": "PATTERN", "value": "\\d+"}
        "variable": {"type": "PATTERN", "value": "[a-zA-Z]\\w*"},
      }
    }
  )JSON");

  if (result.error_type != TSCompileErrorTypeNone) {
    fprintf(stderr, "Compilation failed: %s\n", result.error_message);
    return 1;
  }

  puts(result.code);

  return 0;
}
```

To create the parser, compile this file like this:

```sh
clang++ -std=c++11 \
  -I tree-sitter/include \
  -L tree-sitter/out/Release \
  -l compiler \
  arithmetic_grammar.cc \
  -o arithmetic_grammar
```

Then run the executable to print out the C code for the parser:

```sh
./arithmetic_grammar > arithmetic_parser.c
```

### Using the parser

#### Providing the text to parse

Text input is provided to a tree-sitter parser via a `TSInput` struct, which
contains function pointers for seeking to positions in the text, and for reading
chunks of text. The text can be encoded in either UTF8 or UTF16. This interface
allows you to efficiently parse text that is stored in your own data structure.

#### Querying the syntax tree

The `libruntime` API provides a DOM-style interface for inspecting
syntax trees. Functions like `ts_node_child(node, index)` and `ts_node_next_sibling(node)`
expose every node in the concrete syntax tree. This is useful for operations
like syntax-highlighting, which operate on a token-by-token basis. You can also
traverse the tree in a more abstract way by using functions like
`ts_node_named_child(node, index)` and `ts_node_next_named_sibling(node)`. These
functions don't expose nodes that were specified in the grammar as anonymous
tokens, like `(` and `+`. This is useful when analyzing the meaning of a document.

```c
// test_parser.c

#include <assert.h>
#include <string.h>
#include <stdio.h>
#include "tree_sitter/runtime.h"

// Declare the language function that was generated from your grammar.
TSLanguage *ts_language_arithmetic();

int main() {
  TSDocument *document = ts_document_make();
  ts_document_set_language(document, ts_language_arithmetic());
  ts_document_set_input_string(document, "a + b * 5");
  ts_document_parse(document);

  TSNode root_node = ts_document_root_node(document);
  assert(!strcmp(ts_node_name(root_node, document), "expression"));
  assert(ts_node_named_child_count(root_node) == 1);

  TSNode sum_node = ts_node_named_child(root_node, 0);
  assert(!strcmp(ts_node_name(sum_node, document), "sum"));
  assert(ts_node_named_child_count(sum_node) == 2);

  TSNode product_node = ts_node_child(ts_node_named_child(sum_node, 1), 0);
  assert(!strcmp(ts_node_name(product_node, document), "product"));
  assert(ts_node_named_child_count(product_node) == 2);

  printf("Syntax tree: %s\n", ts_node_string(root_node, document));
  ts_document_free(document);
  return 0;
}
```

To demo this parser's capabilities, compile this program like this:

```sh
clang \
  -I tree-sitter/include \
  -L tree-sitter/out/Debug \
  -l runtime \
  arithmetic_parser.c test_parser.c \
  -o test_parser

./test_parser
```

### References

- [Context Aware Scanning for Parsing Extensible Languages](http://www.umsec.umn.edu/publications/Context-Aware-Scanning-Parsing-Extensible)
- [LR(1) Parser Generation System](http://arxiv.org/pdf/1010.1234.pdf)  (Error recovery scheme)
- [Efficient and Flexible Incremental Parsing](http://harmonia.cs.berkeley.edu/papers/twagner-parsing.ps.gz)
- [Incremental Analysis of Real Programming Languages](http://harmonia.cs.berkeley.edu/papers/twagner-glr.pdf)
