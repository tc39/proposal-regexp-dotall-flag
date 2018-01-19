# ECMAScript proposal: `s` (`dotAll`) flag for regular expressions

## Status

This proposal is at stage 4 of [the TC39 process](https://tc39.github.io/process-document/).

## Motivation

In regular expression patterns, the dot `.` matches a single character, regardless of what character it is. In ECMAScript, there are two exceptions to this:

* `.` doesn’t match astral characters. Setting the `u` (`unicode`) flag fixes that.
* `.` doesn’t match [line terminator characters](https://tc39.github.io/ecma262/#prod-LineTerminator).

ECMAScript recognizes the following line terminator characters:

* U+000A LINE FEED (LF) (`\n`)
* U+000D CARRIAGE RETURN (CR) (`\r`)
* U+2028 LINE SEPARATOR
* U+2029 PARAGRAPH SEPARATOR

However, there are more characters that, depending on the use case, [could be considered as newline characters](https://www.unicode.org/reports/tr14/):

* U+000B VERTICAL TAB (`\v`)
* U+000C FORM FEED (`\f`)
* U+0085 NEXT LINE

This makes the current behavior of `.` problematic:

* By design, it excludes _some_ newline characters, but not all of them, which often does not match the developer’s use case.
* It’s commonly used to match _any_ character, which it doesn’t do.

The proposal you’re looking at right now addresses the latter issue.

Developers wishing to truly match *any* character, including these line terminator characters, cannot use `.`:

```js
/foo.bar/.test('foo\nbar');
// → false
```

Instead, developers have to resort to cryptic workarounds like `[\s\S]` or `[^]`:

```js
/foo[^]bar/.test('foo\nbar');
// → true
```

Since the need to match any character is quite common, other regular expression engines support a mode in which `.` matches any character, including line terminators.

* Engines that support constants to enable regular expression flags implement `DOTALL` or `SINGLELINE`/`s` modifiers.
    * [Java](https://docs.oracle.com/javase/7/docs/api/java/util/regex/Pattern.html#DOTALL) supports `Pattern.DOTALL`.
    * [C# and VB](https://msdn.microsoft.com/en-us/library/system.text.regularexpressions.regexoptions.aspx) support `RegexOptions.Singleline`.
    * Python supports both `re.DOTALL` and [`re.S`](https://docs.python.org/2/library/re.html#re.S).
* Engines that support embedded flag expressions implement `(?s)`.
    * [Java](https://docs.oracle.com/javase/7/docs/api/java/util/regex/Pattern.html#DOTALL)
    * [C# and VB](https://docs.microsoft.com/en-us/dotnet/standard/base-types/regular-expression-options)
* Engines that support regular expression flags implement the flag `s`.
    * [Perl](https://perldoc.perl.org/perlre.html#*s*)
    * [PHP](https://secure.php.net/manual/en/reference.pcre.pattern.modifiers.php#s)

Note the established tradition of naming these modifiers `s` (short for `singleline`) and `dotAll`.

One exception is Ruby, where [the `m` flag (`Regexp::MULTILINE`)](https://ruby-doc.org/core-2.3.3/Regexp.html#method-i-options) also enables `dotAll` mode. Unfortunately, we cannot do the same thing for the `m` flag in JavaScript without breaking backwards compatibility.

## Proposed solution

We propose the addition of a new `s` flag for ECMAScript regular expressions that makes `.` match any character, including line terminators.

```js
/foo.bar/s.test('foo\nbar');
// → true
```

## High-level API

```js
const re = /foo.bar/s; // Or, `const re = new RegExp('foo.bar', 's');`.
re.test('foo\nbar');
// → true
re.dotAll
// → true
re.flags
// → 's'
```

### FAQ

#### What about backwards compatibility?

The meaning of existing regular expression patterns isn’t affected by this proposal since the new `s` flag is required to opt-in to the new behavior.

#### How does `dotAll` mode affect `multiline` mode?

This question might come up since the `s` flag stands for `singleline`, which seems to contradict `m` / `multiline` — except it doesn’t. This is a bit unfortunate, but we’re just following the established naming tradition in other regular expression engines. Picking any other flag name would only cause more confusion. The accessor name `dotAll` gives a much better description of the flag’s effect. For this reason, we recommend referring to this mode as _`dotAll` mode_ rather than _`singleline` mode_.

Both modes are independent and can be combined. `multiline` mode only affects anchors, and `dotAll` mode only affects `.`.

When both the `s` (`dotAll`) and `m` (`multiline`) flags are set, `.` matches any character while still allowing `^` and `$` to match, respectively, just after and just before line terminators within the string.

## Specification

* [Ecmarkup source](https://github.com/tc39/proposal-regexp-dotall-flag/blob/master/spec.html)
* [HTML version](https://tc39.github.io/proposal-regexp-dotall-flag/)

## Implementations

* [V8](https://bugs.chromium.org/p/v8/issues/detail?id=6172), shipping in Chrome 62
* [JavaScriptCore](https://bugs.webkit.org/show_bug.cgi?id=172634), shipping in [Safari Technology Preview 39a](https://developer.apple.com/safari/technology-preview/release-notes/)
* [XS](https://github.com/Moddable-OpenSource/moddable/blob/public/xs/sources/xsre.c), shipping in Moddable as of [the January 17, 2018 update](http://blog.moddable.tech/blog/january-17-2017-big-update-to-moddable-sdk/)
* [regexpu (transpiler)](https://github.com/mathiasbynens/regexpu) with the `{ dotAllFlag: true }` option enabled
    * [online demo](https://mothereff.in/regexpu#input=const+regex+%3D+/foo.bar/s%3B%0Aconsole.log%28%0A++regex.test%28%27foo%5Cnbar%27%29%0A%29%3B%0A//+%E2%86%92+true&dotAllFlag=1)
    * [Babel plugin](https://github.com/mathiasbynens/babel-plugin-transform-dotall-regex)
* [Compat-transpiler of RegExp Tree](https://github.com/dmitrysoshnikov/regexp-tree#using-compat-transpiler-api)
    * [Babel plugin](https://github.com/dmitrysoshnikov/babel-plugin-transform-modern-regexp)
