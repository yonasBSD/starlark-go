# Starlark in Go: Language definition

Starlark is a dialect of Python intended for use as a configuration
language.  A Starlark interpreter is typically embedded within a larger
application, and this application may define additional
domain-specific functions and data types beyond those provided by the
core language.  For example, Starlark is embedded within (and was
originally developed for) the [Bazel build tool](https://bazel.build),
and [Bazel's build language](https://docs.bazel.build/versions/main/skylark/language.html) is based on Starlark.

This document describes the Go implementation of Starlark
at go.starlark.net/starlark.
The language it defines is similar but not identical to
[the Java-based implementation](https://github.com/bazelbuild/bazel/blob/master/src/main/java/net/starlark/java/eval/Starlark.java)
used by Bazel.
We identify places where their behaviors differ, and an
[appendix](#dialect-differences) provides a summary of those
differences.
We plan to converge both implementations on a single specification.

This document is maintained by Alan Donovan <adonovan@google.com>.
It was influenced by the Python specification,
Copyright 1990&ndash;2017, Python Software Foundation,
and the Go specification, Copyright 2009&ndash;2017, The Go Authors.

Starlark was designed and implemented in Java by Laurent Le Brun,
Dmitry Lomov, Jon Brandvin, and Damien Martin-Guillerez, standing on
the shoulders of the Python community.
The Go implementation was written by Alan Donovan and Jay Conrod;
its scanner was derived from one written by Russ Cox.

## Overview

Starlark is an untyped dynamic language with high-level data types,
first-class functions with lexical scope, and automatic memory
management or _garbage collection_.

Starlark is strongly influenced by Python, and is almost a subset of
that language.  In particular, its data types and syntax for
statements and expressions will be very familiar to any Python
programmer.
However, Starlark is intended not for writing applications but for
expressing configuration: its programs are short-lived and have no
external side effects and their main result is structured data or side
effects on the host application.
As a result, Starlark has no need for classes, exceptions, reflection,
concurrency, and other such features of Python.

Starlark execution is _deterministic_: all functions and operators
in the core language produce the same execution each time the program
is run; there are no sources of random numbers, clocks, or unspecified
iterators. This makes Starlark suitable for use in applications where
reproducibility is paramount, such as build tools.

## Lexical elements

A Starlark program consists of one or more modules.
Each module is defined by a single UTF-8-encoded text file.

A complete grammar of Starlark can be found in [grammar.txt](../syntax/grammar.txt).
That grammar is presented piecemeal throughout this document
in boxes such as this one, which explains the notation:

```grammar {.good}
Grammar notation

- lowercase and 'quoted' items are lexical tokens.
- Capitalized names denote grammar productions.
- (...) implies grouping.
- x | y means either x or y.
- [x] means x is optional.
- {x} means x is repeated zero or more times.
- The end of each declaration is marked with a period.
```

The contents of a Starlark file are broken into a sequence of tokens of
five kinds: white space, punctuation, keywords, identifiers, and literals.
Each token is formed from the longest sequence of characters that
would form a valid token of each kind.

```grammar {.good}
File = {Statement | newline} eof .
```

*White space* consists of spaces (U+0020), tabs (U+0009), carriage
returns (U+000D), and newlines (U+000A).  Within a line, white space
has no effect other than to delimit the previous token, but newlines,
and spaces at the start of a line, are significant tokens.

*Comments*: A hash character (`#`) appearing outside of a string
literal marks the start of a comment; the comment extends to the end
of the line, not including the newline character.
Comments are treated like other white space.

*Punctuation*: The following punctuation characters or sequences of
characters are tokens:

```text
+    -    *    /    //   %    =
+=   -=   *=   /=   //=  %=   ==   !=
^    <    >    <<   >>   &    |
^=   <=   >=   <<=  >>=  &=   |=
.    ,    ;    :    ~    **
(    )    [    ]    {    }
```

*Keywords*: The following tokens are keywords and may not be used as
identifiers:

```text
and            elif           in             or
break          else           lambda         pass
continue       for            load           return
def            if             not            while
```

The tokens below also may not be used as identifiers although they do not
appear in the grammar; they are reserved as possible future keywords:

<!-- and to remain a syntactic subset of Python -->

```text
as              except          nonlocal
assert          finally         raise
async           from            try
await           global          with
class           import          yield
del             is   
```
<!-- NB: bazelbuild/starlark puts `while` in the second list -->

<b>Implementation note:</b>
The Go implementation permits `assert` to be used as an identifier,
and this feature is widely used in its tests.

*Identifiers*: an identifier is a sequence of Unicode letters, decimal
 digits, and underscores (`_`), not starting with a digit.
Identifiers are used as names for values.

Examples:

```text
None    True    len
x       index   starts_with     arg0
```

*Literals*: literals are tokens that denote specific values.  Starlark
has string, integer, and floating-point literals.

```text
0                               # int
123                             # decimal int
0x7f                            # hexadecimal int
0o755                           # octal int
0b1011                          # binary int

0.0     0.       .0             # float
1e10    1e+10    1e-10
1.1e10  1.1e+10  1.1e-10

"hello"      'hello'            # string
'''hello'''  """hello"""        # triple-quoted string
r'hello'     r"hello"           # raw string literal
```

Integer and floating-point literal tokens are defined by the following grammar:

```grammar {.good}
int         = decimal_lit | octal_lit | hex_lit | binary_lit .
decimal_lit = ('1' … '9') {decimal_digit} | '0' .
octal_lit   = '0' ('o'|'O') octal_digit {octal_digit} .
hex_lit     = '0' ('x'|'X') hex_digit {hex_digit} .
binary_lit  = '0' ('b'|'B') binary_digit {binary_digit} .

float     = decimals '.' [decimals] [exponent]
          | decimals exponent
          | '.' decimals [exponent]
          .
decimals  = decimal_digit {decimal_digit} .
exponent  = ('e'|'E') ['+'|'-'] decimals .

decimal_digit = '0' … '9' .
octal_digit   = '0' … '7' .
hex_digit     = '0' … '9' | 'A' … 'F' | 'a' … 'f' .
binary_digit  = '0' | '1' .
```

### String literals

A Starlark string literal denotes a string value. 
In its simplest form, it consists of the desired text 
surrounded by matching single- or double-quotation marks:

```python
"abc"
'abc'
```

Literal occurrences of the chosen quotation mark character must be
escaped by a preceding backslash. So, if a string contains several
of one kind of quotation mark, it may be convenient to quote the string
using the other kind, as in these examples:

```python
'Have you read "To Kill a Mockingbird?"'
"Yes, it's a classic."

"Have you read \"To Kill a Mockingbird?\""
'Yes, it\'s a classic.'
```

Literal occurrences of the _opposite_ kind of quotation mark, such as
an apostrophe within a double-quoted string literal, may be escaped
by a backslash, but this is not necessary: `"it's"` and `"it\'s"` are
equivalent.


#### String escapes

Within a string literal, the backslash character `\` indicates the
start of an _escape sequence_, a notation for expressing things that
are impossible or awkward to write directly.

The following *traditional escape sequences* represent the ASCII control
codes 7-13:

```
\a   \x07 alert or bell
\b   \x08 backspace
\f   \x0C form feed
\n   \x0A line feed
\r   \x0D carriage return
\t   \x09 horizontal tab
\v   \x0B vertical tab
```

A *literal backslash* is written using the escape `\\`.

An *escaped newline*---that is, a backslash at the end of a line---is ignored,
allowing a long string to be split across multiple lines of the source file.

```python
"abc\
def"			# "abcdef"
```

An *octal escape* encodes a single byte using its octal value.
It consists of a backslash followed by one, two, or three octal digits [0-7].
It is error if the value is greater than decimal 255.

```python
'\0'			# "\x00"  a string containing a single NUL byte
'\12'			# "\n"    octal 12 = decimal 10
'\101-\132'		# "A-Z"
'\119'			# "\t9"   = "\11" + "9"
```

<b>Implementation note:</b>
The Java implementation encodes strings using UTF-16,
so an octal escape encodes a single UTF-16 code unit.
Octal escapes for values above 127 are therefore not portable across implementations.
There is little reason to use octal escapes in new code.

A *hex escape* encodes a single byte using its hexadecimal value.
It consists of `\x` followed by exactly two hexadecimal digits [0-9A-Fa-f].

```python
"\x00"			# "\x00"  a string containing a single NUL byte
"(\x20)"		# "( )"   ASCII 0x20 = 32 = space

red, reset = "\x1b[31m", "\x1b[0m"	# ANSI terminal control codes for color
"(" + red + "hello" + reset + ")"	# "(hello)" with red text, if on a terminal
```

<b>Implementation note:</b>
The Java implementation does not support hex escapes.

An ordinary string literal may not contain an unescaped newline,
but a *multiline string literal* may spread over multiple source lines.
It is denoted using three quotation marks at start and end.
Within it, unescaped newlines and quotation marks (or even pairs of
quotation marks) have their literal meaning, but three quotation marks
end the literal. This makes it easy to quote large blocks of text with
few escapes.

```
haiku = '''
Yesterday it worked.
Today it is not working.
That's computers. Sigh.
'''
```

Regardless of the platform's convention for text line endings---for
example, a linefeed (\n) on UNIX, or a carriage return followed by a
linefeed (\r\n) on Microsoft Windows---an unescaped line ending in a
multiline string literal always denotes a line feed (\n).

Starlark also supports *raw string literals*, which look like an
ordinary single- or double-quotation preceded by `r`. Within a raw
string literal, there is no special processing of backslash escapes,
other than an escaped quotation mark (which denotes a literal
quotation mark), or an escaped newline (which denotes a backslash
followed by a newline). This form of quotation is typically used when
writing strings that contain many quotation marks or backslashes (such
as regular expressions or shell commands) to reduce the burden of
escaping:

```python
"a\nb"		# "a\nb"  = 'a' + '\n' + 'b'
r"a\nb"		# "a\\nb" = 'a' + '\\' + 'n' + 'b'

"a\
b"		# "ab"
r"a\
b"		# "a\\\nb"
```

It is an error for a backslash to appear within a string literal other
than as part of one of the escapes described above.

TODO: define indent, outdent, semicolon, newline, eof

## Data types

These are the main data types built in to the interpreter:

```python
NoneType                     # the type of None
bool                         # True or False
int                          # a signed integer of arbitrary magnitude
float                        # an IEEE 754 double-precision floating point number
string                       # a byte string
list                         # a modifiable sequence of values
tuple                        # an unmodifiable sequence of values
dict                         # a mapping from values to values
set                          # a set of values
function                     # a function implemented in Starlark
builtin_function_or_method   # a function or method implemented by the interpreter or host application
```

Some functions, such as the iteration methods of `string`, or the
`range` function, return instances of special-purpose types that don't
appear in this list.
Additional data types may be defined by the host application into
which the interpreter is embedded, and those data types may
participate in basic operations of the language such as arithmetic,
comparison, indexing, and function calls.

<!-- We needn't mention the stringIterable type here. -->

Some operations can be applied to any Starlark value.  For example,
every value has a type string that can be obtained with the expression
`type(x)`, and any value may be converted to a string using the
expression `str(x)`, or to a Boolean truth value using the expression
`bool(x)`.  Other operations apply only to certain types.  For
example, the indexing operation `a[i]` works only with strings, lists,
and tuples, and any application-defined types that are _indexable_.
The [_value concepts_](#value-concepts) section explains the groupings of
types by the operators they support.


### None

`None` is a distinguished value used to indicate the absence of any other value.
For example, the result of a call to a function that contains no return statement is `None`.

`None` is equal only to itself.  Its [type](#type) is `"NoneType"`.
The truth value of `None` is `False`.


### Booleans

There are two Boolean values, `True` and `False`, representing the
truth or falsehood of a predicate.  The [type](#type) of a Boolean is `"bool"`.

Boolean values are typically used as conditions in `if`-statements,
although any Starlark value used as a condition is implicitly
interpreted as a Boolean.
For example, the values `None`, `0`, `0.0`, and the empty sequences
`""`, `()`, `[]`, and `{}` have a truth value of `False`, whereas non-zero
numbers and non-empty sequences have a truth value of `True`.
Application-defined types determine their own truth value.
Any value may be explicitly converted to a Boolean using the built-in `bool`
function.

```python
1 + 1 == 2                              # True
2 + 2 == 5                              # False

if 1 + 1:
        print("True")
else:
        print("False")
```

### Integers

The Starlark integer type represents integers.  Its [type](#type) is `"int"`.

Integers may be positive or negative, and arbitrarily large.
Integer arithmetic is exact.
Integers are totally ordered; comparisons follow mathematical
tradition.

The `+` and `-` operators perform addition and subtraction, respectively.
The `*` operator performs multiplication.

The `//` and `%` operations on integers compute floored division and
remainder of floored division, respectively.
If the signs of the operands differ, the sign of the remainder `x % y`
matches that of the divisor, `y`.
For all finite x and y (y ≠ 0), `(x // y) * y + (x % y) == x`.
The `/` operator implements real division, and
yields a `float` result even when its operands are both of type `int`.

Integers, including negative values, may be interpreted as bit vectors.
The `|`, `&`, and `^` operators implement bitwise OR, AND, and XOR,
respectively. The unary `~` operator yields the bitwise inversion of its
integer argument. The `<<` and `>>` operators shift the first argument
to the left or right by the number of bits given by the second argument.

Any bool, number, or string may be interpreted as an integer by using
the `int` built-in function.

An integer used in a Boolean context is considered true if it is
non-zero.

```python
100 // 5 * 9 + 32               # 212
3 // 2                          # 1
3 / 2                           # 1.5
111111111 * 111111111           # 12345678987654321
"0x%x" % (0x1234 & 0xf00f)      # "0x1004"
int("ffff", 16)                 # 65535, 0xffff
```

### Floating-point numbers

The Starlark floating-point data type represents an IEEE 754
double-precision floating-point number.  Its [type](#type) is `"float"`.

Arithmetic on floats using the `+`, `-`, `*`, `/`, `//`, and `%`
 operators follows the IEE 754 standard.
However, computing the division or remainder of division by zero is a dynamic error.

An arithmetic operation applied to a mixture of `float` and `int`
operands works as if the `int` operand is first converted to a
`float`.  For example, `3.141 + 1` is equivalent to `3.141 +
float(1)`.
There are two floating-point division operators:
`x / y ` yields the floating-point quotient of `x` and `y`,
whereas `x // y` yields `floor(x / y)`, that is, the largest
integer value not greater than `x / y`.
Although the resulting number is integral, it is represented as a
`float` if either operand is a `float`.

The `%` operation computes the remainder of floored division.
As with the corresponding operation on integers,
if the signs of the operands differ, the sign of the remainder `x % y`
matches that of the divisor, `y`.

The infinite float values `+Inf` and `-Inf` represent numbers
greater/less than all finite float values.

The non-finite `NaN` value represents the result of dubious operations
such as `Inf/Inf`.  A NaN value compares neither less than, nor
greater than, nor equal to any value, including itself.

All floats other than NaN are totally ordered, so they may be compared
using operators such as `==` and `<`.

Any bool, number, or string may be interpreted as a floating-point
number by using the `float` built-in function.

A float used in a Boolean context is considered true if it is
non-zero.

```python
1.23e45 * 1.23e45                               # 1.5129e+90
1.111111111111111 * 1.111111111111111           # 1.23457
3.0 / 2                                         # 1.5
3 / 2.0                                         # 1.5
float(3) / 2                                    # 1.5
3.0 // 2.0                                      # 1.0
```

### Strings

A string represents an immutable sequence of bytes.
The [type](#type) of a string is `"string"`.

Strings can represent arbitrary binary data, including zero bytes, but
most strings contain text, encoded by convention using UTF-8.

The built-in `len` function returns the number of bytes in a string.

Strings may be concatenated with the `+` operator.

The substring expression `s[i:j]` returns the substring of `s` from
index `i` up to index `j`.  The index expression `s[i]` returns the
1-byte substring `s[i:i+1]`.

Strings are hashable, and thus may be used as keys in a dictionary.

Strings are totally ordered lexicographically, so strings may be
compared using operators such as `==` and `<`.

Strings are _not_ iterable sequences, so they cannot be used as the operand of
a `for`-loop, list comprehension, or any other operation than requires
an iterable sequence.
To obtain a view of a string as an iterable sequence of numeric byte
values, 1-byte substrings, numeric Unicode code points, or 1-code
point substrings, you must explicitly call one of its four methods:
`elems`, `elem_ords`, `codepoints`, or `codepoint_ords`.

Any value may formatted as a string using the `str` or `repr` built-in
functions, the `str % tuple` operator, or the `str.format` method.

A string used in a Boolean context is considered true if it is
non-empty.

Strings have several built-in methods:

* [`capitalize`](#string·capitalize)
* [`codepoint_ords`](#string·codepoint_ords)
* [`codepoints`](#string·codepoints)
* [`count`](#string·count)
* [`elem_ords`](#string·elem_ords)
* [`elems`](#string·elems)
* [`endswith`](#string·endswith)
* [`find`](#string·find)
* [`format`](#string·format)
* [`index`](#string·index)
* [`isalnum`](#string·isalnum)
* [`isalpha`](#string·isalpha)
* [`isdigit`](#string·isdigit)
* [`islower`](#string·islower)
* [`isspace`](#string·isspace)
* [`istitle`](#string·istitle)
* [`isupper`](#string·isupper)
* [`join`](#string·join)
* [`lower`](#string·lower)
* [`lstrip`](#string·lstrip)
* [`partition`](#string·partition)
* [`replace`](#string·replace)
* [`removeprefix`](#string·removeprefix)
* [`removesuffix`](#string·removesuffix)
* [`rfind`](#string·rfind)
* [`rindex`](#string·rindex)
* [`rpartition`](#string·rpartition)
* [`rsplit`](#string·rsplit)
* [`rstrip`](#string·rstrip)
* [`split`](#string·split)
* [`splitlines`](#string·splitlines)
* [`startswith`](#string·startswith)
* [`strip`](#string·strip)
* [`title`](#string·title)
* [`upper`](#string·upper)

<b>Implementation note:</b>
The type of a string element varies across implementations.
There is agreement that byte strings, with text conventionally encoded
using UTF-8, is the ideal choice, but the Java implementation treats
strings as sequences of UTF-16 codes and changing it appears
intractible; see Google Issue b/36360490.

<b>Implementation note:</b>
The Java implementation does not consistently treat strings as
iterable; see `testdata/string.star` in the test suite and Google Issue
b/34385336 for further details.

### Lists

A list is a mutable sequence of values.
The [type](#type) of a list is `"list"`.

Lists are indexable sequences: the elements of a list may be iterated
over by `for`-loops, list comprehensions, and various built-in
functions.

List may be constructed using bracketed list notation:

```python
[]              # an empty list
[1]             # a 1-element list
[1, 2]          # a 2-element list
```

Lists can also be constructed from any iterable sequence by using the
built-in `list` function.

The built-in `len` function applied to a list returns the number of elements.
The index expression `list[i]` returns the element at index i,
and the slice expression `list[i:j]` returns a new list consisting of
the elements at indices from i to j.

List elements may be added using the `append` or `extend` methods,
removed using the `remove` method, or reordered by assignments such as
`list[i] = list[j]`.

The concatenation operation `x + y` yields a new list containing all
the elements of the two lists x and y.

For most types, `x += y` is equivalent to `x = x + y`, except that it
evaluates `x` only once. However, if `x` refers to a list, the statement
`x += y` does not allocate a new list as `x = x + y` would, but instead 
mutates the original list in place, similar to `x.extend(y)`.

Lists are not hashable, so may not be used in the keys of a dictionary.

A list used in a Boolean context is considered true if it is
non-empty.

A [_list comprehension_](#comprehensions) creates a new list whose elements are the
result of some expression applied to each element of another sequence.

```python
[x*x for x in [1, 2, 3, 4]]      # [1, 4, 9, 16]
```

A list value has these methods:

* [`append`](#list·append)
* [`clear`](#list·clear)
* [`extend`](#list·extend)
* [`index`](#list·index)
* [`insert`](#list·insert)
* [`pop`](#list·pop)
* [`remove`](#list·remove)

### Tuples

A tuple is an immutable sequence of values.
The [type](#type) of a tuple is `"tuple"`.

Tuples are constructed using parenthesized list notation:

```python
()                      # the empty tuple
(1,)                    # a 1-tuple
(1, 2)                  # a 2-tuple ("pair")
(1, 2, 3)               # a 3-tuple
```

Observe that for the 1-tuple, the trailing comma is necessary to
distinguish it from the parenthesized expression `(1)`.
1-tuples are seldom used.

Starlark, unlike Python, does not permit a trailing comma to appear in
an unparenthesized tuple expression:

```python
for k, v, in dict.items(): pass                 # syntax error at 'in'
_ = [(v, k) for k, v, in dict.items()]          # syntax error at 'in'
f = lambda a, b, : None                         # syntax error at ':'

sorted(3, 1, 4, 1,)                             # ok
[1, 2, 3, ]                                     # ok
{1: 2, 3:4, }                                   # ok
```

Any iterable sequence may be converted to a tuple by using the
built-in `tuple` function.

Like lists, tuples are indexed sequences, so they may be indexed and
sliced.  The index expression `tuple[i]` returns the tuple element at
index i, and the slice expression `tuple[i:j]` returns a sub-sequence
of a tuple.

Tuples are iterable sequences, so they may be used as the operand of a
`for`-loop, a list comprehension, or various built-in functions.

Unlike lists, tuples cannot be modified.
However, the mutable elements of a tuple may be modified.

Tuples are hashable (assuming their elements are hashable),
so they may be used as keys of a dictionary.

Tuples may be concatenated using the `+` operator.

A tuple used in a Boolean context is considered true if it is
non-empty.


### Dictionaries

A dictionary is a mutable mapping from keys to values.
The [type](#type) of a dictionary is `"dict"`.

Dictionaries provide constant-time operations to insert an element, to
look up the value for a key, or to remove an element.  Dictionaries
are implemented using hash tables, so keys must be hashable.  Hashable
values include `None`, Booleans, numbers, and strings, and tuples
composed from hashable values.  Most mutable values, such as lists,
dictionaries, and sets, are not hashable, even when frozen.
Attempting to use a non-hashable value as a key in a dictionary
results in a dynamic error.

A [dictionary expression](#dictionary-expressions) specifies a
dictionary as a set of key/value pairs enclosed in braces:

```python
coins = {
  "penny": 1,
  "nickel": 5,
  "dime": 10,
  "quarter": 25,
}
```

The expression `d[k]`, where `d` is a dictionary and `k` is a key,
retrieves the value associated with the key.  If the dictionary
contains no such item, the operation fails:

```python
coins["penny"]          # 1
coins["dime"]           # 10
coins["silver dollar"]  # error: key not found
```

The number of items in a dictionary `d` is given by `len(d)`.
A key/value item may be added to a dictionary, or updated if the key
is already present, by using `d[k]` on the left side of an assignment:

```python
len(coins)				# 4
coins["shilling"] = 20
len(coins)				# 5, item was inserted
coins["shilling"] = 5
len(coins)				# 5, existing item was updated
```

A dictionary can also be constructed using a [dictionary
comprehension](#comprehension), which evaluates a pair of expressions,
the _key_ and the _value_, for every element of another iterable such
as a list.  This example builds a mapping from each word to its length
in bytes:

```python
words = ["able", "baker", "charlie"]
{x: len(x) for x in words}	# {"charlie": 7, "baker": 5, "able": 4}
```

Dictionaries are iterable sequences, so they may be used as the
operand of a `for`-loop, a list comprehension, or various built-in
functions.
Iteration yields the dictionary's keys in the order in which they were
inserted; updating the value associated with an existing key does not
affect the iteration order.

```python
x = dict([("a", 1), ("b", 2)])          # {"a": 1, "b": 2}
x.update([("a", 3), ("c", 4)])          # {"a": 3, "b": 2, "c": 4}
```

```python
for name in coins:
  print(name, coins[name])	# prints "quarter 25", "dime 10", ...
```

Like all mutable values in Starlark, a dictionary can be frozen, and
once frozen, all subsequent operations that attempt to update it will
fail.

A dictionary used in a Boolean context is considered true if it is
non-empty.

The binary `|` operation may be applied to two dictionaries.
It yields a new dictionary whose set of keys is the union of the sets
of keys of the two operands. The corresponding values are taken from
the operands, where the value taken from the right operand takes
precedence if both contain a given key.  Iterating over the keys in
the resulting dictionary first yields all keys in the left operand in
insertion order, then all keys in the right operand that were not
present in the left operand, again in insertion order.

There is also an augmented assignment version of the `|` operation.
For two dictionaries `x` and `y`, the statement `x |= y` behaves
similar to `x = x | y`, but updates `x` in place rather than assigning
a new dictionary to it.

Dictionaries may be compared for equality using `==` and `!=`.  Two
dictionaries compare equal if they contain the same number of items
and each key/value item (k, v) found in one dictionary is also present
in the other.  Dictionaries are not ordered; it is an error to compare
two dictionaries with `<`.


A dictionary value has these methods:

* [`clear`](#dict·clear)
* [`get`](#dict·get)
* [`items`](#dict·items)
* [`keys`](#dict·keys)
* [`pop`](#dict·pop)
* [`popitem`](#dict·popitem)
* [`setdefault`](#dict·setdefault)
* [`update`](#dict·update)
* [`values`](#dict·values)

### Sets

A set is a mutable set of values.
The [type](#type) of a set is `"set"`.

Like dictionaries, sets are implemented using hash tables, so the
elements of a set must be hashable.

Sets may be compared for equality or inequality using `==` and `!=`.
Two sets compare equal if they contain the same elements.

Sets are iterable sequences, so they may be used as the operand of a
`for`-loop, a list comprehension, or various built-in functions.
Iteration yields the set's elements in the order in which they were
inserted.

The binary `|` and `&` operators compute union and intersection when
applied to sets.  The right operand of the `|` operator may be any
iterable value.  The binary `in` operator performs a set membership
test when its right operand is a set.

The binary `^` operator performs symmetric difference of two sets.

Sets are instantiated by calling the built-in `set` function, which
returns a set containing all the elements of its optional argument,
which must be an iterable sequence.  Sets have no literal syntax.

A set has these methods:

* [`add`](#set·add)
* [`clear`](#set·clear)
* [`difference`](#set·difference)
* [`discard`](#set·discard)
* [`intersection`](#set·intersection)
* [`issubset`](#set·issubset)
* [`issuperset`](#set·issuperset)
* [`pop`](#set·pop)
* [`remove`](#set·remove)
* [`symmetric_difference`](#set·symmetric_difference)
* [`union`](#set·union)
* [`update`](#set·update)

A set used in a Boolean context is considered true if it is non-empty.

<b>Implementation note:</b>
The Go implementation of Starlark requires the `-set` flag to
enable support for sets.
The Java implementation does not support sets.


### Functions

A function value represents a function defined in Starlark.
Its [type](#type) is `"function"`.
A function value used in a Boolean context is always considered true.

Functions defined by a [`def` statement](#function-definitions) are named;
functions defined by a [`lambda` expression](#lambda-expressions) are anonymous.

Function definitions may be nested, and an inner function may refer to a local variable of an outer function.

A function definition defines zero or more named parameters.
Starlark has a rich mechanism for passing arguments to functions.

<!-- TODO break up this explanation into caller-side and callee-side
     parts, and put the former under function calls and the latter
     under function definitions. Also try to convey that the Callable
     interface sees the flattened-out args and kwargs and that's what
     built-ins get.
-->

The example below shows a definition and call of a function of two
required parameters, `x` and `y`.

```python
def idiv(x, y):
  return x // y

idiv(6, 3)		# 2
```

A call may provide arguments to function parameters either by
position, as in the example above, or by name, as in first two calls
below, or by a mixture of the two forms, as in the third call below.
All the positional arguments must precede all the named arguments.
Named arguments may improve clarity, especially in functions of
several parameters.

```python
idiv(x=6, y=3)		# 2
idiv(y=3, x=6)		# 2

idiv(6, y=3)		# 2
```

<b>Optional parameters:</b> A parameter declaration may specify a
default value using `name=value` syntax; such a parameter is
_optional_.  The default value expression is evaluated during
execution of the `def` statement or evaluation of the `lambda`
expression, and the default value forms part of the function value.
All optional parameters must follow all non-optional parameters.
A function call may omit arguments for any suffix of the optional
parameters; the effective values of those arguments are supplied by
the function's parameter defaults.

```python
def f(x, y=3):
  return x, y

f(1, 2)	# (1, 2)
f(1)	# (1, 3)
```

If a function parameter's default value is a mutable expression,
modifications to the value during one call may be observed by
subsequent calls.
Beware of this when using lists or dicts as default values.
If the function becomes frozen, its parameters' default values become
frozen too.

```python
# module a.star
def f(x, list=[]):
  list.append(x)
  return list

f(4, [1,2,3])           # [1, 2, 3, 4]
f(1)                    # [1]
f(2)                    # [1, 2], not [2]!

# module b.star
load("a.star", "f")
f(3)                    # error: cannot append to frozen list
```

<b>Variadic functions:</b> Some functions allow callers to provide an
arbitrary number of arguments.
After all required and optional parameters, a function definition may
specify a _variadic arguments_ or _varargs_ parameter, indicated by a
star preceding the parameter name: `*args`.
Any surplus positional arguments provided by the caller are formed
into a tuple and assigned to the `args` parameter.

```python
def f(x, y, *args):
  return x, y, args

f(1, 2)                 # (1, 2, ())
f(1, 2, 3, 4)           # (1, 2, (3, 4))
```

<b>Keyword-variadic functions:</b> Some functions allow callers to
provide an arbitrary sequence of `name=value` keyword arguments.
A function definition may include a final _keyword arguments_ or
_kwargs_ parameter, indicated by a double-star preceding the parameter
name: `**kwargs`.
Any surplus named arguments that do not correspond to named parameters
are collected in a new dictionary and assigned to the `kwargs` parameter:

```python
def f(x, y, **kwargs):
  return x, y, kwargs

f(1, 2)                 # (1, 2, {})
f(x=2, y=1)             # (2, 1, {})
f(x=2, y=1, z=3)        # (2, 1, {"z": 3})
```

It is a static error if any two parameters of a function have the same name.

Just as a function definition may accept an arbitrary number of
positional or named arguments, a function call may provide an
arbitrary number of positional or named arguments supplied by a
list or dictionary:

```python
def f(a, b, c=5):
  return a * b + c

f(*[2, 3])              # 11
f(*[2, 3, 7])           # 13
f(*[2])                 # error: f takes at least 2 arguments (1 given)

f(**dict(b=3, a=2))             # 11
f(**dict(c=7, a=2, b=3))        # 13
f(**dict(a=2))                  # error: f takes at least 2 arguments (1 given)
f(**dict(d=4))                  # error: f got unexpected keyword argument "d"
```

Once the parameters have been successfully bound to the arguments
supplied by the call, the sequence of statements that comprise the
function body is executed.

It is a static error if a function call has two named arguments of the
same name, such as `f(x=1, x=2)`. A call that provides a `**kwargs`
argument may yet have two values for the same name, such as
`f(x=1, **dict(x=2))`. This results in a dynamic error.

Function arguments are evaluated in the order they appear in the call.
<!-- see https://github.com/bazelbuild/starlark/issues/13 -->

Unlike Python, Starlark does not allow more than one `*args` argument in a
call, and if a `*args` argument is present it must appear after all
positional and named arguments.

The final argument to a function call may be followed by a trailing comma.

A function call completes normally after the execution of either a
`return` statement, or of the last statement in the function body.
The result of the function call is the value of the return statement's
operand, or `None` if the return statement had no operand or if the
function completed without executing a return statement.

```python
def f(x):
  if x == 0:
    return
  if x < 0:
    return -x
  print(x)

f(1)            # returns None after printing "1"
f(0)            # returns None without printing
f(-1)           # returns 1 without printing
```

<b>Implementation note:</b>
The Go implementation of Starlark requires the `-recursion`
flag to allow recursive functions.


If the `-recursion` flag is not specified it is a dynamic error for a
function to call itself or another function value with the same
declaration.

```python
def fib(x):
  if x < 2:
    return x
  return fib(x-2) + fib(x-1)	# dynamic error: function fib called recursively

fib(5)
```

This rule, combined with the invariant that all loops are iterations
over finite sequences, implies that Starlark programs can not be
Turing complete unless the `-recursion` flag is specified.

<!-- This rule is supposed to deter people from abusing Starlark for
     inappropriate uses, especially in the build system.
     It may work for that purpose, but it doesn't stop Starlark programs
     from consuming too much time or space.  Perhaps it should be a
     dialect option.
-->



### Built-in functions

A built-in function is a function or method implemented in Go by the interpreter
or the application into which the interpreter is embedded.

The [type](#type) of a built-in function is `"builtin_function_or_method"`.

A built-in function value used in a Boolean context is always considered true.

Many [built-in functions](#built-in-constants-and-functions) are predeclared 
in the environment (see [Name binding and variables](#name-binding-and-variables)).
Some built-in functions such as `len` are _universal_, that is,
available to all Starlark programs.
The host application may predeclare additional built-in functions
in the environment of a specific module.

Except where noted, built-in functions accept only positional arguments.
The parameter names serve merely as documentation.

Most built-in functions that have a Boolean parameter require its
argument to be `True` or `False`. Unlike `if` statements, other values
are not implicitly converted to their truth value and instead cause a
dynamic error.


## Name binding and variables

After a Starlark file is parsed, but before its execution begins, the
Starlark interpreter checks statically that the program is well formed.
For example, `break` and `continue` statements may appear only within
a loop; a `return` statement may appear only within a
function; and `load` statements may appear only outside any function.

_Name resolution_ is the static checking process that
resolves names to variable bindings.
During execution, names refer to variables.  Statically, names denote
places in the code where variables are created; these places are
called _bindings_.  A name may denote different bindings at different
places in the program.  The region of text in which a particular name
refers to the same binding is called that binding's _scope_.

Four Starlark constructs bind names, as illustrated in the example below:
`load` statements (`a` and `b`),
`def` statements (`c`),
function parameters (`d`),
and assignments (`e`, `h`, including the augmented assignment `e += 1`).
Variables may be assigned or re-assigned explicitly (`e`, `h`), or implicitly, as
in a `for`-loop (`f`) or comprehension (`g`, `i`).

```python
load("lib.star", "a", b="B")

def c(d):
  e = 0
  for f in d:
     print([True for g in f])
     e += 1

h = [2*i for i in a]
```

The environment of a Starlark program is structured as a tree of
_lexical blocks_, each of which may contain name bindings.
The tree of blocks is parallel to the syntax tree.
Blocks are of five kinds.

<!-- Avoid the term "built-in" block since that's also a type. -->
At the root of the tree is the _predeclared_ block,
which binds several names implicitly.
The set of predeclared names includes the universal
constant values `None`, `True`, and `False`, and
various built-in functions such as `len` and `list`;
these functions are immutable and stateless.
An application may pre-declare additional names
to provide domain-specific functions to that file, for example.
These additional functions may have side effects on the application.
Starlark programs cannot change the set of predeclared bindings
or assign new values to them.

Nested beneath the predeclared block is the _module_ block,
which contains the bindings of the current module.
Bindings in the module block (such as `c`, and `h` in the
example) are called _global_ and may be visible to other modules.
The module block is empty at the start of the file
and is populated by top-level binding statements.

Nested beneath the module block is the _file_ block,
which contains bindings local to the current file.
Names in this block (such as `a` and `b` in the example)
are bound only by `load` statements.
The sets of names bound in the file block and in the module block do not overlap:
it is an error for a load statement to bind the name of a global,
or for a top-level statement to bind a name bound by a load statement.

A file block contains a _function_ block for each top-level
function, and a _comprehension_ block for each top-level comprehension.
Bindings in either of these kinds of block,
and in the file block itself, are called _local_.
(In the example, the bindings for `e`, `f`, `g`, and `i` are all local.)
Additional functions and comprehensions, and their blocks, may be
nested in any order, to any depth.

If name is bound anywhere within a block, all uses of the name within
the block are treated as references to that binding,
even if the use appears before the binding.
This is true even at the top level, unlike Python.
The binding of `y` on the last line of the example below makes `y`
local to the function `hello`, so the use of `y` in the print
statement also refers to the local `y`, even though it appears
earlier.

```python
y = "goodbye"

def hello():
  for x in (1, 2):
    if x == 2:
      print(y) # prints "hello"
    if x == 1:
      y = "hello"
```
It is a dynamic error to evaluate a reference to a local variable
before it has been bound:

```python
def f():
  print(x)              # dynamic error: local variable x referenced before assignment
  x = "hello"
```

The same is true for global variables:

```python
print(x)                # dynamic error: global variable x referenced before assignment
x = "hello"
```

The same is also true for nested loops in comprehensions.
In the (unnatural) examples below, the scope of the variables `x`, `y`, 
and `z` is the entire comprehension block, except the operand of the first
loop (`[]` or `[1]`), which is resolved in the enclosing environment.
The second loop may thus refer to variables defined by the third (`z`),
even though such references would fail if actually executed.

```
[1//0 for x in [] for y in z for z in ()]   # []   (no error)
[1//0 for x in [1] for y in z for z in ()]  # dynamic error: local variable z referenced before assignment
```


<!-- This is similar to Python[23]. Presumed rational: it resembles
     the desugaring to nested loop statements, in which the scope
     of all three variables is the entire enclosing function,
     including the portion before the bindings.

      def f():
        ...
        for x in []:
          for y in z:
            for z in ():
              1//0
-->

It is a static error to refer to a name that has no binding at all.
```
def f():
  if False:
    g()                   # static error: undefined: g
```
(This behavior differs from Python, which treats such references as global,
and thus does not report an error until the expression is evaluated.)

<!-- Consequently, the REPL, which consumes one compound statement at a time,
     cannot resolve forward references such as
             def f(): return K
             K = 1
     because the first chunk has an unresolved reference to K.
-->

It is a static error to bind a global variable already explicitly bound in the file:

```python
x = 1
x = 2                   # static error: cannot reassign global x declared on line 1
```

<!-- The above rule, and the rule that forbids if-statements and loops at
     top level, exist to ensure that there is exactly one statement
     that binds each global variable, which makes cross-referenced
     documentation more useful, the designers assure me, but
     I am skeptical that it's worth the trouble. -->

If a name was pre-bound by the application, the Starlark program may
explicitly bind it, but only once.

An augmented assignment statement such as `x += y` is considered both a
reference to `x` and a binding use of `x`, so it may not be used at
top level.

<b>Implementation note:</b>
The Go implementation of Starlark permits augmented assignments to appear
at top level if the `-globalreassign` flag is enabled.

A function may refer to variables defined in an enclosing function.
In this example, the inner function `f` refers to a variable `x`
that is local to the outer function `squarer`.
`x` is a _free variable_ of `f`.
The function value (`f`) created by a `def` statement holds a
reference to each of its free variables so it may use
them even after the enclosing function has returned.

```python
def squarer():
    x = [0]
    def f():
      x[0] += 1
      return x[0]*x[0]
    return f

sq = squarer()
print(sq(), sq(), sq(), sq()) # "1 4 9 16"
```

An inner function cannot assign to a variable bound in an enclosing
function, because the assignment would bind the variable in the
inner function.
In the example below, the `x += 1` statement binds `x` within `f`,
hiding the outer `x`.
Execution fails because the inner `x` has not been assigned before the
attempt to increment it.

```python
def squarer():
    x = 0
    def f():
      x += 1            # dynamic error: local variable x referenced before assignment
      return x*x
    return f

sq = squarer()
```

(Starlark has no equivalent of Python's `nonlocal` or `global`
declarations, but as the first version of `squarer` showed, this
omission can be worked around by using a list of a single element.)


A name appearing after a dot, such as `split` in
`get_filename().split('/')`, is not resolved statically.
The [dot expression](#dot-expressions) `.split` is a dynamic operation
on the value returned by `get_filename()`.


## Value concepts

Starlark has eleven core [data types](#data-types).  An application
that embeds the Starlark interpreter may define additional types that
behave like Starlark values.  All values, whether core or
application-defined, implement a few basic behaviors:

```text
str(x)		-- return a string representation of x
type(x)		-- return a string describing the type of x
bool(x)		-- convert x to a Boolean truth value
```

### Identity and mutation

Starlark is an imperative language: programs consist of sequences of
statements executed for their side effects.
For example, an assignment statement updates the value held by a
variable, and calls to some built-in functions such as `print` change
the state of the application that embeds the interpreter.

Values of some data types, such as `NoneType`, `bool`, `int`, `float`, and
`string`, are _immutable_; they can never change.
Immutable values have no notion of _identity_: it is impossible for a
Starlark program to tell whether two integers, for instance, are
represented by the same object; it can tell only whether they are
equal.

Values of other data types, such as `list`, `dict`, and `set`, are
_mutable_: they may be modified by a statement such as `a[i] = 0` or
`items.clear()`.  Although `tuple` and `function` values are not
directly mutable, they may refer to mutable values indirectly, so for
this reason we consider them mutable too.  Starlark values of these
types are actually _references_ to variables.

Copying a reference to a variable, using an assignment statement for
instance, creates an _alias_ for the variable, and the effects of
operations applied to the variable through one alias are visible
through all others.

```python
x = []                          # x refers to a new empty list variable
y = x                           # y becomes an alias for x
x.append(1)                     # changes the variable referred to by x
print(y)                        # "[1]"; y observes the mutation
```

Starlark uses _call-by-value_ parameter passing: in a function call,
argument values are assigned to function parameters as if by
assignment statements.  If the values are references, the caller and
callee may refer to the same variables, so if the called function
changes the variable referred to by a parameter, the effect may also
be observed by the caller:

```python
def f(y):
    y.append(1)                 # changes the variable referred to by x

x = []                          # x refers to a new empty list variable
f(x)                            # f's parameter y becomes an alias for x
print(x)                        # "[1]"; x observes the mutation
```


As in all imperative languages, understanding _aliasing_, the
relationship between reference values and the variables to which they
refer, is crucial to writing correct programs.

### Freezing a value

Starlark has a feature unusual among imperative programming languages:
a mutable value may be _frozen_ so that all subsequent attempts to
mutate it fail with a dynamic error; the value, and all other values
reachable from it, become _immutable_.

Immediately after execution of a Starlark module, all values in its
top-level environment are frozen. Because all the global variables of
an initialized Starlark module are immutable, the module may be published to
and used by other threads in a parallel program without the need for
locks. For example, the Bazel build system loads and executes BUILD
and .bzl files in parallel, and two modules being executed
concurrently may freely access variables or call functions from a
third without the possibility of a race condition.

### Hashing

The `dict` and `set` data types are implemented using hash tables, so
only _hashable_ values are suitable as keys of a `dict` or elements of
a `set`. Attempting to use a non-hashable value as the key in a hash
table results in a dynamic error.

The hash of a value is an unspecified integer chosen so that two equal
values have the same hash, in other words, `x == y => hash(x) == hash(y)`.
A hashable value has the same hash throughout its lifetime.

Values of the types `NoneType`, `bool`, `int`, `float`, and `string`,
which are all immutable, are hashable.

Values of mutable types such as `list`, `dict`, and `set` are not
hashable. These values remain unhashable even if they have become
immutable due to _freezing_.

A `tuple` value is hashable only if all its elements are hashable.
Thus `("localhost", 80)` is hashable but `([127, 0, 0, 1], 80)` is not.

Values of the types `function` and `builtin_function_or_method` are also hashable.
Although functions are not necessarily immutable, as they may be
closures that refer to mutable variables, instances of these types
are compared by reference identity (see [Comparisons](#comparisons)),
so their hash values are derived from their identity.


### Sequence types

Many Starlark data types represent a _sequence_ of values: lists,
tuples, and sets are sequences of arbitrary values, and in many
contexts dictionaries act like a sequence of their keys.

We can classify different kinds of sequence types based on the
operations they support.
Each is listed below using the name of its corresponding interface in
the interpreter's Go API.

* `Iterable`: an _iterable_ value lets us process each of its elements in a fixed order.
  Examples: `dict`, `set`, `list`, `tuple`, but not `string`.
* `Sequence`: a _sequence of known length_ lets us know how many elements it
  contains without processing them.
  Examples: `dict`, `set`, `list`, `tuple`, but not `string`.
* `Indexable`: an _indexed_ type has a fixed length and provides efficient
  random access to its elements, which are identified by integer indices.
  Examples: `string`, `tuple`, and `list`.
* `SetIndexable`: a _settable indexed type_ additionally allows us to modify the
  element at a given integer index. Example: `list`.
* `Mapping`: a mapping is an association of keys to values. Example: `dict`.

Although all of Starlark's core data types for sequences implement at
least the `Sequence` contract, it's possible for an application
that embeds the Starlark interpreter to define additional data types
representing sequences of unknown length that implement only the `Iterable` contract.

Strings are not iterable, though they do support the `len(s)` and
`s[i]` operations. Starlark deviates from Python here to avoid a common
pitfall in which a string is used by mistake where a list containing a
single string was intended, resulting in its interpretation as a sequence
of bytes.

Most Starlark operators and built-in functions that need a sequence
of values will accept any iterable.

It is a dynamic error to mutate a sequence such as a list, set, or
dictionary while iterating over it.

```python
def increment_values(dict):
  for k in dict:
    dict[k] += 1			# error: cannot insert into hash table during iteration

dict = {"one": 1, "two": 2}
increment_values(dict)
```


### Indexing

Many Starlark operators and functions require an index operand `i`,
such as `a[i]` or `list.insert(i, x)`. Others require two indices `i`
and `j` that indicate the start and end of a sub-sequence, such as
`a[i:j]`, `list.index(x, i, j)`, or `string.find(x, i, j)`.
All such operations follow similar conventions, described here.

Indexing in Starlark is *zero-based*. The first element of a string
or list has index 0, the next 1, and so on. The last element of a
sequence of length `n` has index `n-1`.

```python
"hello"[0]			# "h"
"hello"[4]			# "o"
"hello"[5]			# error: index out of range
```

For sub-sequence operations that require two indices, the first is
_inclusive_ and the second _exclusive_. Thus `a[i:j]` indicates the
sequence starting with element `i` up to but not including element
`j`. The length of this sub-sequence is `j-i`. This convention is known
as *half-open indexing*.

```python
"hello"[1:4]			# "ell"
```

Either or both of the index operands may be omitted. If omitted, the
first is treated equivalent to 0 and the second is equivalent to the
length of the sequence:

```python
"hello"[1:]                     # "ello"
"hello"[:4]                     # "hell"
```

It is permissible to supply a negative integer to an indexing
operation. The effective index is computed from the supplied value by
the following two-step procedure. First, if the value is negative, the
length of the sequence is added to it. This provides a convenient way
to address the final elements of the sequence:

```python
"hello"[-1]                     # "o",  like "hello"[4]
"hello"[-3:-1]                  # "ll", like "hello"[2:4]
```

Second, for sub-sequence operations, if the value is still negative, it
is replaced by zero, or if it is greater than the length `n` of the
sequence, it is replaced by `n`. In effect, the index is "truncated" to
the nearest value in the range `[0:n]`.

```python
"hello"[-1000:+1000]		# "hello"
```

This truncation step does not apply to indices of individual elements:

```python
"hello"[-6]		# error: index out of range
"hello"[-5]		# "h"
"hello"[4]		# "o"
"hello"[5]		# error: index out of range
```


## Expressions

An expression specifies the computation of a value.

The Starlark grammar defines several categories of expression.
An _operand_ is an expression consisting of a single token (such as an
identifier or a literal), or a bracketed expression.
Operands are self-delimiting.
An operand may be followed by any number of dot, call, or slice
suffixes, to form a _primary_ expression.
In some places in the Starlark grammar where an expression is expected,
it is legal to provide a comma-separated list of expressions denoting
a tuple.
The grammar uses `Expression` where a multiple-component expression is allowed,
and `Test` where it accepts an expression of only a single component.

```grammar {.good}
Expression = Test {',' Test} .

Test = LambdaExpr | IfExpr | PrimaryExpr | UnaryExpr | BinaryExpr .

PrimaryExpr = Operand
            | PrimaryExpr DotSuffix
            | PrimaryExpr CallSuffix
            | PrimaryExpr SliceSuffix
            .

Operand = identifier
        | int | float | string
        | ListExpr | ListComp
        | DictExpr | DictComp
        | '(' [Expression] [,] ')'
        | ('-' | '+') PrimaryExpr
        .

DotSuffix   = '.' identifier .
CallSuffix  = '(' [Arguments [',']] ')' .
SliceSuffix = '[' [Expression] [':' Test [':' Test]] ']' .
```

TODO: resolve position of +x, -x, and 'not x' in grammar: Operand or UnaryExpr?

### Identifiers

```grammar {.good} {.good}
Primary = identifier
```

An identifier is a name that identifies a value.

Lookup of locals and globals may fail if not yet defined.

### Literals

Starlark supports literals of three different kinds:

```grammar {.good}
Primary = int | float | string
```

Evaluation of a literal yields a value of the given type (string, int,
or float) with the given value.
See [Literals](#lexical-elements) for details.

### Parenthesized expressions

```grammar {.good}
Primary = '(' [Expression] ')'
```

A single expression enclosed in parentheses yields the result of that expression.
Explicit parentheses may be used for clarity,
or to override the default association of subexpressions.

```python
1 + 2 * 3 + 4                   # 11
(1 + 2) * (3 + 4)               # 21
```

If the parentheses are empty, or contain a single expression followed
by a comma, or contain two or more expressions, the expression yields a tuple.

```python
()                              # (), the empty tuple
(1,)                            # (1,), a tuple of length 1
(1, 2)                          # (1, 2), a 2-tuple or pair
(1, 2, 3)                       # (1, 2, 3), a 3-tuple or triple
```

In some contexts, such as a `return` or assignment statement or the
operand of a `for` statement, a tuple may be expressed without
parentheses.

```python
x, y = 1, 2

return 1, 2

for x in 1, 2:
   print(x)
```

Starlark (like Python 3) does not accept an unparenthesized tuple
expression as the operand of a list comprehension:

```python
[2*x for x in 1, 2, 3]	       	# parse error: unexpected ','
```

### Dictionary expressions

A dictionary expression is a comma-separated list of colon-separated
key/value expression pairs, enclosed in curly brackets, and it yields
a new dictionary object.
An optional comma may follow the final pair.

```grammar {.good}
DictExpr = '{' [Entries [',']] '}' .
Entries  = Entry {',' Entry} .
Entry    = Test ':' Test .
```

Examples:


```python
{}
{"one": 1}
{"one": 1, "two": 2,}
```

The key and value expressions are evaluated in left-to-right order.
Evaluation fails if the same key is used multiple times.

Only [hashable](#hashing) values may be used as the keys of a dictionary.
This includes all built-in types except dictionaries, sets, and lists;
a tuple is hashable only if its elements are hashable.


### List expressions

A list expression is a comma-separated list of element expressions,
enclosed in square brackets, and it yields a new list object.
An optional comma may follow the last element expression.

```grammar {.good}
ListExpr = '[' [Expression [',']] ']' .
```

Element expressions are evaluated in left-to-right order.

Examples:

```python
[]                      # [], empty list
[1]                     # [1], a 1-element list
[1, 2, 3,]              # [1, 2, 3], a 3-element list
```

### Unary operators

There are three unary operators, all appearing before their operand:
`+`, `-`, `~`, and `not`.

```grammar {.good}
UnaryExpr = '+' PrimaryExpr
          | '-' PrimaryExpr
          | '~' PrimaryExpr
          | 'not' Test
          .
```

```text
+ number        unary positive          (int, float)
- number        unary negation          (int, float)
~ number        unary bitwise inversion (int)
not x           logical negation        (any type)
```

The `+` and `-` operators may be applied to any number
(`int` or `float`) and return the number unchanged.
Unary `+` is never necessary in a correct program,
but may serve as an assertion that its operand is a number,
or as documentation.

```python
if x > 0:
	return +1
else if x < 0:
	return -1
else:
	return 0
```

The `not` operator returns the negation of the truth value of its
operand.

```python
not True                        # False
not False                       # True
not [1, 2, 3]                   # False
not ""                          # True
not 0                           # True
```

The `~` operator yields the bitwise inversion of its integer argument.
The bitwise inversion of x is defined as -(x+1).

```python
~1                              # -2
~-1                             # 0
~0                              # -1
```


### Binary operators

Starlark has the following binary operators, arranged in order of increasing precedence:

```text
or
and
==   !=   <    >   <=   >=   in   not in
|
^
&
<<   >>
-    +
*    /    //   %
```

Comparison operators, `in`, and `not in` are non-associative,
so the parser will not accept `0 <= i < n`.
All other binary operators of equal precedence associate to the left.

```grammar {.good}
BinaryExpr = Test {Binop Test} .

Binop = 'or'
      | 'and'
      | '==' | '!=' | '<' | '>' | '<=' | '>=' | 'in' | 'not' 'in'
      | '|'
      | '^'
      | '&'
      | '-' | '+'
      | '*' | '%' | '/' | '//'
      | '<<' | '>>'
      .
```

#### `or` and `and`

The `or` and `and` operators yield, respectively, the logical disjunction and
conjunction of their arguments, which need not be Booleans.
The expression `x or y` yields the value of `x` if its truth value is `True`,
or the value of `y` otherwise.

```starlark
False or False		# False
False or True		# True
True  or False		# True
True  or True		# True

0 or "hello"		# "hello"
1 or "hello"		# 1
```

Similarly, `x and y` yields the value of `x` if its truth value is
`False`, or the value of `y` otherwise.

```starlark
False and False		# False
False and True		# False
True  and False		# False
True  and True		# True

0 and "hello"		# 0
1 and "hello"		# "hello"
```

These operators use "short circuit" evaluation, so the second
expression is not evaluated if the value of the first expression has
already determined the result, allowing constructions like these:

```python
len(x) > 0 and x[0] == 1		# x[0] is not evaluated if x is empty
x and x[0] == 1
len(x) == 0 or x[0] == ""
not x or not x[0]
```

#### Comparisons

The `==` operator reports whether its operands are equal; the `!=`
operator is its negation.

The operators `<`, `>`, `<=`, and `>=` perform an ordered comparison
of their operands.  It is an error to apply these operators to
operands of unequal type, unless one of the operands is an `int` and
the other is a `float`.  Of the built-in types, only the following
support ordered comparison, using the ordering relation shown:

```shell
NoneType        # None <= None
bool            # False < True
int             # mathematical
float           # as defined by IEEE 754
string          # lexicographical
tuple           # lexicographical
list            # lexicographical
```

Comparison of floating point values follows the IEEE 754 standard,
which breaks several mathematical identities.  For example, if `x` is
a `NaN` value, the comparisons `x < y`, `x == y`, and `x > y` all
yield false for all values of `y`.

When used to compare two `set` objects, the `<=`, and `>=` operators will report
whether one set is a subset or superset of another. Similarly, using `<` or `>` will
report whether a set is a proper subset or superset of another, thus `x > y` is
equivalent to `x >= y and x != y`.

Applications may define additional types that support ordered
comparison.

The remaining built-in types support only equality comparisons.
Values of type `dict` or `set` compare equal if their elements compare
equal, and values of type `function` or `builtin_function_or_method` are equal only to
themselves.

```shell
dict                            # equal contents
set                             # equal contents
function                        # identity
builtin_function_or_method      # identity
```

#### Arithmetic operations

The following table summarizes the binary arithmetic operations
available for built-in types:

```shell
Arithmetic (int or float; result has type float unless both operands have type int)
   number + number              # addition
   number - number              # subtraction
   number * number              # multiplication
   number / number              # real division  (result is always a float)
   number // number             # floored division
   number % number              # remainder of floored division
   number ^ number              # bitwise XOR
   number << number             # bitwise left shift
   number >> number             # bitwise right shift

Concatenation
   string + string
     list + list
    tuple + tuple

Repetition (string/list/tuple)
      int * sequence
 sequence * int

String interpolation
   string % any                 # see String Interpolation

Sets
      int | int                 # bitwise union (OR)
      set | set                 # set union
      int & int                 # bitwise intersection (AND)
      set & set                 # set intersection
      set ^ set                 # set symmetric difference
      set - set                 # set difference


Dict
      dict | dict               # ordered union
```

The operands of the arithmetic operators `+`, `-`, `*`, `//`, and
`%` must both be numbers (`int` or `float`) but need not have the same type.
The type of the result has type `int` only if both operands have that type.
The result of real division `/` always has type `float`.

The `+` operator may be applied to non-numeric operands of the same
type, such as two lists, two tuples, or two strings, in which case it
computes the concatenation of the two operands and yields a new value of
the same type.

```python
"Hello, " + "world"		# "Hello, world"
(1, 2) + (3, 4)			# (1, 2, 3, 4)
[1, 2] + [3, 4]			# [1, 2, 3, 4]
```

The `*` operator may be applied to an integer _n_ and a value of type
`string`, `list`, or `tuple`, in which case it yields a new value
of the same sequence type consisting of _n_ repetitions of the original sequence.
The order of the operands is immaterial.
Negative values of _n_ behave like zero.

```python
'mur' * 2               # 'murmur'
3 * range(3)            # [0, 1, 2, 0, 1, 2, 0, 1, 2]
```

Applications may define additional types that support any subset of
these operators.

The `&` operator requires two operands of the same type, either `int` or `set`.
For integers, it yields the bitwise intersection (AND) of its operands.
For sets, it yields a new set containing the intersection of the
elements of the operand sets, preserving the element order of the left
operand.

The `|` operator likewise computes bitwise, set, or dict unions.
The result of `set | set` is a new set whose elements are the
union of the operands, preserving the order of the elements of the
operands, left before right.
Similarly, the result of `dict | dict` is a new dict whose entries are
the union of the operands, preserving the order in which keys first
appear, but using the value from the right operand for each key
common to both dicts.

The `^` operator accepts operands of either `int` or `set` type.
For integers, it yields the bitwise XOR (exclusive OR) of its operands.
For sets, it yields a new set containing elements of either first or second
operand but not both (symmetric difference).

The `<<` and `>>` operators require operands of `int` type both. They shift
the first operand to the left or right by the number of bits given by the
second operand. It is a dynamic error if the second operand is negative.
Implementations may impose a limit on the second operand of a left shift.

```python
0x12345678 & 0xFF               # 0x00000078
0x12345678 | 0xFF               # 0x123456FF
0b01011101 ^ 0b110101101        # 0b111110000
0b01011101 >> 2                 # 0b010111
0b01011101 << 2                 # 0b0101110100

set([1, 2]) & set([2, 3])       # set([2])
set([1, 2]) | set([2, 3])       # set([1, 2, 3])
set([1, 2]) ^ set([2, 3])       # set([1, 3])
set([1, 2]) - set([2, 3])       # set([1])
```

<b>Implementation note:</b>
The Go implementation of Starlark requires the `-set` flag to
enable support for sets.
The Java implementation does not support sets.


#### Membership tests

```text
      any in     sequence		(list, tuple, dict, set, string)
      any not in sequence
```

The `in` operator reports whether its first operand is a member of its
second operand, which must be a list, tuple, dict, set, or string.
The `not in` operator is its negation.
Both return a Boolean.

The meaning of membership varies by the type of the second operand:
the members of a list, tuple, or set are its elements;
the members of a dict are its keys;
the members of a string are all its substrings.

```python
1 in [1, 2, 3]                  # True
4 in (1, 2, 3)                  # False
4 not in set([1, 2, 3])         # True

d = {"one": 1, "two": 2}
"one" in d                      # True
"three" in d                    # False
1 in d                          # False
[] in d				# False

"nasty" in "dynasty"            # True
"a" in "banana"                 # True
"f" not in "way"                # True
```

#### String interpolation

The expression `format % args` performs _string interpolation_, a
simple form of template expansion.
The `format` string is interpreted as a sequence of literal portions
and _conversions_.
Each conversion, which starts with a `%` character, is replaced by its
corresponding value from `args`.
The characters following `%` in each conversion determine which
argument it uses and how to convert it to a string.

Each `%` character marks the start of a conversion specifier, unless
it is immediately followed by another `%`, in which case both
characters together denote a literal percent sign.

If the `"%"` is immediately followed by `"(key)"`, the parenthesized
substring specifies the key of the `args` dictionary whose
corresponding value is the operand to convert.
Otherwise, the conversion's operand is the next element of `args`,
which must be a tuple with exactly one component per conversion,
unless the format string contains only a single conversion, in which
case `args` itself is its operand.

If the format string contains no conversions, the operand must be a
`Mapping` or an empty tuple.

Starlark does not support the flag, width, and padding specifiers
supported by Python's `%` and other variants of C's `printf`.

After the optional `(key)` comes a single letter indicating what
operand types are valid and how to convert the operand `x` to a string:

```text
%       none            literal percent sign
s       any             as if by str(x)
r       any             as if by repr(x)
d       number          signed integer decimal
i       number          signed integer decimal
o       number          signed octal
x       number          signed hexadecimal, lowercase
X       number          signed hexadecimal, uppercase
e       number          float exponential format, lowercase
E       number          float exponential format, uppercase
f       number          float decimal format, lowercase
F       number          float decimal format, uppercase
g       number          like %e for large exponents, %f otherwise
G       number          like %E for large exponents, %F otherwise
c       string          x (string must encode a single Unicode code point)
        int             as if by chr(x)
```

It is an error if the argument does not have the type required by the
conversion specifier.  A Boolean argument is not considered a number.

Examples:

```python
"Hello %s, your score is %d" % ("Bob", 75)      # "Hello Bob, your score is 75"

"%d %o %x %c" % (65, 65, 65, 65)                # "65 101 41 A" (decimal, octal, hexadecimal, Unicode)

"%(greeting)s, %(audience)s" % dict(            # "Hello, world"
  greeting="Hello",
  audience="world",
)

"rate = %g%% APR" % 3.5                         # "rate = 3.5% APR"
```

One subtlety: to use a tuple as the operand of a conversion in format
string containing only a single conversion, you must wrap the tuple in
a singleton tuple:

```python
"coordinates=%s" % (40.741491, -74.003680)	# error: too many arguments for format string
"coordinates=%s" % ((40.741491, -74.003680),)	# "coordinates=(40.741491, -74.003680)"
```

TODO: specify `%e` and `%f` more precisely.

### Conditional expressions

A conditional expression has the form `a if cond else b`.
It first evaluates the condition `cond`.
If it's true, it evaluates `a` and yields its value;
otherwise it yields the value of `b`.

```grammar {.good}
IfExpr = Test 'if' Test 'else' Test .
```

Example:

```python
"yes" if enabled else "no"
```

### Comprehensions

A comprehension constructs new list or dictionary value by looping
over one or more iterables and evaluating a _body_ expression that produces
successive elements of the result.

A list comprehension consists of a single expression followed by one
or more _clauses_, the first of which must be a `for` clause.
Each `for` clause resembles a `for` statement, and specifies an
iterable operand and a set of variables to be assigned by successive
values of the iterable.
An `if` cause resembles an `if` statement, and specifies a condition
that must be met for the body expression to be evaluated.
A sequence of `for` and `if` clauses acts like a nested sequence of
`for` and `if` statements.

```grammar {.good}
ListComp = '[' Test {CompClause} ']'.
DictComp = '{' Entry {CompClause} '}' .

CompClause = 'for' LoopVariables 'in' Test
           | 'if' Test .

LoopVariables = PrimaryExpr {',' PrimaryExpr} .
```

Examples:

```python
[x*x for x in range(5)]                 # [0, 1, 4, 9, 16]
[x*x for x in range(5) if x%2 == 0]     # [0, 4, 16]
[(x, y) for x in range(5)
        if x%2 == 0
        for y in range(5)
        if y > x]                       # [(0, 1), (0, 2), (0, 3), (0, 4), (2, 3), (2, 4)]
```

A dict comprehension resembles a list comprehension, but its body is a
pair of expressions, `key: value`, separated by a colon,
and its result is a dictionary containing the key/value pairs
for which the body expression was evaluated.
Evaluation fails if the value of any key is unhashable.

As with a `for` loop, the loop variables may exploit compound
assignment:

```python
[x*y+z for (x, y), z in [((2, 3), 5), (("o", 2), "!")]]         # [11, 'oo!']
```

Starlark, following Python 3, does not accept an unparenthesized
tuple or lambda expression as the operand of a `for` clause:

```python
[x*x for x in 1, 2, 3]		# parse error: unexpected comma
[x*x for x in lambda: 0]	# parse error: unexpected lambda
```

Comprehensions in Starlark, again following Python 3, define a new lexical
block, so assignments to loop variables have no effect on variables of
the same name in an enclosing block:

```python
x = 1
_ = [x for x in [2]]            # new variable x is local to the comprehension
print(x)                        # 1
```

The operand of a comprehension's first clause (always a `for`) is
resolved in the lexical block enclosing the comprehension.
In the examples below, identifiers referring to the outer variable
named `x` have been distinguished by subscript.

```python
x₀ = (1, 2, 3)
[x*x for x in x₀]               # [1, 4, 9]
[x*x for x in x₀ if x%2 == 0]   # [4]
```

All subsequent `for` and `if` expressions are resolved within the
comprehension's lexical block, as in this rather obscure example:

```python
x₀ = ([1, 2], [3, 4], [5, 6])
[x*x for x in x₀ for x in x if x%2 == 0]     # [4, 16, 36]
```

which would be more clearly rewritten as:

```python
x = ([1, 2], [3, 4], [5, 6])
[z*z for y in x for z in y if z%2 == 0]     # [4, 16, 36]
```


### Function and method calls

```grammar {.good}
CallSuffix = '(' [Arguments [',']] ')' .

Arguments = Argument {',' Argument} .
Argument  = Test | identifier '=' Test | '*' Test | '**' Test .
```

A value `f` of type `function` or `builtin_function_or_method` may be called using the expression `f(...)`.
Applications may define additional types whose values may be called in the same way.

A method call such as `filename.endswith(".star")` is the composition
of two operations, `m = filename.endswith` and `m(".star")`.
The first, a dot operation, yields a _bound method_, a function value
that pairs a receiver value (the `filename` string) with a choice of
method ([string·endswith](#string·endswith)).

Only built-in or application-defined types may have methods.

See [Functions](#functions) for an explanation of function parameter passing.

### Dot expressions

A dot expression `x.f` selects the attribute `f` (a field or method)
of the value `x`.

Fields are possessed by none of the main Starlark [data types](#data-types),
but some application-defined types have them.
Methods belong to the built-in types `string`, `list`, `dict`, and
`set`, and to many application-defined types.

```grammar {.good}
DotSuffix = '.' identifier .
```

A dot expression fails if the value does not have an attribute of the
specified name.

Use the built-in function `hasattr(x, "f")` to ascertain whether a
value has a specific attribute, or `dir(x)` to enumerate all its
attributes.  The `getattr(x, "f")` function can be used to select an
attribute when the name `"f"` is not known statically.

A dot expression that selects a method typically appears within a call
expression, as in these examples:

```python
["able", "baker", "charlie"].index("baker")     # 1
"banana".count("a")                             # 3
"banana".reverse()                              # error: string has no .reverse field or method
```

But when not called immediately, the dot expression evaluates to a
_bound method_, that is, a method coupled to a specific receiver
value.  A bound method can be called like an ordinary function,
without a receiver argument:

```python
f = "banana".count
f                                               # <built-in method count of string value>
f("a")                                          # 3
f("n")                                          # 2
```

### Index expressions

An index expression `a[i]` yields the `i`th element of an _indexable_
type such as a string, tuple, or list.  The index `i` must be an `int`
value in the range -`n` ≤ `i` < `n`, where `n` is `len(a)`; any other
index results in an error.

```grammar {.good}
SliceSuffix = '[' [Expression] [':' Test [':' Test]] ']' .
```

A valid negative index `i` behaves like the non-negative index `n+i`,
allowing for convenient indexing relative to the end of the
sequence.

```python
"abc"[0]                        # "a"
"abc"[1]                        # "b"
"abc"[-1]                       # "c"

("zero", "one", "two")[0]       # "zero"
("zero", "one", "two")[1]       # "one"
("zero", "one", "two")[-1]      # "two"
```

An index expression `d[key]` may also be applied to a dictionary `d`,
to obtain the value associated with the specified key.  It is an error
if the dictionary contains no such key.

An index expression appearing on the left side of an assignment causes
the specified list or dictionary element to be updated:

```starlark
a = range(3)            # a == [0, 1, 2]
a[2] = 7                # a == [0, 1, 7]

coins["suzie b"] = 100
```

It is a dynamic error to attempt to update an element of an immutable
type, such as a tuple or string, or a frozen value of a mutable type.

### Slice expressions

A slice expression `a[start:stop:stride]` yields a new value containing a
sub-sequence of `a`, which must be a string, tuple, or list.

```grammar {.good}
SliceSuffix = '[' [Expression] [':' Test [':' Test]] ']' .
```

Each of the `start`, `stop`, and `stride` operands is optional;
if present, and not `None`, each must be an integer.
The `stride` value defaults to 1.
If the stride is not specified, the colon preceding it may be omitted too.
It is an error to specify a stride of zero.

Conceptually, these operands specify a sequence of values `i` starting
at `start` and successively adding `stride` until `i` reaches or
passes `stop`. The result consists of the concatenation of values of
`a[i]` for which `i` is valid.`

The effective start and stop indices are computed from the three
operands as follows.  Let `n` be the length of the sequence.

<b>If the stride is positive:</b>
If the `start` operand was omitted, it defaults to -infinity.
If the `end` operand was omitted, it defaults to +infinity.
For either operand, if a negative value was supplied, `n` is added to it.
The `start` and `end` values are then "clamped" to the
nearest value in the range 0 to `n`, inclusive.

<b>If the stride is negative:</b>
If the `start` operand was omitted, it defaults to +infinity.
If the `end` operand was omitted, it defaults to -infinity.
For either operand, if a negative value was supplied, `n` is added to it.
The `start` and `end` values are then "clamped" to the
nearest value in the range -1 to `n`-1, inclusive.

```python
"abc"[1:]               # "bc"  (remove first element)
"abc"[:-1]              # "ab"  (remove last element)
"abc"[1:-1]             # "b"   (remove first and last element)
"banana"[1::2]          # "aaa" (select alternate elements starting at index 1)
"banana"[4::-2]         # "nnb" (select alternate elements in reverse, starting at index 4)
```

Unlike Python, Starlark does not allow a slice expression on the left
side of an assignment.

Slicing a tuple or string may be more efficient than slicing a list
because tuples and strings are immutable, so the result of the
operation can share the underlying representation of the original
operand (when the stride is 1). By contrast, slicing a list requires
the creation of a new list and copying of the necessary elements.

<!-- TODO tighten up this section -->

### Lambda expressions

A `lambda` expression yields a new function value.

```grammar {.good}
LambdaExpr = 'lambda' [Parameters] ':' Test .

Parameters = Parameter {',' Parameter} .
Parameter  = identifier
           | identifier '=' Test
           | '*'
           | '*' identifier
           | '**' identifier
           .
```

Syntactically, a lambda expression consists of the keyword `lambda`,
followed by a parameter list like that of a `def` statement but
unparenthesized, then a colon `:`, and a single expression, the
_function body_.

Example:

```python
def map(f, list):
    return [f(x) for x in list]

map(lambda x: 2*x, range(3))    # [0, 2, 4]
```

As with functions created by a `def` statement, a lambda function
captures the syntax of its body, the default values of any optional
parameters, the value of each free variable appearing in its body, and
the global dictionary of the current module.

The name of a function created by a lambda expression is `"lambda"`.

The two statements below are essentially equivalent, but the
function created by the `def` statement is named `twice` and the
function created by the lambda expression is named `lambda`.

```python
def twice(x):
   return x * 2

twice = lambda x: x * 2
```

## Statements

```grammar {.good}
Statement  = DefStmt | IfStmt | ForStmt | SimpleStmt .
SimpleStmt = SmallStmt {';' SmallStmt} [';'] '\n' .
SmallStmt  = ReturnStmt
           | BreakStmt | ContinueStmt | PassStmt
           | AssignStmt
           | ExprStmt
           | LoadStmt
           .
```

### Pass statements

A `pass` statement does nothing.  Use a `pass` statement when the
syntax requires a statement but no behavior is required, such as the
body of a function that does nothing.

```grammar {.good}
PassStmt = 'pass' .
```

Example:

```python
def noop():
   pass

def list_to_dict(items):
  # Convert list of tuples to dict
  m = {}
  for k, m[k] in items:
    pass
  return m
```

### Assignments

An assignment statement has the form `lhs = rhs`.  It evaluates the
expression on the right-hand side then assigns its value (or values) to
the variable (or variables) on the left-hand side.

```grammar {.good}
AssignStmt = Expression '=' Expression .
```

The expression on the left-hand side is called a _target_.  The
simplest target is the name of a variable, but a target may also have
the form of an index expression, to update the element of a list or
dictionary, or a dot expression, to update the field of an object:

```python
k = 1
a[i] = v
m.f = ""
```

Compound targets may consist of a comma-separated list of
subtargets, optionally surrounded by parentheses or square brackets,
and targets may be nested arbitrarily in this way.
An assignment to a compound target checks that the right-hand value is a
sequence with the same number of elements as the target.
Each element of the sequence is then assigned to the corresponding
element of the target, recursively applying the same logic.

```python
pi, e = 3.141, 2.718
(x, y) = f()
[zero, one, two] = range(3)

[(a, b), (c, d)] = {"a": "b", "c": "d"}.items()
a, b = {"a": 1, "b": 2}
```

The same process for assigning a value to a target expression is used
in `for` loops and in comprehensions.


### Augmented assignments

An augmented assignment, which has the form `lhs op= rhs` updates the
variable `lhs` by applying a binary arithmetic operator `op` (one of
`+`, `-`, `*`, `/`, `//`, `%`, `&`, `|`, `^`, `<<`, `>>`) to the previous
value of `lhs` and the value of `rhs`.

```grammar {.good}
AssignStmt = Expression ('+=' | '-=' | '*=' | '/=' | '//=' | '%=' | '&=' | '|=' | '^=' | '<<=' | '>>=') Expression .
```

The left-hand side must be a simple target:
a name, an index expression, or a dot expression.

```python
x -= 1
x.filename += ".star"
a[index()] *= 2
```

Any subexpressions in the target on the left-hand side are evaluated
exactly once, before the evaluation of `rhs`.
The first two assignments above are thus equivalent to:

```python
x = x - 1
x.filename = x.filename + ".star"
```

and the third assignment is similar in effect to the following two
statements but does not declare a new temporary variable `i`:

```python
i = index()
a[i] = a[i] * 2
```

### Function definitions

A `def` statement creates a named function and assigns it to a variable.

```grammar {.good}
DefStmt = 'def' identifier '(' [Parameters [',']] ')' ':' Suite .
```

Example:

```python
def twice(x):
    return x * 2

str(twice)              # "<function twice>"
twice(2)                # 4
twice("two")            # "twotwo"
```

The function's name is preceded by the `def` keyword and followed by
the parameter list (which is enclosed in parentheses), a colon, and
then an indented block of statements which form the body of the function.

The parameter list is a comma-separated list whose elements are of
several kinds.  First come zero or more required parameters, which are
simple identifiers; all calls must provide an argument value for these parameters.

The required parameters are followed by zero or more optional
parameters, of the form `name=expression`.  The expression specifies
the default value for the parameter for use in calls that do not
provide an argument value for it.

The required parameters are optionally followed by a single parameter
name preceded by a `*`.  This is the called the _varargs_ parameter,
and it accumulates surplus positional arguments specified by a call.
It is conventionally named `*args`.

The varargs parameter may be followed by zero or more
parameters, again of the forms `name` or `name=expression`,
but these parameters differ from earlier ones in that they are
_keyword-only_: if a call provides their values, it must do so as
keyword arguments, not positional ones.

```python
def f(a, *, b=2, c):
  print(a, b, c)

f(1)                    # error: function f missing 1 argument (c)
f(1, 3)                 # error: function f accepts 1 positional argument (2 given)
f(1, c=3)               # "1 2 3"

def g(a, *args, b=2, c):
  print(a, b, c, args)

g(1, 3)                 # error: function g missing 1 argument (c)
g(1, 4, c=3)            # "1 2 3 (4,)"

```

A non-variadic function may also declare keyword-only parameters,
by using a bare `*` in place of the `*args` parameter.
This form does not declare a parameter but marks the boundary
between the earlier parameters and the keyword-only parameters.
This form must be followed by at least one optional parameter.

Finally, there may be an optional parameter name preceded by `**`.
This is called the _keyword arguments_ parameter, and accumulates in a
dictionary any surplus `name=value` arguments that do not match a
prior parameter. It is conventionally named `**kwargs`.

The final parameter may be followed by a trailing comma.

Here are some example parameter lists:

```python
def f(): pass
def f(a, b, c): pass
def f(a, b, c=1): pass
def f(a, b, c=1, *args): pass
def f(a, b, c=1, *args, **kwargs): pass
def f(**kwargs): pass
def f(a, b, c=1, *, d=1): pass

def f(
  a,
  *args,
  **kwargs,
)
```

Execution of a `def` statement creates a new function object.  The
function object contains: the syntax of the function body; the default
value for each optional parameter; the value of each free variable
referenced within the function body; and the global dictionary of the
current module.

<!-- this is too implementation-oriented; it's not a spec. -->


### Return statements

A `return` statement ends the execution of a function and returns a
value to the caller of the function.

```grammar {.good}
ReturnStmt = 'return' [Expression] .
```

A return statement may have zero, one, or more
result expressions separated by commas.
With no expressions, the function has the result `None`.
With a single expression, the function's result is the value of that expression.
With multiple expressions, the function's result is a tuple.

```python
return                  # returns None
return 1                # returns 1
return 1, 2             # returns (1, 2)
```

### Expression statements

An expression statement evaluates an expression and discards its result.

```grammar {.good}
ExprStmt = Expression .
```

Any expression may be used as a statement, but an expression statement is
most often used to call a function for its side effects.

```python
list.append(1)
```

### If statements

An `if` statement evaluates an expression (the _condition_), then, if
the truth value of the condition is `True`, executes a list of
statements.

```grammar {.good}
IfStmt = 'if' Test ':' Suite {'elif' Test ':' Suite} ['else' ':' Suite] .
```

Example:

```python
if score >= 100:
    print("You win!")
    return
```

An `if` statement may have an `else` block defining a second list of
statements to be executed if the condition is false.

```python
if score >= 100:
        print("You win!")
        return
else:
        print("Keep trying...")
        continue
```

It is common for the `else` block to contain another `if` statement.
To avoid increasing the nesting depth unnecessarily, the `else` and
following `if` may be combined as `elif`:

```python
if x > 0:
        result = +1
elif x < 0:
        result = -1
else:
        result = 0
```

An `if` statement is permitted only within a function definition.
An `if` statement at top level results in a static error.

<b>Implementation note:</b>
The Go implementation of Starlark permits `if`-statements to appear at top level
if the `-globalreassign` flag is enabled.


### While loops

A `while` loop evaluates an expression (the _condition_) and if the truth
value of the condition is `True`, it executes a list of statement and repeats
the process until the truth value of the condition becomes `False`.

```grammar {.good}
WhileStmt = 'while' Test ':' Suite .
```

Example:

```python
while n > 0:
    r = r + n
    n = n - 1
```

A `while` statement is permitted only within a function definition.
A `while` statement at top level results in a static error.

<b>Implementation note:</b>
The Go implementation of Starlark permits `while` loops only if the `-recursion` flag is enabled.
A `while` statement is permitted at top level if the `-globalreassign` flag is enabled.


### For loops

A `for` loop evaluates its operand, which must be an iterable value.
Then, for each element of the iterable's sequence, the loop assigns
the successive element values to one or more variables and executes a
list of statements, the _loop body_.

```grammar {.good}
ForStmt = 'for' LoopVariables 'in' Expression ':' Suite .
```

Example:

```python
for x in range(10):
   print(10)
```

The assignment of each value to the loop variables follows the same
rules as an ordinary assignment.  In this example, two-element lists
are repeatedly assigned to the pair of variables (a, i):

```python
for a, i in [["a", 1], ["b", 2], ["c", 3]]:
  print(a, i)                          # prints "a 1", "b 2", "c 3"
```

Because Starlark loops always iterate over a finite sequence, they are
guaranteed to terminate, unlike loops in most languages which can
execute an arbitrary and perhaps unbounded number of iterations.

Within the body of a `for` loop, `break` and `continue` statements may
be used to stop the execution of the loop or advance to the next
iteration.

In Starlark, a `for` loop is permitted only within a function definition.
A `for` loop at top level results in a static error.

<b>Implementation note:</b>
The Go implementation of Starlark permits loops to appear at top level
if the `-globalreassign` flag is enabled.


### Break and Continue

The `break` and `continue` statements terminate the current iteration
of a `for` loop.  Whereas the `continue` statement resumes the loop at
the next iteration, a `break` statement terminates the entire loop.

```grammar {.good}
BreakStmt    = 'break' .
ContinueStmt = 'continue' .
```

Example:

```python
for x in range(10):
    if x%2 == 1:
        continue        # skip odd numbers
    if x > 7:
        break           # stop at 8
    print(x)            # prints "0", "2", "4", "6"
```

Both statements affect only the innermost lexically enclosing loop.
It is a static error to use a `break` or `continue` statement outside a
loop.


### Load statements

The `load` statement loads another Starlark module, extracts one or
more values from it, and binds them to names in the current module.

<!--
The awkwardness of load statements is a consequence of staying a
strict subset of Python syntax, which allows reuse of existing tools
such as editor support. Python import statements are inadequate for
Starlark because they don't allow arbitrary file names for module names.
-->

Syntactically, a load statement looks like a function call `load(...)`.

```grammar {.good}
LoadStmt = 'load' '(' string {',' [identifier '='] string} [','] ')' .
```

A load statement requires at least two "arguments".
The first must be a literal string; it identifies the module to load.
Its interpretation is determined by the application into which the
Starlark interpreter is embedded, and is not specified here.

During execution, the application determines what action to take for a
load statement.
A typical implementation locates and executes a Starlark file,
populating a cache of files executed so far to avoid duplicate work,
to obtain a module, which is a mapping from global names to values.

The remaining arguments are a mixture of literal strings, such as
`"x"`, or named literal strings, such as `y="x"`.

The literal string (`"x"`), which must denote a valid identifier not
starting with `_`, specifies the name to extract from the loaded
module.  In effect, names starting with `_` are not exported.
The name (`y`) specifies the local name;
if no name is given, the local name matches the quoted name.

```python
load("module.star", "x", "y", "z")       # assigns x, y, and z
load("module.star", "x", y2="y", "z")    # assigns x, y2, and z
```

A load statement may not be nested inside any other statement.


## Module execution

Each Starlark file defines a _module_, which is a mapping from the
names of global variables to their values.
When a Starlark file is executed, whether directly by the application
or indirectly through a `load` statement, a new Starlark thread is
created, and this thread executes all the top-level statements in the
file.
Because if-statements and for-loops cannot appear outside of a function,
control flows from top to bottom.

If execution reaches the end of the file, module initialization is
successful.
At that point, the value of each of the module's global variables is
frozen, rendering subsequent mutation impossible.
The module is then ready for use by another Starlark thread, such as
one executing a load statement.
Such threads may access values or call functions defined in the loaded
module.

A Starlark thread may carry state on behalf of the application into
which it is embedded, and application-defined functions may behave
differently depending on this thread state.
Because module initialization always occurs in a new thread, thread
state is never carried from a higher-level module into a lower-level
one.
The initialization behavior of a module is thus independent of
whichever module triggered its initialization.

If a Starlark thread encounters an error, execution stops and the error
is reported to the application, along with a backtrace showing the
stack of active function calls at the time of the error.
If an error occurs during initialization of a Starlark module, any
active `load` statements waiting for initialization of the module also
fail.

Starlark provides no mechanism by which errors can be handled within
the language.


## Built-in constants and functions

The outermost block of the Starlark environment is known as the "predeclared" block.
It defines a number of fundamental values and functions needed by all Starlark programs,
such as `None`, `True`, `False`, and `len`, and possibly additional
application-specific names.

These names are not reserved words so Starlark programs are free to
redefine them in a smaller block such as a function body or even at
the top level of a module.  However, doing so may be confusing to the
reader.  Nonetheless, this rule permits names to be added to the
predeclared block in later versions of the language (or
application-specific dialect) without breaking existing programs.


### None

`None` is the distinguished value of the type `NoneType`.

### True and False

`True` and `False` are the two values of type `bool`.

### abs

`abs(x)` returns the absolute value of its argument `x`, which must be an int or float.
The result has the same type as `x`.

### any

`any(x)` returns `True` if any element of the iterable sequence x has a truth value of true.
If the iterable is empty, it returns `False`.

### all

`all(x)` returns `False` if any element of the iterable sequence x has a truth value of false.
If the iterable is empty, it returns `True`.

### bool

`bool(x)` interprets `x` as a Boolean value---`True` or `False`.
With no argument, `bool()` returns `False`.


### chr

`chr(i)` returns a string that encodes the single Unicode code point
whose value is specified by the integer `i`. `chr` fails unless 0 ≤
`i` ≤ 0x10FFFF.

Example:

```python
chr(65)                         # "A",
chr(1049)                       # "Й", CYRILLIC CAPITAL LETTER SHORT I
chr(0x1F63F)                    # "😿", CRYING CAT FACE
```

See also: `ord`.

<b>Implementation note:</b> `chr` is not provided by the Java implementation.

### dict

`dict` creates a dictionary.  It accepts up to one positional
argument, which is interpreted as an iterable of two-element
sequences (pairs), each specifying a key/value pair in
the resulting dictionary.

`dict` also accepts any number of keyword arguments, each of which
specifies a key/value pair in the resulting dictionary;
each keyword is treated as a string.

```python
dict()                          # {}, empty dictionary
dict([(1, 2), (3, 4)])          # {1: 2, 3: 4}
dict([(1, 2), ["a", "b"]])      # {1: 2, "a": "b"}
dict(one=1, two=2)              # {"one": 1, "two", 1}
dict([(1, 2)], x=3)             # {1: 2, "x": 3}
```

With no arguments, `dict()` returns a new empty dictionary.

`dict(x)` where x is a dictionary returns a new copy of x.

### dir

`dir(x)` returns a new sorted list of the names of the attributes (fields and methods) of its operand.
The attributes of a value `x` are the names `f` such that `x.f` is a valid expression.

For example,

```python
dir("hello")                    # ['capitalize', 'count', ...], the methods of a string
```

Several types known to the interpreter, such as list, string, and dict, have methods, but none have fields.
However, an application may define types with fields that may be read or set by statements such as these:

```text
y = x.f
x.f = y
```

### enumerate

`enumerate(x)` returns a list of (index, value) pairs, each containing
successive values of the iterable sequence xand the index of the value
within the sequence.

The optional second parameter, `start`, specifies an integer value to
add to each index.

```python
enumerate(["zero", "one", "two"])               # [(0, "zero"), (1, "one"), (2, "two")]
enumerate(["one", "two"], 1)                    # [(1, "one"), (2, "two")]
```

### fail

The `fail(*args, sep=" ")` function causes execution to fail
with the specified error message.
Like `print`, arguments are formatted as if by `str(x)` and
separated by a space, unless an alternative separator is
specified by a `sep` named argument.

```python
fail("oops")				# "fail: oops"
fail("oops", 1, False, sep='/')		# "fail: oops/1/False"
```

### float

`float(x)` interprets its argument as a floating-point number.

If x is a `float`, the result is x.
if x is an `int`, the result is the nearest floating point value to x.
If x is a string, the string is interpreted as a floating-point literal.
With no arguments, `float()` returns `0.0`.


### getattr

`getattr(x, name)` returns the value of the attribute (field or method) of x named `name`.
It is a dynamic error if x has no such attribute.

`getattr(x, "f")` is equivalent to `x.f`.

```python
getattr("banana", "split")("a")	       # ["b", "n", "n", ""], equivalent to "banana".split("a")
```

The three-argument form `getattr(x, name, default)` returns the
provided `default` value instead of failing.

### hasattr

`hasattr(x, name)` reports whether x has an attribute (field or method) named `name`.

### hash

`hash(x)` returns an integer hash of a string x
such that two equal strings have the same hash.
In other words `x == y` implies `hash(x) == hash(y)`.

In the interests of reproducibility of Starlark program behavior over time and
across implementations, the specific hash function is the same as that implemented by
[java.lang.String.hashCode](https://docs.oracle.com/javase/7/docs/api/java/lang/String.html#hashCode),
a simple polynomial accumulator over the UTF-16 transcoding of the string:
 ```
s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
```

`hash` fails if given a non-string operand,
even if the value is hashable and thus suitable as the key of dictionary.

### int

`int(x[, base])` interprets its argument as an integer.

If x is an `int`, the result is x.
If x is a `float`, the result is the integer value nearest to x,
truncating towards zero; it is an error if x is not finite (`NaN`,
`+Inf`, `-Inf`).
If x is a `bool`, the result is 0 for `False` or 1 for `True`.

If x is a string, it is interpreted as a sequence of digits in the
specified base, decimal by default.
If `base` is zero, x is interpreted like an integer literal, the base
being inferred from an optional base prefix such as `0b`, `0o`, or
`0x` preceding the first digit.
When the `base` is provided explicitly, a matching base prefix is
also permitted, and has no effect.
Irrespective of base, the string may start with an optional `+` or `-`
sign indicating the sign of the result.

```python
int("11")               # 11
int("11", 0)            # 11
int("11", 10)           # 11
int("11", 2)            # 3
int("11", 8)            # 9
int("11", 16)           # 17

int("0x11", 0)          # 17
int("0x11", 16)         # 17
int("0b1", 16)          # 177 (0xb1)
int("0b1", 2)           # 1
int("0b1", 0)           # 1

int("0x11")             # error: invalid literal with base 10
```

### len

`len(x)` returns the number of elements in its argument.

It is a dynamic error if its argument is not a sequence.

### list

`list` constructs a list.

`list(x)` returns a new list containing the elements of the
iterable sequence x.

With no argument, `list()` returns a new empty list.

### max

`max(x)` returns the greatest element in the iterable sequence x.

It is an error if any element does not support ordered comparison,
or if the sequence is empty.

The optional named parameter `key` specifies a function to be applied
to each element prior to comparison.

```python
max([3, 1, 4, 1, 5, 9])                         # 9
max("two", "three", "four")                     # "two", the lexicographically greatest
max("two", "three", "four", key=len)            # "three", the longest
```

### min

`min(x)` returns the least element in the iterable sequence x.

It is an error if any element does not support ordered comparison,
or if the sequence is empty.

```python
min([3, 1, 4, 1, 5, 9])                         # 1
min("two", "three", "four")                     # "four", the lexicographically least
min("two", "three", "four", key=len)            # "two", the shortest
```


### ord

`ord(s)` returns the integer value of the sole Unicode code point encoded by the string `s`.

If `s` does not encode exactly one Unicode code point, `ord` fails.
Each invalid code within the string is treated as if it encodes the
Unicode replacement character, U+FFFD.

Example:

```python
ord("A")				# 65
ord("Й")				# 1049
ord("😿")					# 0x1F63F
ord("Й"[1:])				# 0xFFFD (Unicode replacement character)
```

See also: `chr`.

<b>Implementation note:</b> `ord` is not provided by the Java implementation.

### print

`print(*args, sep=" ")` prints its arguments, followed by a newline.
Arguments are formatted as if by `str(x)` and separated with a space,
unless an alternative separator is specified by a `sep` named argument.

Example:

```python
print(1, "hi")		       		# "1 hi\n"
print("hello", "world")			# "hello world\n"
print("hello", "world", sep=", ")	# "hello, world\n"
```

Typically the formatted string is printed to the standard error file,
but the exact behavior is a property of the Starlark thread and is
determined by the host application.

### range

`range` returns an immutable sequence of integers defined by the specified interval and stride.

```python
range(stop)                             # equivalent to range(0, stop)
range(start, stop)                      # equivalent to range(start, stop, 1)
range(start, stop, step)
```

`range` requires between one and three integer arguments.
With one argument, `range(stop)` returns the ascending sequence of non-negative integers less than `stop`.
With two arguments, `range(start, stop)` returns only integers not less than `start`.

With three arguments, `range(start, stop, step)` returns integers
formed by successively adding `step` to `start` until the value meets or passes `stop`.
A call to `range` fails if the value of `step` is zero.

A call to `range` does not materialize the entire sequence, but
returns a fixed-size value of type `"range"` that represents the
parameters that define the sequence.
The `range` value is iterable and may be indexed efficiently.

```python
list(range(10))                         # [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
list(range(3, 10))                      # [3, 4, 5, 6, 7, 8, 9]
list(range(3, 10, 2))                   # [3, 5, 7, 9]
list(range(10, 3, -2))                  # [10, 8, 6, 4]
```

The `len` function applied to a `range` value returns its length.
The truth value of a `range` value is `True` if its length is non-zero.

Range values are comparable: two `range` values compare equal if they
denote the same sequence of integers, even if they were created using
different parameters.

Range values are not hashable.  <!-- should they be? -->

The `str` function applied to a `range` value yields a string of the
form `range(10)`, `range(1, 10)`, or `range(1, 10, 2)`.

The `x in y` operator, where `y` is a range, reports whether `x` is equal to
some member of the sequence `y`; the operation fails unless `x` is a
number.

### repr

`repr(x)` formats its argument as a string.

All strings in the result are double-quoted.

```python
repr(1)                 # '1'
repr("x")               # '"x"'
repr([1, "x"])          # '[1, "x"]'
```

### reversed

`reversed(x)` returns a new list containing the elements of the iterable sequence x in reverse order.

```python
reversed(range(5))                              # [4, 3, 2, 1, 0]
reversed("stressed".codepoints())               # ["d", "e", "s", "s", "e", "r", "t", "s"]
reversed({"one": 1, "two": 2}.keys())           # ["two", "one"]
```

### set

`set(x)` returns a new set containing the elements of the iterable x.
With no argument, `set()` returns a new empty set.

```python
set([3, 1, 4, 1, 5, 9])         # set([3, 1, 4, 5, 9])
```

<b>Implementation note:</b>
Sets are an optional feature of the Go implementation of Starlark,
enabled by the `-set` flag.


### sorted

`sorted(x)` returns a new list containing the elements of the iterable sequence x,
in sorted order.  The sort algorithm is stable.

The optional named parameter `reverse`, if true, causes `sorted` to
return results in reverse sorted order.

The optional named parameter `key` specifies a function of one
argument to apply to obtain the value's sort key.
The default behavior is the identity function.

```python
sorted(set("harbors".codepoints()))                             # ['a', 'b', 'h', 'o', 'r', 's']
sorted([3, 1, 4, 1, 5, 9])                                      # [1, 1, 3, 4, 5, 9]
sorted([3, 1, 4, 1, 5, 9], reverse=True)                        # [9, 5, 4, 3, 1, 1]

sorted(["two", "three", "four"], key=len)                       # ["two", "four", "three"], shortest to longest
sorted(["two", "three", "four"], key=len, reverse=True)         # ["three", "four", "two"], longest to shortest
```


### str

`str(x)` formats its argument as a string.

If x is a string, the result is x (without quotation).
All other strings, such as elements of a list of strings, are double-quoted.

```python
str(1)                          # '1'
str("x")                        # 'x'
str([1, "x"])                   # '[1, "x"]'
```

### tuple

`tuple(x)` returns a tuple containing the elements of the iterable x.

With no arguments, `tuple()` returns the empty tuple.

### type

type(x) returns a string describing the type of its operand.

```python
type(None)              # "NoneType"
type(0)                 # "int"
type(0.0)               # "float"
```

### zip

`zip()` returns a new list of n-tuples formed from corresponding
elements of each of the n iterable sequences provided as arguments to
`zip`.  That is, the first tuple contains the first element of each of
the sequences, the second tuple contains the second element of each
of the sequences, and so on.  The result list is only as long as the
shortest of the input sequences.

```python
zip()                                   # []
zip(range(5))                           # [(0,), (1,), (2,), (3,), (4,)]
zip(range(5), "abc".elems())            # [(0, "a"), (1, "b"), (2, "c")]
```

## Built-in methods

This section lists the methods of built-in types.  Methods are selected
using [dot expressions](#dot-expressions).
For example, strings have a `count` method that counts
occurrences of a substring; `"banana".count("a")` yields `3`.

As with built-in functions, built-in methods accept only positional
arguments except where noted.
The parameter names serve merely as documentation.


<a id='dict·clear'></a>
### dict·clear

`D.clear()` removes all the entries of dictionary D and returns `None`.
It fails if the dictionary is frozen or if there are active iterators.

```python
x = {"one": 1, "two": 2}
x.clear()                               # None
print(x)                                # {}
```

<a id='dict·get'></a>
### dict·get

`D.get(key[, default])` returns the dictionary value corresponding to the given key.
If the dictionary contains no such value, `get` returns `None`, or the
value of the optional `default` parameter if present.

`get` fails if `key` is unhashable, or the dictionary is frozen or has active iterators.

```python
x = {"one": 1, "two": 2}
x.get("one")                            # 1
x.get("three")                          # None
x.get("three", 0)                       # 0
```

<a id='dict·items'></a>
### dict·items

`D.items()` returns a new list of key/value pairs, one per element in
dictionary D, in the same order as they would be returned by a `for` loop.

```python
x = {"one": 1, "two": 2}
x.items()                               # [("one", 1), ("two", 2)]
```

<a id='dict·keys'></a>
### dict·keys

`D.keys()` returns a new list containing the keys of dictionary D, in the
same order as they would be returned by a `for` loop.

```python
x = {"one": 1, "two": 2}
x.keys()                               # ["one", "two"]
```

<a id='dict·pop'></a>
### dict·pop

`D.pop(key[, default])` returns the value corresponding to the specified
key, and removes it from the dictionary.  If the dictionary contains no
such value, and the optional `default` parameter is present, `pop`
returns that value; otherwise, it fails.

`pop` fails if `key` is unhashable, or the dictionary is frozen or has active iterators.

```python
x = {"one": 1, "two": 2}
x.pop("one")                            # 1
x                                       # {"two": 2}
x.pop("three", 0)                       # 0
x.pop("four")                           # error: missing key
```

<a id='dict·popitem'></a>
### dict·popitem

`D.popitem()` returns the first key/value pair, removing it from the dictionary.

`popitem` fails if the dictionary is empty, frozen, or has active iterators.

```python
x = {"one": 1, "two": 2}
x.popitem()                             # ("one", 1)
x.popitem()                             # ("two", 2)
x.popitem()                             # error: empty dict
```

<a id='dict·setdefault'></a>
### dict·setdefault

`D.setdefault(key[, default])` returns the dictionary value corresponding to the given key.
If the dictionary contains no such value, `setdefault`, like `get`,
returns `None` or the value of the optional `default` parameter if
present; `setdefault` additionally inserts the new key/value entry into the dictionary.

`setdefault` fails if the key is unhashable, or if the dictionary is frozen or has active iterators.

```python
x = {"one": 1, "two": 2}
x.setdefault("one")                     # 1
x.setdefault("three", 0)                # 0
x                                       # {"one": 1, "two": 2, "three": 0}
x.setdefault("four")                    # None
x                                       # {"one": 1, "two": 2, "three": None}
```

<a id='dict·update'></a>
### dict·update

`D.update([pairs][, name=value[, ...])` makes a sequence of key/value
insertions into dictionary D, then returns `None.`

If the positional argument `pairs` is present, it must be `None`,
another `dict`, or some other iterable.
If it is another `dict`, then its key/value pairs are inserted into D.
If it is an iterable, it must provide a sequence of pairs (or other iterables of length 2),
each of which is treated as a key/value pair to be inserted into D.

For each `name=value` argument present, the name is converted to a
string and used as the key for an insertion into D, with its corresponding
value being `value`.

`update` fails if the dictionary is frozen or has active iterators.

```python
x = {}
x.update([("a", 1), ("b", 2)], c=3)
x.update({"d": 4})
x.update(e=5)
x                                       # {"a": 1, "b": "2", "c": 3, "d": 4, "e": 5}
```

<a id='dict·values'></a>
### dict·values

`D.values()` returns a new list containing the dictionary's values, in the
same order as they would be returned by a `for` loop over the
dictionary.

```python
x = {"one": 1, "two": 2}
x.values()                              # [1, 2]
```

<a id='list·append'></a>
### list·append

`L.append(x)` appends `x` to the list L, and returns `None`.

`append` fails if the list is frozen or has active iterators.

```python
x = []
x.append(1)                             # None
x.append(2)                             # None
x.append(3)                             # None
x                                       # [1, 2, 3]
```

<a id='list·clear'></a>
### list·clear

`L.clear()` removes all the elements of the list L and returns `None`.
It fails if the list is frozen or if there are active iterators.

```python
x = [1, 2, 3]
x.clear()                               # None
x                                       # []
```

<a id='list·extend'></a>
### list·extend

`L.extend(x)` appends the elements of `x`, which must be iterable, to
the list L, and returns `None`.

`extend` fails if `x` is not iterable, or if the list L is frozen or has active iterators.

```python
x = []
x.extend([1, 2, 3])                     # None
x.extend(["foo"])                       # None
x                                       # [1, 2, 3, "foo"]
```

<a id='list·index'></a>
### list·index

`L.index(x[, start[, end]])` finds `x` within the list L and returns its index.

The optional `start` and `end` parameters restrict the portion of
list L that is inspected.  If provided and not `None`, they must be list
indices of type `int`. If an index is negative, `len(L)` is effectively
added to it, then if the index is outside the range `[0:len(L)]`, the
nearest value within that range is used; see [Indexing](#indexing).

`index` fails if `x` is not found in L, or if `start` or `end`
is not a valid index (`int` or `None`).

```python
x = list("banana".codepoints())
x.index("a")                            # 1 (bAnana)
x.index("a", 2)                         # 3 (banAna)
x.index("a", -2)                        # 5 (bananA)
```

<a id='list·insert'></a>
### list·insert

`L.insert(i, x)` inserts the value `x` in the list L at index `i`, moving
higher-numbered elements along by one.  It returns `None`.

As usual, the index `i` must be an `int`. If its value is negative,
the length of the list is added, then its value is clamped to the
nearest value in the range `[0:len(L)]` to yield the effective index.

`insert` fails if the list is frozen or has active iterators.

```python
x = ["b", "c", "e"]
x.insert(0, "a")                        # None
x.insert(-1, "d")                       # None
x                                       # ["a", "b", "c", "d", "e"]
```

<a id='list·pop'></a>
### list·pop

`L.pop([index])` removes and returns the last element of the list L, or,
if the optional index is provided, at that index.

`pop` fails if the index is not valid for `L[i]`,
or if the list is frozen or has active iterators.

```python
x = [1, 2, 3, 4, 5]
x.pop()                                 # 5
x                                       # [1, 2, 3, 4]
x.pop(-2)                               # 3
x                                       # [1, 2, 4]
x.pop(-3)                               # 1
x                                       # [2, 4]
x.pop()                                 # 4
x                                       # [2]
```

<a id='list·remove'></a>
### list·remove

`L.remove(x)` removes the first occurrence of the value `x` from the list L, and returns `None`.

`remove` fails if the list does not contain `x`, is frozen, or has active iterators.

```python
x = [1, 2, 3, 2]
x.remove(2)                             # None (x == [1, 3, 2])
x.remove(2)                             # None (x == [1, 3])
x.remove(2)                             # error: element not found
```

<a id='set·add'></a>
### set·add

`S.add(x)` adds the value `x` to the set `S`. It returns `None`.

It is permissible to `add` a value already present in the set; this leaves the
set `S` unchanged.

`add` fails if the set `S` is frozen or has active iterators, or if `x` is
unhashable.

```python
x = set([1, 2])
x.add(3)                             # None
x                                    # set([1, 2, 3])
x.add(3)                             # None
x                                    # set([1, 2, 3])
```

<a id='set·clear'></a>
### set·clear

`S.clear()` removes all items from the set or fails if the set is non-empty and frozen.

It returns None.

```python
x = set([1, 2, 3])
x.clear(2)                               # None
x                                        # set([])
```

<a id='set·difference'></a>
### set·difference

`S.difference(y)` returns a new set into which have been inserted all the elements of set S which are not in y.

y can be any type of iterable (e.g. set, list, tuple).

```python
x = set([1, 2, 3])
x.difference([3, 4, 5])                   # set([1, 2])
```

<a id='set·discard'></a>
### set·discard

`S.discard(x)` removes the value `x` from the set `S` if present. It returns
`None`.

It is permissible to `discard` a value not present in the set; this leaves the
set `S` unchanged.

`discard` fails if the set `S` is frozen or has active iterators, or if `x` is
unhashable. This applies even if `x` is not a member of the set.

```python
x = set([1, 2, 3])
x.discard(2)                             # None
x                                        # set([1, 3])
x.discard(2)                             # None
x                                        # set([1, 3])
```

<a id='set·intersection'></a>
### set·intersection

`S.intersection(y)` returns a new set into which have been inserted all the elements of set S which are also in y.

y can be any type of iterable (e.g. set, list, tuple).

```python
x = set([1, 2, 3])
x.intersection([3, 4, 5])                # set([3])
```

<a id='set·issubset'></a>
### set·issubset

`S.issubset(y)` returns True if all items in S are also in y, otherwise it returns False.

y can be any type of iterable (e.g. set, list, tuple).

```python
x = set([1, 2])
x.issubset([1, 2, 3])                # True
x.issubset([1, 3, 4])                # False
```

<a id='set·issuperset'></a>
### set·issuperset

`S.issuperset(y)` returns True if all items in y are also in S, otherwise it returns False.

y can be any type of iterable (e.g. set, list, tuple).

```python
x = set([1, 2, 3])
x.issuperset([1, 2])                 # True
x.issuperset([1, 3, 4])              # False
```

<a id='set·pop'></a>
### set·pop

`S.pop()` removes the first inserted item from the set and returns it.

`pop` fails if the set is empty or frozen.

```python
x = set([1, 2])
x.pop()                                 # 1
x.pop()                                 # 2
x.pop()                                 # error: empty set
```

<a id='set·remove'></a>
### set·remove

`S.remove(x)` removes `x` from the set and returns None.

`remove` fails if the set does not contain `x` or is frozen.

```python
x = set([1, 2, 3])
x.remove(2)                             # None
x                                       # set([1, 3])
x.remove(2)                             # error: element not found
```

<a id='set·symmetric_difference'></a>
### set·symmetric_difference

`S.symmetric_difference(y)` creates a new set into which is inserted all of the items which are in S but not y, followed by all of the items which are in y but not S.

y can be any type of iterable (e.g. set, list, tuple).

```python
x = set([1, 2, 3])
x.symmetric_difference([3, 4, 5])         # set([1, 2, 4, 5])
```

<a id='set·union'></a>
### set·union

`S.union(iterable...)` returns a new set into which have been inserted
all the elements of set S and each element of the iterable sequences.

`union` fails if any argument is not an iterable sequence, or if any
sequence element is not hashable.

```python
x = set([1, 2])
y = set([2, 3])
x.union(y)                              # set([1, 2, 3])
```

<a id='set·update'></a>
### set·update

`S.update(iterable...)` adds to S each element of the iterable
sequences. The method will return `None`.

```python
x = set([1, 2])
x.update([2, 3], [4, 5])
```

<a id='string·elem_ords'></a>
### string·elem_ords

`S.elem_ords()` returns an iterable value containing the
sequence of numeric bytes values in the string S.

To materialize the entire sequence of bytes, apply `list(...)` to the result.

Example:

```python
list("Hello, 世界".elem_ords())        # [72, 101, 108, 108, 111, 44, 32, 228, 184, 150, 231, 149, 140]
```

See also: `string·elems`.

<b>Implementation note:</b> `elem_ords` is not provided by the Java implementation.

<a id='string·capitalize'></a>
### string·capitalize

`S.capitalize()` returns a copy of string S with its first code point
changed to its title case and all subsequent letters changed to their
lower case.

```python
"hello, world!".capitalize()		# "Hello, world!"
"hElLo, wOrLd!".capitalize()		# "Hello, world!"
"¿Por qué?".capitalize()		# "¿por qué?"
```

<a id='string·codepoint_ords'></a>
### string·codepoint_ords

`S.codepoint_ords()` returns an iterable value containing the
sequence of integer Unicode code points encoded by the string S.
Each invalid code within the string is treated as if it encodes the
Unicode replacement character, U+FFFD.

By returning an iterable, not a list, the cost of decoding the string
is deferred until actually needed; apply `list(...)` to the result to
materialize the entire sequence.

Example:

```python
list("Hello, 世界".codepoint_ords())        # [72, 101, 108, 108, 111, 44, 32, 19990, 30028]

for cp in "Hello, 世界".codepoint_ords():
   print(chr(cp))  # prints 'H', 'e', 'l', 'l', 'o', ',', ' ', '世', '界'
```

See also: `string·codepoints`.

<b>Implementation note:</b> `codepoint_ords` is not provided by the Java implementation.

<a id='string·count'></a>
### string·count

`S.count(sub[, start[, end]])` returns the number of occurrences of
`sub` within the string S, or, if the optional substring indices
`start` and `end` are provided, within the designated substring of S.
They are interpreted according to Starlark's [indexing conventions](#indexing).

```python
"hello, world!".count("o")              # 2
"hello, world!".count("o", 7, 12)       # 1  (in "world")
```

<a id='string·endswith'></a>
### string·endswith

`S.endswith(suffix[, start[, end]])` reports whether the string
`S[start:end]` has the specified suffix.

```python
"filename.star".endswith(".star")         # True
```

The `suffix` argument may be a tuple of strings, in which case the
function reports whether any one of them is a suffix.

```python
'foo.cc'.endswith(('.cc', '.h'))         # True
```


<a id='string·find'></a>
### string·find

`S.find(sub[, start[, end]])` returns the index of the first
occurrence of the substring `sub` within S.

If either or both of `start` or `end` are specified,
they specify a subrange of S to which the search should be restricted.
They are interpreted according to Starlark's [indexing conventions](#indexing).

If no occurrence is found, `found` returns -1.

```python
"bonbon".find("on")             # 1
"bonbon".find("on", 2)          # 4
"bonbon".find("on", 2, 5)       # -1
```

<a id='string·format'></a>
### string·format

`S.format(*args, **kwargs)` returns a version of the format string S
in which bracketed portions `{...}` are replaced
by arguments from `args` and `kwargs`.

Within the format string, a pair of braces `{{` or `}}` is treated as
a literal open or close brace.
Each unpaired open brace must be matched by a close brace `}`.
The optional text between corresponding open and close braces
specifies which argument to use and how to format it, and consists of
three components, all optional:
a field name, a conversion preceded by '`!`', and a format specifier
preceded by '`:`'.

```text
{field}
{field:spec}
{field!conv}
{field!conv:spec}
```

The *field name* may be either a decimal number or a keyword.
A number is interpreted as the index of a positional argument;
a keyword specifies the value of a keyword argument.
If all the numeric field names form the sequence 0, 1, 2, and so on,
they may be omitted and those values will be implied; however,
the explicit and implicit forms may not be mixed.

The *conversion* specifies how to convert an argument value `x` to a
string. It may be either `!r`, which converts the value using
`repr(x)`, or `!s`, which converts the value using `str(x)` and is
the default.

The *format specifier*, after a colon, specifies field width,
alignment, padding, and numeric precision.
Currently it must be empty, but it is reserved for future use.

```python
"a{x}b{y}c{}".format(1, x=2, y=3)               # "a2b3c1"
"a{}b{}c".format(1, 2)                          # "a1b2c"
"({1}, {0})".format("zero", "one")              # "(one, zero)"
"Is {0!r} {0!s}?".format('heterological')       # 'Is "heterological" heterological?'
```

<a id='string·index'></a>
### string·index

`S.index(sub[, start[, end]])` returns the index of the first
occurrence of the substring `sub` within S, like `S.find`, except
that if the substring is not found, the operation fails.

```python
"bonbon".index("on")             # 1
"bonbon".index("on", 2)          # 4
"bonbon".index("on", 2, 5)       # error: substring not found  (in "nbo")
```

<a id='string·isalnum'></a>
### string·isalnum

`S.isalnum()` reports whether the string S is non-empty and consists only
Unicode letters and digits.

```python
"base64".isalnum()              # True
"Catch-22".isalnum()            # False
```

<a id='string·isalpha'></a>
### string·isalpha

`S.isalpha()` reports whether the string S is non-empty and consists only of Unicode letters.

```python
"ABC".isalpha()                 # True
"Catch-22".isalpha()            # False
"".isalpha()                    # False
```

<a id='string·isdigit'></a>
### string·isdigit

`S.isdigit()` reports whether the string S is non-empty and consists only of Unicode digits.

```python
"123".isdigit()                 # True
"Catch-22".isdigit()            # False
"".isdigit()                    # False
```

<a id='string·islower'></a>
### string·islower

`S.islower()` reports whether the string S contains at least one cased Unicode
letter, and all such letters are lowercase.

```python
"hello, world".islower()        # True
"Catch-22".islower()            # False
"123".islower()                 # False
```

<a id='string·isspace'></a>
### string·isspace

`S.isspace()` reports whether the string S is non-empty and consists only of Unicode spaces.

```python
"    ".isspace()                # True
"\r\t\n".isspace()              # True
"".isspace()                    # False
```

<a id='string·istitle'></a>
### string·istitle

`S.istitle()` reports whether the string S contains at least one cased Unicode
letter, and all such letters that begin a word are in title case.

```python
"Hello, World!".istitle()       # True
"Catch-22".istitle()            # True
"HAL-9000".istitle()            # False
"ǅenan".istitle()		# True
"Ǆenan".istitle()		# False ("Ǆ" is a single Unicode letter)
"123".istitle()                 # False
```

<a id='string·isupper'></a>
### string·isupper

`S.isupper()` reports whether the string S contains at least one cased Unicode
letter, and all such letters are uppercase.

```python
"HAL-9000".isupper()            # True
"Catch-22".isupper()            # False
"123".isupper()                 # False
```

<a id='string·join'></a>
### string·join

`S.join(iterable)` returns the string formed by concatenating each
element of its argument, with a copy of the string S between
successive elements. The argument must be an iterable whose elements
are strings.

```python
", ".join(["one", "two", "three"])      # "one, two, three"
"a".join("ctmrn".codepoints())          # "catamaran"
```

<a id='string·lower'></a>
### string·lower

`S.lower()` returns a copy of the string S with letters converted to lowercase.

```python
"Hello, World!".lower()                 # "hello, world!"
```

<a id='string·lstrip'></a>
### string·lstrip

`S.lstrip()` returns a copy of the string S with leading whitespace removed.

Like `strip`, it accepts an optional string parameter that specifies an
alternative set of Unicode code points to remove.

```python
"  hello  ".lstrip()                    # "hello  "
"  hello  ".lstrip("h o")               # "ello  "
```

<a id='string·partition'></a>
### string·partition

`S.partition(x)` splits string S into three parts and returns them as
a tuple: the portion before the first occurrence of string `x`, `x` itself,
and the portion following it.
If S does not contain `x`, `partition` returns `(S, "", "")`.

`partition` fails if `x` is not a string, or is the empty string.

```python
"one/two/three".partition("/")		# ("one", "/", "two/three")
```

<a id='string·removeprefix'></a>
### string·removeprefix

`S.removeprefix(prefix)` returns a copy of string S with the prefix `prefix`
removed if S starts with `prefix`, otherwise it returns S.

```python
"banana".removeprefix("ban")          # "ana"
"banana".removeprefix("foo")          # "banana"
"foofoobar".removeprefix("foo")       # "foobar"
```

<a id='string·removesuffix'></a>
### string·removesuffix

`S.removesuffix(suffix)` returns a copy of string S with the suffix `suffix`
removed if S ends with `suffix`, otherwise it returns S.

```python
"banana".removesuffix("nana")         # "ba"
"banana".removesuffix("foo")          # "banana"
"banana".removesuffix("na")           # "bana"
```

<a id='string·replace'></a>
### string·replace

`S.replace(old, new[, count])` returns a copy of string S with all
occurrences of substring `old` replaced by `new`. If the optional
argument `count`, which must be an `int`, is non-negative, it
specifies a maximum number of occurrences to replace.

```python
"banana".replace("a", "o")		# "bonono"
"banana".replace("a", "o", 2)		# "bonona"
```

<a id='string·rfind'></a>
### string·rfind

`S.rfind(sub[, start[, end]])` returns the index of the substring `sub` within
S, like `S.find`, except that `rfind` returns the index of the substring's
_last_ occurrence.

```python
"bonbon".rfind("on")             # 4
"bonbon".rfind("on", None, 5)    # 1
"bonbon".rfind("on", 2, 5)       # -1
```

<a id='string·rindex'></a>
### string·rindex

`S.rindex(sub[, start[, end]])` returns the index of the substring `sub` within
S, like `S.index`, except that `rindex` returns the index of the substring's
_last_ occurrence.

```python
"bonbon".rindex("on")             # 4
"bonbon".rindex("on", None, 5)    # 1                           (in "bonbo")
"bonbon".rindex("on", 2, 5)       # error: substring not found  (in "nbo")
```

<a id='string·rpartition'></a>
### string·rpartition

`S.rpartition(x)` is like `partition`, but splits `S` at the last occurrence of `x`.

```python
"one/two/three".partition("/")		# ("one/two", "/", "three")
```

<a id='string·rsplit'></a>
### string·rsplit

`S.rsplit([sep[, maxsplit]])` splits a string into substrings like `S.split`,
except that when a maximum number of splits is specified, `rsplit` chooses the
rightmost splits.

```python
"banana".rsplit("n")                         # ["ba", "a", "a"]
"banana".rsplit("n", 1)                      # ["bana", "a"]
"one two  three".rsplit(None, 1)             # ["one two", "three"]
"".rsplit("n")                               # [""]
```

<a id='string·rstrip'></a>
### string·rstrip

`S.rstrip()` returns a copy of the string S with trailing whitespace removed.

Like `strip`, it accepts an optional string parameter that specifies an
alternative set of Unicode code points to remove.

```python
"  hello  ".rstrip()                    # "  hello"
"  hello  ".rstrip("h o")               # "  hell"
```

<a id='string·split'></a>
### string·split

`S.split([sep [, maxsplit]])` returns the list of substrings of S,
splitting at occurrences of the delimiter string `sep`.

Consecutive occurrences of `sep` are considered to delimit empty
strings, so `'food'.split('o')` returns `['f', '', 'd']`.
Splitting an empty string with a specified separator returns `['']`.
If `sep` is the empty string, `split` fails.

If `sep` is not specified or is `None`, `split` uses a different
algorithm: it removes all leading spaces from S
(or trailing spaces in the case of `rsplit`),
then splits the string around each consecutive non-empty sequence of
Unicode white space characters.
If S consists only of white space, `S.split()` returns the empty list.

If `maxsplit` is given and non-negative, it specifies a maximum number of splits.

```python
"one two  three".split()                    # ["one", "two", "three"]
"one two  three".split(" ")                 # ["one", "two", "", "three"]
"one two  three".split(None, 1)             # ["one", "two  three"]
"banana".split("n")                         # ["ba", "a", "a"]
"banana".split("n", 1)                      # ["ba", "ana"]
"".split("n")                               # [""]
```

<a id='string·elems'></a>
### string·elems

`S.elems()` returns an iterable value containing successive
1-byte substrings of S.
To materialize the entire sequence, apply `list(...)` to the result.

Example:

```python
list('Hello, 世界'.elems())  # ["H", "e", "l", "l", "o", ",", " ", "\xe4", "\xb8", "\x96", "\xe7", "\x95", "\x8c"]
```

See also: `string·elem_ords`.


<a id='string·codepoints'></a>
### string·codepoints

`S.codepoints()` returns an iterable value containing the sequence of
substrings of S that each encode a single Unicode code point.
Each invalid code within the string is treated as if it encodes the
Unicode replacement character, U+FFFD.

By returning an iterable, not a list, the cost of decoding the string
is deferred until actually needed; apply `list(...)` to the result to
materialize the entire sequence.

Example:

```python
list('Hello, 世界'.codepoints())  # ['H', 'e', 'l', 'l', 'o', ',', ' ', '世', '界']

for cp in 'Hello, 世界'.codepoints():
   print(cp)  # prints 'H', 'e', 'l', 'l', 'o', ',', ' ', '世', '界'
```

See also: `string·codepoint_ords`.

<b>Implementation note:</b> `codepoints` is not provided by the Java implementation.

<a id='string·splitlines'></a>
### string·splitlines

`S.splitlines([keepends])` returns a list whose elements are the
successive lines of S, that is, the strings formed by splitting S at
line terminators (currently assumed to be a single newline, `\n`,
regardless of platform).

The optional argument, `keepends`, is interpreted as a Boolean.
If true, line terminators are preserved in the result, though
the final element does not necessarily end with a line terminator.

As a special case, if S is the empty string,
`splitlines` returns the empty list.

```python
"one\n\ntwo".splitlines()       # ["one", "", "two"]
"one\n\ntwo".splitlines(True)   # ["one\n", "\n", "two"]
"".splitlines()                 # [] -- a special case
```

<a id='string·startswith'></a>
### string·startswith

`S.startswith(prefix[, start[, end]])` reports whether the string
`S[start:end]` has the specified prefix.

```python
"filename.star".startswith("filename")         # True
```

The `prefix` argument may be a tuple of strings, in which case the
function reports whether any one of them is a prefix.

```python
'abc'.startswith(('a', 'A'))                  # True
'ABC'.startswith(('a', 'A'))                  # True
'def'.startswith(('a', 'A'))                  # False
```

<a id='string·strip'></a>
### string·strip

`S.strip()` returns a copy of the string S with leading and trailing whitespace removed.

It accepts an optional string argument:
`S.strip(cutset)` instead removes all leading
and trailing Unicode code points contained in `cutset`.

```python
"  hello  ".strip()                     # "hello"
"  hello  ".strip("h o")                # "ell"
```

<a id='string·title'></a>
### string·title

`S.title()` returns a copy of the string S with letters converted to title case.

Letters are converted to upper case at the start of words, lower case elsewhere.

```python
"hElLo, WoRlD!".title()                 # "Hello, World!"
"ǆenan".title()                        # "ǅenan" ("ǅ" is a single Unicode letter)
```

<a id='string·upper'></a>
### string·upper

`S.upper()` returns a copy of the string S with letters converted to uppercase.

```python
"Hello, World!".upper()                 # "HELLO, WORLD!"
```

## Dialect differences

The list below summarizes features of the Go implementation that are
known to differ from the Java implementation of Starlark used by Bazel.
Some of these features may be controlled by global options to allow
applications to mimic the Bazel dialect more closely. Our goal is
eventually to eliminate all such differences on a case-by-case basis.
See [Starlark spec issue 20](https://github.com/bazelbuild/starlark/issues/20).

* String interpolation supports the `[ioxXc]` conversions.
* String elements are bytes.
* Non-ASCII strings are encoded using UTF-8.
* Strings support hex byte escapes.
* Strings have the additional methods `elem_ords`, `codepoint_ords`, and `codepoints`.
* The `chr` and `ord` built-in functions are supported.
* The `set` built-in function is provided (option: `-set`).
* `set & set` and `set | set` compute set intersection and union, respectively.
* `assert` is a valid identifier.
* `if`, `for`, and `while` are permitted at top level (option: `-globalreassign`).
* top-level rebindings are permitted (option: `-globalreassign`).
