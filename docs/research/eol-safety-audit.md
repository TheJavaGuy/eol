# eol 0.10.0 safety audit

Date: 2026-10-10. Subject: `eol` 0.10.0 (fork HEAD `cdd9427`). `eol.js`, `eol.d.ts`, `test.js`, `index.html`, `package.json` and `README.md` are byte-identical to the npm tarball `eol-0.10.0.tgz` (registry integrity `sha512-+w3ktYrOphcIqC1XKmhQYvM+o2uxgQFiimL7B6JPZJlWVxf7Lno9e/JWLPIgbHo7DoZ+b7jsf/NzrUcNe6ZTZQ==`, recomputed locally from the downloaded tarball).

Environment: macOS 26.6.2 (Darwin 25.6.0, arm64, Apple M3 Pro, 18 GiB RAM), Node v24.20.0 (V8 `heap_size_limit` 4288 MiB, `buffer.constants.MAX_STRING_LENGTH` 536870888), npm 11.19.0, Deno 2.9.7, Bun 1.3.11, TypeScript 5.9.3 / 6.0.3 / 7.0.2 (installed outside the repo). Not tested: Windows hosts, real browsers.

Unless noted, commands run from the repo root. `$S` is the session scratch directory outside the repo (`/private/tmp/claude-502/-Users-ivanmilosavljevic-src-eol/483a7220-95ef-4cf9-892f-a655f1439a72/scratchpad`); harness files referenced as `$S/...` are reproduced in the [Appendix](#appendix-harnesses). `$REPO` = `/Users/ivanmilosavljevic/src/eol`.

Severity rubric used here:

- High: crash or silent wrong output through a documented API with ordinary input.
- Medium: crash or silent wrong output through a documented API with unusual input, or declarations/docs that make documented usage fail.
- Low: explicit failure, narrow trigger, or a gap outside documented scope.
- Info: no defect against any stated contract; hazard or hardening observation.

## Summary

| ID | Category | Finding | Severity | Proven |
|----|----------|---------|----------|--------|
| F1 | Correctness | `dub(to)` expands `$$`, `$&`, `` $` ``, `$'` in `to`; output differs from `split().join(dub)` | Medium | Yes |
| F2 | Correctness | No input validation; sibling functions disagree (TypeError vs. silent `"null"`/`"undefined"` insertion); duck-typed objects executed | Medium | Yes |
| F3 | Robustness / DoS | Uncatchable process abort (`FATAL ERROR`, exit 134) on ~1.2e8 line breaks with default heap | Medium | Yes |
| F4 | Correctness | `auto`/`before`/`after` take the platform from the `process` global once, at load: LF in any realm without `process` (browsers on Windows); spoofable | Medium | Yes |
| F5 | API / types | `eol.d.ts` fails to compile on TS 6/7 (TS1540); README `join(eol.auto)` is a type error (TS2345); named imports type-check but fail at runtime in Node ESM | Medium | Yes |
| F6 | Correctness (scope) | NEL, LS, PS, VT, FF not treated as line breaks; `eol.split` disagrees with the engine's own line terminators | Low | Yes |
| F7 | Security / integration | UMD wrapper overwrites global `eol` and an unrelated global `module.exports`; throws on load when evaluated as an ES module | Low | Yes |
| F8 | Performance | Newline-dense input: 42-102 MiB/s vs 538-3031 MiB/s on text; process peak RSS 16-19x (normalizers) to 23-29x (`match`) the input; ns/char grows with size | Low | Yes |
| F9 | Test quality | `test.js:35` assertion can never fail; 7 of 11 behaviour-changing mutants pass the suite; Windows and browser branches never run; no CI | Low | Yes |
| F10 | Supply chain | Committed `package-lock.json` pins 6 dev packages with 20 advisories (npm audit: 1 moderate, 5 high); local `node_modules` diverges from lockfile | Low | Yes |
| F11 | Integration hazard | Not chunk-safe: a CRLF split across chunks becomes two line breaks | Info | Yes |
| F12 | Contract ambiguity | `split` yields a trailing `""` and `[""]` for empty input; count differs from POSIX "line" | Info | Yes |
| F13 | Hardening | Exported object is a mutable singleton; private regex reachable and rewritable via intrinsics | Info | Yes |
| F14 | Packaging | No `files` field (tests, demo page, CI-less `.github` shipped); no npm provenance attestation | Info | Yes |

Confirmed: Critical 0, High 0, Medium 5, Low 5, Info 4.

---

## F1. `dub(to)` treats `to` as a replacement template

Description: `dub` passes `to` directly as the replacement argument of `String.prototype.replace` (`eol.js:21`). Per ECMAScript `GetSubstitution`, `$$`, `$&`, `` $` ``, `$'` (and `$n`, `$<name>` when applicable) in a replacement string are expanded. README states the normalizer makes "linebreaks become <var>text</var>" (`README.md:35`); `index.html:79-81` says it "replaces line endings with text" and "Normalizers are usable as glue to join arrays"; README shows `split(text).join(normalizer) === normalizer(text)` for `auto` (`README.md:65-69`). `change.toString` returns the raw `to` (`eol.js:23-25`), so `dub(to)(text)` and `split(text).join(dub(to))` diverge.

Source: `eol.js:19-27`; ECMAScript §22.1.3.19.1 GetSubstitution, https://tc39.es/ecma262/#sec-getsubstitution ("If templateRemainder starts with "$$", then ... Let refReplacement be "$"", with equivalent branches for `` $` ``, `$&`, `$'`).

Reproduction:

```sh
node - <<'EOF'
const eol = require("./eol.js");
for (const to of ["$$", "$&", "$`", "$'", "<$&>"]) {
  const d = eol.dub(to);
  console.log(`dub(${JSON.stringify(to)})("a\\nb") = ${JSON.stringify(d("a\nb"))}  expected ${JSON.stringify("a" + to + "b")}  split().join(d) = ${JSON.stringify(eol.split("a\nb").join(d))}`);
}
EOF
```

Observed:

```
dub("$$")("a\nb") = "a$b"  expected "a$$b"  split().join(d) = "a$$b"
dub("$&")("a\nb") = "a\nb"  expected "a$&b"  split().join(d) = "a$&b"
dub("$`")("a\nb") = "aab"  expected "a$`b"  split().join(d) = "a$`b"
dub("$'")("a\nb") = "abb"  expected "a$'b"  split().join(d) = "a$'b"
dub("<$&>")("a\nb") = "a<\n>b"  expected "a<$&>b"  split().join(d) = "a<$&>b"
```

Impact: silent output corruption for any custom separator containing `$`; `` $` `` and `$'` copy arbitrary amounts of surrounding input into the output. Built-in `lf`/`cr`/`crlf`/`auto` are unaffected (their `to` has no `$`).

## F2. No input validation; inconsistent behaviour on non-strings

Description: no function checks its argument. `lf`/`cr`/`crlf`/`auto`/`match`/`split` call `text.replace`/`text.match`/`text.split` (`eol.js:21,30,34`): `null`, `undefined`, numbers, Symbols, arrays and Node `Buffer`s throw `TypeError`; `String` objects work; any object exposing a `replace`/`match`/`split` method is executed and its return value is passed through. `before`/`after` use `+` concatenation (`eol.js:11-17`): `null`, `undefined`, objects, arrays and Buffers are silently stringified. `dub()` with no argument inserts the text `"undefined"` for every line break (`test.js:23` calls `eol.dub()` and only checks `typeof`). The only contract is `text: string` in `eol.d.ts`, which plain-JavaScript callers do not get.

Source: `eol.js:11-35`; `eol.d.ts:6-55`.

Reproduction:

```sh
node - <<'EOF'
const eol = require("./eol.js");
const show = (label, f) => { try { console.log(label, "->", JSON.stringify(f())); } catch (e) { console.log(label, "-> throws", e.constructor.name + ":", e.message); } };
show("eol.lf(null)", () => eol.lf(null));
show("eol.lf(Buffer.from('a\\r\\nb'))", () => eol.lf(Buffer.from("a\r\nb")));
show("eol.split(42)", () => eol.split(42));
show("eol.before(null)", () => eol.before(null));
show("eol.after(undefined)", () => eol.after(undefined));
show("eol.after({})", () => eol.after({}));
show("eol.dub()('a\\nb')", () => eol.dub()("a\nb"));
show("eol.lf({replace: () => 'X'})", () => eol.lf({ replace: () => "X" }));
show("eol.match({match: () => ['Y']})", () => eol.match({ match: () => ["Y"] }));
EOF
```

Observed:

```
eol.lf(null) -> throws TypeError: Cannot read properties of null (reading 'replace')
eol.lf(Buffer.from('a\r\nb')) -> throws TypeError: text.replace is not a function
eol.split(42) -> throws TypeError: text.split is not a function
eol.before(null) -> "\nnull"
eol.after(undefined) -> "undefined\n"
eol.after({}) -> "[object Object]\n"
eol.dub()('a\nb') -> "aundefinedb"
eol.lf({replace: () => 'X'}) -> "X"
eol.match({match: () => ['Y']}) -> ["Y"]
```

