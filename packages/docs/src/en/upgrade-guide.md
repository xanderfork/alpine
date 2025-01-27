---
order: 2
title: Upgrade From V2
---

# Upgrade from V2

Below is an exhaustive guide on the breaking changes in Alpine V3, but if you'd prefer something more lively, you can review all the changes as well as new features in V3 by watching the Alpine Day 2021 "Future of Alpine" keynote:

<!-- START_VERBATIM -->
<div class="relative w-full" style="padding-bottom: 56.25%; padding-top: 30px; height: 0; overflow: hidden;">
    <iframe
            class="absolute top-0 left-0 right-0 bottom-0 w-full h-full"
            src="https://www.youtube.com/embed/WixS4JXMwIQ?modestbranding=1&autoplay=1"
            allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
            allowfullscreen
    ></iframe>
</div>
<!-- END_VERBATIM -->

Upgrading from Alpine V2 to V3 should be fairly painless. In many cases, NOTHING has to be done to your codebase to use V3. Below is an exhaustive list of breaking changes and deprecations in descending order of how likely users are to be affected by them:

> Note if you use Laravel Livewire and Alpine together, to use V3 of Alpine, you will need to upgrade to Livewire v2.5.1 or greater.

<a name="breaking-changes"></a>
## Breaking Changes
* [`$el` is now always the current element](#el-no-longer-root)
* [Automatically evaluate `init()` functions defined on data object](#auto-init)
* [Need to call `Alpine.start()` after import](#need-to-call-alpine-start)
* [`x-show.transition` is now `x-transition`](#removed-show-dot-transition)
* [`x-if` no longer supports `x-transition`](#x-if-no-transitions)
* [`x-data` cascading scope](#x-data-scope)
* [`x-init` no longer accepts a callback return](#x-init-no-callback)
* [Returning `false` from event handlers no longer implicitly "preventDefault"s](#no-false-return-from-event-handlers)
* [`x-spread` is now `x-bind`](#x-spread-is-now-x-bind)
* [Use global lifecycle events instead of `Alpine.deferLoadingAlpine()`](#use-global-events-now)
* [IE11 no longer supported](#no-ie-11)
* [`x-html` has been removed](#no-x-html)

<a name="el-no-longer-root"></a>
### `$el` is now always the current element

`$el` now always represents the element that an expression was executed on, not the root element of the component. This will replace most usages of `x-ref` and in the cases where you still want to access the root of a component, you can do so using `x-ref`. For example:

```html
<!-- 🚫 Before -->
<div x-data>
    <button @click="console.log($el)"></button>
    <!-- In V2, $el would have been the <div>, now it's the <button> -->
</div>

<!-- ✅ After -->
<div x-data x-ref="root">
    <button @click="console.log($refs.root)"></button>
</div>
```

For a smoother upgrade experience, you can optionally replace all instances of `$el` with a custom magic called `$root`, then add the following code to your site to mimmick the behavior:

```html
<script>
    document.addEventListener('alpine:initializing', () => {
        Alpine.magic('root', el => {
            let closestRootEl = (node) => {
                if (node.hasAttribute('x-data')) return node

                return closestRootEl(node.parentNode)
            }

            return closestRootEl(el)
        })
    })
</script>
```

[→ Read more about $el in V3](/magics/el)

<a name="auto-init"></a>
### Automatically evaluate `init()` functions defined on data object

A common pattern in V2 was to manually call an `init()` (or similarly named method) on an `x-data` object.

In V3, Alpine will automatically call `init()` methods on data objects.

```html
<!-- 🚫 Before -->
<div x-data="foo()" x-init="init()"></div>

<!-- ✅ After -->
<div x-data="foo()"></div>

<script>
    function foo() {
        return {
            init() {
                //
            }
        }
    }
</script>
```

[→ Read more about auto-evaluating init functions](/globals/alpine-data#init-functions)

<a name="need-to-call-alpine-start"></a>
### Need to call Alpine.start() after import

If you were importing Alpine V2 from NPM, you will now need to manually call `Alpine.start()` for V3. This doesn't affect you if you use Alpine's build file or CDN from a `<template>` tag.

```js
// 🚫 Before
import 'alpinejs'

// ✅ After
import Alpine from 'alpinejs'

window.Alpine = Alpine

Alpine.start()
```

[→ Read more about initializing Alpine V3](/essentials/installation#as-a-module)

<a name="removed-show-dot-transition"></a>
### `x-show.transition` is now `x-transition`

All of the conveniences provided by `x-show.transition...` helpers are still available, but now from a more unified API: `x-transition`:

```html
<!-- 🚫 Before -->
<div x-show.transition="open"></div>
<!-- ✅ After -->
<div x-show="open" x-transition></div>

<!-- 🚫 Before -->
<div x-show.transition.duration.500ms="open"></div>
<!-- ✅ After -->
<div x-show="open" x-transition.duration.500ms></div>

<!-- 🚫 Before -->
<div x-show.transition.in.duration.500ms.out.duration.750ms="open"></div>
<!-- ✅ After -->
<div
    x-show="open"
    x-transition:enter.duration.500ms
    x-transition:leave.duration.750ms
></div>
```

[→ Read more about x-transition](/directives/transition)

<a name="x-if-no-transitions"></a>
### `x-if` no longer supports `x-transition`

The ability to transition elements in and add before/after being removed from the DOM is no longer available in Alpine.

This was a feature very few people even knew existed let alone used.

Because the transition system is complex, it makes more sense from a maintenance perspective to only support transitioning elements with `x-show`.

```html
<!-- 🚫 Before -->
<template x-if.transition="open">
    <div>...</div>
</template>

<!-- ✅ After -->
<div x-show="open" x-transition>...</div>
```

[→ Read more about x-if](/directives/if)

<a name="x-data-scope"></a>
### `x-data` cascading scope

Scope defined in `x-data` is now available to all children unless overwritten by a nested `x-data` expression.

```html
<!-- 🚫 Before -->
<div x-data="{ foo: 'bar' }">
    <div x-data="{}">
        <!-- foo is undefined -->
    </div>
</div>

<!-- ✅ After -->
<div x-data="{ foo: 'bar' }">
    <div x-data="{}">
        <!-- foo is 'bar' -->
    </div>
</div>
```

[→ Read more about x-data scoping](/directives/data#scope)

<a name="x-init-no-callback"></a>
### `x-init` no longer accepts a callback return

Before V3, if `x-init` received a return value that is `typeof` "function", it would execute the callback after Alpine finished initializing all other directives in the tree. Now, you must manually call `$nextTick()` to achieve that behavior. `x-init` is no longer "return value aware".

```html
<!-- 🚫 Before -->
<div x-data x-init="() => { ... }">...</div>

<!-- ✅ After -->
<div x-data x-init="$nextTick(() => { ... })">...</div>
```

[→ Read more about $nextTick](/magics/next-tick)

<a name="no-false-return-from-event-handlers"></a>
### Returning `false` from event handlers no longer implicitly "preventDefault"s

Alpine V2 observes a return value of `false` as a desire to run `preventDefault` on the event. This conforms to the standard behavior of native, inline listeners: `<... oninput="someFunctionThatReturnsFalse()">`. Alpine V3 no longer supports this API. Most people don't know it exists and therefore is surprising behavior.

```html
<!-- 🚫 Before -->
<div x-data="{ blockInput() { return false } }">
    <input type="text" @input="blockInput()">
</div>

<!-- ✅ After -->
<div x-data="{ blockInput(e) { e.preventDefault() }">
    <input type="text" @input="blockInput($event)">
</div>
```

[→ Read more about x-on](/directives/on)

<a name="x-spread-now-x-bind"></a>
### `x-spread` is now `x-bind`

One of Alpine's stories for re-using functionality is abstracting Alpine directives into objects and applying them to elements with `x-spread`. This behavior is still the same, except now `x-bind` (with no specified attribute) is the API instead of `x-spread`.

```html
<!-- 🚫 Before -->
<div x-data="dropdown()">
    <button x-spread="trigger">Toggle</button>

    <div x-spread="dialogue">...</div>
</div>

<!-- ✅ After -->
<div x-data="dropdown()">
    <button x-bind="trigger">Toggle</button>

    <div x-bind="dialogue">...</div>
</div>


<script>
    function dropdown() {
        return {
            open: false,

            trigger: {
                'x-on:click'() { this.open = ! this.open },
            },

            dialogue: {
                'x-show'() { return this.open },
                'x-bind:class'() { return 'foo bar' },
            },
        }
    }
</script>
```

[→ Read more about binding directives using x-bind](/directives/bind#bind-directives)

<a name="use-global-events-now"></a>
### Use global lifecycle events instead of `Alpine.deferLoadingAlpine()`

```html
<!-- 🚫 Before -->
<script>
    window.deferLoadingAlpine = startAlpine => {
        // Will be executed before initializing Alpine.

        startAlpine()

        // Will be executed after initializing Alpine.
    }
</script>

<!-- ✅ After -->
<script>
    document.addEventListener('alpine:initializing', () => {
        // Will be executed before initializing Alpine.
    })

    document.addEventListener('alpine:initialized', () => {
        // Will be executed after initializing Alpine.
    })
</script>
```

[→ Read more about Alpine lifecycle events](/essentials/lifecycle#alpine-initialization)

<a name="no-ie-11"></a>
### IE11 no longer supported

Alpine will no longer officially support Internet Explorer 11. If you need support for IE11, we recommend still using Alpine V2.

<a name="no-x-html"></a>
### `x-html` has been removed

`x-html` was a seldom used directive in Alpine V2. In an effort to keep the API slimmed down to only valued features, V3 is removing this directive.

However, using V3's new custom directive API, it's trivial to reintroduce this functionality using the following polyfill:

```html
<!-- 🚫 Before -->
<div x-data="{ someHtml: '<h1>...</h1>' }">
    <div x-html="someHTML">
</div>

<!-- ✅ After -->
<!-- The above will now work with the following script added to the page: -->
<script>
    document.addEventListener('alpine:initializing', () => {
        Alpine.directive('html', (el, { expression }, { evaluateLater, effect }) => {
            let getHtml = evaluateLater(expression)

            effect(() => {
                getHtml(html => {
                    el.innerHTML = html
                })
            })
        })
    })
</script>
```

## Deprecated APIs

The following 2 APIs will still work in V3, but are considered deprecated and are likely to be removed at some point in the future.

<a name="away-replace-with-outside"></a>
### Event listener modifier `.away` should be replaced with `.outside`

```html
<!-- 🚫 Before -->
<div x-show="open" @click.away="open = false">
    ...
</div>

<!-- ✅ After -->
<div x-show="open" @click.outside="open = false">
    ...
</div>
```

<a name="alpine-data-instead-of-global-functions"></a>
### Prefer `Alpine.data()` to global Alpine function data providers

```html
<!-- 🚫 Before -->
<div x-data="dropdown()">
    ...
</div>

<script>
    function dropdown() {
        return {
            ...
        }
    }
</script>

<!-- ✅ After -->
<div x-data="dropdown">
    ...
</div>

<script>
    document.addEventListener('alpine:initializing', () => {
        Alpine.data('dropdown', () => ({
            ...
        }))
    })
</script>
```
