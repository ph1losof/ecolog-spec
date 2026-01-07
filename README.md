# The Ecolog Dotenv File Format (EDF) Specification

**Document ID:** EDF-SPEC-001  
**Version:** 1.0.1  
**Date:** 2026-01-07  
**Status:** Draft Standard  
**Format:** RFC-style specification  
**Target Audience:** Parser Implementers, DevOps Engineers, IT Specialists

## License

This specification is licensed under the Creative Commons Attribution 4.0
International License (CC BY 4.0).

A copy of the license is available at https://creativecommons.org/licenses/by/4.0/

## 1. Abstract

The EDF Specification defines a strict, portable, and deterministic standard for .env configuration files. It aims to eliminate ambiguity by providing exhaustive rules for parsing, encoding, and interpreting environment variable definitions. This document adheres to RFC 2119 terminology (MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED, MAY, and OPTIONAL).

## 2. Terminology

- **Parser:** Any software or library that reads an EDF-compliant file to extract key-value pairs.
- **Consumer:** An application that uses the values extracted by the Parser.
- **Physical Line:** A sequence of characters terminated by a Newline sequence.
- **Logical Line (Definition):** A complete variable definition which may span multiple Physical Lines (e.g., via Quoted Multiline or Backslash Continuation).
- **Whitespace:** The characters Space (U+0020) and Tab (U+0009).
- **Newline:** The Line Feed (U+000A) character, or the Carriage Return + Line Feed (U+000D U+000A) sequence.

## 3. Encoding & Format

### 3.1. Character Encoding

The file MUST be encoded in UTF-8. Implementations MUST reject files containing invalid UTF-8 byte sequences before any other processing occurs.

**Invalid UTF-8 Sequences:**

```env
KEY=value\xFF\xFE
```

**Behavior:** Parser MUST reject file with error.
**Error Message:** "Invalid UTF-8 byte sequence at position N"

**UTF-8 BOM Handling:**

- Usage of Byte Order Marks (BOM) (U+FEFF) is DISCOURAGED but MAY be safely ignored by the parser at the very beginning of the file.
- If a BOM appears at the start of the file, the parser SHOULD strip it before processing.
- If a BOM appears in the middle of the file, it is invalid and the parser MUST reject the file.

**UTF-8 BOM at Start:**

```env
﻿KEY=value
```

**Behavior:** BOM (U+FEFF) at the very beginning MAY be ignored. Parser should strip it before processing.
**Result:** `KEY` = `value`

**BOM Not at Start:**

```env
KEY=value
﻿OTHER=value
```

**Behavior:** BOM in middle of file is invalid UTF-8 usage. Parser MUST reject the file with an error.
**Error Message:** "UTF-8 BOM found at invalid position (byte N)"

### 3.2. Structural Parsing

The file is processed as a stream of characters, organized into Logical Definitions.

**Empty Files:**
An empty file (zero bytes or only EOF marker) is valid and results in zero key-value pairs.

**Files With Only Comments:**

```env
# Comment 1
# Comment 2
```

**Behavior:** Valid. Results in zero key-value pairs.

**Files With Only Whitespace:**

```env



```

**Behavior:** Valid. All whitespace-only lines are ignored. Results in zero key-value pairs.

**Ignored Content:**

- **Empty Lines:** Physical lines containing only whitespace (spaces and/or tabs) MUST be ignored completely. They do not affect parsing and are not considered part of any variable definition.
- **Comment Lines:** Any Physical Line where the first non-whitespace character is `#` MUST be ignored completely. The entire line, from the `#` to the end of the line, is treated as a comment.

**Comment Line Examples:**

```env
# This is a comment
  # This is also a comment (leading whitespace allowed)
	# Tab-indented comment
```

All of these are valid comments and are completely ignored by the parser.

**Variable Definitions:**

- Any non-ignored line (not empty, not a comment) is the start of a Variable Definition.
- Depending on quoting rules, a Variable Definition MAY consume multiple subsequent Physical Lines (see Section 5).
- Each Variable Definition consists of a Key, an equals sign separator, and a Value.

## 4. Syntax Rules

### 4.1. Variable Definitions

A definition follows the grammar: `[export] <KEY>=<VALUE>`.

This grammar indicates:

- The `export` prefix is optional (indicated by square brackets)
- A KEY is required
- An equals sign `=` is required as the separator
- A VALUE is required (but may be empty string)

#### 4.1.1. The `export` Prefix

To support POSIX shell compatibility (`source .env`), lines MAY start with the keyword `export` followed by one or more whitespace characters.

- **Parsing Rule:** The parser MUST look for the `export` token at the start of the line (after optional leading whitespace). If found, it MUST consume the token and the following whitespace, treating the remainder of the line as the Definition.
- **Semantic Effect:** The `export` keyword is ignored for value resolution and has no effect on the variable's value. It exists solely for shell compatibility.
- **Recognition Rule:** The `export` keyword is only recognized when followed by whitespace (Space or Tab). Keys that start with the literal text "export" but are not followed by whitespace are treated as regular keys.

**Leading Whitespace Before Export:**
Leading whitespace before the `export` keyword is allowed and MUST be stripped before processing the export prefix.

```env
export KEY=value
  export KEY2=value
	export KEY3=value
```

**Behavior:** All are valid. Leading whitespace is stripped before processing.
**Results:**

- Line 1: `KEY` = `value`
- Line 2: `KEY2` = `value` (two spaces stripped)
- Line 3: `KEY3` = `value` (tab stripped)

**Keys Beginning with "export":**

```env
exportKey=value
export_KEY=value
exportedValue=value
```

**Behavior:** The `export` keyword is only recognized when followed by whitespace. If "export" is followed by any non-whitespace character, it is treated as part of the key name.
**Results:**

