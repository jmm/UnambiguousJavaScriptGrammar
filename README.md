# Unambiguous JavaScript Grammar

| Status | DRAFT |
| --- | --- |
| Authors | [@jdalton](https://github.com/jdalton)  [@bmeck](https://github.com/bmeck) |
| Date | June 14, 2016 |

## TL;DR

* CJS and ES modules **just work** without new extensions, extra ceremony, or
  additional scaffolding
* Performance is generally on par or better than existing CJS module loading
* Performance is significantly improved for ES modules over transpilation workflows
* Change JS grammars for Script and Module to be unambiguous
* Determine grammar of `.js` files by parsing as one grammar and falling back to
  the other
* Introduce a field to `package.json` to provide a second entry point for Node
  versions that support ES modules

## Problem

The Script and Module goal of ECMA262 have a grammatical ambiguity where some
code can run in both goals, having the exact same source, but produce different
values. Unlike `"use strict"`, the signal to have a specific behavior is not in
the code, thus the code has a multitude of possible effects which are not
controlled by the programmer.

### Example

```js
function foo(value) {
  value = value || '';
  var args = [].slice.call(arguments);
  args.unshift(this);
  return args;
}
foo(null);
```

Differences:

 | script | module
--- | --- | --- | ---
variable scope of `foo` | global | local
`arguments` object | modified | unmodified
`this` binding of `foo` | global | `undefined`

Since there is no way in source text to enforce the goal with the current grammar;
this leads to the behavior of certain constructs being undefined by the programmer,
and defined by the host environment. In turn, existing code could be run in the
wrong goal and partially function, or function without errors but produce
incorrect values.

## ECMA262 Solution

Require a structure in the Module goal that does not parse in the Script goal.
Having this requirement would prevent any source text written for the Module
goal from being executed in the Script goal by removing the ambiguity at parse
time. This proposal is [on the agenda](https://github.com/tc39/agendas/blob/master/2016/07.md)
for the July 2016 ECMA TC39 meeting.

The proposal is to require that Module source text has at least one `import` or
`export` statement. This should feel natural to developers as most modules import
dependencies and/or export APIs. Modules that do not export anything should
explicitly specify an `export default null` to make their intentions clear.

### Script Example

```js
function foo(value) {
  value = value || '';
  var args = [].slice.call(arguments);
  args.unshift(this);
  return args;
}
foo(null);
```

 | script | module (cannot parse)
--- | --- | --- | ---
variable scope of `foo` | global | n/a
`arguments` object | modified | n/a
`this` binding of `foo` | global | n/a

### Module Example

```js
function foo(value) {
  value = value || '';
  var args = [].slice.call(arguments);
  args.unshift(this);
  return args;
}
foo(null);
export default null;
```

 | script (cannot parse) | module
--- | --- | --- | ---
variable scope of `foo` | n/a | local
`arguments` object | n/a | unmodified
`this` binding of `foo` | n/a | undefined

## Problem

Node currently requires a means for programmers to signal what goal their code
is written to run in.

After much research, leading solutions have either hefty ecosystem tolls,
ceremony, or scaffolding. They lack a way to define the intent of the source text
from the ECMA262 standard.

## Solution

Parse source text as either goal, and if there is a parse error that may allow
the other goal to parse, then parse as the other goal. After this, the goal is
known unambiguously and the environment can safely perform initialization without
the possibility of the source text being run in the wrong goal.

### Algorithm

Note: A host can choose either goal to parse first and, over time, may change
their preferred order. Feel free to swap the order of Script and Module in the
following steps.

1. Bootstrap for Script
2. Parse as Script
3. If Success, return
4. Bootstrap for Module
5. Parse as Module
6. If Success, return
7. Throw error

## Problem

Node needs a way for developers to ship both CJS and ES modules in a single package.

## Solution

Introduce a field to `package.json` mimicing
[`modules.root`](https://github.com/dherman/defense-of-dot-js/blob/master/proposal.md#transitioning-applications-and-large-packages) /
Document Base URI to provide a second entry point for Node versions that support
ES modules *(field name pending investigation)*. This would be introduced at the
same time as ES modules. Any Node version that supports this field would change
the path resolution upon packages to resolve relative to the path defined in this
field. Much like [multiarchitecture binaries](https://en.wikipedia.org/wiki/Fat_binary),
this enables packages to ship both CJS and ES codebases. This preserves their
structure for legacy support and uses `modules.root` for newer Node versions.

### Example

Given a directory structure of:

```
myapp
\- package.json
\- dist
  \- app.js # ES
\ app.js # CJS
```

And `package.json` containing:

```json
{
  "modules.root": "./dist",
  "main": "app.js"
}
```

```js
require('myapp');
// Load dist/app.js if Node supports ES modules
// Load app.js if Node supports only CJS modules
```

### Side effects of field

In the above example scenario:

```js
require('myapp/package.json');
// Fail to load if Node supports ES modules
// Load package.json if Node supports only CJS modules
```

This has the benefit of allowing packages, like `react`, to access internal
modules without exposing them to package consumers.

### Validation of field

The field must not be the values `node_modules`, or `./node_modules`; nor may
the field begin with `node_modules/`, `./node_modules/`, `../`, or `/`
otherwise it will throw. This is to explicitly show the intent of the field
relative to `package.json` and reserve other prefixes for future usage.

### `modules.root` mechanics

`modules.root` is applied upon encountering a package via `node_modules` paths, or if the final file resolves using a `package.json`. Intermediary paths are not checked.

```js
import 'foo/../..';
// Path normalizes to 'foo/../..'.
//
// This would resolve the directory containing the path of `foo` *without*
// realpathing. However, the '../' means `package.json` is not used.
//
// Given `abc/node_modules/foo` then `import 'foo/../..';` would get `abc`.
```

```js
import 'foo/node_modules/bar/baz/..';
// Normalizes to 'foo/node_modules/bar'
// Approximately ~= path.resolve(
//   './node_modules/foo',
//   foo['modules.root'],
//   'node_modules/bar'
// );
//
// This would use `foo/package.json` + `modules.root`.
// This would use `bar/package.json` + `modules.root`.
```

```js
import 'foo/node_modules/bar/node_modules/baz';
// This would use `foo/package.json` + `modules.root`.
// This would not use `bar/package.json`, so no `modules.root`.
// This would use `baz.package.json` + `modules.root`.
```

## Implementation

To improve performance, host environments may want to specify a goal to parse
first. This could take several forms:<br>
a command line flag, a manifest file, HTTP header, file extension, etc.

The recommendation for Node is that we store this in a cache on disk.

Both [@trevnorris](https://github.com/trevnorris) and [@indutny](https://github.com/indutny)
believe caching is doable. Caching removes the sting of parsing and can actually
improve on existing performance through techniques like bytecode caching. While
investigations are in their early stages, there is plenty of room for improvements
and optimizations in this space.

The workflow for loading files would look like:

1. Get path to load as `filename`
2. If cache has `filename` set `goal` from cached value
  1. Validate cache against file for `filename`
  2. If valid
    1. Return cache
3. Else
  1. Set `goal` to a preferred goal (Script for now since most modules are CJS)
4. Load file for `filename` as `source`
5. Bootstrap source for `goal` as `bootstrapped_source`
5. Parse `bootstrapped_source` using `goal` grammar
6. If success
  1. Cache and return results
7. Else
  1. Change `goal` to opposite grammar
8. Bootstrap source for `goal` as `bootstrapped_source`
9. Parse `bootstrapped_source` using `goal` grammar
10. If success
  1. Cache and return results
11. Throw error

## Tooling concerns

Some situations outside of Node do not have a JS parser (Bash programs, some
asset pipelines, etc.). These tools generally operate on files as opaque blobs,
or plain text files. These tools can use the noted methods to get information
about the grammar for a blob via means listed in [Implementation](#implementation).

## External Impact

 * Microsoft packaged web apps can benefit from unambiguous Script and Module
   goals. At the moment there is no way to detect the intended goal of JavaScript
   files so bytecode caches are generated for the Script goal and discarded if
   ES modules are encountered.

## Special Thanks

This proposal would not have been possible without the tireless effort, conviction,
and collaboration of<br>
[@bmeck](https://github.com/bmeck), [@dherman](https://github.com/dherman),
and [@wycats](https://github.com/wycats).

I am also grateful for the feedback from:<br>
 * [@AtOMiCNebula](https://github.com/AtOMiCNebula) (Microsoft Edge)
 * [@brendaneich](https://github.com/brendaneich) (TC39)
 * [@bterlson](https://github.com/bterlson) (Microsoft Chakra / TC39)
 * [@chrisdickinson](https://github.com/chrisdickinson) (Node)
 * [@hzoo](https://github.com/hzoo) (Babel)
 * [@indutny](https://github.com/indutny) (Node)
 * [@josh](https://github.com/josh) (Github / Sprockets)
 * [@leobalter](https://github.com/leobalter) (jQuery Foundation / TC39)
 * [@ljharb](https://github.com/ljharb) (TC39)
 * [@loganfsmyth](https://github.com/loganfsmyth) (Babel)
 * [@mathiasbynens](https://github.com/mathiasbynens) (Lodash)
 * [@rvagg](https://github.com/rvagg) (Node)
 * [@sheerun](https://github.com/sheerun) (Bower)
 * [@sindresorhus](https://github.com/sindresorhus) (AVA / Chalk / Yeoman)
 * [@travisleithead](https://github.com/travisleithead) (Microsoft Edge / W3C)
 * [@trevnorris](https://github.com/trevnorris) (Node)

Thank you! :heart:<br>
  &#8212; [@jdalton](https://github.com/jdalton)
