# Cloud report: React Compiler fix, calls in `switch` case tests

## Branch

| | |
|---|---|
| Fork | `b84330808/react` |
| Fix branch | `claude/determined-gauss-qbwlry` |
| Commit | `a76d69cb11665b400e24ca836a46d7d5e1713667` (the only commit on top of the base) |
| Base | `upstream/main` @ `c5ec5c6d180ad951048668ed113c443211bc2623` (react/react `main`, re-fetched 2026-10-30, unchanged) |
| Author | `b84330808 <b84330808@gmail.com>` |
| Committer | `Claude <noreply@anthropic.com>` (see note) |
| Trailer | `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>` |

**Note on the committer:** the cloud environment's push hook rejects commits whose committer isn't `noreply@anthropic.com`. It suggested `--reset-author`, which would also have replaced the author. I kept you as the author (CLA attribution uses that) and only changed the committer. If you want the committer to be you as well, run this on the branch before opening the PR, then force-push:

```bash
git -c user.name=b84330808 -c user.email=b84330808@gmail.com commit --amend --no-edit
```

```
 .../react_compiler_lowering/src/build_hir.rs       | 143 +++++++++++++++------
 .../src/HIR/BuildHIR.ts                            |  71 ++++++++--
 .../error.todo-switch-case-test-call.expect.md     |  71 ++++++++++
 .../compiler/error.todo-switch-case-test-call.js   |  34 +++++
 ...e-test-call-bailout-preserves-renders.expect.md | 109 ++++++++++++++++
 ...tch-case-test-call-bailout-preserves-renders.js |  37 ++++++
 6 files changed, 415 insertions(+), 50 deletions(-)
```

Most of the Rust line count is rustfmt re-wrapping calls that gained an argument. The logic change is three lines per language: the flag is checked in the call and `new` branches, and the switch passes `false`.

## PR title

```
[compiler] Bail out on calls in switch case tests
```

## PR body

````markdown
## Summary

BuildHIR lowers every `switch` case test before the `switch` terminal, last case first. A case test that calls a function can therefore run eagerly and out of order, instead of lazily in source order. `lowerReorderableExpression` is meant to admit only case tests where that can't be observed, and it already rejects e.g. `case props.kind:`. But `isReorderableExpression` accepts calls and `new` whose callee and arguments are reorderable. That allowance exists for destructuring defaults (#36107), where the default value is still lowered lazily inside a ternary. Switch case tests share the helper, so a later case test can run even when an earlier case matches:

```js
function Component({value}) {
  const labels = [];
  switch (value) {
    case null:
      labels.push('empty');
      break;
    case lower(value):
      labels.push('lowercase');
      break;
    case upper(value):
      labels.push('uppercase');
      break;
  }
  return labels;
}
```

compiles to

```js
  if ($[0] !== value) {
    labels = [];

    const t1 = upper(value);
    const t2 = lower(value);
    bb0: switch (value) {
      case null: {
        labels.push("empty");
        break bb0;
      }
      case t2: {
```

With `value: null` the original returns `["empty"]`. The compiled component calls `upper(null)` before entering the switch and throws during render. Whether the tests get hoisted depends on later passes. Here the switch is inside the reactive scope for `labels`; when nothing forces that, codegen happens to inline the tests back into their cases.

This adds an `allowCalls` flag to `lowerReorderableExpression`/`isReorderableExpression`. Switch case tests pass `false`, so a call or `new` anywhere in a case test now reports the existing "cannot be safely reordered" Todo and the function is left uncompiled. Destructuring defaults pass `true` and behave as before. The same change is made in the Rust port (`build_hir.rs`).

## How did you test this change?

Added two fixtures, following the pattern used for the implicit `arguments` bailout:

- `error.todo-switch-case-test-call`: the Todo error.
- `switch-case-test-call-bailout-preserves-renders` (`@panicThreshold:"none"`): sprout renders the original and the compiler output for `null`, `'a'` and `'A'` and compares them.

With the fix reverted, both fail on `yarn snap` and `yarn snap --rust`:

