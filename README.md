# JCON Configuration Data Format

License: Creative Commons Zero 1.0 Universal

JCON is a superset of JSON that defines JSON-equivalent values, designed for 
use in configuration files. JCON has the following motivating principles:

  * JCON files should be convenient for humans to write.
  * JCON files should be pleasant for humans to read.
  * JSON must be valid JCON syntax in any context.
  * The result of parsing a JCON file must be a JSON-compatible value (no
    additional data types, no metadata, etc).
  * JCON parsing should not be significantly more difficult than JSON parsing

The JSON spec is, in some ways, a marvel of expressive simplicity. It almost
invites you to write your own parser. JCON aspires to keep a similar degree of 
simplicity in the specification, while enabling many more affordances for 
humans to read and write it. Other reasonably sane assumptions have been made.

A JCON file might look very JSON-like, with minor differences:

    // Rolled-My-Own Email Client v0.1 config
    { 
      account: {
        email: "bighair@metalcoder.com"
        delete-folder: "Trash"
        archive-folder: "Keep",
        fetch: "all",
        signature: "--\nFrom the desk of BIGHAIR\n"
      }

      // My badass Norton Commander color scheme
      skin: {
        fg: "#ff88ff"
        bg: "#000088"
        bold: "#ffffff"
      }
    }

Or it could look more more like a TOML or INI file. This file is exactly
equivalent to the previous one:

    /**********************************************************
     * Rolled-My-Own Email Client v0.1 Config
     */

    [account]

    email =           bighair@metalcoder.com
    delete-folder =   Trash
    archive-folder =  Keep
    fetch =           all
    signature = """
    --
    From the desk of BIGHAIR
    """

    [skin]        // My badass Norton Commander color scheme
    
    fg        = #ff88ff
    bg        = #000088
    bold      = #ffffff

JCON offers a number of extensions to the JSON specification. Importantly, the 
extensions do not change the result of parsing a JCON file; a valid JCON file, 
once parsed, can be exactly represented by a valid JSON file. Also, in every 
context in a JCON extension where a value is expected, any valid JSON value 
(embedded whitespace, line feeds, and all) can be used with completely 
unambiguous results. 

There is one significant limitation: JCON files must represent an object. A 
JCON file that defines a different type of data (array, string, etc) is invalid.

The extensions to JSON are:

  * Omit outermost object braces
  * Trailing commas are allowed in objects and arrays
  * Hexadecimal and binary numbers
  * Separators in numbers
  * Unquoted object keys
  * Line-delimited features:
    * Omit comma at end of line
    * Line comments
    * Multi-line comments
    * Assignment syntax for name/value pairs
    * Sections
    * Heredocs

As intimated above, JCON introduces lines as semantic elements in the data 
format, and defines six additional syntax elements that may appear in a 
line-delimited context. These provide support for comments and various 
alternatives for defining objects and the overall structure of the file.

## Omit Outermost Object Braces

A JCON file is not valid unless it represents an object. This implies that every
JCON file must begin with a `{` codepoint and end with a `}` codepoint (aside
from whitespace and comments). And, since they are required elements in every 
file, you can simply omit them and infer their presence if they're missing. So 
these two JCON files are exactly equivalent:

File 1:

    {
      account: {
        email: "bighair@metalcoder.com"
      }
      skin: {
        fg: 0xff88ff
      }
    }

File 2:

    account: {
      email: "bighair@metalcoder.com"
    }
    skin: {
      fg: 0xff88ff
    }

## Hexadecimal and Binary Numbers

A hexadecimal number may be written with the codepoints `0x` followed by 
one or more hexadecimal digits (that is, `0123456789abcdefABCDEF`).

    +----+    +-------------------------------+
    | 0x |----| any of 0123456789abcdefABCDEF |-\
    +----+   7+-------------------------------+  |
            |                                    |
             \----------------------------------/

A binary number may be written with the codepoints `0y` followed by one 
or more binary digits (either `0` or `1`).

    +----+    +-----------+
    | 0y |----| any of 01 |-\
    +----+   7+-----------+  |
            |                |
             \--------------/

