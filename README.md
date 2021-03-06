# File Specifier Resolution in Node.js

**Contributors**: Geoffrey Booth (@GeoffreyBooth), John-David Dalton (@jdalton), Jan Krems (@jkrems), Guy Bedford (@guybedford), Saleh Abdel Motaal (@SMotaal), Bradley Meck (@bmeck)

## TL;DR

In a hurry? Just read the [Proposal](#proposal) section.

## Motivating Examples

- A project where all JavaScript is ESM.

- A project where all source is a transpiled language such as TypeScript or CoffeeScript.

- A project where some source is ESM and some is CommonJS.

- A package that aims to be imported into either a Node.js or a browser environment, without requiring a build step.

## High Level Considerations

- The baseline behavior of relative imports should match a browser’s with a simple file server.

  This implies that `./x` will only ever import exactly the sibling file `"x"` without appending paths or extensions. `"x"` is never resolved to `x.mjs` or `x/index.mjs` (or the `.js` equivalents).

- As browsers support ESM in `import` statements of `.js` files, Node.js also needs to allow ESM in `import` statements of `.js` files.

  To be precise, browsers support ESM in files served via the MIME type `text/javascript`, which is the type associated with the `.js` extension and the MIME type served for `.js` files by all standard web servers.

  This is covered in summary in [nodejs/modules#149][nodejs/modules#149] with links to deeper discussions.

- Node also needs to allow ESM in `.js` files because transpiled languages such as CoffeeScript lack a way to use the file extension as a place to store metadata, the way `.mjs` does double duty both identifying the file as JavaScript and specifying an ESM parse goal.

  The only way CoffeeScript could do the same would be creating a new extension like `.mcoffee`, but this is impractical because of the scope of the ecosystem updates that would be required, with related packages like `gulp-coffee` and `coffee-loader` and so on needing updates.

  TypeScript has similar issues, though its situation is more complex because of its type definition files. This is covered in [nodejs/modules#150][nodejs/modules#150].

- Along with `.js` files needing to be able to contain ESM, they also still need to be able to contain CommonJS.

  We need to preserve CommonJS files’ ability to `require` CommonJS `.js` files, and ESM files need some way to import `.js` CommonJS files.

- The [package exports proposal][jkrems/proposal-pkg-exports] covers how Node should locate an ESM package’s entry point and how Node should locate deep imports of files inside ESM packages.

  File extensions (and filenames or paths) can be irrelevant for deep imports, allowing specifiers like `"lodash/forEach"` to resolve to a path like `./node_modules/lodash/collections/each.js` via the package exports map.

  That proposal covers how such files are _located,_ while this proposal discusses how such files are _parsed._ The two proposals are intended as complements for each other.

- This proposal only covers `import` statement specifiers; this doesn’t aim to also cover `--eval`, STDIN, command line flags, extensionless files or any of the other ways Node can import an entry point into a project or package.

  We intend to build on this proposal with a follow up to cover entry points.

## Real World Data

As part of preparing the package exports proposal, @GeoffreyBooth did [research][npm-packages-module-field-analysis] into public NPM registry packages using ESM syntax already, as identified by packages that define a `"module"` field in their `package.json` files. There are 941 such packages as of 2018-10-22.

A project was created with those packages `npm install`ed, creating a gigantic `node_modules` folder containing 96,923 JavaScript (`.js` or `.mjs`) files.

Code was then written to parse all of those JavaScript files with `acorn` and look for `import` or `export` declarations, and inspect the specifiers used in the `import` or `export ... from` statements. The [code for this][esm-npm-modules-research-code] is in this repo.

### Findings

#### &#x2002; 5,870 `import` statements imported ESM modules<sup>⑴</sup>, as bare specifiers, e.g. `import 'esm-module'`

#### &#x200a; 25,763 `import` statements imported CommonJS modules<sup>⑵</sup>, as bare specifiers, e.g. `import 'cjs-module'`

#### &#x2002; 1,564 `import` statements imported ESM files within packages e.g. `import 'esm-module/file'`

#### &#x2002; 8,140 `import` statements imported CommonJS files within packages, e.g. `import 'cjs-module/file'`

#### &#x200a; 86,001 `import` statements imported relative ESM JavaScript files<sup>⑶</sup>, e.g. `import './esm-file.mjs'`

#### &#x2002; 4,229 `import` statements imported relative CommonJS JavaScript files<sup>⑷</sup>, e.g. `import './cjs-file.js'`

<pre>
 ⑴  packages with a <samp>"module"</samp> field in their package.json.
 ⑵  packages lacking a <samp>"module"</samp> field in their package.json.
 ⑶  files with an <samp>import</samp> or <samp>export</samp> declaration.
 ⑷  files with a <samp>require</samp> call or references to <samp>module.exports</samp>, <samp>exports</samp>, <samp>__filename</samp>, or <samp>__dirname</samp>.
</pre>

## A Note on Defaults

This proposal takes the position that `.js` should be treated as ESM by default within an ESM context. This differs from the default behavior of the current `--experimental-modules` implementation which treats `.js` files to be CommonJS sources and `.mjs` to be ESM.

The rationale behind this position is to move towards directions that can:

1. Improve interoperability with browsers where file extension does not affect how they interpret and load a JavaScript source.

2. Be forward-looking in that ESM is the standard and should therefore be the default behavior within ESM files, rather than something to be opted into.

As of this writing, there is no way to modify Node’s default behavior and affect if and when files with a `.js` (or any other) extension should be treated as ESM instead of CommonJS, or other source types, without having to use a special loader (eg `--loader` with `--experimental-modules` for the time being).

Two proposals (at least) were made to try to address this specifically through declarative fields in the `package.json`, affecting the handling of files within the scope of their respective package:

1. **[`"mode"`][nodejs/node/pull/18392]** proposes a `"mode": "esm"` field to force Node to treat all `.js` files as ESM sources.

2. **[`"mimes"`][nodejs/modules#160]** proposes a `"mimes": { … }` block which defines fine-grained mappings for any extension.

The data shows that `import` statements of CommonJS `.js` files appear to be far less popular comapred to imports of ESM `.js` files, which are 19 times more common. From this, we can make an assumption that users in general may be more inclined to “intuitively” prefer `import` statements of `.js` files to be used to import from ESM sources over CommonJS ones. However, it is also the position of the authors that the `.mjs` file extension should retain it's current connotation to be by default always treated as an ESM source, unless otherwise reconfigured.

## Proposal

There are (at least) two parts to module resolution: _location_ and _interpretation._ Location is covered by the [resolver specification][nodejs/ecmascript-modules:esm.md#resolver-algorithm], and involves things like taking the specifier string `'underscore'` and finding its package entry point file `./node_modules/underscore/underscore.js`.

This proposal covers only the interpretation, or what Node should do once the file is found. For our purposes, interpretation means whether Node should load the package or file as ESM or as CommonJS.

### Parsing files as ESM or as CommonJS

There are four types of `import` statement specifiers:

- _Bare specifiers_ like `'lodash'`

  > They refer to an entry point of a package by the package name.

- _Deep import specifiers_ like `'lodash/lib/shuffle.mjs'`

  > They refer to a file within a package prefixed by the package name.

- _Relative file specifiers_ like `'./startup.js'` or `'../config.mjs'`

  > They refer to a file relative to the location of the importing file.

- _Absolute URL file specifiers_ like `'file:///opt/nodejs/config.js'`

  > They refer directly and explicity to a file by it's location.

#### Procedure

In all cases, first Node follows its algorithm to locate a file to load. Once the file is found, Node must then decide whether to load it as ESM or as CommonJS. The algorithm goes as follows:

```
If the file is a package entry point
    And the package’s package.json is detected to be ESM
        Load the file as ESM.
    Else
        Load the file as CommonJS.
Else
    If there is a package.json in the folder where the file is
        And the package.json is detected to be ESM
            Load the file as ESM.
        Else
            Load the file as CommonJS.
    Else
        Go into the parent folder and look for a package.json there
        Repeat until we either find a package.json or hit the file system root
        If we found a package.json
            And the package.json is detected to be ESM
                Load the file as ESM.
            Else
                Load the file as CommonJS.
        Else we reach the file system root without finding a package.json
            Load the file as ESM.
```

A `package.json` file is detected as ESM if it contains a key that signifies ESM support, such as the `"exports"` field from the [package exports proposal][jkrems/proposal-pkg-exports] or another ESM-signifying field like [`"mode"`][nodejs/node/pull/18392]

The folder containing the located `package.json` and its subfolders are the _package scope,_ and the parent folder is on the other side of a _package boundary._ There can be multiple `package.json` files in a path, creating multiple package boundaries.

#### Example

<!--
```
/usr/src/app/package.json - contains "exports" field, starts ESM package scope
/usr/src/app/index.js - parsed as ESM
/usr/src/app/startup/init.js - parsed as ESM

/usr/src/app/node_modules/sinon/package.json - contains "exports" with
                                               {"": "./dist/index.mjs",
                                                "/stub": "./dist/stub/index.mjs"}
/usr/src/app/node_modules/sinon/dist/index.mjs - parsed as ESM
/usr/src/app/node_modules/sinon/dist/stub/index.mjs - parsed as ESM

/usr/src/app/node_modules/sinon/node_modules/underscore/package.json - no "exports", starts CJS scope
/usr/src/app/node_modules/sinon/node_modules/underscore/underscore.js - parsed as CommonJS

/usr/src/app/node_modules/request/package.json - no "exports", starts CJS package scope
/usr/src/app/node_modules/request/index.js - parsed as CommonJS
/usr/src/app/node_modules/request/lib/cookies.js - parsed as CommonJS
```
-->

```
 ├─ /usr/src/app/                        <- ESM package scope
 │    package.json {                        created by package.json with "exports" field
 │      "exports": { ... }
 │    }
 │
 ├─ index.js                             <- parsed as ESM
 │
 ├─┬─ startup/
 │ │
 │ └─ init.js                            <- parsed as ESM
 │
 └─┬ node_modules/
   │
   ├─┬─ sinon/                           <- ESM package scope
   │ │    package.json {                    created by package.json with "exports" field
   │ │      "exports": {
   │ │        "": "./dist/index.mjs",
   │ │        "/stub": "./dist/stub/index.mjs"
   │ │      }
   │ │    }
   │ │
   │ ├─┬─ dist/
   │ │ │
   │ │ ├─ index.mjs                      <- parsed as ESM
   │ │ │
   │ │ └─┬─ stub/
   │ │   │
   │ │   └─ index.mjs                    <- parsed as ESM
   │ │
   │ └─┬ node_modules/
   │   │
   │   └─┬ underscore/                   <- CommonJS package scope
   │     │   package.json {                 created by package.json with no "exports" field
   │     │     "main": "underscore.js"
   │     │   }
   │     │
   │     └─ underscore.js                <- parsed as CommonJS
   │
   └─┬ request/                          <- CommonJS package scope
     │   package.json {                     created by package.json with no "exports" field
     │     "main": "index.js"
     │   }
     │
     ├─ index.js                         <- parsed as CommonJS
     │
     └─┬─ lib/
       │
       └─ cookies.js                     <- parsed as CommonJS
```

The following `import` statements from the above `/usr/src/app/index.js` would parse as follows:

```js
// Package entry points
import sinon from 'sinon'; // ESM
import request from 'request'; // CommonJS

// Deep imports
import stub from 'sinon/stub'; // ESM
import cookies from 'request/lib/cookies.js'; // CommonJS

// File specifiers: relative
import './startup/init.js'; // ESM
import cookies from './node_modules/request/lib/cookies.js'; // CommonJS
import stub from './node_modules/sinon/dist/stub/index.mjs'; // ESM
import _ from './node_modules/sinon/node_modules/underscore/underscore.js'; // CommonJS

// File specifiers: absolute
import 'file:///usr/src/app/startup/init.js'; // ESM
import cookies from 'file:///usr/src/app/node_modules/request/lib/cookies.js'; // CommonJS
```

File extensions are still relevant. While either a `.js` or an `.mjs` file can be loaded as ESM, only `.js` files can be loaded as CommonJS. If the above example’s `cookies.js` was renamed `cookies.mjs`, the theoretical `import cookies from 'request/lib/cookies.mjs'` would throw.

The CommonJS automatic file extension resolution or folder `index.js` discovery are not supported for `import` statements, even when referencing files inside CommonJS packages. Both `import cookies from 'request/lib/cookies'` and `import request from './node_modules/request'` would throw. Automatic file extension resolution or folder `index.js` discovery _are_ still supported for `package.json` `"main"` field specifiers, however, to preserve backward compatibility.

#### Important Notes

- **Specifiers starting with `/` or `//`**

  These are currently unsupported but reserved for future use. Browsers support specifiers like `'/app.js'` to be relative to the base URL of the page, whereas in CommonJS a specifier starting with `/` refers to the root of the file system. We would like to find a solution that conforms Node closer to browsers for `/`-leading specifiers. That may not necessarily involve a leading `/`, for example if Node adopts the [`import:` proposal][import-urls] or something like it; but we would like to preserve design space for a future way to conveniently refer to the root of the current package.

- **`createRequireFromPath` in ESM**

  Packages or files located from `require` calls created via [`module.createRequireFromPath`][nodejs-docs-modules-create-require-from-path] are _always_ parsed as CommonJS, following how `require` behaves now.

- **“Dual mode” packages**

  A package can be “dual mode” if its `package.json` contains both a `"main"` field and an `"exports"` field (or some other ESM-signifying field). An `import` statement of such a package will treat the package as ESM and ignore the `"main"` field. To explicitly import a dual-mode package via its CommonJS entry point, [`module.createRequireFromPath`][nodejs-docs-modules-create-require-from-path] could be used.

  The ESM and CommonJS versions of a dual-mode package are really two distinct packages, and would be treated as such by Node if both were imported. It is an implementation detail to be worked out what Node should do in such a situation. Of particular concern are race conditions that could occur if it might be unpredictable which version (ESM or CommonJS) of a dual-mode package gets loaded first.

  Different modes of the same package _should_ be importable into different package scopes, for example if a user’s project imports ESM `lodash` and that project has a dependency which itself imports CommonJS `lodash`. This corresponds with how different package scopes today can import different versions of the same package, such a user’s project having dependencies of `lodash@2` and `request`, with `request` then having a dependency of `lodash@1`, and both the user’s project and `request` each receive the version of `lodash` that they expect.

#### Further Considerations

<details><summary>“Double importing” of files</summary>

There is the possibility of `import` and `createRequireFromPath` both importing the same file into the same package scope, potentially the former as ESM and the latter as CommonJS. Allowing this would likely cause issues, and a solution would need to be worked out to handle this situation.

</details>

<details><summary>“Loose” CommonJS files (files outside of packages)</summary>

Currently, `module.createRequireFromPath` can be used to import CommonJS files that aren’t inside a CommonJS package scope. To import the file via an `import` statement, a symlink could also be created from inside a CommonJS package scope to the desired “loose” CommonJS file, or the file could simply be moved inside a CommonJS package scope. Seeing as there is low user demand for ESM files importing CommonJS files outside of CommonJS packages, we feel that these options are sufficient.

</details>

<details><summary>CommonJS files inside an ESM package scope</summary>

One of our use cases is a package in transition, for example that’s migrating from CommonJS to ESM but the migration is not yet complete. This raises the issue of both parse goals (ESM and CommonJS) existing within an ESM package scope; but `import` statements will always treat JavaScript files within such a scope as ESM.

One way to import such CommonJS files is to use `module.createRequireFromPath`; another would be to move the CommonJS files into or under a folder with an empty `package.json` file, which would create a CommonJS package scope. `import` statements of symlinks inside a CommonJS package scope could also be used.

Of course, perhaps the most straightforward solution would be for users transitioning a package to simply transpile with a tool like Babel, like they do now, until the migration is complete.

We feel that these options are sufficient for now, but if user demand grows such that we want to provide a way to use `import` statements to import CommonJS files that are in an ESM package scope, we have a few options:

1. We could introduce a `.cjs` extension that Node always interprets as JavaScript with a CommonJS parse goal, the mirror of `.mjs`. (This might be a good thing to add in any case, for design symmetry.) Users could then rename their CommonJS `.js` files to use `.cjs` extensions and import them via `import` statements. We could also support symlinks, so that a `foo.cjs` symlink pointing at `foo.js` would be treated as CommonJS when imported via `import './foo.cjs';`, to support cases where users can’t rename their files for whatever reason.

2. We could implement the `"mimes"` proposal from [nodejs/modules#160][nodejs/modules#160], which lets users control how Node treats various file extensions within a package scope. This would let users create a configuration to tell Node to treat certain extensions as ESM and others as CommonJS, for example the `--experimental-modules` pattern of `.mjs` as ESM and `.js` as CommonJS.

3. Presumably loaders would be able to enable this functionality, deciding to treat a file as CommonJS either based on file extension or some detection inside the file source.

4. We could create some other form of configuration to enable this, like a section in `package.json` that explicitly lists files to be loaded as CommonJS.

Again, we think that user demand for this use case is so low as to not warrant supporting it any more conveniently for now, especially since there are several other potential solutions that remain possible in the future within the design space of this proposal.

</details>

<details><summary>CommonJS files importing ESM</summary>

CommonJS import of ESM packages or files is outside the scope of this proposal. We presume it will be enabled via `import()`, where any specifier inside `import()` is treated like an ESM `import` statement specifier. We assume that CommonJS `require` of ESM will never be natively supported.

</details>

<details><summary>Constancy expectations when loading files</summary>

Files are read as part of the module loading process: source code files and the `package.json` files used to locate those source files or determine those source files’ parse goals. Once a file is loaded for a particular resolved URL, or a `package.json` is read as part of resolving that URL, those files are not read again. If the source file or `package.json` changes on disk, or the virtual representation of the file on disk changes, Node is unaware of the change and will continue to use the cached versions of the files as they existed when Node first read them.

</details>

## Prior Art

- [Package exports proposal][jkrems/proposal-pkg-exports]
- [`"mimes"` field proposal][nodejs/modules#160]
- [Import Maps Proposal][domenic/import-maps]
- [Node.js ESM Resolver Specification][nodejs/ecmascript-modules/pull/12]
- [HTML spec for resolving module specifiers][whatwg-js-module-integration]

[esm-npm-modules-research-code]: ./esm-npm-modules-research 'The code to scrub packages (local)'

[jkrems/proposal-pkg-exports]: https://github.com/jkrems/proposal-pkg-exports '[Krems et al] Package Exports Proposal (repo)'

[nodejs/modules#160]: https://github.com/nodejs/modules/pull/160 "[Booth] \"mimes\" Field Proposal #160 (pr)"

[nodejs/modules#149]: https://github.com/nodejs/modules/issues/149 '[Booth] Web compatibility and ESM in .js files #149 (discussion)'

[nodejs/modules#150]: https://github.com/nodejs/modules/pull/150 '[Booth] ESM in .js files proposals #150 (pr)'

[domenic/import-maps]: https://github.com/domenic/import-maps '[Denicola] Import Maps Proposal (repo)'

[import-urls]: https://github.com/domenic/import-maps#import-urls '[Denicola] Import Maps Proposal section on `import:` URLs'

[nodejs-docs-modules-create-require-from-path]: https://nodejs.org/docs/latest/api/modules.html#modules_module_createrequirefrompath_filename '[Node.js] Documentation - Modules - createRequireFromPath (doc)'

[nodejs/ecmascript-modules:esm.md]: https://github.com/nodejs/ecmascript-modules/blob/esm-resolver-spec/doc/api/esm.md '[Node.js] Documentation - ECMAScript Modules (doc)'

[nodejs/ecmascript-modules:esm.md#resolver-algorithm]: https://github.com/nodejs/ecmascript-modules/blob/esm-resolver-spec/doc/api/esm.md#resolver-algorithm '[Node.js] Documentation - ECMAScript Modules - Resolver Algorithm (doc)'

[nodejs/ecmascript-modules/pull/12]: https://github.com/nodejs/ecmascript-modules/pull/12 '[Bedford] ESM Resolver Specification #12 (pr)'

[whatwg-js-module-integration]: https://html.spec.whatwg.org/multipage/webappapis.html#integration-with-the-javascript-module-system '[WHATWG] HTML - 8.1.3.8. Integration with the JavaScript module system'

[npm-packages-module-field-analysis]: https://gist.github.com/GeoffreyBooth/1b0d7a06bae52d124ace313634cb2f4a '[Booth] Analysis of public NPM packages using the“module”field (gist)'

[nodejs/node/pull/18392]: https://github.com/nodejs/node/pull/18392 '[Bedford] ESM: Implement esm mode flag #18392 (pr)'