Full matrix (9 input kinds x 8 functions): `node $S/r01_nonstring.js` (Appendix).

Impact: a missing value reaching `before`/`after`/`dub` produces plausible-looking corrupted text (`"null"`, `"undefined"`) with no error. The README's own integrations (`edit-file`, `map-file`, `README.md:99-119`) and Node `fs.readFile` without an encoding yield `Buffer`s, which throw.

## F3. Uncatchable process abort on newline-dense input

Description: `match`/`split` materialise one array element per line break, and the normalizers' transient heap grows with the number of matches (F8). On newline-dense strings the process aborts with a V8 `FATAL ERROR` (SIGABRT, exit 134). `try/catch` does not intercept it. With a larger heap (`--max-old-space-size=12000`), normalizers instead throw a catchable `RangeError: Invalid string length` (string limit, `buffer.constants.MAX_STRING_LENGTH` = 536870888 here; "the largest length that a string primitive can have", https://nodejs.org/api/buffer.html#bufferconstantsmax_string_length) while `split`/`match` still abort at 134217724 elements ("invalid array length").

Source: `eol.js:21,30,34`.

Reproduction (helper `$S/limits.js`, Appendix; default Node flags, heap limit 4288 MiB; exit code printed after each run):

```sh
OP=split N=100000000 node - < $S/limits.js
OP=split N=117440512 node - < $S/limits.js
OP=match N=117440512 node - < $S/limits.js
OP=lf    N=122000000 node - < $S/limits.js
OP=lf    N=130000000 node - < $S/limits.js
OP=crlf  N=130000000 node - < $S/limits.js
```

Observed:

```
eol.split("\n" x 100000000) returned length 100000001
process survived                                                      exit=0
FATAL ERROR: invalid array length Allocation failed - JavaScript heap out of memory      exit=134
FATAL ERROR: invalid array length Allocation failed - JavaScript heap out of memory      exit=134
eol.lf("\n" x 122000000) returned length 122000000                    exit=0
FATAL ERROR: Ineffective mark-compacts near heap limit Allocation failed - JavaScript heap out of memory   exit=134
FATAL ERROR: Ineffective mark-compacts near heap limit Allocation failed - JavaScript heap out of memory   exit=134
```

With `node --max-old-space-size=12000 $S/r11_limits.js <op> <n>` (Appendix): `crlf 268435455` -> `caught: RangeError: Invalid string length`, `process survived`; `split 134217724` -> `FATAL ERROR: invalid array length Allocation failed - JavaScript heap out of memory`, exit 134; `match 134217728` -> same, exit 134.

Impact: a ~117-130 MB input consisting mostly of line breaks (or short lines such as `"1\n"` repeated) kills the host process; callers cannot recover in-process. Thresholds depend on the heap limit, which V8 derives from host memory. The library documents no size limit.

## F4. Platform detection for `auto`, `before`, `after`

Description: `isWindows = typeof process != "undefined" && "win32" === process.platform` is evaluated once when the module body runs (`eol.js:7-8`). Consequences: (a) in any realm without a `process` global (browsers, including on Windows; web workers) `auto` produces LF, contradicting "Normalize line endings in text to match the current operating system" (`README.md:15`, `eol.d.ts:3`), while `index.html` invites browser use; (b) any object named `process` with `platform: "win32"`, or a redefinition of `process.platform` before first `require`, switches the output to CRLF; (c) later changes are ignored by the already-loaded instance, so two instances in one process can disagree. Node, Deno 2.9.7 and Bun 1.3.11 all expose `process` (verified with `$S/r05_auto_runtimes.mjs`: all returned LF on darwin). Node documents `process.platform` values `'aix' 'darwin' 'freebsd' 'linux' 'openbsd' 'sunos' 'win32'` (plus experimental `'android'`), https://nodejs.org/api/process.html#processplatform; nothing outside Node defines the property.

Source: `eol.js:7-8,11-17,40`.

Reproduction:

```sh
node - <<'EOF'
const fs = require("fs"), vm = require("vm");
const src = fs.readFileSync("./eol.js", "utf8");
const load = g => { const c = vm.createContext(g); vm.runInContext(src, c); return c.eol; };
console.log("host:", process.platform, "-> auto:", JSON.stringify(require("./eol.js").auto("a\r\nb")));
console.log("realm without process (browser, any OS) -> auto:", JSON.stringify(load({}).auto("a\r\nb")));
console.log("realm with process={platform:'win32'} -> auto:", JSON.stringify(load({ process: { platform: "win32" } }).auto("a\nb")));
const d = Object.getOwnPropertyDescriptor(process, "platform");
Object.defineProperty(process, "platform", { value: "win32" });
delete require.cache[require.resolve("./eol.js")];
const spoofed = require("./eol.js");
Object.defineProperty(process, "platform", d);
console.log("platform spoofed at load, then restored -> auto:", JSON.stringify(spoofed.auto("a\nb")), "(process.platform now", process.platform + ")");
EOF
```

Observed:

```
host: darwin -> auto: "a\nb"
realm without process (browser, any OS) -> auto: "a\nb"
realm with process={platform:'win32'} -> auto: "a\r\nb"
platform spoofed at load, then restored -> auto: "a\r\nb" (process.platform now darwin)
```

Impact: `auto`, `before`, `after` and `String(eol.auto)` produce host-dependent bytes for identical input; the result is decided by mutable global state at load time. The Windows branch (`eol.js:8` `"\r\n"`) is never executed by the test suite on non-Windows hosts (F9).

## F5. Type declarations disagree with the compiler, the docs and the runtime

Description:

