Fearless syntax highlighting: design proposal
Written overnight by an agent, for Marco to review. Not merged anywhere; this
is a discussion starter, not a decision.

1. What the real grammar actually supports
===========================================

Source: Frontend/FearlessFrontend/src/fearlessParser/TokenKind.java,
Token.java, and the relevant bits of Parser.java. This is the ground truth,
not a guess.

1.1 True reserved keywords (lexically fixed strings, never identifiers)
- Reference capabilities: mut imm iso read readH mutH, plus the single
  combined token read/imm (matched before read so it is not split).
  These are the only tokens that are keywords in the conventional sense:
  fixed spellings, matched by fixed alternation, never overridable by a
  program.
- There is no other reserved word. That is the whole keyword set.

1.2 Contextual keywords (context-sensitive, not lexical)
  use, map, as, in
  These are NOT separate token kinds. Per TokenKind.java's own comment,
  _use/_map/_as/_in are "tokens that are never considered for matching,
  but useful for asserts and for labelling special cases". Parser.java
  confirms this: it calls expectValidate(..., LowercaseId, _as) etc. -
  i.e. the tokenizer produces an ordinary LowercaseId token, and the
  parser only checks the token's *content* against these words, and only
  at specific positions (package-rename headers: "map"/"use" at the very
  start of a package's header clause, "as"/"in" inside it). Elsewhere,
  "use", "map", "as", "in" are perfectly legal method/parameter names
  (e.g. a list's ".map" method - very likely to exist in real code and
  must not be painted as a keyword).

1.3 Type names - and why literals count as types
  Token.typeName = { UppercaseId, SignedFloat, UnSignedFloat, SignedInt,
  UnsignedInt, SStr, UStr }
  This literally confirms Marco's framing: 123, "ho", `ho`, +1.5 etc. are
  themselves type names in Fearless (each value's most precise type is a
  singleton type named after its literal spelling). So "type name" as a
  highlighting category should include:
  - identifiers matching UppercaseId: optional single lowercase
    package-prefix + dot (reserved device names con/prn/aux/nul and
    com1-9/lpt1-9 excluded, a Windows-filename-safety carryover), then
    underscores*, an uppercase letter, then alnum/underscore, then
    trailing primes ('). This whole prefixed form is ONE token, so a
    highlighter never needs to separately color the package prefix.
  - numeric literals (signed/unsigned int and float, digits with
    underscore separators, optional exponent, optional trailing "soft")
  - string literals ("...") and backtick literals (`...`), each confined
    to one line, no escape sequences.
  All of these are 100% lexer-detectable by regex. This is the strongest,
  safest category for any highlighter, cheap or not.

1.4 Method names ARE lexer-detectable too (via the dot)
  DotName's regex is "\._*[a-z][A-Za-z0-9_]*'*" - the dot and the
  lowercase name are glued into a single token by the tokenizer itself.
  So ".foo", ".step2", ".get" are unambiguous method-name tokens purely
  from spelling: a dot immediately followed by a lowercase identifier,
  no space, is always a method name (declaration or call - Fearless does
  not need to distinguish those visually; both are ".name" and both are
  genuinely "the method called/declared here"). Operator methods
  (".+", ".==") are named with Op-class characters after the dot instead
  of letters; same rule, different character class.
  This means "method name" is NOT the hard category people might assume
  from "identifiers are contextual" - the leading dot removes the
  ambiguity entirely at the lexer level.

1.5 Parameter names - one category, every bare LowercaseId
  Parameter declarations are bare LowercaseId tokens: Parser.java's
  parseXPat/parseParam call expect("parameter name", LowercaseId) with
  no distinguishing punctuation of their own other than being followed by
  ":" in a parameter list, e.g. ".cmp[R:**](t0: read T, t1: read T)".
  The same token kind, LowercaseId, is also used for "let"-sugar bindings
  ("pod" in ".let pod={...}"), for every later use/read of a parameter or
  binding inside the method body ("t0" appearing again after its
  declaration), and for the implicit receiver name "this" (not a keyword
  at all - Parser.java special-cases the literal string "this" only for
  name-resolution bookkeeping; lexically it is just another LowercaseId,
  the implicit parameter zero, so a user could shadow it and the
  tokenizer would not care). All four of these are, by design, the SAME
  highlighting category: Fearless draws no visual distinction between
  declaring a name and reading it back, so this proposal draws none
  either. That collapses what could look like a scope-tracking problem
  into a plain regex: any bare LowercaseId, in any position, is a
  parameter name. The only carve-outs are the ones already covered by
  other categories - the leading dot of a method name removes its
  LowercaseId from consideration entirely (1.4), the lowercase
  package-prefix of a dotted type name is absorbed into UppercaseId's own
  regex and never becomes a separate LowercaseId token (so that
  ambiguity does not arise - good news for a regex highlighter), and the
  four contextual keywords (1.2) take priority over the parameter
  category when they appear as bare words.

1.6 Comments - three doc-comment kinds, and how they're really told apart
  Confirmed both in TokenKind.java (Ws/LineComment/BlockComment are the
  only comment TokenKinds the tokenizer itself knows) and, more
  precisely, in Coordinator/src/docBuilder/SourceDocs.java, which is the
  actual consumer that gives the three prefixes their distinct meaning:
    ///  -> doc-comment prose (attached to the following/preceding
            declaration, used to generate <pkg>.txt and to build
            per-type "_<Type>_Examples" test suites)
    //>  -> a doc-comment line that is also a *runnable example*
            (turned into part of the auto-generated test suite AND shown
            in generated docs)
    //-  -> a doc-comment line that is a *test-only* example (used to
            generate tests, but not printed in the generated docs)
    //   -> an ordinary, non-doc comment (thrown away)
    /* ... */ -> an ordinary block comment, no nesting, can span lines
  This is 100% resolved by looking at the first three characters after
  the two slashes, checked in this priority order (SourceDocs.scan()
  checks "///" before "//>" before "//-" before generic "//"). This is
  exactly as regex-friendly as it looks - no parser state needed beyond
  "am I inside a string/block-comment already", which any regex-based
  line-oriented highlighter already has to track anyway. Note //- is
  defined and real but I did not find a live example of it in the current
  StandardLibrary/base sources (grep only turns up "///-" i.e. a "-"
  inside ordinary /// prose, e.g. a markdown bullet); Marco confirms it is
  still an intended feature (section 5), so it keeps its own comment kind
  even though currently unused in practice.

1.7 Punctuation / "everything else"
  Structural tokens with fixed spelling: -> ( ) { } [ ] , ; :: : ' _
  (Underscore is its own token, used as a wildcard pattern, e.g.
  ".some c -> ...; .empty -> ...", and also as a leading "throwaway"
  prefix on identifiers, e.g. "_x" in a pattern.)
  Operators are an open, regex-defined character class run:
  \ / # * - + % < > = ! & ^ ~ ? |  (one or more, with a
  no-comment-marker lookahead built directly into the token: it must not
  start consuming into "/*", "*/" or "//" mid-run). This exclusion is not
  a cosmetic detail - real StandardLibrary code relies on it (e.g.
  "Magic!//CacheReprF$2..." in base/caching.fear, where "!" is an
  operator immediately followed by a real comment; naive regex without
  the same lookahead swallows the comment marker into the operator run,
  as the POC below hit and had to fix).

