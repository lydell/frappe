Frappe
======

JavaScript with some nice fluff on top of it.

Frappe is a modest compile-to-JS language inspired by [CoffeeScript], without
taking it very far from plain JavaScript.

**Note:** Frappe has no implementation. I once read something like this on the
Internet (unfortunately I couldn’t find where when writing this):

> Whenever you feel like creating your own programming language, lay down and
> try to resist the urge.

Perhaps it’d be lots of fun. Perhaps it’d be a huge waste of time. Perhaps it’d
be useful. Will it happen? I have no idea.

[CoffeeScript]: http://coffeescript.org/

Golden rules
============

1. Everything should work and have exactly the same syntax as in JavaScript
   module code (script code is not suppored), except the things noted below,
   which should not be too many. Frappe rather leaves things out than adding new
   features.

2. There should be syntax for everything that [Babel] can parse.

3. Use the [UNIX philosophy]. Keep Frappe as small as possible and use it in
   conjunction with other tools (such as [Babel]).

[Babel]: http://babeljs.io/
[UNIX philosophy]: http://www.faqs.org/docs/artu/ch01s06.html


Syntax
======

Aliases
-------

- `and` -> `&&`
- `or` -> `||`
- `not` -> `!`
- `is` -> `===`
- `isnt` -> `!==`

  In English, “isn’t” is an abbreviation for “is not”. Should `is not` → `!==`,
  too?

  There are two ways to go here. One is to allow `is not` → `!==`. The other is
  to add a separate operator, like `isnt`. If we _don’t_ allow `is not` → `!==`,
  it doesn’t matter what the separate operator is—we will _always_ have the
  possible `is not` → `!==` vs. `is not` → `=== !` confusion.

  If we _do_ allow `is not` → `!==` we trade one confusion for two others. `not
  expression` would no longer always compile to `!expression`—it’d be dependent
  on if the previous token is `is`—and `is` would no longer always compile to
  `===`—it’d be dependent on if the next token is `not`. This has been [debated
  in CoffeeScript] before.

  Note that Python has all three of `is`, `not` and `is not`, but in Python `==`
  and `is` mean different things and `a is (not b)` is pointless, so looking at
  Python is no help.

  In the end, the easiest way to go is to just do it like CoffeeScript. After
  all, lots of people are used to it. I also find `expression1 isnt expression2`
  a more obvious negated comparison than `expression1 is not expression2`,
  because of the extra space. It’s better to have the linter catch `a is not b`
  as unclear, possibly erraneous code.

- `@` → `this.`. Unlike CoffeeScript, `@` strictly means `this.`, so `@` by
  itself is invalid (use `this` instead), as is `@[foo]` and `@()` (use
  `this[foo]` and `this()` instead). This (no pun intended) should not
  interfere with JavaScript’s proposed decorators, since they can only be used
  before class method definitions.

[debated in CoffeeScript]: https://github.com/jashkenas/coffeescript/issues/3201

Operators
---------

- `a not instanceof b` → `!(a instanceof b)`
- `a not in b` → `!(a in b)`
- `a < b < c` → `a < b && b < c` (also applies to `>`, `<=` and `>=`)
- `a %% b` → `(a % b + b) % b`. “Useful” modulo.


Conditionals and loops
----------------------

- Parentheses are not required around the condition (see also [Significant
  indentation]):

  ```
  if condition // valid
     block
  if condition statement // invalid
  ```

  This applies to `if`, `while` and their aliases below, as well as `for`,
  `switch` and `catch`.

- `unless expression` → `if not (expression)`
- `until expression` → `while not (expression)`
- `statement if expression` → `if (expression) statement` (also applies to
  `unless`)
- `statement while expression` → `while (expression) statement` (also applies to
  `until` and `for`)

[Significant indentation]: #significant-indentation

Parentheses-less function calls when used as statements
-------------------------------------------------------

    array.push item // valid
    console.log 'item', item // valid
    console.log 'item', item.indexOf 'foo', 'bar' // invalid
    let t = window.setTimeout function(){}, 100 // invalid

Sane “semicolon insertion”
--------------------------

    a = b
    ['one', 'two', 'three'].forEach(…)

In Frappe, the above means the same as if you’d put a semicolon after the `b`
(or before the `[`), not `a = b['one', 'two', 'three'].forEach(…)`. This
probably means not allowing `a = b [0]` and `a = fn (arg)`, but that’s okay
since that style is uncommon, as far as I can tell.

Strings
-------