- `exportKey=value` → Key: `exportKey`, Value: `value` (no space after "export")
- `export_KEY=value` → Key: `export_KEY`, Value: `value` (underscore, not space)
- `exportedValue=value` → Key: `exportedValue`, Value: `value` (regular key)

Only `export KEY=value` (with whitespace after "export") is treated as using the export prefix.

**Export Without Definition:**

```env
export
KEY=value
```

**Behavior:** An `export` keyword on its own line (without a KEY=VALUE following on the same line) is invalid. Parser SHOULD reject with error or ignore the line.
**Error Message:** "Export keyword without definition at line N"

**Export With Only Whitespace:**

```env
export
KEY=value
```

**Behavior:** An `export` followed only by whitespace with no key-value pair on the same line MUST be rejected or ignored (no definition follows).
**Error Message:** "Export keyword without definition at line N"

#### 4.1.2. Keys

Keys are identifiers that name environment variables. They follow strict naming rules to ensure compatibility across systems.

- **Syntax:** Keys MUST match the regular expression `^[a-zA-Z_][a-zA-Z0-9_]*$`
- **Allowed Characters:** Uppercase letters (A-Z), Lowercase letters (a-z), Digits (0-9), and Underscore (\_).
- **Start Character:** The key MUST start with either a letter (uppercase or lowercase) or an underscore. The key MUST NOT start with a digit.
- **Case Sensitivity:** Keys are case-sensitive. `API_KEY` and `api_key` are treated as two distinct variables.
- **Parsing Rule:** The Key ends at the first `=` character encountered.
- **Length:** Keys MUST have at least one character. Empty keys are forbidden.

**Valid Key Examples:**

```env
KEY=value
_PRIVATE=value
API_KEY=value
database_url=value
CONFIG123=value
_123=value
VALID="=value"
ALSO_VALID='=value'
```

All of these have valid key names.

**Invalid Key Examples:**

```env
123KEY=value
my-key=value
my.key=value
my key=value
```

**Errors:**

- `123KEY`: Starts with digit (invalid)
- `my-key`: Contains hyphen (invalid character)
- `my.key`: Contains period (invalid character)
- `my key`: Contains space (invalid character)

**Leading Whitespace Before Key:**
Optional whitespace before the key (or `export` keyword) is allowed and MUST be stripped before processing.

```env
KEY=value
  KEY2=value
	KEY3=value
```

**Behavior:** All valid. Leading whitespace is stripped.
**Results:**

- `KEY` = `value`
- `KEY2` = `value` (leading spaces stripped)
- `KEY3` = `value` (leading tab stripped)

**Empty Keys:**

```env
=value
```

**Behavior:** Keys with zero characters are FORBIDDEN. Lines with empty keys (e.g., `=value`) MUST be rejected with a parse error.
**Error Message:** "Empty key at line N"

**Whitespace-Only Keys:**

```env
   =value
```

**Behavior:** MUST be rejected. After leading whitespace is stripped, this results in an empty key.
**Error Message:** "Empty key at line N"

**Whitespace Around Equals:**
Whitespace is FORBIDDEN between the Key and the `=` separator. Any whitespace in this position makes the line invalid.

```env
KEY =value
KEY= value
KEY = value
```

**Behavior:** ALL INVALID. Whitespace is not allowed between the key and the equals sign.
**Error Message:** "Whitespace between key and equals at line N"

This rule ensures unambiguous parsing and prevents accidental errors.

**Multiple Equals Signs:**

When multiple `=` characters appear in a line, the first `=` separates the key from the value. Subsequent `=` characters appearing later in the value are treated as part of the value itself.

**Double Equals Detection:**
If the parser encounters two consecutive equals signs immediately after the key (`KEY==`), it MUST reject the file with a parse error. This prevents common typos and ensures shell compatibility.

**Valid Examples:**

```env
KEY=value=with=equals
DATABASE_URL=postgresql://user:pass@host:5432/db?param=value
EQUATION=x=y+z
```

**Behavior:** First `=` separates key from value. Subsequent `=` characters within the value are literal.
**Results:**

- `KEY=value=with=equals` → Key: `KEY`, Value: `value=with=equals`
- `DATABASE_URL=...` → Key: `DATABASE_URL`, Value: `postgresql://user:pass@host:5432/db?param=value`
- `EQUATION=x=y+z` → Key: `EQUATION`, Value: `x=y+z`

**Invalid Example:**

```env
KEY==value
```

**Behavior:** Double equals immediately after key is FORBIDDEN. Parser MUST reject with error.
**Error Message:** "Double equals sign detected at line N. To set a value starting with '=', use quotes: KEY=\"=value\""
**Rationale:** This is almost always a typo and creates files incompatible with shell sourcing.

**Values Starting With Equals:**
If you need a value that starts with an equals sign, use quotes:
Valid:

```env
VALID="=value"
ALSO_VALID='=value'
```

**Results:**

- `VALID` = `=value`
- `ALSO_VALID` = `=value`

Invalid:

```env
KEY==value
SETTING==true
```

**Errors:**

- `KEY==value`: Double equals not allowed (use quotes if value should start with `=`)
- `SETTING==true`: Double equals not allowed

````

**Duplicate Keys:**
When a key appears multiple times in the file, the last occurrence wins. The final value overwrites all previous values for that key. This behavior matches most existing dotenv implementations and allows for file composition patterns (base configuration + overrides).

