---
---

## New Parser

The parser in liquid-rust has been a hurdle for improvements.  It included a
regex-based lexer and a random assortment of functions to parse the tokens.
These functions were particularly a hurdle because logic was duplicated when it
shouldn't, shared when it shouldn't, etc making it difficult to grok, extend,
or refactor.

There are two big challenges with parsing in liquid:
- There is no official grammar or compliance tests
- Language plugins exist and are heavily used.

Liquid has the concept of filters, plugins to modify a value:
{% raw %}
```liquid
{{ data | uniq | first }}
```
{% endraw %}
These are relatively simple.

In contrast, tags and blocks declare their own grammar, for both parameters and a block's content.
{% raw %}
```liquid
Standard include: {% include variable %}
```
{% endraw %}
In contrast, a Jekyll-style include would pass the variable as {%raw%}```{{ variable }}```{%endraw%}.

[Goncalerta][Goncalerta] took on the monumental task of replacing all of this
with a [pest][pest]-based parser written from the ground up.

No matter the parser library, language plugins are a chalenge.  In Pest's case,
it has a grammar file that gets turned into Rust code.  For now, we've taken
the approach of expressing all the parts of the native grammar and where
plugins are involved, we expose more of a higher-level token stream than
before. This does present a challenge that our parser has to support every
feature of every plugin.

My thought for moving forward on this it to parse chunks of data at a time.
This means the parser only needs to be able to handle as much data as it has
knowledge of the gramar.  If the following chunk is from a plugin, we can then
delegate to the plugin's parser.

The challenge with Pest is I've not seen how we can expose subsections of our
grammar so that our plugins can combine them into their custom grammar.

Some other misc thoughts on Pest
- Using a single `enum` for to store all grammar rules makes it so bad combination of rules get turned into runtime errors rather than compile errors.
- We probably need to get more experience with pest to find the right style but
  all of the `into_inner`s make it hard to tell at a glance how the grammar is
  being used in Rust.

## Path to 1.0