```
FAIL: switch-case-test-call-bailout-preserves-renders
Found differences in evaluator results
Non-forget (expected):
(kind: ok) ["empty"]
["lowercase"]
["uppercase"]
Forget:
(kind: ok) [[ (exception in render) Error: upper: missing value ]]
["lowercase"]
["uppercase"]

FAIL: error.todo-switch-case-test-call
Expected an error to be thrown for fixture: `error.todo-switch-case-test-call`, remove the 'error.' prefix if an error is not expected.
```

With the fix, both pass on both backends.

- `yarn snap`: 1846/1846 (1844/1844 on `main`).
- `yarn snap --rust`: 1846/1846 (1844/1844 on `main`).
- `compiler/scripts/test-rust-port.ts` over all fixtures: 1845/1845, every pass matching. On the two new fixtures: 1/1 each.
- `yarn workspace babel-plugin-react-compiler lint`, prettier, and `cargo fmt --check` are clean.

I used Claude Code to write this change.
````

## Fail-before (fix reverted, fixtures kept)

I restored only `compiler/packages/babel-plugin-react-compiler/src/HIR/BuildHIR.ts` and `compiler/crates/react_compiler_lowering/src/build_hir.rs` from `upstream/main` (`git checkout upstream/main -- <those two files>`). I kept both fixtures and their `.expect.md` files and ran each fixture on each backend. All four runs exit 1.

### TS: `yarn snap -p switch-case-test-call-bailout-preserves-renders.js`

```
$ node dist/main.js -p switch-case-test-call-bailout-preserves-renders.js
FAIL: switch-case-test-call-bailout-preserves-renders
 >> Unexpected error during test:
Found differences in evaluator results
Non-forget (expected):
(kind: ok) ["empty"]
["lowercase"]
["uppercase"]
Forget:
(kind: ok) [[ (exception in render) Error: upper: missing value ]]
["lowercase"]
["uppercase"]
1 Tests, 0 Passed, 1 Failed
```

### TS: `yarn snap -p error.todo-switch-case-test-call.js`

```
$ node dist/main.js -p error.todo-switch-case-test-call.js
FAIL: error.todo-switch-case-test-call
 >> Unexpected error during test:
Expected an error to be thrown for fixture: `error.todo-switch-case-test-call`, remove the 'error.' prefix if an error is not expected.
1 Tests, 0 Passed, 1 Failed
```

### Rust: `yarn snap --rust -p switch-case-test-call-bailout-preserves-renders.js`

```
$ node dist/main.js --rust -p switch-case-test-call-bailout-preserves-renders.js
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 1.12s
Built Rust compiler successfully
FAIL: switch-case-test-call-bailout-preserves-renders
 >> Unexpected error during test:
Found differences in evaluator results
Non-forget (expected):
(kind: ok) ["empty"]
["lowercase"]
["uppercase"]
Forget:
(kind: ok) [[ (exception in render) Error: upper: missing value ]]
["lowercase"]
["uppercase"]
1 Tests, 0 Passed, 1 Failed
```

### Rust: `yarn snap --rust -p error.todo-switch-case-test-call.js`

```
$ node dist/main.js --rust -p error.todo-switch-case-test-call.js
Built Rust compiler successfully
FAIL: error.todo-switch-case-test-call
 >> Unexpected error during test:
Expected an error to be thrown for fixture: `error.todo-switch-case-test-call`, remove the 'error.' prefix if an error is not expected.
1 Tests, 0 Passed, 1 Failed
```

### Compiled output without the fix (TS backend; the Rust backend fails with the same evaluator difference above)

```js
function Component(t0) {
  const $ = _c(2);
  const { value } = t0;
  let labels;
  if ($[0] !== value) {
    labels = [];

    const t1 = upper(value);
    const t2 = lower(value);
    bb0: switch (value) {
      case null: {
        labels.push("empty");
        break bb0;
      }
      case t2: {
        labels.push("lowercase");
        break bb0;
      }
      case t1: {
        labels.push("uppercase");
      }
    }
    $[0] = value;
    $[1] = labels;
  } else {
    labels = $[1];
  }
  return labels;
}
```

## Pass-after (fix applied)