1. `eol.d.ts:1` declares a namespace with the `module` keyword (`declare module eol {`). TypeScript 6.0.3 and 7.0.2 reject the file with TS1540 for every consumer that does not set `skipLibCheck`. Sources: TypeScript 6.0 release notes, "Deprecated: legacy module Syntax for namespaces" ("In TypeScript 6.0, using module where namespace is expected is now a hard deprecation"), https://devblogs.microsoft.com/typescript/announcing-typescript-6-0/#deprecated-legacy-module-syntax-for-namespaces; extension to the `declare module foo` form, https://github.com/microsoft/TypeScript/pull/62876.
2. README documents `eol.split(text).join(eol.auto)` (`README.md:69`, also `index.html` "Normalizers are usable as glue"). `eol.d.ts` types `auto` as `(text: string) => string`, so `join(eol.auto)` is TS2345 on 5.9.3, 6.0.3, 7.0.2.
3. `eol.d.ts` exposes `export function lf` etc. inside an `export =` namespace, so `import { lf } from "eol"` type-checks (exit 0 on 5.9.3; exit 0 on 6/7 with `--skipLibCheck`), but Node derives CommonJS named exports from "a heuristic static analysis ... against the source text" (https://nodejs.org/api/esm.html#commonjs-namespaces), which finds none in `module.exports = make()`, and fails at link time.
4. README documents `import eol from "eol"` (`README.md:9`). On TS 5.9.3 with `--module commonjs` and no `esModuleInterop` this is TS1259.

Source: `eol.d.ts:1,6,31,58-60`; `eol.js:2`; `README.md:9,65-71`.

Reproduction (in `$S/ts`; `node_modules/eol` is a copy of the repo's `eol.js`, `eol.d.ts`, `package.json`, `cmp`-verified identical; case files in Appendix):

```sh
cd $S/ts && npm i ts5@npm:typescript@5.9.3 ts6@npm:typescript@6.0.3 ts7@npm:typescript@7.0.2
mkdir -p node_modules/eol && cp $REPO/eol.js $REPO/eol.d.ts $REPO/package.json node_modules/eol/
node node_modules/ts6/bin/tsc --noEmit --strict --module nodenext --moduleResolution nodenext cases/declfile.ts
node node_modules/ts5/bin/tsc --noEmit --strict --module nodenext --moduleResolution nodenext cases/join.ts
node node_modules/ts5/bin/tsc --noEmit --strict --module nodenext --moduleResolution nodenext cases/named_import.mts; echo "exit=$?"
node node_modules/ts5/bin/tsc --noEmit --strict --module commonjs --moduleResolution node10 cases/default_import.ts
```

Observed:

```
node_modules/eol/eol.d.ts(1,16): error TS1540: A 'namespace' declaration should not be declared using the 'module' keyword. Please use the 'namespace' keyword instead.
cases/join.ts(3,54): error TS2345: Argument of type '(text: string) => string' is not assignable to parameter of type 'string'.
exit=0
cases/default_import.ts(2,8): error TS1259: Module '"eol"' can only be default-imported using the 'esModuleInterop' flag
```

(TS1540 is identical on 7.0.2; TS2345 identical on 6.0.3 and 7.0.2.)

Runtime half of item 3:

```sh
printf 'import { lf } from "%s/eol.js";\nconsole.log(lf("a\\r\\nb"));\n' "$PWD" > $S/named.mjs && node $S/named.mjs
```

```
SyntaxError: Named export 'lf' not found. The requested module '/Users/ivanmilosavljevic/src/eol/eol.js' is a CommonJS module, which may not support all module.exports as named exports.
```

Impact: builds fail on current TypeScript unless library checking is disabled; code that type-checks crashes at module link time.

## F6. Unicode and ECMAScript line terminators are not recognised

Description: the only pattern is `/\r\n|\r|\n/g` (`eol.js:9`). U+0085 NEL, U+2028 LS, U+2029 PS, U+000B VT, U+000C FF pass through unchanged and do not split lines. UAX #14 (Revision 57, Unicode 18.0.0) class BK "Mandatory Break" lists U+000B, U+000C, U+2028, U+2029; class NL (U+0085) "acts like BK in all respects"; rule LB5: "Treat CR followed by LF, as well as CR, LF, and NL as hard line breaks" (https://www.unicode.org/reports/tr14/#BK, #NL, #LB5). The Unicode Standard §5.8 "Newline Guidelines", Table 5-1, lists CR, LF, CRLF, NEL, VT, FF, LS, PS (https://www.unicode.org/versions/latest/core-spec/chapter-5/#G10213). ECMAScript §12.3, Table 32, treats LF, CR, LS, PS as line terminators (https://tc39.es/ecma262/#sec-line-terminators), so `eol.split` and `/^.*$/gm` on the same string disagree. The README never claims Unicode coverage; it lists only `\r\n`, `\n`, `\r` (`README.md:16,21,26,31`). Severity Low for that reason; the failure is silent.

Source: `eol.js:9`.

Reproduction:

```sh
node - <<'EOF'
const eol = require("./eol.js");
for (const [n, c] of [["VT U+000B", "\v"], ["FF U+000C", "\f"], ["NEL U+0085", "\u0085"], ["LS U+2028", "\u2028"], ["PS U+2029", "\u2029"]]) {
  const s = "a" + c + "b";
  console.log(n.padEnd(11), "eol.split len", eol.split(s).length, "| eol.match len", eol.match(s).length, "| eol.lf changed", eol.lf(s) !== s, "| engine /^.*$/gm lines", s.match(/^.*$/gm).length);
}
EOF
```

Observed:

```
VT U+000B   eol.split len 1 | eol.match len 0 | eol.lf changed false | engine /^.*$/gm lines 1
FF U+000C   eol.split len 1 | eol.match len 0 | eol.lf changed false | engine /^.*$/gm lines 1
NEL U+0085  eol.split len 1 | eol.match len 0 | eol.lf changed false | engine /^.*$/gm lines 1
LS U+2028   eol.split len 1 | eol.match len 0 | eol.lf changed false | engine /^.*$/gm lines 2
PS U+2029   eol.split len 1 | eol.match len 0 | eol.lf changed false | engine /^.*$/gm lines 2
```

Impact: line counts and line boundaries differ from Unicode and from the JavaScript engine for text containing these characters (e.g. EBCDIC-derived NEL, word-processor LS/PS); "normalized" output still contains line separators.

## F7. UMD wrapper side effects

Description: if `module` is undefined or has a falsy `exports`, the wrapper assigns `root[name] = make()` with `root = this` (`eol.js:1-4`). (a) In a classic script it overwrites any existing global `eol` without a no-conflict path. (b) If the realm has an unrelated global `module` with an `exports` object, that object is replaced by the eol API. (c) When the file is evaluated as an ES module (Deno importing the file, a `.mjs` copy, Node `vm.SourceTextModule`; `<script type="module">` has the same top-level `this === undefined` semantics but was not run in a browser) or inside a strict-mode function wrapper, `this` is `undefined` and loading throws.

Source: `eol.js:1-4`; `index.html:133`.

Reproduction:

```sh
node - <<'EOF'
const fs = require("fs"), vm = require("vm");
const src = fs.readFileSync("./eol.js", "utf8");
let c = vm.createContext({ eol: { owner: "other-lib" } }); vm.runInContext(src, c);
console.log("existing global eol.owner after load:", c.eol.owner, "| keys:", Object.keys(c.eol).join(","));
c = vm.createContext({ module: { exports: { important: true } } }); vm.runInContext(src, c);
console.log("unrelated global module.exports.important after load:", c.module.exports.important, "| global eol defined:", "eol" in c);
EOF
cp eol.js $S/eol_copy.mjs && node $S/eol_copy.mjs
printf 'import * as m from "%s/eol.js";\n' "$PWD" > $S/deno_import.mjs && deno run --allow-read $S/deno_import.mjs
node --experimental-vm-modules -e 'const vm=require("vm");const src=require("fs").readFileSync("./eol.js","utf8");const m=new vm.SourceTextModule(src);m.link(()=>{}).then(()=>m.evaluate()).then(()=>console.log("ok"),e=>console.log("THROWS",e.constructor.name+":",e.message))'
node -e 'const vm=require("vm");const src=require("fs").readFileSync("./eol.js","utf8");try{vm.runInContext("(function(){\"use strict\";\n"+src+"\n}).call(undefined)",vm.createContext({}));console.log("ok")}catch(e){console.log("strict wrapper: THROWS",e.constructor.name+":",e.message)}'
```

Observed:

```
existing global eol.owner after load: undefined | keys: lf,cr,crlf,auto,before,after,match,split,dub
unrelated global module.exports.important after load: undefined | global eol defined: false
TypeError: Cannot set properties of undefined (setting 'eol')        (node, exit 1)
error: Uncaught (in promise) TypeError: Cannot set properties of undefined (setting 'eol')   (deno)
THROWS TypeError: Cannot set properties of undefined (setting 'eol')                      (vm.SourceTextModule)
strict wrapper: THROWS TypeError: Cannot set properties of undefined (setting 'eol')
```

Impact: global namespace clobbering in script contexts; hard load failure in ES-module contexts. No input-driven exploitation path was found (see "Investigated, not confirmed").

## F8. Throughput and memory on newline-dense input

Description: measured time and transient heap of `replace`/`match`/`split` scale with the number of line breaks rather than the number of bytes. Text with ordinary line lengths is fast; newline-dense text is ~10-50x slower per byte and drives process peak RSS to 16-19x (normalizers) and 23-29x (`match`) the input size (ranges over two runs). Per-character cost of `lf` on `"\r"` repeated grows with size (doubling n multiplies time by 2.2-2.7, not 4: superlinear from GC pressure, not quadratic).

Source: `eol.js:21,30,34`.

Reproduction: `node --expose-gc $S/r10_perf.js <op> <MiB> <text|dense>` for op in lf, crlf, cr, auto, match, split, before, after and MiB in 10, 100; `node --max-old-space-size=12000 $S/r09b_scaling.js lf` (Appendix). One process per measurement. "text" = lines of 0-119 chars with endings cycling CRLF/LF/CR; "dense" = `"\r\n"` repeated.

Observed (second run, selected; full output of both harnesses in Appendix):

| op | input | time | throughput | process peak RSS |
|----|-------|------|-----------|----------|
| lf | 10 MiB text | 3.6 ms | 2780 MiB/s | 114 MiB |
| lf | 100 MiB text | 185.9 ms | 538 MiB/s | 700 MiB |
| split | 100 MiB text | 55.9 ms | 1787 MiB/s | 614 MiB |
| match | 100 MiB text | 33.0 ms | 3031 MiB/s | 624 MiB |
| lf | 100 MiB dense | 1975.9 ms | 51 MiB/s | 1574 MiB |
| crlf | 100 MiB dense | 1814.1 ms | 55 MiB/s | 1850 MiB |
| split | 100 MiB dense | 981.7 ms | 102 MiB/s | 1680 MiB |
| match | 100 MiB dense | 2379.5 ms | 42 MiB/s | 2293 MiB |
| before/after | 100 MiB | ~0 ms | n/a (V8 cons-string; cost deferred to first flatten) | unchanged |

First run of the same harness: lf 100 MiB dense 1814 ms / 1849 MiB, match 100 MiB dense 2350 ms / 2915 MiB.

Scaling of `lf` on `"\r"` x n (12 GB heap): 9.66 ns/char at 1e6, 37.52 at 1.6e7, 66.34 at 1.28e8 (8491 ms). With default heap the 1.28e8 step aborts with `FATAL ERROR: Ineffective mark-compacts near heap limit` (F3).

Impact: worst-case latency and memory are input-shape dependent and unbounded by the library; combined with F3, memory use ends in process abort.

## F9. Test suite defects and gaps

Description:

1. `test.js:35` asserts `String(eol.auto) === isWindows ? "\r\n" : "\n"`. `===` binds tighter than `?:`, so the asserted value is always `"\r\n"` or `"\n"` (truthy); `aok` coerces with `!!` (`node_modules/aok/aok.js:107`). The assertion cannot fail. Present since `b23e98c` (2017-05-01).
2. Mutation probe (darwin): 11 single-line mutants of a copy of `eol.js` run against the unmodified `test.js`; 7 behaviour changes pass the suite.
3. V8 precise coverage under `test.js`: uncovered ranges `eol.js:2-3` (`else root[name] = make()`) and `eol.js:8` (`? "\r\n"`).
4. Not tested at all: non-string input, Unicode terminators, `$` in `dub`, `"\r\r\n"`, `split("")`/`match("")`, trailing line breaks, the browser/UMD path, ES-module import, `eol.d.ts`, large inputs, Windows (`before`/`after` are only checked after `eol.lf()` normalisation, `test.js:63-66`, which hides the platform line break).
5. No CI: `.github/` contains only `FUNDING.yml`. `eslint.config.js` enables only `indent` and `quotes`.

Source: `test.js:23,35,63-66`; `eslint.config.js:1-6`; `.github/FUNDING.yml`.

Reproduction:

```sh
npm test 2>&1 | tail -1
node - <<'EOF'
for (const autoValue of ["\n", "\r\n", "WRONG"]) for (const isWindows of [false, true]) {
  const asserted = String({ toString: () => autoValue }) === isWindows ? "\r\n" : "\n"; // expression from test.js:35
  console.log(`String(auto)=${JSON.stringify(autoValue)} isWindows=${isWindows} -> asserted ${JSON.stringify(asserted)} passes=${!!asserted}`);
}
EOF
node $S/r14_mutants.js
NODE_V8_COVERAGE=$S/cov node test.js >/dev/null && node $S/r13_coverage.js $S/cov
```

Observed:

```
All tests passed =)            (39 "ok" lines; exit 0; eslint 10.12.0 from local node_modules)
String(auto)="\n" isWindows=false -> asserted "\n" passes=true
String(auto)="\n" isWindows=true -> asserted "\n" passes=true
String(auto)="\r\n" isWindows=false -> asserted "\n" passes=true
String(auto)="\r\n" isWindows=true -> asserted "\n" passes=true
String(auto)="WRONG" isWindows=false -> asserted "\n" passes=true
String(auto)="WRONG" isWindows=true -> asserted "\n" passes=true

SURVIVED  baseline (no mutation)          <- control: unmodified eol.js passes
SURVIVED  before() ignores platform: linebreak -> "\r\n"
SURVIVED  after() ignores platform: linebreak -> "\r\n"
SURVIVED  Windows detection disabled
killed    Windows detection inverted <- Error: FAILED TEST: auto normalizes
SURVIVED  regex also matches U+2028/U+2029/NEL
SURVIVED  dub(): replacement as function (disables $-patterns)
SURVIVED  normalizers coerce input with String()
SURVIVED  browser export branch broken
killed    match() returns null when none <- TypeError: Cannot read properties of null (reading 'length')
killed    split() drops empty lines <- Error: FAILED TEST: split join
killed    lf -> \r\n <- Error: FAILED TEST: lf repeat newlines intact

UNCOVERED range in (anonymous): lines 2-3: "else root[name] = make()"
UNCOVERED range in (anonymous): lines 8-8: "? \"\\r\\n\""
functions: (anon)=1 (anon)=1 (anon)=1 before=2 after=2 dub=5 change=29 change.toString=7 match=3 split=8
```

Impact: the suite does not constrain platform behaviour, input handling, `dub` semantics or the browser export; regressions in those areas would ship.

## F10. Development dependency advisories and tree drift

Description: `npm audit` against the committed lockfile reports 6 vulnerable packages (1 moderate, 5 high; 20 advisories). All are transitive devDependencies of `eslint@9.9.1` (lockfile parents: `ajv` <- `eslint`, `@eslint/eslintrc`; `minimatch` <- `eslint`, `@eslint/config-array`, `@eslint/eslintrc`; `brace-expansion` <- `minimatch`; `js-yaml` <- `@eslint/eslintrc`; `flatted` <- `flat-cache`; `cross-spawn` <- `eslint`; all `"dev": true`); `eol` has zero runtime dependencies (`package.json` has no `dependencies`; audit metadata `prod: 1` = the root). A fresh `npm ci` installs the locked vulnerable versions. Upstream Dependabot PR #44 (js-yaml 4.1.1) has been open and unmerged since 2025-11-21. The local `node_modules` does not match the lockfile (`eslint@10.12.0` installed, lockfile and `package.json` say `9.9.1`/`^9.9.1`), so `npm test` in this checkout does not exercise the locked toolchain.

Source: `package.json:44-47`; `package-lock.json` (`node_modules/brace-expansion` 1.1.11, `minimatch` 3.1.2, `js-yaml` 4.1.0, `flatted` 3.3.1, `cross-spawn` 7.0.3, `ajv` 6.12.6).

Reproduction:

```sh
npm audit --json | node -e 'let s="";process.stdin.on("data",d=>s+=d).on("end",()=>{const a=JSON.parse(s);console.log(JSON.stringify(a.metadata.vulnerabilities));for(const[k,v]of Object.entries(a.vulnerabilities))console.log(k,v.severity,v.range,v.via.filter(x=>typeof x==="object").map(x=>x.url.split("/").pop()).join(" "))})'
npm ls --all 2>&1 | grep invalid | head -1
```

Observed:

```
{"info":0,"low":0,"moderate":1,"high":5,"critical":0,"total":6}
ajv moderate <6.14.0 GHSA-2g4f-4pwh-qvx6
brace-expansion high <=1.1.20 GHSA-v6h2-p8h4-qcjw GHSA-f886-m6hf-6m8v GHSA-3jxr-9vmj-r5cp GHSA-mh99-v99m-4gvg GHSA-rgw5-rvv9-x895 GHSA-q2hr-2g5m-vwhr GHSA-qhr7-859c-m2p7 GHSA-6j4f-fj2g-mc7p
cross-spawn high 7.0.0 - 7.0.4 GHSA-3xgq-45jj-v275
flatted high <=3.4.1 GHSA-25h7-pfq9-p65f GHSA-rf6f-7fwh-wjgh
js-yaml high 4.0.0 - 4.3.1 GHSA-mh29-5h37-fv8m GHSA-h67p-54hq-rp68 GHSA-52cp-r559-cp3m GHSA-5p4m-2wfm-xmqj GHSA-2883-xcg3-v3hh
minimatch high <=3.1.3 GHSA-3ppc-4f35-3m26 GHSA-7r86-cg39-jmmj GHSA-23c5-xmqv-rm74
└─┬ eslint@10.12.0 invalid: "^9.9.1" from the root project
```

Advisory pages: `https://github.com/advisories/<GHSA-id>`.

Impact: affects developer and CI machines running lint, not consumers of the published package.

## F11. Not safe for chunked input

Description: each call is stateless. A CR at the end of one chunk and LF at the start of the next are counted as two line breaks. The README integrations operate on whole files, so this is a hazard for stream users, not a contract violation.

Source: `eol.js:9,21`.

Reproduction:

```sh
node - <<'EOF'
const eol = require("./eol.js");
const chunks = ["A\r", "\nB\r", "\n"];
console.log("whole  :", JSON.stringify(eol.lf(chunks.join(""))));
console.log("chunked:", JSON.stringify(chunks.map(eol.lf).join("")));
EOF
```

Observed:

```
whole  : "A\nB\n"
chunked: "A\n\nB\n\n"
```

Impact: a `Transform` stream applying `eol.lf` per chunk doubles line breaks at chunk boundaries (verified with `stream.Transform`: 4 line breaks instead of 2).

## F12. `split` line-count semantics

Description: `split` is `String.prototype.split` with the regex (`eol.js:33-35`): a trailing line break yields a trailing empty element and empty input yields `[""]`. README calls the result an "array of lines" (`README.md:61`). POSIX.1-2024 §3.185 "Line": "A sequence of zero or more non-<newline> characters plus a terminating <newline> character" (https://pubs.opengroup.org/onlinepubs/9799919799/basedefs/V1_chap03.html#tag_03_185), so `"a\n"` is one POSIX line but two `split` elements.

Reproduction:

```sh
node - <<'EOF'
const eol = require("./eol.js");
for (const t of ["", "a", "a\n", "a\nb\n"]) console.log(`eol.split(${JSON.stringify(t)}) = ${JSON.stringify(eol.split(t))}`);
EOF
```

Observed:

```
eol.split("") = [""]
eol.split("a") = ["a"]
eol.split("a\n") = ["a",""]
eol.split("a\nb\n") = ["a","b",""]
```

Impact: off-by-one line counts for callers assuming POSIX semantics. Invariant `split(s).length === match(s).length + 1` holds (verified by fuzz, see below).

## F13. Mutable singleton and reachable private regex

Description: the exported object is a plain mutable object shared through the `require` cache (`eol.js:6,37-46`). The private regex (`eol.js:9`) is reachable by any same-realm code through `RegExp.prototype[Symbol.replace]` and can be rewritten in place with Annex B `RegExp.prototype.compile`. Requires code execution in the same realm; not exploitable through input.

Reproduction:

```sh
node - <<'EOF'
const a = require("./eol.js");
require("./eol.js").lf = () => "TAMPERED";   // any other module in the process
console.log("a.lf('x') =", a.lf("x"), "| Object.isFrozen =", Object.isFrozen(a));
let rx; const orig = RegExp.prototype[Symbol.replace];
RegExp.prototype[Symbol.replace] = function (...args) { rx = this; return orig.apply(this, args); };
a.crlf("x"); RegExp.prototype[Symbol.replace] = orig;
rx.compile("b", "g");                          // Annex B RegExp.prototype.compile mutates the private regex
console.log("after compile: a.crlf('a\\nb') =", JSON.stringify(a.crlf("a\nb")), "| a.split('abc') =", JSON.stringify(a.split("abc")));
EOF
```

Observed:

```
a.lf('x') = TAMPERED | Object.isFrozen = false
after compile: a.crlf('a\nb') = "a\n\r\n" | a.split('abc') = ["a","c"]
```

Impact: no isolation from other code in the process (any dependency can alter behaviour for all consumers).

## F14. Packaging

Description: `package.json` has no `files` field; the published tarball contains 11 files including `test.js`, `index.html`, `style.css`, `eslint.config.js`, `.editorconfig`, `.github/FUNDING.yml` (upstream PR #38 "Exclude index.html from published NPM package" was closed). The registry entry for `eol@0.10.0` carries a registry signature but no provenance attestation (`dist.attestations` absent). Single npm maintainer (`ryanve`).

Reproduction:

```sh
npm view eol@0.10.0 dist --json --registry=https://registry.npmjs.org/
mkdir -p $S/pub && cd $S/pub && npm pack eol@0.10.0 --registry=https://registry.npmjs.org/ && tar tzf eol-0.10.0.tgz
echo "sha512-$(openssl dgst -sha512 -binary eol-0.10.0.tgz | base64)"
tar xzf eol-0.10.0.tgz && for f in eol.js eol.d.ts test.js index.html package.json README.md; do cmp -s package/$f $REPO/$f && echo "$f identical"; done
```

Observed: `dist` keys `integrity,shasum,tarball,fileCount,unpackedSize,signatures` (no `attestations`), `fileCount: 11`; tarball lists `package/.editorconfig package/style.css package/index.html package/eol.js package/eslint.config.js package/test.js package/package.json package/LICENSE.md package/README.md package/eol.d.ts package/.github/FUNDING.yml`; recomputed sha512 equals the registry `integrity`; all six files print `identical`.

Impact: none at runtime; no build-provenance link between the published file and the source repository (byte equality was verified manually here).

---

## Investigated, not confirmed

| Suspicion | Evidence | Result |
|-----------|----------|--------|
| ReDoS in `/\r\n\|\r\|\n/g` | Pattern is an alternation of three fixed literals with no quantifiers. Timings for `lf`/`crlf`/`match`/`split` on `"a"`x n, `"\r"`x n, `"\r\n"`, `"\n\r"`, `"\r\r\n"` repeated, n = 1e4..1e7 (`r09_redos.js`): 1e6 -> 1e7 ratio x9.7-x28.6; e.g. `lf` on `"\r"` x 1e7 = 413 ms, no-match input x 1e7 = 0.34 ms. Doubling series to 1.28e8 grows x2.2-2.7 per doubling. | No catastrophic backtracking; linear match work with GC-driven superlinear constant (reported as F8). |
| `lastIndex` state bugs from the shared `/g` regex | `RegExp.prototype[%Symbol.replace%]` and `[%Symbol.match%]` reset `lastIndex` to 0 for global regexes; `[%Symbol.split%]` builds a new splitter via `SpeciesConstructor` (ECMAScript §22.2.6.11 https://tc39.es/ecma262/#sec-regexp.prototype-%25symbol.replace%25 "If global is true, then Perform ? Set(regexp, "lastIndex", +0, true)"; §22.2.6.8 https://tc39.es/ecma262/#sec-regexp.prototype-%25symbol.match%25; §22.2.6.14 https://tc39.es/ecma262/#sec-regexp.prototype-%25symbol.split%25). 1e5 iterations interleaving all ops with an exception thrown mid-call: drift 0. Setting the captured regex's `lastIndex = 5` before calls: outputs unchanged. | Not a bug. |
| Mixed sequences `"\r\r\n"`, `"\n\r"`, `"\r\n\r"` | `"\r\r\n"` -> `["\r","\r\n"]`; `"\n\r"` -> `["\n","\r"]`; CRLF is treated as one unit, consistent with the Unicode newline guidelines. Differential fuzz: 200,000 random strings over `{\r,\n,a,U+2028,U+D800,$}` vs. an independent state machine for `split`, `match`, `lf`, `crlf`, `cr`, idempotence of `lf`/`crlf`, and `split.length === match.length + 1`: 0 mismatches (`r03_sequences.js`). | Core CR/LF/CRLF handling correct. |
| Lone surrogates | `"\ud800\n\udc00"` -> lf `"\ud800\n\udc00"`, split `["\ud800","\udc00"]`. Regex has no `u` flag; code units pass through. | No corruption beyond the input's own ill-formedness. |
| Empty string | `lf("")`=`""`, `match("")`=`[]`, `split("")`=`[""]` (see F12). | No defect. |
| Prototype pollution | `eol.js` writes only constant keys on a fresh `{}` (`eol.js:6,37-45`) and `change.toString`; no input-derived property writes; no `__proto__`/`constructor`/`Object.assign`/`eval` (`git grep`). | None. |
| XSS / third-party code in `index.html` | Only `<script src="eol.js">` (local, `index.html:133`) and `<link rel="stylesheet" href="style.css">`; no inline scripts, event handlers, `javascript:` URLs, `innerHTML`, or external subresources; 11 external links are navigation only. | None. |
| Install-time code / integrity | All 90 lockfile entries have `sha512` integrity, resolve from `registry.npmjs.org`, none has `hasInstallScript`; `npm audit signatures`: 78 installed packages have verified registry signatures; published tarball equals repo files. | None. |
| Deno / Bun via `require` | Deno 2.9.7 and Bun 1.3.11 expose `process`; `auto` matched Node on darwin. Bun's ESM import of `eol.js` works; Deno's does not (F7). | Only the ESM-evaluation failure (F7). |
| Quadratic behaviour | See ReDoS row; `before`/`after` O(1) (cons strings). | Not observed. |

## Safety-critical fitness

What a nuclear I&C deployment is measured against (official pages; clause numbers deliberately not quoted because the standards are paywalled and were not read in full):

- IEC 61508-3:2010, Ed. 2.0, "Functional safety of electrical/electronic/programmable electronic safety-related systems - Part 3: Software requirements"; "applies to any software forming part of a safety-related system or used to develop a safety-related system within the scope of IEC 61508-1 and IEC 61508-2". https://webstore.iec.ch/en/publication/5517
- IEC 60880:2006, Ed. 2.0, "Nuclear power plants - Instrumentation and control systems important to safety - Software aspects for computer-based systems performing category A functions"; "Provides requirements for the purpose of achieving highly reliable software." https://webstore.iec.ch/en/publication/3795
- IEC 62138:2018, Ed. 2.0, software for I&C systems performing category B or C functions "as defined by IEC 61226". https://webstore.iec.ch/en/publication/30172
- IAEA Safety Standards Series No. SSG-39, "Design of Instrumentation and Control Systems for Nuclear Power Plants" (2016, STI/PUB/1694). https://www.iaea.org/publications/10838/design-of-instrumentation-and-control-systems-for-nuclear-power-plants. IAEA lists a revision (DS514) as under development; no superseding edition was found.

Which standard applies depends on the safety category the integrator assigns to the function (IEC 61226 categorisation is referenced by IEC 62138's abstract).

What this repository provides, as verified:

| Item | Evidence |
|------|----------|
| Requirements / specification | None. Behaviour is described only by `README.md` (API bullets), `eol.d.ts` JSDoc and `index.html`. No defined input domain, error behaviour, size limits or platform semantics (F2, F3, F4). |
| Design documentation, traceability | None. |
| Verification | 39 assertions in one file; no CI; V8 coverage leaves the Windows and browser branches unexecuted; 7/11 mutants survive; one assertion is a tautology (F9). No tests of `eol.d.ts`. |
| Static analysis | ESLint with 2 style rules (`indent`, `quotes`). No type checking of `eol.js`. |
| Determinism | `auto`/`before`/`after` output depends on the host and on mutable global state at load (F4). Memory and latency depend on input shape; abort is possible (F3, F8). |
| Language / runtime | JavaScript on V8: dynamic typing, garbage collection, JIT; failure modes include uncatchable V8 `FATAL ERROR` aborts (F3). |
| Versioning | `0.10.0`. SemVer: "Major version zero (0.y.z) is for initial development. Anything MAY change at any time. The public API SHOULD NOT be considered stable." (https://semver.org/#spec-item-4) |
| Maintenance | 79 commits, 2014-03-12 to 2024-09-01 upstream (plus 1 fork commit 2026-10-10). Last functional change to `eol.js`: 2022-02-28 (`7045668`, `27f5f2a`); 2024 change was quote style only. Upstream (`gh api repos/ryanve/eol`, 2026-10-10): not archived, 40 stars, 5 forks, 2 open PRs (Dependabot #44 since 2025-11-21; #46 by TheJavaGuy), last release v0.10.0 2024-09-01. Single npm maintainer. |
| Warranty / liability | MIT: "THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND" (`LICENSE.md:14`). |
| Supply chain | Zero runtime dependencies; dev toolchain lockfile carries 6 vulnerable packages (F10); no provenance attestation (F14). |

Factual conclusion: the repository contains no requirements specification, design description, safety plan, or verification and validation records; the standards above are written around producing such artefacts. Using this package in a safety function would leave the integrator to produce all of that evidence, fix F1-F7, and justify the JavaScript runtime itself. For scale: `eol.js` is 47 lines, and its CR/LF/CRLF transformation matched an independent reference implementation on 200,000 fuzzed inputs (0 mismatches).

---

## Appendix: harnesses

All files were created in `$S`, not in the repo.

<details><summary>limits.js (F3)</summary>

```js
const eol = require("/Users/ivanmilosavljevic/src/eol/eol.js");
const { OP, N } = process.env;
try { const r = eol[OP]("\n".repeat(+N)); console.log(`eol.${OP}("\\n" x ${N}) returned length ${r.length}`); }
catch (e) { console.log(`eol.${OP}("\\n" x ${N}) caught ${e.constructor.name}: ${e.message}`); }
console.log("process survived");
```
</details>

<details><summary>r11_limits.js (F3, larger heap)</summary>

```js
// Engine-limit failure modes. Usage: node r11_limits.js <op> <count> [char]
const eol = require("/Users/ivanmilosavljevic/src/eol/eol.js");
const [op, count, ch = "\n"] = [process.argv[2], Number(process.argv[3]), process.argv[4] === "cr" ? "\r" : "\n"];
const s = ch.repeat(count);
console.log(`input: ${count} x ${JSON.stringify(ch)} (${(count / 1048576).toFixed(0)} MiB one-byte string); op=eol.${op}`);
try {
  const r = eol[op](s);
  console.log("returned, length =", r.length);
} catch (e) {
  console.log("caught:", e.constructor.name + ": " + e.message);
}
console.log("process survived");
```
</details>

<details><summary>r01_nonstring.js (F2 full matrix)</summary>

```js
// Non-string inputs against every exported function
const eol = require("/Users/ivanmilosavljevic/src/eol/eol.js");
const fns = ["lf", "cr", "crlf", "auto", "before", "after", "match", "split"];
const inputs = [
  ["null", null], ["undefined", undefined], ["123", 123], ["{}", {}],
  ["Symbol()", Symbol("s")], ["Buffer('a\\r\\nb')", Buffer.from("a\r\nb")],
  ["new String('a\\r\\nb')", new String("a\r\nb")], ["['a\\nb']", ["a\nb"]],
  ["{replace:()=>'X',match:()=>['Y'],split:()=>['Z']}", { replace: () => "X", match: () => ["Y"], split: () => ["Z"] }],
];
for (const f of fns) {
  const row = [];
  for (const [label, v] of inputs) {
    try { row.push(`${label} -> ${JSON.stringify(eol[f](v))}`); }
    catch (e) { row.push(`${label} -> THROWS ${e.constructor.name}: ${e.message}`); }
  }
  console.log(`eol.${f}:\n  ` + row.join("\n  "));
}
console.log("eol.dub()('a\\nb') ->", JSON.stringify(eol.dub()("a\nb")));
console.log("eol.dub(null)('a\\nb') ->", JSON.stringify(eol.dub(null)("a\nb")));
console.log("eol.dub(0)('a\\nb') ->", JSON.stringify(eol.dub(0)("a\nb")));
```
</details>

<details><summary>esc.js (helper for r03)</summary>

```js
module.exports = function esc(v) {
  return JSON.stringify(v, null, 0).replace(/[\u007f-￿]/g, c => "\\u" + c.charCodeAt(0).toString(16).padStart(4, "0"));
};
```
</details>

<details><summary>r03_sequences.js (mixed sequences, surrogates, differential fuzz)</summary>

```js
// Mixed sequences, lone surrogates, regex statefulness, differential fuzz vs reference state machine
const eol = require("/Users/ivanmilosavljevic/src/eol/eol.js");
const esc = require("./esc.js");
const cases = ["", "\r\r\n", "\n\r", "\r\n\r", "\r\n\n", "\n\r\n", "a\r", "\ud800\n\udc00", "\udfff\r\n\ud83d"];
for (const c of cases)
  console.log(esc(c).padEnd(22), "lf=" + esc(eol.lf(c)).padEnd(20), "crlf=" + esc(eol.crlf(c)).padEnd(26), "split=" + esc(eol.split(c)).padEnd(26), "match=" + esc(eol.match(c)));

// Reference: Unicode 5.8 NLF recognition restricted to CR, LF, CRLF (CRLF is one unit)
function refSplit(s) { const out = []; let cur = ""; for (let i = 0; i < s.length; i++) { const ch = s[i];
  if (ch === "\r") { out.push(cur); cur = ""; if (s[i + 1] === "\n") i++; } else if (ch === "\n") { out.push(cur); cur = ""; } else cur += ch; }
  out.push(cur); return out; }
function refBreaks(s) { const out = []; for (let i = 0; i < s.length; i++) { if (s[i] === "\r") { if (s[i + 1] === "\n") { out.push("\r\n"); i++; } else out.push("\r"); } else if (s[i] === "\n") out.push("\n"); } return out; }
const alphabet = ["\r", "\n", "a", "\u2028", "\ud800", "$"];
let seed = 12345; const rnd = () => (seed = (seed * 1103515245 + 12345) >>> 0) / 2 ** 32;
let mismatches = 0, N = 200000;
for (let n = 0; n < N; n++) {
  let s = ""; const len = Math.floor(rnd() * 12); for (let i = 0; i < len; i++) s += alphabet[Math.floor(rnd() * alphabet.length)];
  const rs = refSplit(s), rb = refBreaks(s);
  const ok = JSON.stringify(eol.split(s)) === JSON.stringify(rs) && JSON.stringify(eol.match(s)) === JSON.stringify(rb)
    && eol.lf(s) === rs.join("\n") && eol.crlf(s) === rs.join("\r\n") && eol.cr(s) === rs.join("\r")
    && eol.lf(eol.lf(s)) === eol.lf(s) && eol.crlf(eol.crlf(s)) === eol.crlf(s)
    && eol.split(s).length === eol.match(s).length + 1;
  if (!ok) { mismatches++; if (mismatches < 5) console.log("MISMATCH", esc(s)); }
}
console.log(`differential fuzz: ${N} random strings, ${mismatches} mismatches`);
```
</details>

<details><summary>r05_auto_runtimes.mjs (F4; run with node, deno run --allow-read, bun)</summary>

```js
import { createRequire } from "node:module";
const require = createRequire(import.meta.url);
const eol = require("/Users/ivanmilosavljevic/src/eol/eol.js");
const rt = typeof Deno !== "undefined" ? "deno " + Deno.version.deno : typeof Bun !== "undefined" ? "bun " + Bun.version : "node " + process.version;
console.log(rt, "typeof process =", typeof process, "platform =", typeof process !== "undefined" ? process.platform : "-", "auto('a\\r\\nb') =", JSON.stringify(eol.auto("a\r\nb")));
```
</details>

<details><summary>r09_redos.js (ReDoS probe)</summary>

```js
// ReDoS probe for /\r\n|\r|\n/g (eol.js:9): time vs input length for adversarial shapes.
const eol = require("/Users/ivanmilosavljevic/src/eol/eol.js");
const shapes = {
  "a*n (no match)": n => "a".repeat(n),
  "\\r*n": n => "\r".repeat(n),
  "\\r\\n*n/2": n => "\r\n".repeat(n / 2),
  "\\n\\r*n/2": n => "\n\r".repeat(n / 2),
  "(\\r\\r\\n)*n/3": n => "\r\r\n".repeat(Math.floor(n / 3)),
};
const ops = { lf: s => eol.lf(s), crlf: s => eol.crlf(s), match: s => eol.match(s).length, split: s => eol.split(s).length };
const sizes = [1e4, 1e5, 1e6, 1e7];
console.log("shape".padEnd(16), "op".padEnd(6), sizes.map(n => ("n=" + n.toExponential(0)).padStart(11)).join(""), "  ratio(1e7/1e6)");
for (const [sn, mk] of Object.entries(shapes)) for (const [on, op] of Object.entries(ops)) {
  const t = sizes.map(n => { const s = mk(n); op(s); const reps = n <= 1e5 ? 20 : n <= 1e6 ? 5 : 3; const ts = [];
    for (let i = 0; i < reps; i++) { const t0 = process.hrtime.bigint(); op(s); ts.push(Number(process.hrtime.bigint() - t0) / 1e6); }
    ts.sort((a, b) => a - b); return ts[Math.floor(ts.length / 2)]; });
  console.log(sn.padEnd(16), on.padEnd(6), t.map(x => (x.toFixed(2) + "ms").padStart(11)).join(""), ("  x" + (t[3] / t[2]).toFixed(1)));
}
```

Output:

```
shape            op          n=1e+4     n=1e+5     n=1e+6     n=1e+7   ratio(1e7/1e6)
a*n (no match)   lf          0.00ms     0.00ms     0.03ms     0.34ms   x10.1
a*n (no match)   crlf        0.00ms     0.00ms     0.04ms     0.34ms   x9.8
a*n (no match)   match       0.00ms     0.00ms     0.04ms     0.34ms   x9.7
a*n (no match)   split       0.00ms     0.00ms     0.03ms     0.34ms   x10.0
\r*n             lf          0.14ms     1.52ms    14.43ms   413.45ms   x28.6
\r*n             crlf        0.13ms     1.26ms    12.75ms   331.98ms   x26.0
\r*n             match       0.13ms     1.68ms    14.25ms   151.63ms   x10.6
\r*n             split       0.16ms     1.98ms    19.47ms   210.84ms   x10.8
\r\n*n/2         lf          0.06ms     0.63ms     6.43ms   171.70ms   x26.7
\r\n*n/2         crlf        0.06ms     0.63ms     6.33ms   167.55ms   x26.5
\r\n*n/2         match       0.07ms     0.72ms     7.65ms   190.01ms   x24.9
\r\n*n/2         split       0.09ms     0.96ms     9.40ms    91.19ms   x9.7
\n\r*n/2         lf          0.06ms     0.64ms     6.40ms   169.96ms   x26.5
\n\r*n/2         crlf        0.06ms     0.63ms     6.34ms   154.61ms   x24.4
\n\r*n/2         match       0.07ms     0.71ms     7.67ms   163.83ms   x21.4
\n\r*n/2         split       0.09ms     1.00ms     9.54ms    94.96ms   x10.0
(\r\r\n)*n/3     lf          0.08ms     0.84ms     8.48ms   226.06ms   x26.7
(\r\r\n)*n/3     crlf        0.08ms     0.84ms     8.59ms   225.41ms   x26.3
(\r\r\n)*n/3     match       0.09ms     1.01ms    10.17ms   178.75ms   x17.6
(\r\r\n)*n/3     split       0.12ms     1.22ms    12.94ms   134.02ms   x10.4
```
</details>

<details><summary>r09b_scaling.js (F8 scaling; run with --max-old-space-size=12000)</summary>

```js
// Scaling check: is the 1e6->1e7 jump quadratic or a constant-factor step? Doubling series.
const eol = require("/Users/ivanmilosavljevic/src/eol/eol.js");
const op = process.argv[2] || "lf";
for (let n = 1e6; n <= 1.28e8; n *= 2) {
  const s = "\r".repeat(n); eol[op](s);
  const ts = []; for (let i = 0; i < 3; i++) { const t0 = process.hrtime.bigint(); eol[op](s); ts.push(Number(process.hrtime.bigint() - t0) / 1e6); }
  ts.sort((a, b) => a - b);
  console.log(`${op} "\\r"x${n.toExponential(2)}: ${ts[1].toFixed(1)} ms  (${(ts[1] * 1e6 / n).toFixed(2)} ns/char)`);
}
```

Output:

```
lf "\r"x1.00e+6: 9.7 ms  (9.66 ns/char)
lf "\r"x2.00e+6: 18.0 ms  (9.02 ns/char)
lf "\r"x4.00e+6: 86.2 ms  (21.55 ns/char)
lf "\r"x8.00e+6: 225.5 ms  (28.19 ns/char)
lf "\r"x1.60e+7: 600.4 ms  (37.52 ns/char)
lf "\r"x3.20e+7: 1363.2 ms  (42.60 ns/char)
lf "\r"x6.40e+7: 3147.9 ms  (49.19 ns/char)
lf "\r"x1.28e+8: 8491.5 ms  (66.34 ns/char)
```
</details>

<details><summary>r10_perf.js (F8 throughput/memory; second run output)</summary>

```js
// Throughput + memory. Usage: node r10_perf.js <op> <MiB> <shape>
// shape "text": lines of 0..119 'x' chars, endings cycling \r\n,\n,\r ; shape "dense": all "\r\n"
const eol = require("/Users/ivanmilosavljevic/src/eol/eol.js");
const [op, mib, shape] = [process.argv[2], Number(process.argv[3]), process.argv[4] || "text"];
const target = mib * 1048576;
let s;
if (shape === "dense") s = "\r\n".repeat(target / 2);
else { const parts = []; let len = 0, i = 0, seed = 7; const ends = ["\r\n", "\n", "\r"];
  while (len < target) { seed = (seed * 1103515245 + 12345) >>> 0; const line = "x".repeat(seed % 120) + ends[i++ % 3]; parts.push(line); len += line.length; }
  s = parts.join("").slice(0, target); }
global.gc && global.gc();
const before = process.memoryUsage(), rss0 = process.resourceUsage().maxRSS;
const t0 = process.hrtime.bigint();
const r = eol[op](s);
const ms = Number(process.hrtime.bigint() - t0) / 1e6;
const after = process.memoryUsage();
const outLen = typeof r === "string" ? r.length : r.length;
console.log(JSON.stringify({ op, shape, inMiB: mib, ms: +ms.toFixed(1), MiBps: +(mib / (ms / 1000)).toFixed(0),
  out: (typeof r === "string" ? "string len " : "array len ") + outLen,
  heapDeltaMiB: +((after.heapUsed - before.heapUsed) / 1048576).toFixed(0),
  peakRssMiB: +(process.resourceUsage().maxRSS / 1024).toFixed(0), rssBeforeOpMiB: +(rss0 / 1024).toFixed(0) }));
```

Output:

```
{"op":"lf","shape":"text","inMiB":10,"ms":3.6,"MiBps":2780,"out":"string len 10426106","heapDeltaMiB":16,"peakRssMiB":114,"rssBeforeOpMiB":114}
{"op":"crlf","shape":"text","inMiB":10,"ms":3.8,"MiBps":2652,"out":"string len 10605067","heapDeltaMiB":16,"peakRssMiB":114,"rssBeforeOpMiB":114}
{"op":"cr","shape":"text","inMiB":10,"ms":3.8,"MiBps":2662,"out":"string len 10426106","heapDeltaMiB":16,"peakRssMiB":114,"rssBeforeOpMiB":114}
{"op":"auto","shape":"text","inMiB":10,"ms":3.6,"MiBps":2758,"out":"string len 10426106","heapDeltaMiB":16,"peakRssMiB":114,"rssBeforeOpMiB":114}
{"op":"match","shape":"text","inMiB":10,"ms":3.3,"MiBps":3022,"out":"array len 178961","heapDeltaMiB":8,"peakRssMiB":118,"rssBeforeOpMiB":114}
{"op":"split","shape":"text","inMiB":10,"ms":5,"MiBps":1982,"out":"array len 178962","heapDeltaMiB":12,"peakRssMiB":117,"rssBeforeOpMiB":114}
{"op":"before","shape":"text","inMiB":10,"ms":0,"MiBps":226203,"out":"string len 10485761","heapDeltaMiB":0,"peakRssMiB":114,"rssBeforeOpMiB":114}
{"op":"after","shape":"text","inMiB":10,"ms":0,"MiBps":247678,"out":"string len 10485761","heapDeltaMiB":0,"peakRssMiB":114,"rssBeforeOpMiB":114}
{"op":"lf","shape":"text","inMiB":100,"ms":185.9,"MiBps":538,"out":"string len 104261792","heapDeltaMiB":157,"peakRssMiB":700,"rssBeforeOpMiB":585}
{"op":"crlf","shape":"text","inMiB":100,"ms":145.1,"MiBps":689,"out":"string len 106049216","heapDeltaMiB":157,"peakRssMiB":700,"rssBeforeOpMiB":575}
{"op":"cr","shape":"text","inMiB":100,"ms":144.8,"MiBps":690,"out":"string len 104261792","heapDeltaMiB":157,"peakRssMiB":701,"rssBeforeOpMiB":575}
{"op":"auto","shape":"text","inMiB":100,"ms":146.7,"MiBps":682,"out":"string len 104261792","heapDeltaMiB":157,"peakRssMiB":700,"rssBeforeOpMiB":575}
{"op":"match","shape":"text","inMiB":100,"ms":33,"MiBps":3031,"out":"array len 1787424","heapDeltaMiB":69,"peakRssMiB":624,"rssBeforeOpMiB":585}
{"op":"split","shape":"text","inMiB":100,"ms":55.9,"MiBps":1787,"out":"array len 1787425","heapDeltaMiB":106,"peakRssMiB":614,"rssBeforeOpMiB":575}
{"op":"before","shape":"text","inMiB":100,"ms":0,"MiBps":2266289,"out":"string len 104857601","heapDeltaMiB":0,"peakRssMiB":575,"rssBeforeOpMiB":575}
{"op":"after","shape":"text","inMiB":100,"ms":0,"MiBps":2531646,"out":"string len 104857601","heapDeltaMiB":0,"peakRssMiB":586,"rssBeforeOpMiB":586}
{"op":"lf","shape":"dense","inMiB":10,"ms":176,"MiBps":57,"out":"string len 5242880","heapDeltaMiB":170,"peakRssMiB":317,"rssBeforeOpMiB":45}
{"op":"crlf","shape":"dense","inMiB":10,"ms":171.9,"MiBps":58,"out":"string len 10485760","heapDeltaMiB":170,"peakRssMiB":317,"rssBeforeOpMiB":45}
{"op":"cr","shape":"dense","inMiB":10,"ms":170.7,"MiBps":59,"out":"string len 5242880","heapDeltaMiB":170,"peakRssMiB":317,"rssBeforeOpMiB":45}
{"op":"auto","shape":"dense","inMiB":10,"ms":169.3,"MiBps":59,"out":"string len 5242880","heapDeltaMiB":170,"peakRssMiB":317,"rssBeforeOpMiB":45}
{"op":"match","shape":"dense","inMiB":10,"ms":190.3,"MiBps":53,"out":"array len 5242880","heapDeltaMiB":269,"peakRssMiB":416,"rssBeforeOpMiB":45}
{"op":"split","shape":"dense","inMiB":10,"ms":105.7,"MiBps":95,"out":"array len 5242881","heapDeltaMiB":128,"peakRssMiB":184,"rssBeforeOpMiB":45}
{"op":"before","shape":"dense","inMiB":10,"ms":0,"MiBps":666667,"out":"string len 10485761","heapDeltaMiB":0,"peakRssMiB":45,"rssBeforeOpMiB":45}
{"op":"after","shape":"dense","inMiB":10,"ms":0,"MiBps":930233,"out":"string len 10485761","heapDeltaMiB":0,"peakRssMiB":45,"rssBeforeOpMiB":45}
{"op":"lf","shape":"dense","inMiB":100,"ms":1975.9,"MiBps":51,"out":"string len 52428800","heapDeltaMiB":1700,"peakRssMiB":1574,"rssBeforeOpMiB":45}
{"op":"crlf","shape":"dense","inMiB":100,"ms":1814.1,"MiBps":55,"out":"string len 104857600","heapDeltaMiB":1700,"peakRssMiB":1850,"rssBeforeOpMiB":45}
{"op":"cr","shape":"dense","inMiB":100,"ms":1820,"MiBps":55,"out":"string len 52428800","heapDeltaMiB":1700,"peakRssMiB":1914,"rssBeforeOpMiB":45}
{"op":"auto","shape":"dense","inMiB":100,"ms":1799.2,"MiBps":56,"out":"string len 52428800","heapDeltaMiB":1700,"peakRssMiB":1914,"rssBeforeOpMiB":45}
{"op":"match","shape":"dense","inMiB":100,"ms":2379.5,"MiBps":42,"out":"array len 52428800","heapDeltaMiB":2587,"peakRssMiB":2293,"rssBeforeOpMiB":45}
{"op":"split","shape":"dense","inMiB":100,"ms":981.7,"MiBps":102,"out":"array len 52428801","heapDeltaMiB":1624,"peakRssMiB":1680,"rssBeforeOpMiB":45}
{"op":"before","shape":"dense","inMiB":100,"ms":0,"MiBps":7571169,"out":"string len 104857601","heapDeltaMiB":0,"peakRssMiB":45,"rssBeforeOpMiB":45}
{"op":"after","shape":"dense","inMiB":100,"ms":0,"MiBps":12766501,"out":"string len 104857601","heapDeltaMiB":0,"peakRssMiB":45,"rssBeforeOpMiB":45}
```
</details>

<details><summary>r13_coverage.js (F9 coverage)</summary>

```js
// Block/function coverage of eol.js from V8 precise coverage (NODE_V8_COVERAGE) while running test.js
const fs = require("fs"), path = require("path");
const dir = process.argv[2], target = "file:///Users/ivanmilosavljevic/src/eol/eol.js";
const src = fs.readFileSync("/Users/ivanmilosavljevic/src/eol/eol.js", "utf8");
const lineOf = off => src.slice(0, off).split("\n").length;
for (const f of fs.readdirSync(dir)) {
  const j = JSON.parse(fs.readFileSync(path.join(dir, f)));
  for (const s of j.result) if (s.url === target) {
    for (const fn of s.functions) for (const r of fn.ranges) if (r.count === 0)
      console.log(`UNCOVERED range in ${fn.functionName || "(anonymous)"}: lines ${lineOf(r.startOffset)}-${lineOf(r.endOffset)}: ${JSON.stringify(src.slice(r.startOffset, r.endOffset).trim().slice(0, 60))}`);
    console.log("functions:", s.functions.map(f => (f.functionName || "(anon)") + "=" + f.ranges[0].count).join(" "));
  }
}
```
</details>

<details><summary>r14_mutants.js (F9 mutation probe)</summary>

```js
// Mutation probe: apply single-line mutants to a COPY of eol.js and run the unmodified test.js against it.
const fs = require("fs"), path = require("path"), { spawnSync } = require("child_process");
const REPO = "/Users/ivanmilosavljevic/src/eol", W = path.join(__dirname, "mut");
fs.rmSync(W, { recursive: true, force: true }); fs.mkdirSync(W);
fs.copyFileSync(path.join(REPO, "test.js"), path.join(W, "test.js"));
fs.copyFileSync(path.join(REPO, "package.json"), path.join(W, "package.json"));
const orig = fs.readFileSync(path.join(REPO, "eol.js"), "utf8");
const mutants = [
  ["baseline (no mutation)", null, null],
  ["before() ignores platform: linebreak -> \"\\r\\n\"", "return linebreak + text", 'return "\\r\\n" + text'],
  ["after() ignores platform: linebreak -> \"\\r\\n\"", "return text + linebreak", 'return text + "\\r\\n"'],
  ["Windows detection disabled", '"win32" === process.platform', "false"],
  ["Windows detection inverted", '"win32" === process.platform', '"win32" !== process.platform'],
  ["regex also matches U+2028/U+2029/NEL", "/\\r\\n|\\r|\\n/g", "/\\r\\n|\\r|\\n|\\u2028|\\u2029|\\u0085/g"],
  ["dub(): replacement as function (disables $-patterns)", "text.replace(newline, to)", "text.replace(newline, function() { return to })"],
  ["normalizers coerce input with String()", "text.replace(newline, to)", "String(text).replace(newline, to)"],
  ["browser export branch broken", "else root[name] = make()", "else root[name] = null"],
  ["match() returns null when none", "text.match(newline) || []", "text.match(newline)"],
  ["split() drops empty lines", "text.split(newline)", "text.split(newline).filter(Boolean)"],
  ["lf -> \\r\\n", 'api["lf"] = dub("\\n")', 'api["lf"] = dub("\\r\\n")'],
];
for (const [name, from, to] of mutants) {
  let code = orig;
  if (from) { if (!code.includes(from)) { console.log("PATTERN NOT FOUND:", name); continue; } code = code.replace(from, to); }
  fs.writeFileSync(path.join(W, "eol.js"), code);
  const r = spawnSync(process.execPath, ["test.js"], { cwd: W, env: { ...process.env, NODE_PATH: path.join(REPO, "node_modules") }, encoding: "utf8" });
  const failed = (r.stderr.match(/^(?:Error|TypeError): .*$/m) || [""])[0];
  console.log((r.status === 0 ? "SURVIVED" : "killed  ").padEnd(9), name, r.status === 0 ? "" : "<- " + failed.trim());
}
```
</details>

<details><summary>ts/cases/join.ts (F5)</summary>

```ts
import eol = require("eol");
// README.md:69 documents: eol.split(text).join(eol.auto) // === eol.auto(text)
export const joined: string = eol.split("a\nb").join(eol.auto);
```
</details>

<details><summary>ts/cases/named_import.mts (F5)</summary>

```ts
// Accepted by the type checker, fails at runtime in Node ESM (see r07_esm_named.mjs)
import { lf } from "eol";
export const out: string = lf("a\r\nb");
```
</details>

<details><summary>ts/cases/default_import.ts (F5)</summary>

```ts
// README.md:9 documents: import eol from "eol"
import eol from "eol";
export const out: string = eol.lf("a\r\nb");
```
</details>

<details><summary>ts/cases/declfile.ts (F5)</summary>

```ts
/// <reference path="../node_modules/eol/eol.d.ts" />
export {};
```
</details>