```env
KEY=value1
KEY=value2
KEY=value3
````

**Behavior:** Last value wins. All previous values are overwritten.
**Result:** `KEY` = `value3`

**Key Order:**
Parsers MUST preserve the order of variable definitions as they appear in the file. This is important for consumers that care about definition order, even though the last-wins rule applies to duplicates. The order preservation refers to the sequence in which unique keys are first encountered or last defined.

### 4.2. Values

The Value starts immediately after the `=` separator. Parsing continues until a termination condition is met (Newline, Comment, or closing quote).

Values can be:

- Empty strings
- Unquoted strings
- Single-quoted strings (literal, no escaping)
- Double-quoted strings (with escape sequences)
- Multiline strings (via quotes or backslash continuation)

#### 4.2.1. Empty Values

An empty value is a valid value that results in an empty string.

- **Syntax:** `KEY=` (nothing after equals sign, or only whitespace which gets trimmed)
- **Behavior:** Results in an empty string value.
- **Use Case:** To explicitly set a variable to empty, or to unset a variable in override files.

**Empty Value Examples:**

```env
EMPTY=
EMPTY=""
EMPTY=''
```

**Behavior:** All three syntaxes are valid and produce the same result.
**Result:** All produce `EMPTY` = `` (empty string)

**Only Whitespace After Equals:**

```env
KEY=
KEY=
```

**Behavior:** Whitespace after `=` in an unquoted value is trimmed from the end (trailing whitespace).
**Result:** Both = `` (empty string)

#### 4.2.2. Unquoted Values

Unquoted values are the simplest form, suitable for alphanumeric strings without special characters or spaces.

- **Usage:** Simple alphanumeric strings, numbers, URLs, paths, and other values without spaces or special characters.
- **Termination:** The value ends at the first whitespace character (space or tab) or newline character.
- **Comment Rule:** A `#` character preceded by whitespace starts an inline comment (see Section 4.3.2).
- **No Escape Sequences:** Backslashes and other special characters are treated as literals (except backslash at end of line, which triggers continuation).

**Whitespace Handling:**

- Leading whitespace after `=` is preserved as part of the value
- Trailing whitespace (before comment/newline) MUST be trimmed
- Internal whitespace terminates the value (remainder becomes comment if `#` is present with required preceding whitespace)

**Unquoted Value Examples:**

Simple value:

```env
SIMPLE=Value
```

**Result:** `Value`

Value with leading whitespace:

```env
LEADING=  Value
```

**Result:** `  Value` (leading spaces preserved as part of the value)

Value with trailing whitespace:

```env
TRAILING=Value
TRAILING2=Value
```

**Result:** Both = `Value` (trailing whitespace trimmed)

URL with fragment:

```env
URL=http://example.com#home
```

**Result:** `http://example.com#home` (fragment preserved because `#` not preceded by whitespace)

Hex color code:

```env
COLOR=#333
```

**Result:** `#333` (hex color preserved because `#` is at start, not preceded by whitespace)

Value with inline comment:

```env
KEY=Val #comment
```

**Result:** `Val` (value), `#comment` (ignored as comment)

**CR+LF vs LF Line Endings:**
Both Line Feed (U+000A) and Carriage Return + Line Feed (U+000D U+000A) are valid newline sequences. Parsers MUST accept both as line terminators.

```env
KEY=value\r\n
KEY=value\n
```

**Behavior:** Both are valid and treated identically. The newline ends the value.
**Result:** Both produce the same result (newline terminates the unquoted value).

**Line Ending Preservation:**
In quoted values, the original line ending sequences from the file MUST be preserved as literal characters. Parsers MAY normalize line endings internally during processing, but the actual newline characters within quoted string values must reflect the file's original format. This ensures that multiline values maintain their original line ending style.

**Backslash in Unquoted Values:**
In unquoted values, backslash characters are treated as literal characters UNLESS they appear as the last character on the line (which triggers continuation). Escape sequences like `\n`, `\t`, `\r` are NOT interpreted in unquoted values—they remain as literal backslash followed by the character.

```env
KEY=value\text
KEY=foo\nbar
KEY=foo\\bar
PATH=C:\Users\Name
```

**Behavior:** Backslashes are literal (not escape sequences) except when at line end.
**Results:**

- `KEY=value\text` → `value\text` (literal backslash)
- `KEY=foo\nbar` → `foo\nbar` (literal backslash and 'n', NOT a newline character)
- `KEY=foo\\bar` → `foo\\bar` (two literal backslashes)
- `PATH=C:\Users\Name` → `C:\Users\Name` (Windows path preserved)

#### 4.2.3. Single Quoted Values (`'...'`)

Single-quoted values provide complete literal string handling with no interpretation of any special characters.

- **Usage:** Used for literal strings where no escaping or interpolation is desired. Ideal for passwords, tokens, or any string containing special characters that should be preserved exactly as written.
- **Literal Mode:** Inside single quotes, NO characters are interpreted specially. Everything is literal.
  - `\` is a literal backslash (not an escape character)
  - `$` is a literal dollar sign (no interpolation)
  - `#` is a literal hash character (not a comment)
  - `'` cannot be escaped (it always closes the value)
  - Newlines are literal and preserved
  - All other characters are literal

**Parsing Rule:**

- The value starts immediately after the opening `'`.
- The value ends at the next `'` character.
- NO ESCAPING: It is IMPOSSIBLE to include a single quote character inside a single-quoted string. There is no escape sequence for this.
- Everything between the opening and closing `'` is taken literally, byte-for-byte.

**Quote Immediately After Equals:**

```env
KEY='value'
```

**Behavior:** Valid. The opening quote must immediately follow `=` with no space to enter quoted mode.
**Result:** `value`

**EOF Handling:**

```env
KEY='unclosed
```

**Behavior:** If EOF (End of File) is reached before the closing `'`, the parser MUST reject the file with an error.
**Error Message:** "Unclosed single quote starting at line N, column M"

