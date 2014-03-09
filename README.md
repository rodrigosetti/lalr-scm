# lalr-scm

Lalr-scm is yet another LALR(1) parser generator written in Scheme. In contrast
to other such parser generators, this one implements an efficient algorithm for
computing the lookahead sets. The algorithm is the same as used in Bison (GNU
yacc) and is described in the following paper:

> Efficient Computation of LALR(1) Look-Ahead Set, F. DeRemer and T. Pennello,
> TOPLAS, vol. 4, no. 4, october 1982.

As a consequence, it is not written in a fully functional style. In fact, much
of the code is a direct translation from C to Scheme of the Bison sources.

## Portability

The program has been successfully tested on a number of Scheme interpreters and
compilers, including Gambit v4.0, Bigloo 3.0, Chicken 2.6.

It should be portable to any Scheme interpreter or compiler supporting
low-level, non-hygienic macros à la define-macro. If you port lalr-scm to a new
Scheme system and you want this port to be included in the next releases,
please contact the maintainer.

## Parse Definition

### Loading the generator

The lalr.scm files declares a macro called lalr-parser:

   (lalr-parser [options] tokens rules ...)

This macro, when given appropriate arguments, generates an LALR(1) syntax
analyzer. The macro accepts at least two arguments. The first is a list of
symbols which represent the terminal symbols of the grammar. The remaining
arguments are the grammar production rules. See ParserSyntax for further
details.

### Running the parser

The parser generated by the lalr-parser macro is a function that takes two
parameters. The first parameter is a lexical analyzer while the second is an
error procedure.

#### The lexical analyzer

The lexical analyzer is zero-argument function (a thunk) invoked each time the
parser needs to look-ahead in the token stream. A token is either a symbol, or
a record created by the function make-lexical-token:

   (make-lexical-token category source value)

Once the end of file is encountered, the lexical analyzer must always return
the symbol `'*eoi*` each time it is invoked.

A lexical token record has three fields: category, which must be a symbol, a
source location object source, and a semantic value associated with the token.
For example, a string token would have a category set to 'STRING while its
semantic value is set to the string value "hello". The field accessors are:
lexical-token-category, lexical-token-source, and lexical-token-value.

The source location object is not used by the parser itself, but is usually a
record constructed by the function make-source-location:

   (make-source-location input line column offset length)

The input argument is an object describing the source input (the name of the
input file, for example), while the other arguments give respectively the line
number, the column number, the offset (in character) from the beginning of the
file, and the length of the token. The field accessors are:
source-location-input, source-location-line, source-location-column,
source-location-offset, and source-location-length.

Note: It is the responsibility of the lexical analyzer to provider the source
location information to the parser.

#### The error procedure

The error procedure must be a function that accepts at least one parameter, the
first of which is an error message (a string), while the second, if present, is
the lexical token that caused the error.

## Parser syntax

The grammar is specified by first giving the list of terminals and the list of
non-terminal definitions. Each non-terminal definition is a list where the
first element is the non-terminal and the other elements are the right-hand
sides (lists of grammar symbols). In addition to this, each rhs can be followed
by a semantic action.

For example, consider the following (yacc) grammar for a very simple expression
language:

    e : e '+' t
      | e '-' t
      | t
      ;
    t : t '*' f
      : t '/' f
      | f
      ;
    f : ID
      ;

The same grammar, written for the scheme parser generator, would look like this
(with semantic actions)

    (define expr-parser
      (lalr-parser
       ; Terminal symbols
       (ID + - * /)
       ; Productions
       (e (e + t)    : (+ $1 $3)
          (e - t)    : (- $1 $3)
          (t)        : $1)
       (t (t * f)    : (* $1 $3)
          (t / f)    : (/ $1 $3)
          (f)        : $1)
       (f (ID)       : $1)))

In semantic actions, the symbol `$n` refers to the synthesized attribute value
of the nth symbol in the production. The value associated with the non-terminal
on the left is the result of evaluating the semantic action (it defaults to
`#f`).

### Operator precedence and associativity

The above grammar implicitly handles operator precedences. It is also possible
to explicitly assign precedences and associativity to terminal symbols and
productions à la Yacc. Here is a modified (and augmented) version of the
grammar:

    (define expr-parser
     (lalr-parser
      ; Terminal symbols
      (ID
       (left: + -)
       (left: * /)
       (nonassoc: uminus))
      (e (e + e)              : (+ $1 $3)
         (e - e)              : (- $1 $3)
         (e * e)              : (* $1 $3)
         (e / e)              : (/ $1 $3)
         (- e (prec: uminus)) : (- $2)
         (ID)                 : $1)))

The left: directive is used to specify a set of left-associative operators of
the same precedence level, the right: directive for right-associative
operators, and nonassoc: for operators that are not associative. Note the use
of the (apparently) useless terminal uminus. It is only defined in order to
assign to the penultimate rule a precedence level higher than that of * and /.
The prec: directive can only appear as the last element of a rule. Finally,
note that precedence levels are incremented from left to right, _i.e._ the
precedence level of + and - is less than the precedence level of * and / since
the formers appear first in the list of terminal symbols (token definitions).

### Options

The following options are available.

 * `(output: name filename)` - copies the parser to the given file. The parser is given the name name.
 * `(out-table: filename)` - outputs the parsing tables in filename in a more readable format.
 * `(expect: n)` - don't warn about conflicts if there are n or less conflicts.
 * `(driver: glr)` - generates a GLR parser instead of a regular LALR parser.

### Error recovery

lalr-scm implements a very simple error recovery strategy. A production can be
of the form

   (rulename
      ...
      (error TERMINAL) : action-code)

(There can be several such productions for a single rulename.) This will cause
the parser to skip all the tokens produced by the lexer that are different than
the given TERMINAL. For a C-like language, one can synchronize on semicolons
and closing curly brackets by writing error rules like these:

   (stmt
      (expression SEMICOLON) : ...
      (LBRACKET stmt RBRACKET) : ...
      (error SEMICOLON)
      (error RBRACKET))

### A final note on conflict resolution

Conflicts in the grammar are handled in a conventional way. In the absence of
precedence directives, Shift/Reduce conflicts are resolved by shifting, and
Reduce/Reduce conflicts are resolved by choosing the rule listed first in the
grammar definition.

## GRL Parsing

GLR parsing (which stands for Generalized LR parsing) is an extension of the
traditional LR parsing technique to handle highly ambiguous grammars. It is
especially interesting for natural language processing, the context in which
the technique has been devised.

### Declaration

To generate a GLR parser instead of a regular LALR parser, simply put the
following option in the options section of the grammar declaration:

  (driver: glr)

### Running the parser

GLR parsers are run in exactly the same way as regular LALR parsers. The only
difference is that the result of the parsing is a (possibly empty) list of
parses instead of a single parse.

### Limitations

Since the parsing of a phrase can lead to many potential parses, errors cannot
be detected as easily as with deterministic LR parsing. For this reason, it is
advised to not put error productions in the grammar (they will be ignored
anyway). Moreover, GLR parsers are usually not meant for interactive parsers
(like the calculator example that comes with the distribution).