In JavaScript there are three different types of strings: `'`-quoted, `"`-quoted
and `` ` ``-quoted. The first two work exactly the same way. Using one or the
other is a stylistic choice or to avoid escaping of the delimiter. The latter
has mostly different semantics.

In Frappe, all of those three types have _exactly_ the same behavior. The one
that results in the least escaping should be used. For example, if a string
contains lots of `'`s, use either a `"` or `` ` `` as the delimiter. If it
contains lots of `'`s _and_ `"`s, use a `` ` `` string.

Frappe’s string semantics are a mixture of JavaScript’s `` ` `` strings and
CoffeeScript’s strings.

- When strings contain neither newlines nor `${` they work exactly like
  JavaScript `'` and `"` strings.

- When strings contain a newline (but does not start with it):

      throw new Error('Some long error message
                       written on several lines')
      // Equivalent to:
      throw new Error('Some long error message written on several lines')

  Trailing whitespace of the first line, the newline and any whitespace (which
  includes empty lines) up to the next non-whitespace character are replaced by
  a single space. That single space can be suppressed by ending the line with a
  backslash.

      console.log 'https://example.com/some/\
                   really/long/url'
      // Equivalent to:
      console.log 'https://example.com/some/really/long/url'

  This works exactly like CoffeeScript.

- When strings start with a newline (optionally with trailing
  non-newline-whitespace before it):

      console.log '
        Usage: cli-program [options] file

          -h, --help  Display this help message
          -c          Really long description \
                      written on several lines
      '
      // Equivalent to:

      console.log 'Usage: cli-program [options] file\n\n  -h, --help  Display this help message\n  -c          Really long description  written on several lines'

  The shortest indentation is stripped from each line, and newlines are
  retained—or rather _replaced with_ `\n` newlines (regardless if the input
  used for example `\r\n`). This works like CoffeeScript’s `'''` and `"""`
  strings.

  Trailing whitespace and _singleline comments_ (`// comment`) are stripped
  from each line! If you _want_ trailing whitespace in a line, a variety of
  techniques can be used:

      '
        a	\t  // tab escape → “a” followed by two tabs
        a    \x20  // space escape → “a” followed by five spaces
          \x20 // space escape → a line consisting of three spaces
        a  ${} // empty interpolation (see below) → “a” followed by two spaces
      '

  Not adopting CoffeeScript’s triple-quoting syntax has the benefit of
  not having to define what for example the following should mean:

      ''' a
        b
      '''

  How much indentation should be stripped from each line? Besides, there’s
  never any need to start a single-quoted string with a newline:

      '
        a
      '
      // Equivalent to (in CoffeeScript):
      'a'

  So using triple-quoting is just redundant. Another use case for
  CoffeeScript’s triple-quoted strings, though, is to avoid escaping if your
  string contains lots of `'`s and `"`s. In Frappe you could just use a `` `
  `` instead. But what if you’ve got lots of `` ` ``s as well? That’s about the
  same as if your CoffeeScript string contains lots of `'''`s and `"""`s.

- All strings can contain interpolation, using the `${}` syntax from
  JavaScript’s `` ` `` strings.

      'a${2}b'
      // Equivalent to:
      "a${2}b"
      // Equivalent to:
      `a${2}b`
      // Equivalent to:
      'a2b'

  CoffeeScript only allows interpolation in `"` strings. I’ve never stumbled
  upon a case where there has been so many literal `#{`s (CoffeeScript’s
  interpolation syntax) in the string that simply escaping them to `\${` would
  be too cumbersome. Besides, [CoffeeLint] has a rule that forces you to escape
  `#{}` to `\#{}` in `'` strings to clearly show that you didn’t make the
  mistake of adding interpolation to a string but forgot to change the
  delimiters to `"`.

  Allowing interpolation in all strings allows to choose the delimiter based on
  coding style or to avoid escaping, and still having interpolation if needed.

- All strings can be tagged, not just `` ` `` strings. `tag'a'`, `tag"a"` and
  `` tag`a` `` all mean the same thing.

- All strings can be used everywhere. In JavaScript, `` ` `` strings cannot be
  used in some places:

      import foo from `./foo.js` // valid Frappe, invalid JavaScript
      // While allowing ` strings, interpolation and tags of course still
      // not valid:
      import foo from `./${foo}.js` // invalid Frappe
      import foo from tag'./foo.js' // invalid Frappe

      let obj = {
        `a`: 1, // valid Frappe, invalid JavaScript
        // Frappe even allows interpolation and tags in object literals!
        `${a}`: 1, // valid Frappe → ``[`${a}`]: 1``
        tag'a': 1, // valid Frappe → `[tag`a`]: 1``
      }

Regex
-----

Regex literals quickly become difficult to read in JavaScript. CoffeeScript
provides `///`-delimited regexes (similar to `"""`-delimited strings), which a
step forward, but the triple-slash delimiter is clunky.

In addition to JavaScript’s `/regex/g` syntax, Frappe allows putting a `#` in
front of any (un-tagged) string literal, making it a regex. The string literal
works exactly like they do otherwise, with one exception: unescaped whitespace
is removed.

Regex flags are put between the `#` and the string literal. Having them before
the regex itself lets you know if it is for example case sensitive before trying
to understand what the regex does.

    integer = #'\d+'
    protocol = #i"^ [a-z]+ ://"
    string = #g`
      (["'])         // start delimiter
      (?:
        (?!\1) [^\\] // any character except the delimiter and backslashes
        |
        \\\\[^]      // any escaped character
      )*
      \1             // end delimiter
    `
    mention = #'\b@${regexEscape(username)}\b'
    assert #`\ `.test(' ')

[CoffeeLint]: http://coffeelint.org/

Consistently short arrow function syntax
----------------------------------------

JavaScript’s `x => x * 2` is great because it is such a short syntax for lambda
functions. Unfortunately, it often gets longer because of multiple arguments or
destructuring, which requires parentheses. Therefore Frappe has an alternate
syntax for arrow functions in addition to the standard one.

    array.map(x => x * 2)
    // Equivalent to:
    array.map(|x> x * 2)

    array.map((x, index) => x * index)
    // Equivalent to:
    array.map(|x, index> x * index)

    array.map(({value}) => value * 2)
    // Equivalent to:
    array.map(|{value}> value * 2)

    window.setTimeout(() => alert 'Hello, world!', 1000)
    // Equivalent to:
    window.setTimeout(|> alert 'Hello, world!', 1000)

This way you don’t need to constantly remove and re-add those parentheses when
your function changes.

Frappe also has arrow generators: `*|>` → `function*(){}.bind(this)`.

Sane member access on number literals
-------------------------------------

`5.toString()` → `5..toString()`

Number and string member access
-------------------------------

- `a.5` → `a[5]`
- `a.'prop-name'` → `a['prop-name']`
- `a.'prop-${b}'` → ``a[`prop-${b}`]``

“Automatic comma insertions”
----------------------------

In arrays, objects and parameter lists:

    let array = [
      'one'
      2
      three
    ]
    let object = {
      a: 1
      b: 2
    }
    foo
      arg1
      arg2
      arg3


Significant indentation
-----------------------

Frappe does not use braces for blocks (but still require them for object
literals), and requires a newline to start blocks.

```
if condition
  foo = bar

if (condition) // You can of course wrap any expression in parentheses if you want to.
  foo = bar

if (condition) foo = bar // Invalid. Newline required.
foo = bar if condition // Valid.

function add(a, b)
  return a + b

function add(a, b) return a + b // Invalid. Newline required.

add = (a, b) => a + b // Valid.
add = |a, b> a + b // Valid.

class Person
  constructor(@firstName, @lastName)
    @fullName = "${@firstName} ${@lastName}"

  present() console.log "Hi, I’m ${@fullName}!" // Invalid. Newline required.

  present()
    console.log "Hi, I’m ${@fullName}!" // Valid.
```

Passing multiple function literals as arguments to a function, or as items of an
array:

```
foo arg1, arg2, |>
  bar // Passing more arguments after this function literal is not possible.

foo arg1, arg2, (|>
  bar
), arg4 // You need to wrap in parentheses.

foo
  arg1, arg2
  |>
    bar
  arg4 // Use the indented style with “automatic comma insertions”.
```

Undecided features
------------------

- `a?.b?()` etc. Let’s see what the JavaScript proposals here come up with.
  Related: `a ?= b`, `a ? b` (the last one conflicts with `a ? b : c`).

Considered features intentionally left out
------------------------------------------

- `#` for single line comments. `#` is beautiful and short and used in many
  languages, but it deviates from JavaScript for little reason. Better to keep
  `//` doing what it does in JavaScript, and use `#` (one of the few unused
  ASCII characters) for something more useful.

- “Automatic comma inseriton” between operator-less expressions in arrays and
  objects (and possibly parameter lists).

      ['one' 2 three fn() a.b (1 + 2)] // valid
      [1 1 + 2 3 a instanceof b] // invalid

  It’s too far from JavaScript, and those commas don’t hurt that much.

### “Switches and if-else-es”

- Automatic `break`s in `case`s, and disallow `break` to break the
  `switch`—instead make `break` always break loops. Use `case-fallthrough` for
  fallthrough.

      while a < b {
        switch foo {
          case 1:
            bar()
            // Automatic break
          case 2:
            break // breaks the `while` loop
          case 3:
            bar()
            case-fallthrough
          case 4:
            baz()
        }
      }

  This would be nice, but is too confusing compared to JavaScript.

- `switch { cases }` to switch on thruthiness.

- Some statements being expressions, especially `if-else`, `switch` and `try`,
  but not loops (see [Intentionally left out CoffeeScript features]). But
  disallow `a = b if c` (use `a = if (c) b else undefined` or `a = c ? b :
  undefined` instead). Let’s see what happens with the JavaScript proposals on
  this subject.

      let type = switch {
        case foo.bar(): 'bar'
        case foo.baz is 5: 'baz'
        default: 'unknown'
      }

The only thing I miss here from CoffeeScript is this:

```coffee
value = switch
  when value.type is 'element' then foo
  when value instanceof Dice then bar
  when value.length > 0 then baz
  else null
```

What we need is a nice alternative to nested ternaries:

```coffee
let value =
  value.type === 'element' ? foo :
  value instanceof Dice ? bar :
  value.length > 0 ? baz :
  null
```

I have no good ideas yet, though.


Intentionally left out CoffeeScript features
--------------------------------------------

- `yes` → `true`, `on` → `true`, `no` → `false`, `off` → `false`. Frappe strives
  to take as few departures from plain JavaScript as possible, and then those
  aliases provide too little to be justified. Unlike the `and`, `or`, `not` and
  `is` aliases, which _word_ operators in addition to _symbol_ operators, `yes`,
  `on`, `no` and `off` are words in additon to _other words_ (`true` and
  `false`), so they make less of a difference.

- CoffeeScript’s `class` semantics. JavaScript’s `class` semantics are used
  instead.

- The `extends` operator. Use JavaScript’s classes or good old manual prototypal
  inheritance instead.

- The `::` → `.prototype.` operator. Not used enough to warrant that shortcut.
  Will be used even less now that JavaScript has `class` syntax. There is also a
  proposal to use [`::` as a bind operator, which has already been implemented
  in Babel][bind-operator].

- `a in b` → `b.indexOf(a)`. Too confusing for Frappe to change one of
  JavaScript’s operators. Use `b.includes(a)` instead.

- `==` → `===`. Again, too confusing to change an operator. Frappe does not try
  to make you avoid things many consider bad parts of JavaScript (although it
  avoids Script code entirely, only Module code is supported). That’s the job of
  linters. Use `===` or `is` instead.

- Loops being expressions returning arrays. Use the proposed JavaScript
  comprehensions instead (supported by Babel), or `.map`, `.filter` and friends.

- CoffeeScript’s `for in` loops. Use `for of` and `.forEach` and friends
  instead.

- Literate CoffeeScript. Would be better to create a Literate Anything compiler,
  that takes a markdown document as input, extracts and joins the code in it,
  uses a passed in compiler to compile it and then adjusts the source map back
  to the literate code.

- Implicit returns in all functions. Too far away from JavaScript.

- Automatic variable declaration. Too far away from JavaScript. Let’s you use
  JavaScript features such as `const` without being inconsistent. Avoids the
  debate around how scoping should work.

- Array slicing and splicing syntax: `a[0..2]`. While I like slicing syntax,
  Frappe wants to be minimal. _Every_ little CoffeeScript feature cannot go in.
  This one does not make it because it is clear what `.slice` and `.splice`
  does, while `..` and `...` exclusive/inclusive-ness can be debated.

- ` a // b` floored division. Never used it. The syntax conflicts with comments,
  and I don’t feel like inventing a new syntax. `Math.floor(a / b)` is easy
  enough to remember.

- Embedded JavaScript. Only time I’ve used it is to use new JavaScript features
  not supported by CoffeeScript, such as `` `[...new Set(array)]` ``. Frappe
  should rapidly adopt new things, which should be significantly easier for
  Frappe than for CoffeeScript, since Frappe departs less from JavaScript and is
  less opinionated.

- `loop` → `while true`. Not worth it.

[Undecided features]: #undecided-features
[Intentionally left out CoffeeScript features]: #intentionally-left-out-coffeescript-features
[bind-operator]: https://github.com/babel/babel/issues/1287

Other
=====

- Provide everything needed to create a good linter. A Frappe linter should
  ideally only have to deal with Frappe-related things, while one of the
  established JavaScript linters could be used after it.
