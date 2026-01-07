### Links: 
https://vuejs.org/guide/essentials/template-syntax
### Notas:
#### Raw HTML
The double mustaches interpret the data as plain text, not HTML. In order to output real HTML, you will need to use the `v-html` directive.
```html
<p>Using text interpolation: {{ rawHtml }}</p>
<p>Using v-html directive: <span v-html="rawHtml"></span></p>
```
![[ejemplo1Vue.png]]![[warning1Vue.png]]

#### Attribute Bindings
Mustaches cannot be used inside HTML attributes. Instead, use a `v-bind` directive. The `v-bind` directive instructs Vue to keep the element's `id` attribute in sync with the component's `dynamicId` property. If the bound value is `null` or `undefined`, then the attribute will be removed from the rendered element. Because `v-bind` is so commonly used, it has a dedicated shorthand syntax `:id`.
```html
<div v-bind:id="dynamicId"></div>
```
##### Boolean Attributes.
They are attributes that can indicate true / false values by their presence on an element. For example, disabled is one of the most commonly used boolean attributes.`v-bind` works a bit differently in this case, the `disabled` attribute will be included if `isButtonDisabled` has a ***truthy value***. It will also be included if the value is an empty string, maintaining consistency with `<button disabled="">`. For other ***falsy values*** the attribute will be omitted.
```html
<button :disabled="isButtonDisabled">Button</button>
```
##### Dynamically Binding Multiple Attributes
A JavaScript object representing multiple attributes can be binded them to a single element by using `v-bind` without an argument.
```javascript
const objectOfAttrs = { 
	id: 'container', 
	class: 'wrapper', 
	style: 'background-color:green' 
}
```
```html
<div v-bind="objectOfAttrs"></div>
```


ARREGLAR DESDE ACÁ

#### Using JavaScript Expressions

So far we've only been binding to simple property keys in our templates. But Vue actually supports the full power of JavaScript expressions inside all data bindings:

template

```
{{ number + 1 }}

{{ ok ? 'YES' : 'NO' }}

{{ message.split('').reverse().join('') }}

<div :id="`list-${id}`"></div>
```

These expressions will be evaluated as JavaScript in the data scope of the current component instance.

In Vue templates, JavaScript expressions can be used in the following positions:

- Inside text interpolations (mustaches)
- In the attribute value of any Vue directives (special attributes that start with `v-`)

##### Expressions Only

Each binding can only contain **one single expression**. An expression is a piece of code that can be evaluated to a value. A simple check is whether it can be used after `return`.

Therefore, the following will **NOT** work:

template

```
<!-- this is a statement, not an expression: -->
{{ var a = 1 }}

<!-- flow control won't work either, use ternary expressions -->
{{ if (ok) { return message } }}
```