Hexadecimal or binary numbers may be written anywhere a JSON number could be
written. When converted to JSON, hexadecimal and binary numbers are represented
as integer decimal values.

## Separators in Numbers

The underscore `_` codepoint may be used as a separator in any type of number,
provided only that each separator must be preceded _and_ followed by 
a digit of the number. Separators are purely cosmetic, and do not imply anything
about the value of the digits they separate. All separators are discarded before
determining the value of a number.
 
So, the following are all valid numbers:

    16_384
    3.141_593
    0xdead_beef

Some of these are also valid numbers, but the style police will come for you:

    1_._000_0020
    16__384_
    _007_183_440        // not a number, it's actually a string (or an error)

## Unquoted Names

Anywhere within a JCON file where a name string is expected as part of a 
name/value pair, you may instead use an unquoted name.

Unquoted names must begin with an identifier-like or number-like
codepoint, namely one of `a-zA-Z_$0-9-`. However they may be continued with any 
codepoints except `:`, `=`, `,`, ` ` (however no control codepoints are 
allowed.) This means, for example, that any name containing whitespace must be 
quoted. There is no escape sequence or other method to represent non-literal 
codepoints in an unquoted name; just use a regular quoted string for the name 
if necessary.

Unquoted name:

    +-------------------------+    +-----------------------+
    | any of a-zA-Z_$0-9-     |----| any but :,=<WS><CTRL> |-\
    +-------------------------+   /+-----------------------+  |
                                 |                            |
                                  \--------------------------/

Some examples of unquoted names:

    { 
        fancy: "pants", ur-a: "monster", -moz-crap: "whatever"
        0: 1, 1: 1

        // using assignment syntax
        feeble[0] = minded
        -flags = -i, -d, --fast-math
        2 = 2
        3 = 3
        4 = 5
        5 = 8
        // ...etc...
    }

## Trailing Commas

It is considered valid syntax for a comma to follow the final entry in an 
object or array. This extends the JSON collection specifications.

Examples of trailing commas:

    [1, 2, 3, ]             // A 3-element array

or

    feature-flags: {
      "banner-test": true,
      "new-ad-conversion-monitor": true,
      "dark-revenue-pattern-7": "cohort 7/10",
    }

## Line Delimiters

All line-delimited features rely on the notion of "lines" in the JCON file.
Any of the following codepoint sequences is a valid line terminator anywhere
within a JCON file:

  * `CR`, `LF`
  * `LF`, `CR`
  * `CR`
  * `LF`

The longest matching sequence of codepoints is always given preference 
when identifying a line terminator.

The goal with this syntax is extreme compatibility: Configuration files are 
reknowned for their cobbled-together nature, and modern editors can be quite
permissive in their interpretation of "line endings". JCON is designed to 
support your preferred or emergent line ending convention without judgement.

## Omit Comma at End of Line

When defining a collection, it is valid to separate two elements with a line
terminator instead of a comma. For example:

    active-feature-flags: [
      "banner-test"
      "new-ad-conversion-monitor"
      "dark-revenue-pattern-7"
    ]
    logging: {
        style: "webhook"
        base-url: "https://logsink.metalcoder.com"
    }

## Comments

In keeping with JSON's origins, JCON uses Javascript conventions for comments.

Line comments may appear at the beginning of a line, or after any whitespace
(outside of a quoted string) on a line:

    +----------+   +----+    +---------------+      +---+
    | BOL or   |---| // |----| any codepoint |------|EOL|
    |whitespace|   +----+   /| except CR,LF  | \    +---+
    +----------+           | +---------------+  |
                           |                    |
                            \------------------/

Multi-line comments use the "/*" syntax from Javascript, but with a significant
limitation: JCON does not allow multi-line comments to begin or end on a line
that contains any data element. The intended purposes for this syntax are
blocks of documentary text or to hide blocks of lines from the data parser.