**Single Quote Examples:**

Password with special characters:

```env
PASSWORD='p@ss\word'
```

**Result:** `p@ss\word` (backslash is literal, not an escape)

Literal variable reference:

```env
LITERAL='${VAR}'
```

**Result:** `${VAR}` (dollar and braces are literal, no interpolation)

Hash character:

```env
HASH='#333'
```

**Result:** `#333` (hash is literal, not a comment)

Multiline in single quotes:

```env
TEXT='Line 1
Line 2
Line 3'
```

**Result:** `Line 1` + newline + `Line 2` + newline + `Line 3` (newlines preserved literally)

**Limitation - No Single Quote Inside:**
Because there is no escape mechanism in single-quoted strings, it is IMPOSSIBLE to include a single quote character within the value. If you need a value containing single quotes, use double-quoted strings instead:

```env
# Impossible - syntax error
TEXT='can't do this'

# Correct way
TEXT="can't do this"
```

#### 4.2.4. Double Quoted Values (`"..."`)

Double-quoted values are the standard mechanism for complex strings, providing escape sequence support and multiline capabilities.

- **Usage:** The recommended quoting style for complex strings. Supports escape sequences for special characters and multiline values.
- **Comment Protection:** `#` characters inside double quotes are treated as literal content, NOT as comment markers, regardless of surrounding whitespace.
- **Escape Sequences:** Backslash escape sequences are processed to produce special characters.

**Parsing Rule:**

- The value starts immediately after the opening `"`.
- The value ends at the next unescaped `"` (a `"` not preceded by an odd number of backslashes).
- Escape sequences are processed according to the rules below.

**Quote Immediately After Equals:**

```env
KEY="value"
```

**Behavior:** Valid. The opening quote must immediately follow `=` with no space to enter quoted mode.
**Result:** `value`

**Escape Sequences:**
The parser MUST interpret the following escape sequences within double-quoted strings:

