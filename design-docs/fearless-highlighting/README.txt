Fearless syntax highlighting design - overnight task, 2026-09-12/13

Files in this folder:
- proposal.md
    The design write-up: what the real Frontend grammar supports, why
    "type name" and "method name" are lexer-detectable but "parameter
    name" fundamentally is not without a real parser, three concrete
    design options with tradeoffs, and open questions for Marco.
- fearless.tmLanguage.json
    A working TextMate grammar implementing design option A/B from the
    proposal: comments (including the /// //> //- split), the 6
    reference-capability keywords, type names (identifiers and all 4
    literal forms), method names (the ".name" token), a best-effort
    parameter-declaration heuristic, punctuation and operators.
    Tested (see below) against all 88 real .fear files currently in
    StandardLibrary (base, rt, and the integrationTests sample projects):
    every non-blank character in every file lands in some named scope,
    i.e. nothing silently falls through unclassified, and a manual check
    of representative files (comparators.fear, math.fear, var.fear,
    caching.fear) confirms the categorization matches the real tokens
    Frontend's own tokenizer would produce.
    Not yet wired into an actual editor extension (no package.json /
    language-configuration.json / VS Code extension scaffold) - this is
    the grammar file itself, ready to be dropped into one.
- (this file)

How it was tested: a small Node.js harness using the real vscode-textmate
+ vscode-oniguruma packages (the same engine VS Code itself uses to
interpret .tmLanguage.json) tokenized every line of every .fear file
under StandardLibrary and flagged any non-whitespace character that fell
through to the bare "source.fearless" scope with no more specific scope
attached. One real bug was caught and fixed this way: the operator-run
pattern's character class includes "/", so a naive version of it swallowed
a real "//" comment marker whenever a comment immediately followed an
operator character with no space (e.g. "Magic!//CacheReprF$2..." in
base/caching.fear) - fixed by adding the same
"stop before /*, */, or //" lookahead Frontend's own Op token regex uses.
The harness itself was not kept (throwaway script in the session
scratchpad); rerunning the same check just needs vscode-textmate +
vscode-oniguruma and a short tokenize-and-scan loop.

Nothing here has been opened as a PR or committed to any of the seven
Fearless repos - this is an open design discussion, meant to be read and
argued with, not merged.