The Rust module was rebuilt from the fixed source before these runs (the second `cargo build` line below).

```
$ node dist/main.js -p switch-case-test-call-bailout-preserves-renders.js
1 Tests, 1 Passed, 0 Failed

$ node dist/main.js -p error.todo-switch-case-test-call.js
1 Tests, 1 Passed, 0 Failed

$ node dist/main.js --rust -p switch-case-test-call-bailout-preserves-renders.js
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 1.51s
Built Rust compiler successfully
1 Tests, 1 Passed, 0 Failed

$ node dist/main.js --rust -p error.todo-switch-case-test-call.js
Built Rust compiler successfully
1 Tests, 1 Passed, 0 Failed
```

`switch-case-test-call-bailout-preserves-renders.expect.md` eval output:

```
(kind: ok) ["empty"]
["lowercase"]
["uppercase"]
```

`error.todo-switch-case-test-call.expect.md` error (listed last case first, because BuildHIR lowers cases in reverse):

```
Found 2 errors:

Todo: (BuildHIR::node.lowerReorderableExpression) Expression type `CallExpression` cannot be safely reordered

error.todo-switch-case-test-call.ts:24:9
> 24 |     case upper(value):
     |          ^^^^^^^^^^^^ (BuildHIR::node.lowerReorderableExpression) Expression type `CallExpression` cannot be safely reordered

Todo: (BuildHIR::node.lowerReorderableExpression) Expression type `CallExpression` cannot be safely reordered

error.todo-switch-case-test-call.ts:21:9
> 21 |     case lower(value):
     |          ^^^^^^^^^^^^ (BuildHIR::node.lowerReorderableExpression) Expression type `CallExpression` cannot be safely reordered
```

## Full suites vs. `main`

| Run | `upstream/main` (`c5ec5c6`) | Fix branch (`a76d69c`) |
|---|---|---|
| `yarn snap` | 1844 passed, 0 failed | 1846 passed, 0 failed |
| `yarn snap --rust` | 1844 passed, 0 failed | 1846 passed, 0 failed |
| `test-rust-port.ts` (all fixtures, through `PruneHoistedContexts`) | not run | 1845 passed, 0 failed, frontier none |

The failure set is empty on both sides. The two extra tests are the new fixtures, and no existing `.expect.md` changed.

The full `test-rust-port.ts` run used the fix's exact source with an earlier version of the two fixtures (a logging example, later replaced by the throwing one). I reran it on the final two fixtures individually: 1/1 each, all 20 passes matching.

Other checks on the fix branch:

- `yarn workspace babel-plugin-react-compiler lint`: clean.
- prettier (root `.prettierrc.js`) on `BuildHIR.ts` and both fixture files: clean.
- `cargo fmt -p react_compiler_lowering -- --check`: clean. `upstream/main`'s `build_hir.rs` was already rustfmt-clean, so the only re-wrapping is in touched code.
- `cargo clippy -p react_compiler_lowering`: no new warnings. The ones near the edited lines (`clone` on a `Copy` type, `Ok(...?)` in `lower_reorderable_expression`) are pre-existing.

Environment notes for reproducing locally:

- `test-rust-port.ts` is normally run with `npx tsx`, which wasn't installed in the container and couldn't be fetched there. I used the compiler workspace's ts-node: `node_modules/.bin/ts-node-transpile-only -O '{"module":"commonjs","esModuleInterop":true}' scripts/test-rust-port.ts PruneHoistedContexts <fixture>`.
- Root `node_modules` isn't installed, so prettier ran as `compiler/node_modules/.bin/prettier --config ../.prettierrc.js` (prettier 3.3.3 with the compiler's hermes plugin 0.26; the root pins 0.36). Running `yarn prettier` locally before opening the PR is cheap insurance.

## Duplicate search

### PRs: all states, via git

This cloud session's GitHub tools are scoped to the fork. `react/react` couldn't be attached as a second repository because it would share the `react` checkout directory. So I didn't use the GitHub API against upstream. I used the `upstream` git remote instead:

