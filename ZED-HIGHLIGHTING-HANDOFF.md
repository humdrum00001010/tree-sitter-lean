# Zed Lean highlighting handoff

## Reproduction and diagnosis

The reproduction file is `/Users/phihu/Desktop/dasein/StereoMatching.lean`.
It compiles with the installed Lean 4.34.1 toolchain and the Lean LSP returns
semantic tokens for the file, including `lowerCost`, its parameters, and the
match-arm locals. No Lean LSP fork is needed.

The old grammar revision pinned by the Zed extension lost declaration
structure after valid `do` syntax it did not model. In particular, it could
not parse `if condition then` without `else` in a `do` body, assignments in
`match` arms, or `let pat := value | continue`. Later definitions became
children of a broad `ERROR` node, so the extension's declaration/type captures
never ran for them. The extension query also had obsolete node names for the
newer grammar.

## Changes in this fork

- Added `do_if` and `do_return` nodes for `if … then` and `return` statements
  in do blocks.
- Allowed block assignments and do-style `if` bodies in match arms.
- Added the `let … | fallback` guard form.
- Added corpus regressions for do-match mutation and guarded lets.
- Bumped `tree-sitter-cli` to 0.27.0; 0.26.8 generation exceeded 11 minutes
  and 16 GB resident memory during this work, so that attempt was interrupted.

The parser changes are in commit `824e3a3c5c362287cb9d81a9b343cec3cb432a1a`
on `humdrum00001010/tree-sitter-lean` `main`.

## Verification

- `tree-sitter test`: 301/301 corpus cases pass.
- `tree-sitter build --wasm`: succeeds.
- The Zed extension's updated `languages/lean4/highlights.scm` compiles
  against this grammar and the reproduction file. It now captures `def`,
  function names, applied and arrow types, and local names in the previously
  affected definitions.
- `lean StereoMatching.lean`: succeeds.

## Remaining parser issue and cost

The declarations after `lowestCost` now parse independently. `searchDisparity`
still has two local `ERROR` nodes at the assignments `low := …` and
`high := …` in a multi-statement match arm. Fixing this fully requires
supporting a sequence of do statements as a match-arm body; keep that change
narrow because the grammar state count grows quickly.

The generated `src/parser.c` is 61.56 MB, up from roughly 44 MB, and GitHub
warned that it exceeds its recommended 50 MB file size. Generation with
0.27.0 took several minutes and peaked around 9 GB resident memory. The
README's current under-5-MB state-budget note is now stale. Reducing the
grammar's state/memory cost should be the next task before further broadening
the syntax coverage.

## Zed extension integration

The Zed extension fork at `/Users/phihu/Desktop/zed-lean4` now points to this
grammar repository and revision in `extension.toml`. Its update is pushed to
`humdrum00001010/zed-lean4` at commit `08ec949` (the local branch is five
commits ahead of `origin/main`, which is the upstream repository).

The installed `lean4` dev extension is a symlink to that checkout. In Zed, run
`zed: rebuild dev extension` to fetch/build the pinned grammar, then reload the
Lean buffer. The grammar fork is also open in Zed at
`/Users/phihu/Desktop/tree-sitter-lean`.