The multi-line comment opener must be the first non-whitespace element on
a line:

    +---+  +----+    +---------------+    +---+
    |BOL|--| /* |----| any codepoint |----|EOL|
    +---+  +----+  / | except CR,LF  | \  +---+
                  |  +---------------+  |  
                  |                     |
                   \-------------------/

The multi-line comment terminator must be the last non-whitespace element on a
line.

    +---+      +---------------+    +----+   +---+
    |BOL|------| any codepoint |----| */ |---|EOL|
    +---+    / | except CR,LF  | \  +----+   +---+
            |  +---------------+  |  
            |                     |
             \-------------------/

## Assignment Syntax For Objects

This line-delimited syntax for defining an object's name/value pairs uses an 
equal sign (`=`) instead of a colon (`:`) to separate the name from the value. 
It is defined as:

    +---+  +------+  +---+  +-------+  +----------+  +---+
    |BOL|--| name |--+ = |--| value |--| comment  |--|EOL|
    +---+  +------+  +---+  +-------+  |(optional)|  +---+
                                       +----------+

Horizontal whitespace may optionally appear before or after any element in the 
line.

It looks like this:

    email =           bighair@metalcoder.com      // hey that's me!
    delete-folder =   Trash
    // gotta quote this next one because it looks like a comment
    mailroot =        "//c/Users/bighair/.mail"

When using assignment syntax only, you may substitute an _unquoted value_
for a normal JSON value. An unquoted value is a sequence of
codepoints that does not begin or represent a valid JSON value.
Unquoted values are always strings, and are subject to the following 
restrictions:

  * May not contain control characters other than horizontal TAB
  * May not contain the `//` or `/*` codepoint sequences
  * May not begin with the `[`, `{`, `"`, or `=` codepoints
  * Any whitespace is removed from the beginning and end of the value
  * The values `true`, `false`, and `null` strictly represent their JSON values

Some examples of what not to do and what works with assignment syntax:

    // this is a syntax error since "b" is not the first name on the line
    a:5, b = 2
    // This is valid but misleading. "b" is set to the value "2, c:3, d:4"
    b=2, c:3, d:4
    // This is a syntax error. Unquoted values may not contain `=`
    b=2, c=3, d=4
    // But you could do this
    b="2, c=3, d=4"
    // this doesn't make an array
    e = 4, 5, 6
    // but this does
    f = [7, 8, 9]
    // and so does this
    f = [
        7
        8 
        9
    ]
    // this also makes an array but looks a bit odd
    g = [10, 
    11, 12]
    // this is a number
    h = 13
    // these are strings:
    i = 14.
    j = 3.1415.9
    k = 1: Intro to Science

## Sections

Sections are defined as:

    +---+  +---+  +---------------+  +---+  +----------+  +---+
    |BOL|--| [ |--| string or     |--| ] |--| comment  |--|EOL|
    +---+  +---+  | unquoted name |  +---+  |(optional)|  +---+
                  +---------------+         +----------+

Horizontal whitespace may appear between any of the elements.

Sections provide an alternate syntax for defining the top-level name/value
pairs in a JCON file. When used, sections separate second-level object 
definitions. They look like this:

    [account]

    email =           bighair@metalcoder.com
    fetch =           all

    [skin]        // My badass Norton Commander color scheme
    
    fg        = #ee77ee
    bg        = #000044

    [hotkeys]

    reply           = ctrl+enter
    reply-all       = ctrl+shift+enter

The parsed result, re-saved as a JSON file, might look like this:

    {
      "account": {
        "email": "bighair@metalcoder.com",
        "fetch": "all"
      }, "skin": {
        "fg": "#ee77ee",
        "bg": "#000044"
      }, "hotkeys": {
        "reply": "ctrl+enter",
        "reply-all": "ctrl+shift+enter"
      }
    }

It is intentional that, after parsing, you cannot tell whether the original file
used section syntax or object syntax. Also note that, like objects, sections
represent an _unordered_ collection. Additionally:

  * A section may only appear where an object name is expected.
  * Placing a section inside an explicit collection is an error.
  * If sections are used, the first section must appear before any other
    name/value pairs.