- `git fetch --filter=tree:0 upstream '+refs/pull/*/head:refs/upstream-pr/*'`: 20,941 PR head refs, PRs #1 to #38052. GitHub keeps `refs/pull/N/head` for open, closed and merged PRs, so this covers every state. It only gives commit messages, not PR titles, bodies or comments.
- Full `upstream/main` history: 21,747 commits.
- `git log --glob='refs/upstream-pr/*' upstream/main -i -E --grep=<pattern>` with:
  - `switch`
  - `case test|switch case|case expression|case clause|switch statement|isReorderable|reorderable`
  - `eagerly|evaluation order|order of evaluation|evaluated (out of order|in reverse)|reverse order|lowerReorderable|lower_reorderable`

**Result:** no PR in any state addresses calls in switch case tests or the eager, reversed evaluation of case tests. Related but different:

- #36107 (merged 2026-04): added `NewExpression` to `isReorderableExpression` for `{value = new Number()}` defaults. That's where the call/`new` allowance the switch inherits comes from.
- #33788 (merged 2025-07): reorderability of `TSAsExpression`/`TSNonNullExpression`.
- Rust-port commits like `[rust-compiler] Fix BuildReactiveFunction switch case scheduling` are porting work, not this bug.

### Issues: not searched, please run before opening the PR

Issues aren't reachable through git, and I couldn't use the GitHub API on upstream from this session. Please run these locally, all states:

```bash
gh search issues --repo react/react --include-prs "switch case" compiler --state all
```

```bash
gh search issues --repo react/react --include-prs "switch" "React Compiler" --state all
```

```bash
gh search issues --repo react/react --include-prs "cannot be safely reordered" --state all
```

```bash
gh search issues --repo react/react --include-prs "isReorderableExpression" --state all
```

```bash
gh search issues --repo react/react --include-prs "case" "missing value" compiler --state all
```

### Candidates I found and dropped

| Candidate | What I found | Why dropped |
|---|---|---|
| ConstantPropagation forwards a `LoadGlobal` past a reassignment of that global (`const id = nextId; nextId = id + 1; return id` returns `id + 1`, also inside effects) | Real miscompile on TS and Rust. I had a fix passing 1845/1845 on both before finding the duplicates | Duplicate of #37889 (fixes issue #37888, includes the Rust port) and the older #35469 (issue #34899) |
| `catch ({message})` hits Invariant "Could not find binding for declaration" | Already documented by `error.bug-invariant-couldnt-find-binding-for-decl` | PRs #37913, #37890 |
| `const {[k]: v, ...rest} = obj` hits Invariant "Expected value kind to be initialized" | Already documented by `todo.error.object-pattern-computed-key` and `error._todo.computed-lval-in-destructure` | PR #34902 |
| `for (; i < n; i++)` hits codegen Invariant "Expected a variable declaration" | Real | PR #37493 fixes the omitted-init case in the same `codegenForInit` code. `for (i = 0; ...)` still hits the invariant and masks BuildHIR's Todo, but a fix would conflict with #37493 |
| Logical assignment (`??=`, `\|\|=`, `&&=`) | Todo bailout, not a bug | PR #36733 adds support |
| Compound assignment double evaluation, for..of/for..in existing variable, postfix member update | | Yours (#38028, #38047) or taken (#35205 / #37347), per the brief |

## Probes that found nothing

Compiled-vs-original sprout probes that matched the original:

- **Evaluation order:** computed member assignment, destructuring assignment to member targets, object literal with computed keys and spread, call/argument order, JSX attribute/spread/children order, destructuring default order, template/binary/logical/conditional order, assignment chains, `new` order, optional chains.
- **Control flow:** labeled block break, `default` in the middle of a switch, labeled continue out of a switch inside a loop, do-while with continue, for-in with continue, loop closures over `let`/`const`.
- **Other:** swap destructuring, `delete`/`void`/`in`, hoisted function declarations, JSX attribute strings with quotes and entities, IIFE with extra arguments, `typeof` of an undeclared global, a tagged template whose tag is a member of a module-level object.

A parenthesized optional call `(obj?.get)()` couldn't be exercised, because object methods that use `this` are already a Todo.