2. The hard part, stated plainly
=================================
A highlighter with only lexer-level information (a TextMate grammar, the
kind VS Code / most editors / GitHub's own syntax highlighting use) can
regex-match tokens but cannot track *bindings* or *scope*. Concretely:
- "type name" needs only regex: safe, cheap, and correct virtually always
  (UppercaseId + the four literal shapes).
- "method name" also needs only regex, thanks to the glued dot: safe and
  cheap.
- "parameter name" would need a symbol table - which LowercaseId
  occurrences are the *same* binding introduced by which declaration, and
  in which scope - if the goal were to color a declaration differently
  from its later uses, or to tell one parameter's uses apart from
  another's. That is not the goal here: since every bare LowercaseId
  (declaration, let-sugar binding, later use, or "this") shares one
  color, no scope tracking is needed at all - a plain regex gets 100% of
  this category right, not just an approximation of it.
- What a TextMate grammar still structurally cannot do is *semantic*
  highlighting proper: distinguishing an unresolved name from a real one,
  telling a shadowed "this" apart from the outer one, or coloring a name
  differently because it resolves to a field vs a local. That needs a
  live binder, which only a language server (LSP) backed by Frontend's
  own parser/name-resolution can give - see design C below. Nothing in
  the six-category scheme this proposal targets needs that; it would only
  matter for a *richer* category set than the one asked for here.

3. Recommended color categories (refining Marco's list)
=========================================================
Marco's original six buckets survive almost unchanged; the grammar
suggests one addition and one clarification.
  1. Types - identifiers (UppercaseId) AND all four literal forms
     (numbers, "strings", `backtick strings`). Confirmed correct by
     Token.typeName - literals really are types in Fearless, so lumping
     them with type names is not just a convenient POC shortcut, it is
     linguistically accurate. A theme is free to give literals a
     slightly different shade of the "type" color if desired, but they
     should not be a wholly separate category from types conceptually.
  2. Method names - the ".name" / ".op" token, dot included in the
     colored span (the dot and the name are one token from the lexer's
     own point of view; the grammar colors them as one span, not two).
  3. Parameter names - every bare LowercaseId: declaration sites,
     let-sugar bindings, every later use/read, and "this". Fearless
     itself draws no distinction between declaring a name and reading it
     back, so the highlighting draws none either - see section 1.5/2.
  4. Comments - split into the four kinds the tooling already
     distinguishes: /// (doc), //> (doc + runnable example), //- (doc,
     test-only example), // (plain). A theme can give all four the same
     color, or vary emphasis (e.g. //> in a slightly brighter shade to
     mark "this is executable").
  5. Reference-capability keywords - mut/imm/iso/read/readH/mutH/read-imm.
     These really are keywords (fixed spelling, never identifiers), so
     they deserve their own "keyword" bucket distinct from punctuation.
  6. Everything else - punctuation (braces/brackets/commas/colons/arrow)
     AND the operator-symbol run. Marco's list already puts punctuation
     and "keywords like mut/imm" in the same bucket; this proposal splits
     them (5 vs 6) since mut/imm are real reserved words and punctuation
     is not, but a single merged "everything else" bucket is equally
     defensible if simplicity is preferred.
  One optional addition worth considering: contextual keywords
  (use/map/as/in) as a distinct, clearly-marked-as-approximate 7th
  bucket, since a regex grammar cannot avoid false-highlighting a
  parameter or method literally named "map" - see section 4, design B.

4. Three concrete design alternatives
=======================================

A. Cheap TextMate-grammar approximation (what the POC below implements)
   - One static .tmLanguage.json, no build step, works instantly in any
     TextMate-based editor/viewer (VS Code, GitHub, many others).
   - Gets exactly right: types (identifiers + all 4 literal forms),
     method names, parameter names (every declaration, let-sugar binding,
     later use, and "this" - one color, no positional heuristic needed,
     see section 1.5/2), all 4 comment kinds, the 6 reference-capability
     keywords, punctuation, operators.
   - Approximates: the 4 contextual keywords (use/map/as/in), which will
     occasionally mis-highlight a parameter/method that happens to share
     one of those 4 spellings.
   - Cost: near zero - a few hours to write, test, and iterate; no
     runtime dependency; ships as a single file.
   - Recommended as the immediate, no-regrets first cut: it visibly
     improves on "no highlighting at all" or "generic C-like
     highlighting" (which currently mis-colors 123/"ho" as literals
     rather than types, and has no concept of the /// //> //- split) at
     essentially zero engineering cost, and it does not block B or C -
     an LSP can be layered on top later without discarding this file
     (many editors merge TextMate scopes with LSP semantic tokens,
     using the grammar as a fast first pass and letting semantic tokens
     override once the language server catches up).

B. Cheap grammar, but honest about the contextual keywords
   - Same as A, but drop the use/map/as/in "keyword" highlighting
     entirely rather than approximate it with a false-positive-prone
     regex. Rationale: mis-coloring a real ".map" method or a "map"
     parameter as a keyword is arguably worse than not highlighting the
     package-header syntax at all, since it happens far more often (any
     functional-style code with a .map/.use method) than the rare
     package-rename header actually needing the color.
   - Same cost as A, marginally simpler grammar and fewer false
     positives; loses a minor, rarely-seen highlight.
   - This is a one-line reversible choice (keep or drop the "keywords"
     rule in the repository), described here mainly so Marco can pick
     without needing a second implementation.

C. LSP-backed semantic highlighting using Frontend's own parser
   - A language server wrapping Frontend's tokenizer + parser + name
     resolution, emitting the standard LSP semantic-tokens protocol
     (textDocument/semanticTokens). VS Code (and other LSP-aware
     editors) then color tokens using the *real* parse tree instead of
     regex guesses.
   - Gets right everything A/B get right, PLUS what a lexer structurally
     cannot within a *richer* category set than the six used here - e.g.
     color an unresolved/erroring name differently, grey out an
     unreachable branch, or distinguish a shadowed "this" from the outer
     one. It does not add anything for the six categories themselves:
     design A already gets parameter uses and "this" right with plain
     regex (section 1.5/2).
   - Cost: substantial - this is a real, ongoing piece of software (an
     incremental reparse-on-edit pipeline hung off Frontend, a
     semantic-tokens encoder, a server process/protocol handler,
     editor-side wiring). It also could double as the foundation for
     Fearless "real" IDE features later (go-to-definition,
     rename-refactor, live type errors in the editor) since it is the
     same infrastructure those need - so if any of those are on the
     horizon anyway, the incremental cost of *also* getting semantic
     highlighting is smaller than building highlighting-only tooling
     from scratch.
   - Recommended as the eventual target if Fearless tooling grows an
     editor story beyond "syntax coloring" (go-to-definition,
     rename-refactor, live type errors), but clearly not a same-night
     deliverable, and not needed just for highlighting since design A
     already covers the six categories correctly.

Suggested path: ship A (or B) now as a low-cost, immediate improvement;
treat C as a separate, larger initiative to consider only if/when
Fearless invests in broader editor tooling (since C's cost is justified
mainly by everything else it would unlock, not by highlighting alone).

5. Resolved by Marco
=====================
- //- is still meant to be used in practice; keep it as its own
  doc-comment kind even though no live example currently exists in
  StandardLibrary/base.
- Parameter names are a single category covering declarations,
  let-sugar bindings, every later use, and "this" (the implicit
  parameter zero) - no distinction between declaring a name and reading
  it back. See section 1.5/2 and bucket 3 in section 3.
- The lowercase package-prefix of a dotted type name belongs to the type
  name category, as already implemented (section 1.5, 1.3).
- Method names are colored dot-included: the "." and the name/operator
  that follows it are one colored span, not two.

6. Open questions for Marco
=============================
- Design A vs B: is a false-positive-prone highlight for use/map/as/in
  worse than no highlight for them? (Section 4.B)
- Bucket 5 vs 6: keep reference-capability keywords as their own color,
  or fold them into "everything else" as originally sketched?
- Should literal values (numbers/strings) share the exact type color, or
  get a distinguishable shade within the same category (section 3.1)?