- `\n` → Line Feed (U+000A) - newline character
- `\r` → Carriage Return (U+000D) - carriage return character
- `\t` → Tab (U+0009) - horizontal tab character
- `\\` → Literal Backslash (`\`) - produces a single backslash
- `\"` → Literal Double Quote (`"`) - produces a quote without closing the string
- `\$` → Literal Dollar Sign (`$`) - useful for consumers that perform interpolation

**Unrecognized Escape Sequences:**
All other instances of `\` followed by any character SHOULD be preserved literally (both the backslash and the following character). This ensures forward compatibility and prevents data loss.

Examples of unrecognized sequences:

- `\a` → `\a` (both characters preserved)
- `\x` → `\x` (both characters preserved)
- `\0` → `\0` (both characters preserved)

**EOF Handling:**

```env
KEY="unclosed
```

**Behavior:** If EOF is reached before the closing `"`, the parser MUST reject the file with an error.
**Error Message:** "Unclosed double quote starting at line N, column M"

**Double Quote Examples:**

JSON content:

```env
JSON="{\"key\": \"value\"}"
```

**Result:** `{"key": "value"}` (escaped quotes produce literal quote characters)

Newline in value:

```env
NEWLINE="Line1\nLine2"
```

**Result:** `Line1` + LF + `Line2` (escape sequence produces actual newline character)

Escaped dollar sign:

```env
ESCAPED="\${VAR}"
```

**Result:** `${VAR}` (dollar sign escaped, produces literal $, prevents interpolation by consumers)

Hash character (literal, not comment):

```env
COLOR="#333"
```

**Result:** `#333` (inside quotes, # is always literal)

Hash with spaces (still literal):

```env
TEXT="value #not a comment"
```

**Result:** `value #not a comment` (inside quotes, # is literal even with spaces)

Windows path:

```env
PATH="C:\\Users\\Name"
```

**Result:** `C:\Users\Name` (escaped backslashes produce literal backslashes)

All escape sequences:

```env
ALL="tab:\there\nquote:\"slash:\\dollar:\$"
```

**Result:** `tab:` + TAB + `here` + LF + `quote:"slash:\dollar:$`

**Mixed Quote Types:**

```env
KEY="value'
```

**Behavior:** Opening `"` must be closed by `"`. The `'` is part of the value as a literal character. If EOF is reached without closing `"`, parser must reject.
**Result (if properly closed later):** Value contains literal single quote character.

**Nested Quotes:**

```env
KEY="outer 'inner' end"
KEY='outer "inner" end'
```

**Behavior:** Valid. Inner quotes are literal characters. Quote characters of the opposite type within quoted strings are always treated as literal characters. There is no limit to nesting depth as the parser tracks only the outermost quote type.
**Results:**

- `KEY="outer 'inner' end"` → Value: `outer 'inner' end`
- `KEY='outer "inner" end'` → Value: `outer "inner" end`

**Complex Nesting Example:**

```env
KEY="a 'b "c" d' e"
```

**Result:** `a 'b "c" d' e` (the inner `"c"` quotes are literal characters inside the double-quoted string)

**Space Before Opening Quote:**

```env
KEY= "value"
```

**Behavior:** The space after `=` starts an unquoted value context. When a quote character (`"` or `'`) appears within an unquoted value, it is treated as a literal character, not as a quote delimiter. The parser remains in unquoted mode and does not enter quoted string parsing.
**Result:** ` "value"` (space, quote character, the word value, quote character - all as literal text)
**Error on Unclosed:** Since the quotes are literal characters in unquoted mode, there is no "unclosed quote" error. The value ends at whitespace or newline as per unquoted value rules.
**Recommendation:** Always place the opening quote immediately after `=` with no space to avoid confusion.

**Quotes Within Unquoted Values:**
Quote characters (`"` or `'`) that appear within an unquoted value (not immediately after `=`) are treated as literal characters, not as quote delimiters.

```env
KEY=foo"bar"baz
KEY=foo'bar'baz
KEY=foo"bar
```

**Results:**

- `foo"bar"baz` (quotes are literal)
- `foo'bar'baz` (quotes are literal)
- `foo"bar` (no closing quote needed, quote is literal)

**Consecutive Backslashes:**

```env
KEY="value\\"
KEY="value\\\\"
```

**Behavior:**

- `"value\\"` → `value\` (one escaped backslash produces one literal backslash)
- `"value\\\\"` → `value\\` (two escaped backslashes produce two literal backslashes)

**Trailing Escaped Quote:**

```env
KEY="value\"
more"
```

**Behavior:** `\"` at the end of a line within double quotes is treated as an escaped quote character (literal `"`), not as closing the quoted string. The parser continues reading the next line as part of the quoted value.
**Result:** `KEY` = `value"` + newline + `more`

This allows for quotes within multiline strings that span across line boundaries.

### 4.3. Comments

Comments provide a way to document .env files and temporarily disable variables without deleting them.

#### 4.3.1. Line Comments

A line is a complete comment if the first non-whitespace character is `#`. The entire line MUST be ignored by the parser.

**Behavior:**

- Leading whitespace before `#` is allowed
- Everything from `#` to the end of the line is ignored
- Comment lines do not affect parsing of other lines

**Line Comment Examples:**

```env
# This is a comment
  # This is also a comment (leading whitespace allowed)
	# Tab-indented comment
### Multiple hash marks
# TODO: Add more configuration
```

All of these are valid comment lines and are completely ignored by the parser.

#### 4.3.2. Inline Comments

Inline comments allow you to add comments at the end of a variable definition line.

**Definition:** An inline comment starts when a # character is preceded by at least one whitespace character (Space U+0020 or Tab U+0009) and occurs outside of a quoted value.

**Scope:** When recognized, an inline comment extends from the # to the End of the Physical Line and is ignored for the purpose of parsing the variable definition.

**Critical Rule:** Inline comments are recognized only after a complete Variable Value has been parsed; they MUST NOT affect how the value itself is interpreted.

**Parsing Rules:**

For unquoted values, the value consists of all characters from the start of the value up to, but not including, the whitespace that precedes an inline comment.

**Unquoted Values:** `#` MUST be preceded by whitespace to start a comment.

```env
KEY=Val#ue
```

**Result:** `Val#ue` (literal `#` preserved, no space before it)

```env
KEY=Val #comment
KEY=Val  #comment
```

**Result:** Both = `Val` (value), `#comment` (ignored as comment)
Multiple spaces before `#` are fine.

```env
KEY=Val	#comment
```

**Result:** `Val` (value), `#comment` (comment)
Tab character counts as whitespace for comment detection.

```env
URL=http://example.com#home
```

**Result:** `http://example.com#home` (preserves URL fragments, no space before `#`)

```env
COLOR=#333
```

**Result:** `#333` (preserves hex colors, `#` at start, no preceding space)

```env
START=#value
```

**Result:** `#value` (# not preceded by space, so it's literal)

```env
API_KEY=sk-abc123#prod
```

**Result:** `sk-abc123#prod` (# not preceded by space, part of the value)

**Quoted Values:** Inside quotes (single or double), `#` is ALWAYS treated as literal content, regardless of surrounding whitespace. Quotes protect the `#` from being interpreted as a comment marker.

```env
KEY="Val#ue"
KEY='Val #ue'
KEY="Val #comment"
```

**Results:**

- `"Val#ue"` → `Val#ue` (# is literal)
- `'Val #ue'` → `Val #ue` (space and # both literal)
- `"Val #comment"` → `Val #comment` (this is NOT a comment! The # and everything after it are part of the value)

**Tabs vs Spaces:**
Both Space (U+0020) and Tab (U+0009) count as whitespace for comment detection. Either one before `#` will trigger inline comment parsing.

```env
KEY=value	#tab before hash
KEY=value #space before hash
```

**Behavior:** Both are valid inline comments.
**Result:** Both produce `KEY` = `value` with the comment ignored.

**Ambiguity Resolution:**
If you need a value containing both `#` and trailing content after whitespace, use quotes to protect it:

```env
# This will be split
KEY=Val #not-a-comment

# Use quotes to preserve everything
KEY="Val #not-a-comment"
```

### 4.4. Variable Interpolation

Variable interpolation is the substitution of variable references like `${VAR}` or `$VAR` with their actual values.

**Scope:** Variable interpolation (substitution of `${VAR}` or `$VAR` syntax) is explicitly OUT OF SCOPE for the EDF specification.

**Parser Behavior:** EDF-compliant parsers MUST return variable reference syntax as LITERAL TEXT without performing any substitution. The parser treats `$` and `{}` as regular characters and does not attempt to resolve variable references.

**Rationale:**

- **Security:** Interpolation introduces complexity and security risks including arbitrary code execution, environment variable leakage, and circular reference attacks
- **Portability:** Different environments require different interpolation semantics (shell vs Docker vs systemd)
- **Interoperability:** Interpolation rules vary widely across tools (Docker Compose, systemd, shell scripts)
- **Separation of Concerns:** Separating parsing from interpolation allows for clearer security boundaries and easier testing
- **Consumer Control:** Applications can implement interpolation with their own security model and rules

**Consumer Responsibility:**

- Higher-level tools and consumers MAY implement their own interpolation logic after parsing
- Consumers that implement interpolation SHOULD document their specific rules and security implications
- Recommended approach: Parse with EDF → Validate → Apply environment-specific interpolation

**Examples:**

Variable references returned as literal text:

```env
DATABASE_URL=${DB_HOST}:${DB_PORT}/${DB_NAME}
```

**Parser returns:** `"DATABASE_URL" = "${DB_HOST}:${DB_PORT}/${DB_NAME}"` (literal string)

Single-quoted literal (no difference from parser perspective):

```env
TEMPLATE='Price: $${AMOUNT}'
```

**Parser returns:** `"TEMPLATE" = "Price: $${AMOUNT}"` (literal string)

Escaped dollar sign in double quotes:

```env
ESCAPED="\${NOT_VARIABLE}"
```

**Parser returns:** `"ESCAPED" = "${NOT_VARIABLE}"` (literal string)
**Note:** `\$` was processed as escape sequence during parsing, result is literal `$` character

**Dollar Sign in Various Contexts:**

Dollar sign at end of value:

```env
KEY=value$
KEY="value$"
```

**Behavior:** Both produce literal `$` at end. No interpolation is performed by parser.
**Results:**

- Unquoted: `KEY` = `value$`
- Quoted: `KEY` = `value$`

Dollar sign before non-variable patterns:

```env
KEY=$123
KEY="$ price"
KEY=${
```

**Behavior:** All produce literal output (no interpolation by parser). Parser doesn't attempt to validate or process variable syntax.
**Results:**

- `KEY=$123` → `$123`
- `KEY="$ price"` → `$ price`
- `KEY=${` → `${`

**Note on Escape Sequences:**
The `\$` escape sequence in double quotes (Section 4.2.4) produces a literal `$` character during parsing, which prevents consumers from treating it as interpolation syntax if they implement interpolation later. This provides an explicit way to escape variable references for consumers that do interpolation.

### 4.5. Backtick Command Substitution

Backtick command substitution (`` `command` ``) is legacy shell syntax for capturing command output. Like variable interpolation (Section 4.4), command substitution execution is explicitly OUT OF SCOPE for EDF parsers.

**Scope:** EDF-compliant parsers MUST return backtick syntax as LITERAL TEXT without executing any commands. The parser treats backticks as regular characters.

**Parser Behavior:**

| Context        | Parser Behavior                                     |
| -------------- | --------------------------------------------------- |
| Unquoted value | Backticks are literal characters, part of the value |
| Double-quoted  | Backticks are literal characters, part of the value |
| Single-quoted  | Backticks are literal characters, part of the value |

**Rationale:**

- **Security:** Command execution introduces severe security risks
- **Portability:** Command behavior varies across shells and environments
- **Separation of Concerns:** Parsing must be separate from execution

**Examples:**

```env
CMD=`whoami`
```

**Parser returns:** `CMD` = `` `whoami` `` (literal string)

```env
MSG="Hello `whoami`"
```

**Parser returns:** `MSG` = `` Hello `whoami` `` (literal string)

```env
MSG='`literal`'
```

**Parser returns:** `MSG` = `` `literal` `` (literal string)

**Consumer Responsibility:**

Consumers MAY implement command substitution after parsing. When doing so, consumers SHOULD follow shell-compatible semantics:

| Context        | Consumer Behavior (if implementing substitution)     |
| -------------- | ---------------------------------------------------- |
| Unquoted value | `` `cmd` `` MAY be executed and output substituted   |
| Double-quoted  | `` `cmd` `` MAY be executed and output substituted   |
| Single-quoted  | `` `cmd` `` MUST remain literal (no substitution)    |

Consumers that implement command substitution MUST:

- Provide appropriate sandboxing and security controls
- Document which commands are allowed
- Handle execution errors gracefully
- Consider the security implications of arbitrary command execution

**Token Recognition for Tooling:**

Parser implementations MAY recognize backtick-delimited sequences (`` `...` ``) as distinct tokens for tooling purposes (syntax highlighting, linting, IDE features). This is an implementation choice and does not affect the parsed value.

When recognized as tokens:

- In **unquoted values**: Parser MAY emit `` `...` `` sequences as `backtick_substitution` tokens
- In **double-quoted values**: Parser MAY emit `` `...` `` sequences as `backtick_substitution` tokens
- In **single-quoted values**: Backticks SHOULD NOT receive special token treatment (consistent with single-quote literal semantics)

**Unclosed Backtick Sequences:**

If a parser recognizes backtick sequences as tokens and encounters an unclosed backtick:

- The parser MAY treat the backtick as a literal character (no error)
- The parser MAY emit a warning or diagnostic for tooling purposes
- The parser MUST NOT reject the file solely due to an unclosed backtick

This permissive behavior ensures compatibility with values that legitimately contain single backtick characters.

## 5. Multiline & Backslash Continuation

The specification provides two mechanisms for multiline values: quoted multiline (recommended) and backslash continuation (legacy).

### 5.1. Quoted Multiline

Double-quoted strings are the primary and recommended mechanism for multiline values. They provide clean, readable multiline support with escape sequence processing.

**Parsing Logic:**

1. Parser encounters opening `"`.
2. Parser enters QUOTED_STATE.
3. In this state, Newlines (LF characters U+000A) are treated as literal characters and appended to the value.
4. Parser continues reading across Physical Lines until it encounters an unescaped closing `"`.
5. Escape sequences are processed according to Section 4.2.4.

**Characteristics:**

- Clean and readable syntax
- Preserves exact line breaks
- Supports escape sequences
- No special continuation markers needed
- Works with both single and double quotes (single quotes are also literal multiline)

**Valid PEM Key Example (Newlines preserved):**

```env
PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----
MIIEpQIBAAKCAQEA3Tz2MR7SZiAMfQyuvBjM9Oi..
ZjvcNMi09mM9NzQ3M...
-----END RSA PRIVATE KEY-----"
```

**Result:** The value contains the entire PEM key with all newlines preserved exactly as written.

**Valid JSON Payload:**

```env
CONFIG="{
  \"enable\": true,
  \"retries\": 3,
  \"timeout\": 30
}"
```

**Result:** The value contains the JSON with all formatting, indentation, and newlines preserved. The escaped quotes become literal `"` characters in the final value.

**Empty Lines in Quoted Strings:**
Empty lines within quoted strings are preserved as part of the value.

```env
TEXT="Line1

Line3"
```

**Result:** `Line1\n\nLine3` (preserves empty line - two consecutive newline characters)

**Single-Quote Multiline:**
Single-quoted strings also preserve newlines as literal characters, with no escape sequence processing.

```env
LITERAL='Line 1
Line 2
Line 3'
```

**Result:** `Line 1` + newline + `Line 2` + newline + `Line 3` (all newlines literal)

### 5.2. Backslash Continuation

A mechanism to split a single logical line into multiple physical lines.

**Syntax:** A backslash `\` as the last character of the **contiguous value content** (immediately preceding the newline).

**Strict Whitespace Rule:**
Unquoted values strictly terminate at the first whitespace character (Space or Tab). Therefore:

1.  A backslash can only trigger continuation if it is **part of the value** (not after a space).
2.  **Inline comments are NOT supported** on unquoted continuation lines (because the required space before `#` would terminate the value before the backslash).

**Parsing Algorithm (Strict Order of Operations):**

1.  **Scan Value:** Read characters until Whitespace, Newline, or EOF.
2.  **Terminate:** The value ends _immediately_ upon encountering any whitespace. Everything after that whitespace (comments, garbage, backslashes) is ignored.
3.  **Check Continuation:** If the value ended at a Newline (not whitespace) and the last character is `\`:
    - Remove the `\` marker.
    - Discard the following newline.
    - Append the next line (preserving leading whitespace).
    - Yield to the scan loop again.

**Examples & Cases:**

#### Basic Continuation

The backslash must be contiguous with the value.

**Valid:**

```env
PATH=/usr/bin:\
/usr/local/bin
```

Result: `/usr/bin:/usr/local/bin`

**Invalid (Shell strictness)**

# Space before backslash terminates value early

```
COMMAND="docker run \
  --detach \
  nginx"
```

Result: `docker run   --detach   nginx`
Explanation: Inside quotes, whitespace is preserved, so the backslash at the end of the line successfully triggers continuation.

#### Inline Comments

Cannot be used with unquoted continuation. Because inline comments require a preceding space, and space terminates the value, the parser will never see the backslash as part of the value.

```
# Attempting inline comment
KEY=Value\ # comment
Continued
```

Result: `Value\`
Explanation: The parsing stops at the space before `#`. The value ends with `\`. Because it didn't end at a newline (it ended at space), continuation is NOT triggered. The \ is treated as a literal character.

#### Empty Lines in Continuation

Empty lines encountered during an active continuation are consumed.

```
KEY=Part1\

Part2
```

Result: `Part1Part2`
Explanation: `Part1\` triggers continuation. The empty line is consumed. `Part2` is appended.

#### Consecutive Backslashes

```
KEY=Value\\
More
```

Result: `Value\More`
Explanation: The first `\` escapes the second. The value ends with a literal `\`. Since the last effective character is a literal `\` (not an escaping `\`), continuation is NOT triggered. The newline terminates the value.

#### Backslash at End of File

If a backslash is the very last character of the file (no newline):

```
KEY=Value\
```

Result: `Value\` Explanation: There is no "next line" to continue onto, so it is treated as a literal backslash.

## 6. Security Considerations

This section addresses security concerns that parser implementers must consider to create secure, robust implementations.

### 6.1. UTF-8 Validation

**Threat:** Malicious actors may craft files with invalid UTF-8 sequences to:

- Conceal malicious keys or values from visual inspection (using control characters or invalid sequences)
- Exploit parser bugs that mishandle encoding errors (buffer overflows, incorrect state transitions)
- Cause different interpretations between validation tools and production parsers (validation bypass)
- Inject hidden characters that change parsing behavior
- Cause denial of service through encoding edge cases

**Requirement:**

- Parsers MUST validate UTF-8 encoding before any other processing
- Parsers MUST reject files with invalid UTF-8 byte sequences
- Parsers MUST NOT attempt to "repair" or guess at invalid sequences (no automatic fixing or substitution)
- UTF-8 validation must be the first step before any parsing logic executes

**Rationale:** Invalid UTF-8 can hide malicious content, cause parser vulnerabilities, or lead to different interpretations between systems. Rejecting invalid UTF-8 upfront ensures consistent, safe parsing.

**Implementation Note:** Use well-tested UTF-8 validation libraries rather than implementing your own validation logic.

### 6.2. Newline Injection in Quoted Strings

**Threat:** Newlines inside quoted strings must not be re-interpreted as new definitions. An attacker might try to inject additional variable definitions by including newlines in a quoted value, hoping the parser will re-parse the content and create unintended variables.

**Attack Example:**

```env
SAFE="value
KEY2=malicious"
```

**Correct Behavior:** `SAFE` contains the literal string `value\nKEY2=malicious`. No second variable named `KEY2` is created. The newline and everything after it are part of the SAFE variable's value.

**Requirement:**

- Parsers MUST treat newlines inside quotes as literal characters only
- Parsers MUST NOT re-parse the content of quoted strings as if it were new definitions
- The closing quote MUST be the only termination condition for quoted strings
- No special characters inside quotes should trigger parser state changes (except the closing quote and escape sequences)

**Rationale:** Re-parsing quoted content would allow injection of arbitrary variables, breaking security boundaries and allowing configuration override attacks.

### 6.3. Error Message Information Leakage

**Threat:** Detailed error messages could expose sensitive information such as passwords, API keys, or secrets to logs, monitoring systems, or error reporting services.

**Risk Scenarios:**

- Error messages logged to centralized logging systems
- Error messages displayed in web interfaces or APIs
- Error messages included in crash reports or diagnostics
- Error messages exposed to users without proper access control

**Requirement:**

- Error messages MUST NOT include variable values
- Error messages SHOULD include line numbers and column positions for debugging
- Error messages SHOULD include context (e.g., "Unclosed quote") but not the actual content

**Examples:**

**Bad (leaks value):**

```
Error: Failed to parse KEY=secret_password_123 at line 42
Error: Invalid character in value "confidential_token_xyz"
Error: Unclosed quote in API_KEY="sk-live-secret..."
```

**Good (safe):**

```
Error: Whitespace between key and equals at line 42, column 8
Error: Invalid character in key at line 15, column 3
Error: Unclosed double quote starting at line 23, column 12
Error: Invalid UTF-8 byte sequence at position 456
```

**Implementation Guidance:**

- Never concatenate user input directly into error messages
- Use structured error objects that separate location info from content
- Provide a debug mode flag that can optionally include values when needed for troubleshooting

### 6.4. Parser Output Safety

**Principle:** EDF parsers output literal strings without interpretation. The parser's job is to extract key-value pairs; interpretation is the consumer's responsibility.

**Implications:**

- `${VAR}` syntax is preserved as-is in parser output (literal text)
- Dollar signs, backslashes, and other special characters in values are output as-is (after escape sequence processing for double quotes)
- Consumers are responsible for any further processing (interpolation, shell execution, etc.)
- Security analysis can focus on the parsing layer separately from application logic
- The parser does not execute code, does not access environment, does not perform network requests

**Security Boundary:**
The parser creates a clean security boundary:

- **Input:** Untrusted .env file content
- **Processing:** Deterministic parsing with well-defined rules
- **Output:** Key-value pairs as literal strings
- **Consumer:** Responsible for validation, interpolation, and secure usage

**Consumer Responsibility:**
Consumers of parsed values must implement their own security measures:

- **Interpolation Security:** If implementing variable interpolation:
  - Protect against circular references (A=${B}, B=${A})
  - Limit interpolation depth to prevent stack overflow
  - Validate variable names to prevent injection
  - Document which variables can be interpolated

- **Shell Execution:** If using values in shell commands:
  - Properly escape or quote values before passing to shell
  - Use parameterized execution when possible
  - Validate that values don't contain command injection sequences
  - Consider using restricted shells or sandboxing

- **Input Validation:** Always validate parsed values:
  - Check types (numbers, URLs, paths)
  - Validate ranges and formats
  - Whitelist acceptable values when possible
  - Reject unexpected or suspicious content

## 7. Compliance

### 7.1. Compliance Statement

A parser implementation claiming **EDF 1.0.0 Compliance** MUST adhere to ALL requirements specified in this document. This is a strict, all-or-nothing compliance model.

**Requirements for Compliance:**

An implementation MUST satisfy:

- **ALL syntax validation rules** including rejection of double equals signs immediately after keys
- **ALL normative requirements** using RFC 2119 keywords (MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT)
- **ALL parsing rules** for keys, values, quotes, comments, and multiline handling as specified in Section 4
- **ALL error handling requirements** including error message formats and rejection conditions as specified throughout this document
- **ALL security requirements** including UTF-8 validation, newline injection prevention, and error message safety as specified in Section 6
- **ALL edge case behaviors** as specified throughout this document

**Non-Compliance:**
Partial implementation or "mostly compliant" parsers MUST NOT claim EDF compliance. The specification is designed to be followed completely to ensure interoperability and predictable behavior across all implementations.

**Allowed Variations:**
Implementations MAY vary in:

- Performance characteristics
- Internal implementation details
- API design and language-specific interfaces
- Additional features beyond the specification (clearly documented as extensions)

**Prohibited Variations:**
Implementations MUST NOT vary in:

- Parsing rules and outcomes
- Error conditions and rejection criteria
- Security requirements
- Any behavior specified with MUST, MUST NOT, REQUIRED, SHALL, or SHALL NOT

### 7.2. Version Compatibility

This document specifies **EDF 1.0.1**.

Implementations MUST clearly state which version of the EDF specification they implement.

**Semantic Versioning:**

- **Major version changes** (e.g., 1.x.x → 2.x.x) indicate breaking changes to parsing behavior
  - May introduce incompatible parsing rules
  - May change the interpretation of existing syntax
  - Files may parse differently between major versions

- **Minor version changes** (e.g., 1.0.x → 1.1.x) add clarifications or new optional features without breaking existing behavior
  - Add new optional syntax or features
  - Clarify ambiguous cases
  - Add new SHOULD/MAY recommendations
  - All valid 1.0.x files remain valid and parse identically in 1.1.x

- **Patch version changes** (e.g., 1.0.0 → 1.0.1) fix documentation errors without changing semantics
  - Correct typos
  - Improve examples
  - Fix formatting
  - No behavioral changes

**Version Detection:**
There is no in-file version declaration. The specification version is determined by the parser implementation version, not the file content. All files are parsed according to the rules of the parser's implemented specification version.

---

## Disclaimer

This specification is provided "as is" and without any warranties of any kind, whether express or implied, including but not limited to warranties of merchantability, fitness for a particular purpose, or non-infringement.

In no event shall the authors or copyright holders be liable for any claim, damages, or other liability, whether in an action of contract, tort, or otherwise, arising from, out of, or in connection with this specification or the use or implementation thereof.
