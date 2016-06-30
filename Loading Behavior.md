# Loading Behavior for CommonJS and ES Modules

The TypeScript team has looked at various different facets of the module loading interop between CommonJS and ES.
Our perspective is shaped by the following needs:

* Long term compatibility between the existing *and future* Node ecosystem.
* Ease of migration to ES-style modules.
* The ability for transpilers to be consumed properly from ES and CommonJS modules.

## No property plucking

The current proposal set forward states that a CommonJS module is primarily made available as a default import.
The proposal then further has the notion of property plucking, where properties on the default import are also made available as named imports.
This process is also called "hoisting".

This practice is likely to have certain negative consequences.
One major issue is that it is not clear what host object a "plucked" named import is bound to (i.e. it isn't clear what the `this` value is).
It is also not clear how get-accessors are treated under this system.

A more tangible issue for Node users is that this makes it difficult for library authors from migrate to ES modules, because naively doing so would cause breaks for ES consumers.
For instance, consider the following file `foo.js`

```ts
module.exports = function() {
    // ...
};

module.exports.bar = "hello";
```

Imagine a consumer that is written using ES modules:

```ts
import f, { bar } from "./foo.js";

// 'f' is callable.
f();

// 'f' has a member named 'bar'.
f.bar.toLowerCase();

// We can also use 'bar' as a named import.
bar.toLowerCase();
```

Notice that because of property plucking from the `default`, `bar` was accessible as a named import as well.

Now when `foo.js` wants to migrate to ES module syntax, the author would likely write something like the following:

```ts
export default function() {
    // ...
};

export var bar = "hello";
```

However, breaks the usage of `f.bar.toLowerCase()` in the above example!
Another naive fix might have been the following:

```ts
function d() {
    // ...
};
d.bar = "hello";

export default d;
```

However, this breaks the usage of `bar` as a named import!
Instead, the library author must re-export each member of their `default` export to maintain compatibility with ES consumers.
This is a strong disincentive for moving to ES modules.

We believe that by default, a CommonJS module should only be made available using a default import.

## Transpilation Support

Interop can work well enough natively between ES modules and any existing module system if it plans for it.
That is, we can plan out the interop behavior between CommonJS and ES modules for the future, but that means that there is still a gap for people on older versions of Node.
Transpilers like TypeScript and Babel fill in that gap so that users can still author in ES but target older versions of Node.

If CommonJS modules are only brought in as a default, it becomes impossible to define named exports for ES consumers.
Modules *also* need to be able to affect the shape of the namespace import.

One way to enable this is to "pluck" properties as above, however:

* As mentioned above, this makes it difficult to upgrade to ES modules.
* Also as mentioned above, there are various complications with orphaning methods and accessors.
* It makes it complicated for tools to make a `default` export available because named exports potentially need to be exposed as properties on the default object, making certain types of modules impossible to write.

The `__esModule` property is something that both Babel and TypeScript emit in some capacity today, and is also recognized in SystemJS.
The necessity was recognized by [Guy Bedford in 2013](https://github.com/esnext/es6-module-transpiler/issues/86) in his work on es6-module-transpiler.
Basically what it boils down to is that CommonJS modules need to be able to dictate whether their shape should describe a default import or the namespace import.

In the case that an `__esModule` property is present on the `module.exports` object, it should act as a signal to the loader that the value of `module.exports` describes the namespace import.

For instance:

```ts
// CJS library a.js
module.exports.greeting = "hello!";

// CJS library b.js
module.exports.farewell = "hello!";
module.exports.__esModule

//
// ES consumer:
//
import a from "./a.js";
import * as b from "./b.js";

// 'greeting' is accessible on the default import.
a.greeting.toLowerCase();

// 'farewell' is accessible on the namespace import.
b.farewell.toUpperCase();
```

## Default Substitution for `require`

We mentioned above that authors are likely to convert `module.exports = ...` to `export default ...`.
Problematically, this means CommonJS consumers are immediately broken if `default` exports are only accessible through `require(...).defualt`.

As a fix to avoid breaking CommonJS consumers, `require` should adopt two steps prior to returning:

1. If the `require`'d module is an ES module, and a property named `__esModule` is not exposed on the result, a non-enumerable property of that name is added and set to `true`.
2. If `__esModule` is present and the only other property on the result is named `default`, then the value of `default` is returned instead.

This means that for the following ES module

```ts
export default function(a, b, c) {
    // ...
};
```

you may import it as follows:

```js
var foo = require("./foo");
foo(1, 2, 3);
```

This makes compatibility easy for users on the current runtime.
It also makes it possible to patch up the behavior of old runtimes to allow transpiled modules to work the same.

## Summary

1. If an ES Module is `require(...)`'d, then it automatically gets a non-enumerable property named `__esModule`.
2. If a CommonJS module is imported by an ES module, then
    * If the CJS module has a `__esModule` property on its `module.exports` object, that `module.exports` object is used in place of the namespace export.
    * Otherwise, a CJS module is *only* made available as a `default` import. No named properties are made available.
3. If the result of a call to `require(...)` has only a property named `default` as well as a property named `__esModule`, then `default` is used in place of the original result.