- [Break up `Renderble`'s `Context`][liquid#195]
- [Split into multiple crates][liquid#199] to reduce compatibility breakages
  for API items that might be exposed in other crates' APIs.

## Conformance

Our goal is to be conformant with the Ruby implementation of Liquid with
- `strict`
- `strict_variables`
- `strict_filters`

What we did:
- Values
  - We improved [value coercion][liquid#158], including coercing [`now` and `today` to a date][liquid#182].
  - We added [whole number support to values][liquid#158].
  - Added [`.first` and `.last` to arrays][liquid#218].
  - Add support for indexing into a variable with a variable (e.g. `foo[bar]`) [(1)][liquid#214] [(2)][liquid#231].
- Fixed ranges (e.g. `1..4`) to be [inclusive][liquid#212].
- Indexing into variables
  - Support [this for filters][liquid#208]
- Filters
  - [`compact`][liquid#158]
  - [`at_least` and `at_most`][liquid#210]
- Blocks
  - Extend `if` blocks with [`and` and `or`][liquid#217]
  - Extend `for`-loops to support [iterating over `Object`s for
    `for`-loop][liquid#203] and accept [variables for `offset`,
    `limit`][liquid#213]
  - Support for [`ifchanged`][liquid#211]
  - Support `tablerow`[(1)][liquid#211] [(2)][liquid#213] [(3)][liquid#215]
- Tags
  - [`increment` and `decrement`][liquid#211]

Regarding the [jekyll library][jekyll-liquid]
- Filters
  - [`push`][liquid#220]
  - [`pop`][liquid#220]
  - [`unshift`][liquid#220]
  - [`shift`][liquid#220]
  - [`array_to_sentence_string`][liquid#220]

## Flexibility

- Selectively add tags, filters, and blocks [from the stdlib][liquid#182].

## Template Usability

We've been iterating on the usability of template errors.  I blogged in the
past on [error handling in
Rust](https://epage.github.io/blog/2018/03/redefining-failure/) and am finding
the approach being taken here is offering a balance of good usability with
relatively low overhead.

Specifically, some things we've done:
- [Include a liquid backtrace][liquid#164]
- Include context, like variable values, in backtrace [(1)][liquid#164] [(2)][liquid#233]
- Fix up the tone and use more clear terms [(1)][liquid#231] [(2)][liquid#233]

As we mentioned before, we are also conforming to `strict_variables`.  Some
times, you need to be able to check if a variable exists though.  So we
[loosened up `strict_variables` specifically when doing `if var`
checks][liquid-#168].

## Performance


To help guide performance improvements, We [ported handlebar's and tera's
benchmarks][liquid#234] to Liquid:
- Save us from thinking about representative use cases
- Of all the arbitrary baselines, this seemed like one of the better ones.

We've made quite a good number of performance improvements
- Recognized that tags, blocks, and filters will be statically defined and we
  [do not need to support dynamic names][liquid#158].  Initially, this was an
  even bigger performance gain because when parsing, we were `clone`ing the
  language definition.
- Make extensive use of `Cow`.
  - For the variables passed in for being rendered, we found it common enough
    that people create them in-memory with `&'static str`, that we switched the
    [value keys to `Cow<'static, str>`][liquid#189].
  - While no benchmark covers this, We went ahead and made [value strings
    `Cow<'static, str>`][liquid#193] to help with those using purely in-memory, staticall
    defined, data structures (one user is doing code-gen for his project).
  - We've now been using `Cow<'static, str>` in enough projects, We're considering
    creating a custom string crate to help with this.
- User variables passed to `Template::render`
  - Take variables [by reference][liquid#197]. It
    has been great help relying on the compiler for lifetime correctness
    especially as we consider further optimizations along these lines.
  - Use [a trait for passing in variables][liquid#197]. Some use cases, like
    [cobalt][cobalt], collect the variables from different sources, some being
    one off and some being reused.  This let's us avoid `clone`ing the
    variables for each one off case by instead storing them in a struct,
    references for the reused and owned values for the one-offs.
- Offer `Renderable` trait to [render to a `Write`][liquid#192], giving users
  the opportunity to control the backend and allocations. This might cause us
  problems with Liquid's [string trimming][liquid#244] but we have ideas on how
  to handle this.
- [Reduce allocations within `impl Display`][liquid#200].
- Have parallel code paths for `Option` and `Return`, optimizing for the
  failure case by avoiding expensive error reporting. [(1)][liquid#233]
- Play [whac-a-mole][whac-a-mole] with `clone()`s, finding ways to use
  references instead. [(1)][liquid#233]

These numbers were gathered on my laptop running under WSL.  To help show the
variability of my computer, We included the baseline from both the old and new
liquid runs.

- Liquid 0.13.7
- Liquid 0.18.0
- Handlebars 1.1.0
- Tera 0.11.20

Parsing Benchmarks

| Library                         | Liquid's `parse_test` |  Handlebar's `parse_template`  | Tera's `parsing_basic_template` |
|---------------------------------|---------------|----------------------------------------|---------------------------|
| Baseline Run 1                  |               |    20,730 ns/iter (+/- 4,811)          |24,612 ns/iter (+/- 8,620)
| Baseline Run 2                  |               |    23,517 ns/iter (+/- 3,386)          |24,425 ns/iter (+/- 13,926)
| Liquid v0.13.7 |     2,670 ns/iter (+/- 992)    |    26,518 ns/iter (+/- 14,115)         |24,720 ns/iter (+/- 12,812)
| Liquid v0.18.0 |     1,612 ns/iter (+/- 1,123)  |    24,812 ns/iter (+/- 13,214)         |19,690 ns/iter (+/- 9,187)

Rendering Benchmarks

| Library                         | Liquid's `render_test` |  Handlebar's `render_template`  | Tera's `rendering_basic_template` | Tera's `rendering_only_variable` |
|---------------------------------|---------------|------------------------------------------|-----------------------------------|----------------------------------|
| Baseline Run 1                  |               |    29,105 ns/iter (+/- 14,701) |6,181 ns/iter (+/- 3,112) |2,100 ns/iter (+/- 722)   |
| Baseline Run 2                  |               |    35,703 ns/iter (+/- 4,859)  |6,687 ns/iter (+/- 1,886) |2,345 ns/iter (+/- 1,678) |
| Liquid v0.13.7 |     2,379 ns/iter (+/- 610)    |    16,500 ns/iter (+/- 6,603)  |5,677 ns/iter (+/- 2,408) |3,285 ns/iter (+/- 1,401) |
| Liquid v0.18.0 |       280 ns/iter (+/- 60)     |    11,112 ns/iter (+/- 1,527)  |2,845 ns/iter (+/- 1,216) |525 ns/iter (+/- 186)     |

Variable Access Benchmarks

| Library        | Tera's `access_deep_object` | Tera's `access_deep_object_with_literal` |
|----------------|-----------------------------|------------------------------------------|
| Tera Run 1     |8,626 ns/iter (+/- 5,309)  |10,212 ns/iter (+/- 4,452)
| Tera Run 2     |8,201 ns/iter (+/- 4,476)  |9,490 ns/iter (+/- 6,118)
| Liquid v0.13.7 |13,100 ns/iter (+/- 1,489) |**Unsupported**
| Liquid v0.18.0 |6,863 ns/iter (+/- 1,252)  |8,225 ns/iter (+/- 3,567)

Other Handlebar Benchmarks

| Library                         | `large_loop_helper`|
|---------------------------------|--------------------|
| Baseline 1 | 3,983,150 ns/iter (+/- 509,024)
| Baseline 2 | 4,501,250 ns/iter (+/- 774,025)
| Liquid v0.13.7 | 1,432,650 ns/iter (+/- 309,862)
| Liquid v0.18.0 | 1,013,237 ns/iter (+/- 140,163)

Other Tera Benchmarks

| Library                         |  `big_loop_big_object`  | `big_table`                         | `teams`               |
|---------------------------------|----------------|-------------------------------------|-----------------------|
| Tera Run 1     |    389,225 ns/iter (+/- 46,426) | 3,377,200 ns/iter (+/- 447,750)     |     8,273 ns/iter (+/- 3,958)
| Tera Run 2     |    436,381 ns/iter (+/- 84,278) | 3,409,950 ns/iter (+/- 493,350)     |     9,064 ns/iter (+/- 925)
| Liquid v0.13.7 |  1,064,837 ns/iter (+/- 192,820)|  18,901,500 ns/iter (+/- 4,674,060) |      18,732 ns/iter (+/- 6,218)
| Liquid v0.18.0 |  2,592,400 ns/iter (+/- 266,987)|  11,997,400 ns/iter (+/- 3,410,070) |      10,518 ns/iter (+/- 4,084)

[liquid#158]: https://github.com/cobalt-org/liquid-rust/pull/158
[liquid#164]: https://github.com/cobalt-org/liquid-rust/pull/164
[liquid#168]: https://github.com/cobalt-org/liquid-rust/pull/168
[liquid#182]: https://github.com/cobalt-org/liquid-rust/pull/182
[liquid#189]: https://github.com/cobalt-org/liquid-rust/pull/189
[liquid#192]: https://github.com/cobalt-org/liquid-rust/pull/192
[liquid#193]: https://github.com/cobalt-org/liquid-rust/pull/193
[liquid#195]: https://github.com/cobalt-org/liquid-rust/pull/195
[liquid#197]: https://github.com/cobalt-org/liquid-rust/pull/197
[liquid#199]: https://github.com/cobalt-org/liquid-rust/pull/199
[liquid#200]: https://github.com/cobalt-org/liquid-rust/pull/200
[liquid#203]: https://github.com/cobalt-org/liquid-rust/pull/203
[liquid#208]: https://github.com/cobalt-org/liquid-rust/pull/208
[liquid#210]: https://github.com/cobalt-org/liquid-rust/pull/210
[liquid#211]: https://github.com/cobalt-org/liquid-rust/pull/211
[liquid#212]: https://github.com/cobalt-org/liquid-rust/pull/212
[liquid#213]: https://github.com/cobalt-org/liquid-rust/pull/213
[liquid#214]: https://github.com/cobalt-org/liquid-rust/pull/214
[liquid#215]: https://github.com/cobalt-org/liquid-rust/pull/215
[liquid#217]: https://github.com/cobalt-org/liquid-rust/pull/217
[liquid#218]: https://github.com/cobalt-org/liquid-rust/pull/218
[liquid#220]: https://github.com/cobalt-org/liquid-rust/pull/220
[liquid#221]: https://github.com/cobalt-org/liquid-rust/pull/221
[liquid#231]: https://github.com/cobalt-org/liquid-rust/pull/231
[liquid#233]: https://github.com/cobalt-org/liquid-rust/pull/233
[liquid#234]: https://github.com/cobalt-org/liquid-rust/pull/234
[liquid#244]: https://github.com/cobalt-org/liquid-rust/pull/244
[whac-a-mole]: https://en.wikipedia.org/wiki/Whac-A-Mole
[jekyll-liquid]: https://jekyllrb.com/docs/liquid/
[cobalt]: https://cobalt-org.github.io/
[Goncalerta]: https://github.com/Goncalerta
[pest]: https://pest.rs/
