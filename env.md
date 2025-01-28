# .ENV

### Environment Variables File Format Specification

Authored by [Jon Schlinkert](https://github.com/jonschlinkert), et al.

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
- [Parser Reliability Requirements](#parser-reliability-requirements)
- [Variable Expansion](#variable-expansion)
- [Filename Extension](#filename-extension)
- [MIME Type](#mime-type)

## Spec

- .env files MUST be valid UTF-8 encoded documents
- Files SHOULD be named `.env` (see [Filename Extension](#filename-extension) for variations)
- Whitespace means `tab` (`0x09`) or `space` (`0x20`)
- Newline means `LF` (`0x0A`) or `CRLF` (`0x0D` `0x0A`)
- Every non-empty line MUST be either:
  - A comment
  - A key/value pair
  - A line continuation of a previous value (value only, keys MUST NOT span multiple lines)
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

The primary building block of an .env file is the key/value pair. Each pair MUST be on its own line unless using a line continuation character (see [Multi-line Values](#multi-line-values)). Keys MUST NOT span multiple lines.

```env
KEY=value
```

Key/value pairs MUST be separated by an equals sign (`=`). The first `=` character on a line serves as the separator. Additional `=` characters in the value are treated as part of the value:

```env
URL=https://example.com/path?foo=bar&baz=qux  # Valid, only first = is separator
```

## Keys

Keys are case-sensitive and MUST:

- Begin with a letter (`A-Z`, `a-z`) or underscore (`_`)
- Contain only letters, numbers, and underscores
- Not be empty
- Not span multiple lines

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
FOO\
BAR=value      # Cannot span multiple lines
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

Values may be enclosed in single (`'`) or double (`"`) quotes. Quotes are required if the value:

- Contains leading or trailing whitespace you want to preserve
- Contains a comment character that should be treated as part of the value
- Contains line breaks (see [Multi-line Values](#multi-line-values))

```env
MESSAGE='Hello World'
PATH="C:\Program Files\App"
HASH="my#password"  # Without quotes, #password would be a comment
```

## Data Types

While all values in .env files are technically strings, parsers MAY implement optional type coercion for common data types. If type coercion is implemented:

1. It MUST be optional and easily disabled
2. The default behavior SHOULD be string-only unless type coercion is an established standard in the ecosystem
3. The implementation MUST document its coercion rules
4. Coercion MUST NOT affect the original string value

When implemented, type coercion SHOULD follow these conventions:

### Boolean

The following string values MAY be interpretable as booleans:

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

## Multi-line Values

Long values can span multiple lines using one of two methods. Note that while values can span multiple lines, keys MUST NOT:

### Quoted Multi-line

Values (but not keys) enclosed in quotes can contain newlines:

```env
PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----
MIIBOgIBAAJBAOsfi5AGYhdRs/x6q5H7kScxA0Kzrw
...
-----END RSA PRIVATE KEY-----"

INVALID_KEY="MULTI
LINE"=value  # Invalid: keys cannot contain newlines
```

### Line Continuation

A backslash (`\`) at the end of a line indicates the value continues on the next line. This can only be used for values, not keys:

```env
LONG_MESSAGE=first line \
second line \
third line

INVALID_KEY=MULTI \
LINE=value  # Invalid: keys cannot use line continuation
```

For line continuations:

- The backslash MUST be the last character on the line (excluding whitespace)
- Leading whitespace on continuation lines is included in the value
- Empty continuation lines are preserved
- Comments are not allowed on continuation lines

Invalid continuation examples that MUST trigger ENV005:

```env
SECRET=password\ # comment
NEXT=value      # Must trigger ENV005: comment after continuation

VALUE=first\    # Must trigger ENV005: dangling continuation at EOF

INVALID=test \  # Must trigger ENV005: whitespace before continuation
    next line
```

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

- Description: A backslash line continuation followed by a comment or EOF, or with preceding whitespace
- Required behavior: Parser MUST throw an error

```env
SECRET=password\ # comment
NEXT=value      # Should trigger ENV005
```

### ENV006: Multi-line Key

- Description: Attempt to define a key across multiple lines (either through quotes or line continuation)
- Required behavior: Parser MUST throw an error
- Example:

  ```env
  MULTI\
  LINE_KEY=value

  "MULTI
  LINE"=value
  ```

### ENV007: Invalid Encoding

- Description: File contains non-UTF-8 characters
- Required behavior: Parser MUST throw an error

## Parser Reliability Requirements

The .env format is used for configuration that directly affects application behavior and security. Therefore, parsers MUST prioritize reliability and predictability over convenience. This section outlines common pitfalls and required parser behavior.

### Silent Failures

Parsers MUST NOT silently ignore malformed input. The following examples show problematic input that MUST produce errors:

```env
FOO           # Invalid: no assignment
BAR=value     # This line is valid, but parser must still error due to FOO

KEY VALUE     # Invalid: missing assignment operator
PORT=8080     # This line is valid, but parser must still error due to previous line

USER=john\    # Invalid: dangling continuation
PASSWORD=     # This line is valid, but parser must still error due to previous line
```

### Line Independence

Each line MUST be validated independently before being combined with any other lines. Parsers MUST NOT:

- Combine non-empty lines that lack assignment operators
- Join lines unless explicitly continued with a backslash
- Skip validation of any non-empty, non-comment line

Examples of invalid line joining that MUST produce errors:

```env
FOO
BAR=value     # Some parsers incorrectly return { BAR: 'value' } or { 'FOO\nBAR': 'value' }
              # Must error due to invalid FOO line

A=1\n
B             # Some implementations might try to join B with previous line

FOO=value\n
BAR           # Must not be joined with previous line

KEY=a
VALUE         # Must error, not combine into KEY=aVALUE
```

### Error Consistency

Parsers MUST provide consistent error behavior:

- Same input MUST produce same errors
- Partial results MUST NOT be returned if any line is invalid
- Error messages SHOULD identify the specific line and issue
- Error codes MUST match those specified in Error Handling section

Example of incorrect partial results that MUST be avoided:

```env
VALID_KEY=value
INVALID KEY=foo  # Parser encounters invalid syntax here
ANOTHER_VALID=bar
# Parser must NOT return { VALID_KEY: 'value' } and then error
# Must error immediately without returning any values
```

### Implementation Requirements

To ensure reliable behavior, implementations MUST:

1. Validate entire file before processing any assignments
2. Throw appropriate error (see Error Handling) for first encountered issue
3. Not attempt recovery from invalid syntax
4. Not provide configuration options that bypass these requirements

The presence of valid lines in a file MUST NOT affect the parser's responsibility to error on invalid lines. This helps catch configuration issues early and prevents unpredictable behavior across different environments and implementations.

## Variable Expansion

Variable expansion (e.g., `FOO=${BAR}`) is explicitly NOT part of this specification. Implementations MAY provide variable expansion as an optional feature, but if they do:

1. It MUST be disabled by default
2. It MUST be explicitly documented
3. It MUST define clear rules for:
   - Syntax (e.g., `${VAR}` vs `$VAR` vs `%VAR%`)
   - Scope (current file only, system environment, or both)
   - Error handling for undefined variables
   - Circular reference detection
   - Maximum expansion depth

## Filename Extension

The canonical filename is `.env`. Common variations include:

- `.env.local`
- `.env.development`
- `.env.test`
- `.env.production`
- `.env.example`

## MIME Type

When transferring .env files over the internet, the appropriate MIME type is `application/env`. This designation is justified by the following characteristics that make .env files distinct from other formats:

1. **Distinct Format**

   - Specific syntax rules for key-value pairs
   - Defined comment styles
   - Special handling of quotes and whitespace
   - Line continuation mechanisms
   - Rules for multi-line values

2. **Specific Processing Model**

   - Strict parsing requirements
   - Defined error conditions and handling
   - Special security considerations for environment variables
   - Load-time behavior distinct from other configuration formats

3. **Unique Semantic Meaning**
   - Files represent environment variable declarations
   - Values have specific runtime implications
   - Format carries distinct security and operational considerations

While .env files may appear superficially similar to plain text or other configuration formats, their unique combination of syntax, processing requirements, and semantic meaning warrants a distinct MIME type. This is analogous to how `application/json` exists despite JSON being representable as plain text, because the specific structure and processing model require unique identification.

The `application/env` MIME type serves to:

- Properly identify .env files in transport
- Signal correct processing requirements to consuming systems
- Enable appropriate security handling
- Facilitate correct caching behavior
- Allow proper content negotiation

When transferring .env files over the internet, implementations MUST use the `application/env` MIME type.
