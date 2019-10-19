---
layout: post
title: The Subtle Ergonomics of the Python Parser
catagories: Python, Programming Languages
---

[Python](https://www.python.org/) has established itself as a popular programming language for both
programming newcomers and veterans. This is due to how easy the language is to
both read and write. While writing a Python parser for
[Jeroo](https://jeroo.org/beta), I discovered many subtle things that the Parser and
Lexer do that make the language very ergonomic to use. 

# Dynamic Indentation Levels

Code blocks in Python have to start with a new indentation level and end with a dedentation level.
For example:

```python
if True:
    print("true")
print("always")
```

The indentation level can be of any length, so long as that it is more
whitespace than the previously seen indentation level. Additionally, tabs count
as 8 spaces, so it is possible to parse a file with both tabs and spaces.

# Auto-Closing Dedentation Tokens

[The Python language spec](https://docs.python.org/3/reference/lexical_analysis.html#indentation)
specifies that all indentation levels should be automatically be closed before 
the lexer emits an EOF token. 

```python
while True:
    print("loop")

```

This is quite useful, otherwise it would be very difficult to write programs
with multiple code blocks.

```python
if True:
    if True:
        if True:
            if True:
```

The programmer would have to make 4 newlines, all with one less whitespace
character than the previous.

# Logical Newlines

[The Python language spec](https://docs.python.org/3/reference/lexical_analysis.html#logical-lines)
also describes logical newlines. They are defined by this regular expression
`((whitespace* comment? newline)* whitespace* comment?) newline`, which says 
that logical newlines are whitespace, comments, and at least one newline. 

Without this property, newlines would need to be handled in the parser, which
could lead to ambiguities. 

# EOF as a Valid Newline

Python programs consider an EOF token as a valid end to a statement. 
As a result, not every program needs to end with a newline character, 
which also makes programs easier to write. For example, this:

```python
while True:
    print("loop")
```

is a valid program, even though it doesn't have a newline at the end of the
print statement.