-  Calling Functions[​](https://vuejs.org/guide/essentials/template-syntax#calling-functions)

It is possible to call a component-exposed method inside a binding expression:

template

```
<time :title="toTitleDate(date)" :datetime="date">
  {{ formatDate(date) }}
</time>
```

TIP

Functions called inside binding expressions will be called every time the component updates, so they should **not** have any side effects, such as changing data or triggering asynchronous operations.

##### Restricted Globals Access
Template expressions are sandboxed and only have access to a [restricted list of globals](https://github.com/vuejs/core/blob/main/packages/shared/src/globalsAllowList.ts#L3). The list exposes commonly used built-in globals such as `Math` and `Date`.

Globals not explicitly included in the list, for example user-attached properties on `window`, will not be accessible in template expressions. You can, however, explicitly define additional globals for all Vue expressions by adding them to [`app.config.globalProperties`](https://vuejs.org/api/application#app-config-globalproperties).

#### Directives
Directives are special attributes with the `v-` prefix. Vue provides a number of [built-in directives](https://vuejs.org/api/built-in-directives), including `v-html` and `v-bind` which we have introduced above.

Directive attribute values are expected to be single JavaScript expressions (with the exception of `v-for`, `v-on` and `v-slot`, which will be discussed in their respective sections later). A directive's job is to reactively apply updates to the DOM when the value of its expression changes. Take [`v-if`](https://vuejs.org/api/built-in-directives#v-if) as an example:

template

```
<p v-if="seen">Now you see me</p>
```

Here, the `v-if` directive would remove or insert the `<p>` element based on the truthiness of the value of the expression `seen`.

##### Arguments
Some directives can take an "argument", denoted by a colon after the directive name. For example, the `v-bind` directive is used to reactively update an HTML attribute:

template

```
<a v-bind:href="url"> ... </a>

<!-- shorthand -->
<a :href="url"> ... </a>
```

Here, `href` is the argument, which tells the `v-bind` directive to bind the element's `href` attribute to the value of the expression `url`. In the shorthand, everything before the argument (i.e., `v-bind:`) is condensed into a single character, `:`.

Another example is the `v-on` directive, which listens to DOM events:

template

```
<a v-on:click="doSomething"> ... </a>

<!-- shorthand -->
<a @click="doSomething"> ... </a>
```

Here, the argument is the event name to listen to: `click`. `v-on` has a corresponding shorthand, namely the `@` character. We will talk about event handling in more detail too.

##### Dynamic Arguments
It is also possible to use a JavaScript expression in a directive argument by wrapping it with square brackets:

template

```
<!--
Note that there are some constraints to the argument expression,
as explained in the "Dynamic Argument Value Constraints" and "Dynamic Argument Syntax Constraints" sections below.
-->
<a v-bind:[attributeName]="url"> ... </a>

<!-- shorthand -->
<a :[attributeName]="url"> ... </a>
```

Here, `attributeName` will be dynamically evaluated as a JavaScript expression, and its evaluated value will be used as the final value for the argument. For example, if your component instance has a data property, `attributeName`, whose value is `"href"`, then this binding will be equivalent to `v-bind:href`.

Similarly, you can use dynamic arguments to bind a handler to a dynamic event name:

template

```
<a v-on:[eventName]="doSomething"> ... </a>

<!-- shorthand -->
<a @[eventName]="doSomething"> ... </a>
```

In this example, when `eventName`'s value is `"focus"`, `v-on:[eventName]` will be equivalent to `v-on:focus`.

###### Dynamic Argument Value Constraints

Dynamic arguments are expected to evaluate to a string, with the exception of `null`. The special value `null` can be used to explicitly remove the binding. Any other non-string value will trigger a warning.

###### Dynamic Argument Syntax Constraints

Dynamic argument expressions have some syntax constraints because certain characters, such as spaces and quotes, are invalid inside HTML attribute names. For example, the following is invalid:

template

```
<!-- This will trigger a compiler warning. -->
<a :['foo' + bar]="value"> ... </a>
```

If you need to pass a complex dynamic argument, it's probably better to use a [computed property](https://vuejs.org/guide/essentials/computed), which we will cover shortly.

When using in-DOM templates (templates directly written in an HTML file), you should also avoid naming keys with uppercase characters, as browsers will coerce attribute names into lowercase:

template

```
<a :[someAttr]="value"> ... </a>
```

The above will be converted to `:[someattr]` in in-DOM templates. If your component has a `someAttr` property instead of `someattr`, your code won't work. Templates inside Single-File Components are **not** subject to this constraint.

##### Modifiers
Modifiers are special postfixes denoted by a dot, which indicate that a directive should be bound in some special way. For example, the `.prevent` modifier tells the `v-on` directive to call `event.preventDefault()` on the triggered event:

template

```
<form @submit.prevent="onSubmit">...</form>
```

You'll see other examples of modifiers later, [for `v-on`](https://vuejs.org/guide/essentials/event-handling#event-modifiers) and [for `v-model`](https://vuejs.org/guide/essentials/forms#modifiers), when we explore those features.

And finally, here's the full directive syntax visualized
![[vueDirectiveSyntax.png]]