Here are some examples of improper use of sections. First, an invalid JCON file 
that is missing a section declaration before the first key/value pair:

    flags: ["-a","--verbose=3"]

    [Disk]   // this is an error - what section would "flags" be in?
    mount: "/dev/sda2"

An invalid JCON file that declares a section when a value is expected:

    flags: 

    [Disk]   // this is an error, not a valid JSON value    
    mount: "/dev/sda2"
    
A valid JCON file that might not yield the expected result:

    ["1. Introduction"]
    color: 

    [2]   // this is NOT an error, it's a valid JSON value for the preceding
          // name
    highlight: "#ff0000"

You cannot use a heredoc to define a section name.

## Heredocs

Anywhere in a JCON file where a quoted string could be used as a value, you may 
instead use a heredoc. Heredocs are a line-delimited syntax that allow the use 
of a literal fragment of the JCON file as a string. They look like this:

    mysql_config = """
    [mysqld]
    # The directory where MySQL stores its data files.
    datadir=/var/lib/mysql

    # The port on which the MySQL server listens for incoming connections.
    port=3306
    """

There is no escape character or other mechanism for representing
non-literal codepoints in a heredoc. A heredoc's value is taken from the lines
between the opener and the terminator. Its string value is always an
exact copy of that portion of the JCON file. This definition
has several useful implications:

  - A heredoc can be a completely empty string, if the terminator immediately
    follows the opener
  - The consumer of the JCON data may implement its own character escaping
    and/or value substitution conventions without any conflicts
  - Potentially more efficient and straightforward implementation of heredoc
    strings, since they are simply substrings of the JCON file
  - It is possible (insane, but possible) to use different line terminating
    codepoints in a heredoc than are used in the rest of the file, and the
    difference will be captured in its value

A heredoc opener is defined as:

    +-----+   +---------------+   +----------+  +---+
    | """ |+++| unquoted name |---| comment  |--|EOL|
    +-----+   |   (optional)  |   |(optional)|  +---+
              +---------------+   +----------+

Whitespace is _not_ allowed between the triple quote and the unquoted name, if
present. Horizontal whitespace may optionally appear between other elements in 
the heredoc opener. A heredoc opener may appear anywhere a JSON string value
would appear (except the name part of a name/value pair).

A heredoc terminator looks like this:

    +---+   +-----+   +---------------+   +----------+  +---+
    |BOL|---| """ |+++| unquoted name |---| comment  |--|EOL|
    +---+   +-----+   |   (optional)  |   |(optional)|  +---+
                      +---------------+   +----------+

Again, whitespace is _not_ allowed between the triple quote and the unquoted 
name, if present. Horizontal whitespace may optionally appear between other 
elements in the heredoc terminator.

In order for a heredoc terminator to actually terminate an open heredoc, the
unquoted name must exactly match the name used to open the heredoc. If no name
was used to open the heredoc, the terminator must not include a name either.

Heredocs represent JSON strings, and contain the literal file contents from 
the line following the heredoc opener through to and including the line 
terminator that precedes the heredoc terminator.

You may not define the name part of a name/value pair with a heredoc. Also,
you may not use a heredoc insteaad of a quoted string as a section name.

An example of a heredoc that includes a potentially conflicting line, and so 
uses an unquoted name to disambiguate the heredoc terminator:

    script: """code
        retval = """
        This Python string spans
        multiple lines
        """
    """code

Note that the whitespace at the beginning of each line in the JCON heredoc
above would be included in the string that it is parsed into.

## Appendix 1: Notes on Parsing JCON

A primary goal of the JCON specification is to keep the complexity of parsing 
a JCON file to within the same order of magnitude as parsing a JSON file. On
the surface, there are many new syntatic elements and some ambiguity that 
can only be resolved by examining the entirety of a line. The specification
has been carefully laid out to make it easier for a practical parser to make
decisions on the fly without needing to backtrack any more than is needed for
JSON.

Some hints for how to handle the trickier bits in a stream-oriented parser
follow. If you're using a parser compiler, none of this will be helpful because
the grammar accounts for all of these cases already.

