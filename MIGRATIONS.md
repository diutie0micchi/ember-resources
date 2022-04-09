# Migrations

- [5.0](#5.0)
  - [tl;dr](#tldr)
  - [Overview](#overview)
  - [Nomenclature changes](#nomenclature-changes)
  - [LifecycleResource](#lifecycleresource)
  - [Resource](#resource)
    - [args](#args)
    - [setup](#setup)
    - [update](#update)
    - [teardown](#teardown)
  - [Utilities](#utilities)
    - [useTask](#usetask)
    - [useFunction](#usefunction)
    - [useHelper](#usehelper)
    - [useResource](#useresource);
  - [References](#references)


## 5.0

### tl;dr:

_Migration during the v4 series is available via different imports_.

**Upcoming breaking changes**

- No more `ember-concurrency@v1` support (though compatibility may still work)
- Removed exports:
  - `LifecycleResource`
  - constructor-oriented `Resource`
  - `@use`
- Renamed utilities:
  - `useResource` => `Resource.of`
  - `useHelper` => `helper`
  - `useTask` => `task` with alias `trackedTask`
- Changed behavior:
  - `trackedFunction`
    - no longer receives the previous value
    - will return `null` instead of `undefined` before resolving
    - no longer holds on to the previous return value when re-running

**New features**

- opt-in svelte-able imports, but lazy tree-shakable imports still available (import everything from `'ember-resources'`)
- `Array.prototype.map` as a resource
- new `Resource` class with sole `modify` hook
- `trackedFunction` now wraps [ember-async-data][e-async-data] for a richer way to inspect pending and error state


[e-async-data]: https://github.com/chriskrycho/ember-async-data

-----------------------------------

### Overview


Migrating to 5.0 requires some adjustments to how folks author Resources
 - `LifecycleResource` will be renamed to `Resource` and there will be a single `modify` hook
 - `Resource` will be removed
 - to opt-in to non-deprecated behaviors, there will be new import paths to use. Once 5.0 hits,
   the current top-level imports will re-export the classes and utilities from the new paths
   introduced in as a part of this migration effort (for convenience, totally optional)

For library authors wanting to implement these changes, they can _probably_ be done in a minor release,
as the reactivity and general APIs behave the same -- however, if there are any **potentially** breaking
changes in any of the APIs, they'll be called out below.


Primary goals of this migration:
- to align with the broader ecosystem -- specifically [ember-modifier](https://github.com/ember-modifier/ember-modifier), and simplifying class-based APIs
- improving semantics and nomenclature for resources, i.e.: not relying on other ecosystem's nomenclature for describing the utility APIs (e.g.: `use*`)
- provide an easy module-svelting approach for folks not yet using tree-shaking, but don't want every utility in the `ember-resources` package (i.e.: if you don't use it, you don't pay for it)

### Nomenclature changes

_`use*` is now either `*Of` or dropped entirely_

The reason for this is that the "useThing" isn't descriptive of what behavior is actually happening.
In many cases, folks are using resources to mean "a class that participates in auto-tracking" and while
there may be lifecycle-esque behaviors involved, depending on which implementation is in use, those are
ultimately an implementation detail of the resource author.

_Using a class that participates in autotracking_ may be as simple as adding something like this
in your component:

```js
@tracked foo;
@tracked bar;

@cached
get selection() {
  return new Selection(this.foo, this.bar);
}
```
Alternatively, because the above will create a new instance of `Selection` every time `this.foo` or
`this.bar` changes, you may want to individually reactive arguments to `Selection` so that
the initial returned instance of `Selection` is stable.
```js
@tracked foo;
@tracked bar;

@cached
get selection() {
  return new Selection({
    foo: () => this.foo,
    bar: () => this.bar,
  });
}
```
depending on your performance requirements, the above pattern can be very uplifting when you need
to write vanilla JS, have encapsulated state, and auto-tracked derived data within that encapsulated state.


**But back to "use"**, all of this is _using_ `Selection` -- and with the v4 and earlier APIs of `ember-resources`,
the correlating usage would be:
```js
selection = useResource(this, Selection, () => { /* ... */ });
```
or, following the "provide a single import to your consumers recommendation",
```js
selection = useSelection(this, { /* ... */ });
```

As a library author, you want APIs to be as straight-forward as possible, meeting people where their mental
models are at, without any extra noise -- this may be a provided API that _avoids_ `use`
```js
selection = someClass(this, { /* ... */ });
```

**Why "of"?**

`of` is already common nomenclature in JavaScript.
While `of` exists as a keyword in [`for ... of`][for-of] and [`for await ... of`][for-await-of],
the usage that is most similar to the changes proposed for `ember-resources`
v5 (introduced in a v4 minor) is [`Array.of`][array-of] and [`TypedArray.of`][typed-array-of].
For both `Array` and `TypedArray`, the `of` word, in-spirit, means "Create an instance (of myself) of this configuration".

For `Resource.of`, it's the same, "Create an instance of a 'Resource' with this configuration".

**Why "from"?

`from` is also common nomenclature in JavaScript.
The usage in JavaScript that is most similar to the changse proposed for `ember-resources`
v5 (introduced in a v4 minor) is [`Array.from`][array-from] and [`TypedArray.from`][typed-array-from].

This is very similar to `of`, except that for a `Resource`, we do not
need to pass the class that we want to instantiate a resource for.

For example, the difference:
```js
class Foo {
  a = Resource.of(this, SomeResource, () => {/* ... */});
  a = SomeResource.from(this, () => {/* ... */});
}
```




[array-of]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/of
[for-of]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for...of
[for-await-of]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for-await...of
[typed-array-of]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray/of
[array-from]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/from
[object-fromEntries]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/fromEntries
[typed-array-from]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray/from

**Why omit any specifier?**

Consumers of your library do not need to and should not need to care about the specifics of how a `Resource`
is constructed.

For example, you're maybe providing a `Selection` Resource, a user will grok
`mySelection = selection(this, { /* ... */ })` much more easily than anything with additional words.
The omission of extra words is important, because it's less things to explain.
The lazy alternative may be `mySelection = Resource.of(this, Selection, () => { /* ... */ })`; multiple imports, a class, what's a Resource?, etc. Consumers of your library shouldn't need to know the specifics
of the implementation (the fact that resources are even a thing).

### LifecycleResource

The `LifecycleResource` is no more, but there was great value in having a way to hook in to when args change.
The new `Resource` preserves that value, while simplifying the overall API of the class.

#### `args`

The new `modify()` lifecycle hook receives the positional and named arguments to the modifier as its first and second parameters.
Previously, these were available as `this.args.positional` and `this.args.named` respectively,
and became available to use in that position after calling `super(owner, args)` in the constructor.
Now, the args are always available in the `modify()` hook directly.

This change helps alleviate issues with needing to compare previous/next args, due to how the args' containing object from the framework is the same between updates.

Before

```js
import { LifecycleResource } from 'ember-resources';

class MyResource extends LifecycleResource {
  get someNamedArg() {
    // No way to get previous?
    return this.args.named.someNamedArg;
  }
}
```

After
```js
import { Resource } from 'ember-resources/core';

class MyResource extends Resource {
  modify(positional, { someNamedArg }) {
    // Update local property only when the *value* differs.
    if (this.someNamedArg !== someNamedArg) {
      this.someNamedArg = someNamedArg;
    }
  }
}
```

#### `setup()`

`modify()` is called on initial setup and subsequent updates. If what you need to do is cheap, you can let it happen each update.
If it is expensive, or if the operation is not [idempotent][idempotent], you can set a flag to avoid doing it again.

[idempotent]: https://en.wikipedia.org/wiki/Idempotence

Before
```js
import { LifecycleResource } from 'ember-resources';

class MyResource extends LifecycleResource {
  setup() {
    // do some expensive thing
  }
}
```

After
```js
import { Resource } from 'ember-resources/core';

class MyResource extends Resource {
  didSetup = false;

  modify() {
    if (!didSetup) {
      // do some expensive thing
    }
  }
}
```

#### `update()`

`modify()` is called on initial setup and subsequent updates. If what you need to do is cheap, you can let it happen each update.

Before
```js
import { LifecycleResource } from 'ember-resources';

class MyResource extends LifecycleResource {
  update() {
    // do some updating
    // this.args is always "current"
  }
}
```

After

```js
import { Resource } from 'ember-resources/core';

class MyResource extends Resource {
  modify(positional, named) {
    // do some updating

    if (this.old !== this.positional[0]) {
      // only do some update when a value changes
    }
  }
}
```


#### `teardown()`

Since ember-source@3.22, we no longer need to have teardown hooks implemented.
the [`@ember/destroyable`][e-destroybale] APIs allow us to consistently have
destruction / cleanup behavior on any class/object.

[e-destroyable]: https://api.emberjs.com/ember/release/modules/@ember%2Fdestroyable

Before
```js
import { LifecycleResource } from 'ember-resources';

class MyResource extends LifecycleResource {
  update() {}
  setup() {}
  teardown() {}
}
```
After
```js
import { Resource } from 'ember-resources/core';
import { registerDestructor } from '@ember/destroyable';

class MyResource extends Resource {
  constructor(owner, args) {
    super(owner, args);

    registerDestructor(this, () => {
      // cleanup
    });
  }

  modify(positionalArgs, namedArgs) {
    // update
  }
}
```

### Resource

Previously, the `Resource` crammed too much responsibility into the `constructor`,
which lead to some confusion aronud how to do the most basic of behaviors.
(This is a fault of the design, not the users).
Additionally, the old `Resource` had no way to have a final teardown.

Before
```js
import { Resource } from 'ember-resources';
import { registerDestructor } from '@ember/destroyable';

class MyResource extends Resource {
  constructor(owner, args, previous) {
    super(owner, args, previous);

    if (!previous) {
      // initial setup
    } else {
      // update
    }

    registerDestructor(this, () => {
      // teardown function for each instance
    });
  }
}
```

After
```js
import { Resource } from 'ember-resources/core';
import { registerDestructor } from '@ember/destroyable';

class MyResource extends Resource {
  constructor(owner, args) {
    super(owner, args);

    // initial setup

    registerDestructor(this, () => {
      // 🎵 it's the final teardown
    });
  }

  modify(positional, named) {
    // cleanup function for each update
    this.cleanup();

    // update
  }

  // ... ✂️  ...
}
```

### Utilities

#### `useTask`

_in v5 `ember-concurrency@v1` will no longer be supported_. This does not mean that `ember-concurrency@v1`
won't work, but it does mean that maintenance in `ember-resources` regarding `ember-concurrency@v1`
is no longer worth the effort.

_ember-concurrency@v1 is also not compatible with ember-source@v4+_

#### `trackedFunction`

tbd: replacement not implemented yet.
API will be the same, but behavior will be slightly different, and there will be additional properties.


#### `use`

The `@use` decorator did not see much of any public usage and will be removed in `ember-resources@v5`

#### `useFunction`

Already deprecated in favor of `trackedFunction`. Removed in v5.


#### `useHelper`

Renamed to `helper`, as per the nomenclature thoughts above.

Before

```js
import { tracked } from '@glimmer/tracking';
import { useHelper } from 'ember-resources';
import intersect from 'ember-composable-helpers/addon/helpers/intersect';

class Foo {
  @tracked listA = [1, 2, 3];
  @tracked listB = [3, 4, 5];

  myHelper = useHelper(this, intersect, () => [this.listA, this.listB])

  get intersection() {
    return this.myHelper.value;
  }
}
```
```hbs
{{log this.intersection}}
```

After

```js
import { tracked } from '@glimmer/tracking';
import { helper } from 'ember-resources/util/helper';
import intersect from 'ember-composable-helpers/addon/helpers/intersect';

class Foo {
  @tracked listA = [1, 2, 3];
  @tracked listB = [3, 4, 5];

  myHelper = helper(this, intersect, () => [this.listA, this.listB])

  get intersection() {
    return this.myHelper.value;
  }
}
```
```hbs
{{log this.intersection}}
```


#### `useResource`

Removed in favor of static method on `Resource`, `from`.
Additionally, the class no longer needs to be passed separately.

Before
```js
export function findAll(destroyable, modelName, thunk) {
  return useResource(destroyable, FindAll, () => {
    let reified = thunk?.() || {};
    let options = 'options' in reified ? reified.options : reified;

    return {
      positional: [modelName],
      named: {
        options,
      },
    };
  });
}
```

After
```js
export function findAll(destroyable, modelName, thunk) {
  return Resource.of(destroyable, FindAll, () => {
    let reified = thunk?.() || {};
    let options = 'options' in reified ? reified.options : reified;

    return {
      positional: [modelName],
      named: {
        options,
      },
    };
  });
}
```


## References

Decisions are influenced by the [Code Search for `ember-resources`](https://emberobserver.com/code-search?codeQuery=ember-resources) on [Ember Observer](https://emberobserver.com) as well as internal usage and evolution within @NullVoxPopuli's work (as open source does not contain _everything_).

  Additional searches:
  - [`LifecycleResource`](https://emberobserver.com/code-search?codeQuery=LifecycleResource)
  - [`useResource`](https://emberobserver.com/code-search?codeQuery=useResource)
  - [`useFunction`](https://emberobserver.com/code-search?codeQuery=useFunction)
  - [`useHelper`](https://emberobserver.com/code-search?codeQuery=useHelper)
  - [`import { use }`](https://emberobserver.com/code-search?codeQuery=import%20%7B%20use%20%7D)
  - [`import { use`](https://emberobserver.com/code-search?codeQuery=import%20%7B%20use)


The [ember-modifiers v4 migration guide](https://github.com/ember-modifier/ember-modifier/blob/master/MIGRATIONS.md) that much of this document is based off of.


