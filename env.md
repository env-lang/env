# .ENV v1.0.0

### Environment Variables File Format Specification

Authored by Jon Schlinkert, et al.

## Objectives

The .env file format aims to be a minimal, unambiguous configuration file format primarily used for setting environment variables in development environments. While the format has existed for many years with various implementations, this specification aims to standardize the expected behavior and provide clear guidance for parser implementations.

## Table of contents

- [Spec](#spec)
- [Comments](#comments)
- [Key/Value Pair](#keyvalue-pair)
- [Keys](#keys)
- [Values](#values)
- [String](#string)
- [Data Types](#data-types)
- [Multi-line Values](#multi-line-values)
- [Error Handling](#error-handling)
- [Filename Extension](#filename-extension)
- [MIME Type](#mime-type)

## Spec

- `.env` files MUST be valid UTF-8 encoded documents
- Files SHOULD be named `.env` (see [Filename Extension](#filename-extension) for variations)
- Whitespace means tab (`0x09`) or space (`0x20`)
- Newline means LF (`0x0A`) or CRLF (`0x0D 0x0A`)
- Every non-empty line MUST be either:
  - A comment
  - A key/value pair
  - A line continuation of a previous value
- Empty lines are allowed and MUST be ignored
- Parsers MUST preserve empty values (e.g., `FOO=`)
- Parsers MUST preserve whitespace in values unless explicitly trimmed by quotes or other mechanisms
- Leading and trailing whitespace on a line MUST be ignored

## Comments

Comments allow humans to add notes to .env files without affecting their parsing. The following comment styles are supported, listed in order of prevalence:

### Primary Comment Format

A hash symbol (`#`) marks the rest of the line as a comment:

```env
# This is a full-line comment
FOO=bar # This is an end-of-line comment
```

### Alternative Comment Formats

The following comment formats are also recognized by some parsers. Implementation of these formats is OPTIONAL:

```env
; This is a semicolon comment
// This is a double-slash comment
```

Comments MUST NOT be recognized within values unless escaped:

```env
SECRET="password#123"  # The '#' is part of the value
MESSAGE="Hello # World" # The '#' after Hello is part of the value
```

## Key/Value Pair

The primary building block of an .env file is the key/value pair. Each pair MUST be on its own line unless using a line continuation character (see [Multi-line Values](#multi-line-values)).

```env
KEY=value
```

Key/value pairs MUST be separated by an equals sign (`=`). The first `=` character on a line serves as the separator. Additional `=` characters in the value are treated as part of the value:

```env
URL=https://example.com/path?foo=bar&baz=qux  # Valid, only first = is separator
```

## Keys

Keys are case-sensitive and MUST:

- Begin with a letter (A-Z, a-z) or underscore
- Contain only letters, numbers, and underscores
- Not be empty

```env
# Valid keys
FOO=value
foo=value
FOO_BAR=value
_FOO=value

# Invalid keys
123FOO=value   # Cannot start with number
FOO-BAR=value  # Cannot contain hyphens
.FOO=value     # Cannot start with period
```

Traditional environment variable naming conventions RECOMMEND:

- Using uppercase letters
- Using underscores as separators
- Avoiding special characters

## Values

Values may contain any UTF-8 characters. Leading and trailing whitespace in values is significant unless the value is quoted:

```env
FOO=bar         # value is "bar"
FOO= bar        # value is " bar"
FOO=bar baz     # value is "bar baz"
FOO="bar"       # value is "bar"
FOO=" bar "     # value is " bar "
```

Empty values are valid and MUST be preserved:

```env
EMPTY=
EMPTY=""        # Same as above
```

## String

Strings are the primary value type in .env files. They can be represented in two ways:

### Unquoted Strings

Unquoted values extend from the first non-whitespace character after the `=` to the end of the line (or until a comment):

```env
UNQUOTED=value with spaces
PATH=/usr/local/bin:/usr/bin:/bin
```

### Quoted Strings

Values may be enclosed in single or double quotes. Quotes are required if the value:

- Contains leading or trailing whitespace you want to preserve
- Contains a comment character that should be treated as part of the value
- Contains line breaks (see [Multi-line Values](#multi-line-values))

```env
MESSAGE='Hello World'
PATH="C:\Program Files\App"
HASH="my#password"  # Without quotes, #password would be a comment
```

## Data Types

While all values in .env files are technically strings, many implementations provide type coercion for common data types. This behavior is OPTIONAL but when implemented SHOULD follow these conventions:

### Boolean

The following string values SHOULD be interpretable as booleans:

```env
DEBUG=true   # true
DEBUG=false  # false
DEBUG=True   # true
DEBUG=False  # false
DEBUG=TRUE   # true
DEBUG=FALSE  # false
```

### Numbers

String values that represent valid numbers MAY be coerced to numeric types:

```env
PORT=8080        # Integer
TIMEOUT=12.5     # Float
SCIENTIFIC=1e-10 # Scientific notation
```

### Null Values

The following string values MAY be interpreted as null:

```env
NULL=null
NULL=NULL
NULL=
```

Implementations that support type coercion MUST document their coercion rules and SHOULD provide a way to disable automatic type coercion.

## Multi-line Values

Long values can span multiple lines using one of two methods:

### Quoted Multi-line

Values enclosed in quotes can contain newlines:

```env
PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----
MIIBOgIBAAJBAOsfi5AGYhdRs/x6q5H7kScxA0Kzrw
...
-----END RSA PRIVATE KEY-----"
```

### Line Continuation

A backslash at the end of a line indicates the value continues on the next line:

```env
LONG_MESSAGE=first line \
second line \
third line
```

For line continuations:

- The backslash MUST be the last character on the line (excluding whitespace)
- Leading whitespace on continuation lines is included in the value
- Empty continuation lines are preserved
- Comments are not allowed on continuation lines

## Error Handling

Parsers MUST implement the following error handling:

### ENV001: Invalid Line Format

- Description: A line that is not empty, not a comment, and does not contain a valid key/value pair
- Example: `foo\nbar\nbaz=qux`
- Required behavior: Parser MUST throw an error and MUST NOT combine invalid lines

### ENV002: Duplicate Key

- Description: When the same key appears multiple times
- Required behavior: Parser MUST either:
  a) Use the last value (most common)
  b) Throw an error
  c) Use the first value
- Implementation MUST document which behavior it uses

### ENV003: Invalid Key Format

- Description: Key contains invalid characters or starts with a number
- Required behavior: Parser MUST throw an error

### ENV004: Unclosed Quote

- Description: A quoted value missing its closing quote
- Required behavior: Parser MUST throw an error

### ENV005: Invalid Line Continuation

- Description: A backslash line continuation followed by a comment or EOF
- Required behavior: Parser MUST throw an error

## Filename Extension

The canonical filename is `.env`. Common variations include:

- `.env.local`
- `.env.development`
- `.env.test`
- `.env.production`
- `.env.example`

## MIME Type

When transferring .env files over the internet, the appropriate MIME type is `application/x-env`.