### Restrictions

Some of the restrictions on the use of particular syntaxes are present purely
to make it easier to catch inadvertent errors in the data, while some are 
necessary to prevent ambiguity in the results of parsing. It is strongly 
reccommended to implement all of the restrictions in the specification.

Some examples of restrictions that are not strictly necessary for disambiguation 
but help ensure that some human mistakes are caught:

  * No whitespace or `,` in an unquoted name
  * No `=` allowed at the beginning of an unquoted value
  * No `//` or `/*` allowed within unquoted names or values
  * No heredocs as section names or as names in name/value pairs

The unquoted names and values are hugely valuable for their ease of use, but
they do come with the potential for valid-but-unexpected results.

### Lines

JCON handles lines somewhat differently from most other text formats. The line
termination codepoint sequences are explicitly defined; you cannot rely (at 
least not entirely) on your language runtime or OS definition of "line" when 
parsing the file.

It might be easiest to read the file in a literal or "binary" mode that 
preserves the line terminator codepoints in the data, and do the line 
terminator detection yourself. Here is one way to approach that that does not 
rely on peeking ahead in the data:

  * Whenever a `<CR>` or `<LF>` codepoint is encountered, end the current line
    and save the codepoint you found.
  * If you are at the beginning of a line and you read a `<CR>` or `<LF>`, 
    check to see whether it is different from the saved codepoint from the
    previous line. If it is, silently absorb it. Otherwise end the current 
    line (you have an empty line).

### Initial Data

At the beginning of a JCON file, once you've passed any whitespace and comments,
there are four possibilities for the first element you will encounter:

  - `[` starts a section (not an array because JCON files are always objects)
  - `"` begins a quoted name or heredoc
  - `a-zA-Z_$0-9-` begin an unquoted name
  - `{` is an explicit definition of the top-level object
  - any other code point would be an error

If the first element you encounter is not a section, then there will be no
sections in the file (or if there are it's an error).

### Unquoted Values

Unquoted values are only valid when using assignment syntax. There are just
too many weird and ambiguous situations that can arise if unquoted values
are allowed in a regular name/value pair, or in a normal array definition.

Unquoted values can be tricky because they can be ambiguous until the end of
the value has been identified. A simplistic but complete approach might look 
like this, starting after the `=` that denotes assignment syntax:

  * bypass any whitespace
  * depending on the next character, you might not have an unquoted value:
    * if `"`, begin a string or heredoc
    * if `[`, begin an array (may continue to following lines)
    * if `{`, begin an object (may continue to following lines)
    * if `=` emit an error
  * otherwise, begin defining an unquoted value string
  * the string is complete once you encounter
    * a line terminator
    * end of file, or 
    * horizontal whitespace followed by the `//` codepoints
  * trim any whitespace off the end of the unquoted value
  * if the string is empty, emit a missing value error
  * if the string contains the `//` or `/*` sequences, emit an error
  * if it is `true`, `false`, or `null` then substitute the JSON value
  * if it looks like a valid JCON number then re-parse it as a number
  * otherwise the string you have is the value

There are some odd-looking edge cases with comments:

    not_a_comment=//error, unquoted value may not contain `//`
    is_a_comment= //this *is* a comment (and a missing value error)
    not_a_file =  //c/Users/this_is_also_a_comment_and_error.txt
    is_a_file = "//c/Users/gotta_quote_it.txt"  // filename
    probably_wrong =  rm //c/Users/this_part_is_a_comment.txt

## Appendix 2: Roadmap

### Future

This specification is a work in progress. The chosen features have been 
carefully winnowed from the universe of possible extensions to JSON, with the 
goal of introducing some useful legibility aids with the least possible 
additional complexity.

Possible future additions, if suitably simple, unambiguous, and expressive 
implementations can be found, include:

  * Indented Heredocs
  * Value References

### Past

Some ideas that have died a noble death and should be left to rest in peace:

  * Indented Collection Definition
  * Line Continuation Syntax
  * Preprocessing Macros
  * Binary Blobs
  * Type Metadata Syntax



