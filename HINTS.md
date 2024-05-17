## Notes on Parsing JCON

A primary goal of the JCON specification is to keep the difficulty of parsing 
a JCON file to roughly the same order of magnitude as that of parsing a JSON 
file. On
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

## Roadmap

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
