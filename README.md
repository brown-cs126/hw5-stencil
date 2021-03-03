# Homework 5: Fun with files

In this homework, you'll implement string support and file handling. You'll get
more practice dealing with data on the heap and learn how to extend your
compiler's functionality via the C runtime.

## Strings

Implement support for strings. For now, this just means adding support for
string literals (i.e., s-expressions built with the `Str` constructor). String
literals are sequences of characters enclosed in double-quotes. Strings should
be displayed in the same double-quote-enclosed representation.

### Escaping

Strings may contain characters with special meaning (namely, newlines and
double-quotes). When displaying strings, these should be escaped as `\n` and
`\"` to ensure the resulting expression is well-formed. For instance, a string
containing only a double-quote should be displayed as `"\""` and not `"""`.

You do *not* need to support special characters besides quote and newline.

### Interpreter

Add string support to the interpreter by adding a constructor to `value` and
extending the interpreter's `display_value` function to display strings (use
`String.escaped` to escape special characters).

### Compiler

In the compiler, we'll represent strings like C does: nul-terminated sequences
of characters. The runtime value for a string should be a pointer to the first
character of the sequence, tagged with `0b011`.

Since we're only concerned with string literals for now, you can implement
string support using `DqString`, which will embed a string literal into the
compiled program as data. As with `DqLabel`, you should be sure that program
execution never reaches this directive--it's just data, not instructions.
Instead, you should get the address of the embedded string and use it to create
a runtime representation of the string.

String literals embedded in this way must be placed at 8-byte aligned addresses,
since you will need to tag the pointers with `0b011`.  You can use the `Align`
directive to ensure the next directive will be aligned properly.

Extend the runtime with support for displaying strings. In order to properly
escape newlines and double-quotes, implement this as a loop over the string's
characters, and check if each needs to be escaped.

## File I/O using channels

You're going to add a miniature version of OCaml's channel-based I/O to your
language. First, you'll need to add support for channel types. Channels can be
either input channels or output channels. Input channels can be read from;
output channels can be written to.

The symbols `stdin` and `stdout` refer to the program's input and output.
`stdin` is an input channel; `stdout` is an output channel.

-   In the interpreter, implement input channels with `in_channel` and output
    channels with `out_channel`. In the top-level environment, the symbols
    `stdin` and `stdout` should be initialized to `!input_channel` and
    `!output_channel`, respectively (this is necessary to make tests work).
-   In the compiler, implement both types of channels with the same mask,
    `0b111111111`. Input channels should have tag `0b011111111`; output channels
    should have tag `0b001111111`. `stdin` should be represented at runtime as
    `0` shifted left and tagged with the input-channel tag; `stdout` should be
    represented as `1` shifted left and tagged with the output channel tag.

Channels should be printed as `<in-channel>` or `<out-channel>`.

Now you'll implement some operations to open, close, and read and write via
channels.

-   `(open-in filename)` takes in a string representing a filename and opens it
    as an input channel, returning the channel
-   `(open-out filename)` takes in a string representing a filename and opens it
    as an output channel, returning the channel
-   `(close-in ch)` takes in an input channel and closes it, returning `true`
-   `(close-out ch)` takes in an output channel and closes it, returning `true`
-   `(input ch n)` reads `n` bytes from input channel `ch`, returning a string
    of those bytes (plus a null terminator)
-   `(output ch s)` writes the (null-terminated) string `s` out to output
    channel `ch`, returning `true`
    
In the interpreter, these should be implemented using OCaml's built-in functions:

- `open-in` should be implemented using `open_in`
- `open-out` should be implemented using `open_out`
- `close-in` should be implemented using `close_in`
- `close-out` should be implemented using `close_out`
- `input` should be implemented using `input` (you can use something like
  `Bytes.make n '\000'` to create a buffer of the appropriate size)
- `output` should be implemented using `output_string`

In the compiler, all of these operations will be implemented with C functions
that you add to the runtime. You should define a C function for each operation
(except for `close-in` and `close-out`, which should be implemented identically
in the runtime and can use the same C function).

-   Your functions for `open-in` and `open-out` should use the
    `open_for_reading` and `open_for_writing` helper functions we have provided
    in `runtime.c`. Both functions return a file descriptor as an integer. Turn
    this file descriptor into a channel by shifting it and tagging it with the
    correct channel tag.
-   Your function for `close-in` and `close-out` should use the `close` C
    function, passing in the file descriptor obtained from `open`. Input and
    output channels can be handled exactly the same.
-   Your function for `input` should use the `read_all` helper function we've
    provided in `runtime.c`; this helper function takes care of reading the
    correct number of bytes, adding a null terminator, and handling
    errors. `read_all` reads into a buffer; as this buffer, you'll need to pass
    in a pointer to the Lisp heap (which means you'll have to pass the heap
    pointer as an argument to your function). After you call `read_all` you'll
    need to adjust your heap pointer to make sure that subsequent allocations
    don't overwrite this string; you'll also need to make sure that the heap
    pointer remains a multiple of 8. We recommend doing this by returning the
    heap adjustment from your C function as an integer (taking care to make sure
    it's a multiple of 8), but there are other ways of doing it.
-   `output` should be implemented using the `write_all` helper function we have
    provided in `runtime.c`.

## Testing

This assignment adds a couple of testing features.

### Providing program input

Your programs can now read from standard input, using both the `read-num`
operator defined in class and the `input` operator you'll write on this
assignment. For testing, you should provide inputs by writing `.in` files for
each `.lisp` file that expects an input. For example, if we defined a program
like this in `examples/read-num.lisp`:

```
(print (pair (read-num) (pair (read-num) (read-num))))
```

We could define its input in `examples/read-num.in`:

```
8
13
21
```

The testing system will provide this as the input to both the interpreter and the compiler.

Additionally, for tests involving relatively small programs and inputs, the testing system
will handle rows in `examples/examples.csv` of the form

```csv
<PROGRAM>,<INPUT1>,<OUTPUT1>,<INPUT2>,<OUTPUT2>,...
```

For instance, to test a program that echoes single characters read from stdin to stdout:

```csv
(output stdout (input stdin 1)), a, a, b, b
```

### The scratch directory

When testing reading/writing to files, it can be easy for state to get mixed up between
tests (e.g. one test creates a file and writes some things to it, while the test that runs
after it expects the file not to exist). We've set up a directory `/tmp/csci1260` that the
testing harness will clear out before it runs each test. To testing reading/writing files,
use files inside `/tmp/csci1260`. This will work both locally and on Gradescope.

## Grammar

The grammar your new interpreter and compiler should support is as follows:

```diff
<expr> ::= <num>
         | <id>
+        | <string>
         | true
         | false
+        | stdin
+        | stdout
         | ()
         | (<z-prim>)
         | (<un_prim> <expr>)
         | (<bin_prim> <expr> <expr>)
         | (if <expr> <expr> <expr>)
         | (let ((<id> <expr>)) <expr>)
         | (do <expr> <expr> ...)

<z-prim> ::= read-num | newline

<un_prim> ::= add1
            | sub1
            | zero?
            | num?
            | not
            | pair?
            | left
            | right
            | print
+           | open-in
+           | open-out
+           | close-in
+           | close-out

<bin_prim> ::= +
             | -
             | =
             | <
             | pair
+            | input
+            | output

```
