### Links:
https://vuejs.org/guide/essentials/forms.html
### Notas:

ARREGLAR

When dealing with forms on the frontend, we often need to sync the state of form input elements with corresponding state in JavaScript. It can be cumbersome to manually wire up value bindings and change event listeners:

template

```
<input
  :value="text"
  @input="event => text = event.target.value">
```

The `v-model` directive helps us simplify the above to:

template

```
<input v-model="text">
```

In addition, `v-model` can be used on inputs of different types, `<textarea>`, and `<select>` elements. It automatically expands to different DOM property and event pairs based on the element it is used on:

- `<input>` with text types and `<textarea>` elements use `value` property and `input` event;
- `<input type="checkbox">` and `<input type="radio">` use `checked` property and `change` event;
- `<select>` uses `value` as a prop and `change` as an event.

Note

`v-model` will ignore the initial `value`, `checked` or `selected` attributes found on any form elements. It will always treat the current bound JavaScript state as the source of truth. You should declare the initial value on the JavaScript side, using [reactivity APIs](https://vuejs.org/api/reactivity-core#reactivity-api-core).

## Basic Usage[​](https://vuejs.org/guide/essentials/forms.html#basic-usage)

### Text[​](https://vuejs.org/guide/essentials/forms.html#text)

template

```
<p>Message is: {{ message }}</p>
<input v-model="message" placeholder="edit me" />
```

Message is:

[Try it in the Playground](https://play.vuejs.org/#eNo9jUEOgyAQRa8yYUO7aNkbNOkBegM2RseWRGACoxvC3TumxuX/+f+9ql5Ez31D1SlbpuyJoSBvNLjoA6XMUCHjAg2WnAJomWoXXZxSLAwBSxk/CP2xuWl9d9GaP0YAEhgDrSOjJABLw/s8+NJBrde/NWsOpWPrI20M+yOkGdfeqXPiFAhowm9aZ8zS4+wPv/RGjtZcJtV+YpNK1g==)

Note

For languages that require an [IME](https://en.wikipedia.org/wiki/Input_method) (Chinese, Japanese, Korean, etc.), you'll notice that `v-model` doesn't get updated during IME composition. If you want to respond to these updates as well, use your own `input` event listener and `value` binding instead of using `v-model`.

### Multiline Text[​](https://vuejs.org/guide/essentials/forms.html#multiline-text)

template

```
<span>Multiline message is:</span>
<p style="white-space: pre-line;">{{ message }}</p>
<textarea v-model="message" placeholder="add multiple lines"></textarea>
```

Multiline message is:

[Try it in the Playground](https://play.vuejs.org/#eNo9jktuwzAMRK9CaON24XrvKgZ6gN5AG8FmGgH6ECKdJjB891D5LYec9zCb+SH6Oq9oRmN5roEEGGWlyeWQqFSBDSoeYYdjLQk6rXYuuzyXzAIJmf0fwqF1Prru02U7PDQq0CCYKHrBlsQy+Tz9rlFCDBnfdOBRqfa7twhYrhEPzvyfgmCvnxlHoIp9w76dmbbtDe+7HdpaBQUv4it6OPepLBjV8Gw5AzpjxlOJC1a9+2WB1IZQRGhWVqsdXgb1tfDcbvYbJDRqLQ==)

Note that interpolation inside `<textarea>` won't work. Use `v-model` instead.

template

```
<!-- bad -->
<textarea>{{ text }}</textarea>

<!-- good -->
<textarea v-model="text"></textarea>
```

### Checkbox[​](https://vuejs.org/guide/essentials/forms.html#checkbox)

Single checkbox, boolean value:

template

```
<input type="checkbox" id="checkbox" v-model="checked" />
<label for="checkbox">{{ checked }}</label>
```

false

[Try it in the Playground](https://play.vuejs.org/#eNpVjssKgzAURH/lko3tonVfotD/yEaTKw3Ni3gjLSH/3qhUcDnDnMNk9gzhviRkD8ZnGXUgmJFS6IXTNvhIkCHiBAWm6C00ddoIJ5z0biaQL5RvVNCtmwvFhFfheLuLqqIGQhvMQLgm4tqFREDfgJ1gGz36j2Cg1TkvN+sVmn+JqnbtrjDDiAYmH09En/PxphTebqsK8PY4wMoPslBUxQ==)

We can also bind multiple checkboxes to the same array or [Set](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set) value:

js

```
const checkedNames = ref([])
```

template

```
<div>Checked names: {{ checkedNames }}</div>

<input type="checkbox" id="jack" value="Jack" v-model="checkedNames" />
<label for="jack">Jack</label>

<input type="checkbox" id="john" value="John" v-model="checkedNames" />
<label for="john">John</label>

<input type="checkbox" id="mike" value="Mike" v-model="checkedNames" />
<label for="mike">Mike</label>
```

Checked names: []

JackJohnMike

In this case, the `checkedNames` array will always contain the values from the currently checked boxes.

[Try it in the Playground](https://play.vuejs.org/#eNqVkUtqwzAURbfy0CTtoNU8KILSWaHdQNWBIj8T1fohyybBeO+RbOc3i2e+vHvuMWggHyG89x2SLWGtijokaDF1gQunbfAxwQARaxihjt7CJlc3wgmnvGsTqAOqBqsfabGFXSm+/P69CsfovJVXckhog5EJcwJgle7558yBK+AWhuFxaRwZLbVCZ0K70CVIp4A7Qabi3h8FAV3l/C9Vk797abpy/lrim/UVmkt/Gc4HOv+EkXs0UPt4XeCFZHQ6lM4TZn9w9+YlrjFPCC/kKrPVDd6Zv5e4wjwv8ELezIxeX4qMZwHduAs=)

### Radio[​](https://vuejs.org/guide/essentials/forms.html#radio)

template

```
<div>Picked: {{ picked }}</div>

<input type="radio" id="one" value="One" v-model="picked" />
<label for="one">One</label>

<input type="radio" id="two" value="Two" v-model="picked" />
<label for="two">Two</label>
```

Picked:

OneTwo

[Try it in the Playground](https://play.vuejs.org/#eNqFkDFuwzAMRa9CaHE7tNoDxUBP0A4dtTgWDQiRJUKmHQSG7x7KhpMMAbLxk3z/g5zVD9H3NKI6KDO02RPDgDxSbaPvKWWGGTJ2sECXUw+VrFY22timODCQb8/o4FhWPqrfiNWnjUZvRmIhgrGn0DCKAjDOT/XfCh1gnnd+WYwukwJYNj7SyMBXwqNVuXE+WQXeiUgRpZyaMJaR5BX11SeHQfTmJi1dnNiE5oQBupR3shbC6LX9Posvpdyz/jf1OksOe85ayVqIR5bR9z+o5Qbc6oCk)

### Select[​](https://vuejs.org/guide/essentials/forms.html#select)

Single select:

template

```
<div>Selected: {{ selected }}</div>

<select v-model="selected">
  <option disabled value="">Please select one</option>
  <option>A</option>
  <option>B</option>
  <option>C</option>
</select>
```

Selected:

Please select oneABC

[Try it in the Playground](https://play.vuejs.org/#eNp1j7EOgyAQhl/lwmI7tO4Nmti+QJOuLFTPxASBALoQ3r2H2jYOjvff939wkTXWXucJ2Y1x37rBBvAYJlsLPYzWuAARHPaQoHdmhILQQmihW6N9RhW2ATuoMnQqirPQvFw9ZKAh4GiVDEgTAPdW6hpeW+sGMf4VKVEz73Mvs8sC5stoOlSVYF9SsEVGiLFhMBq6wcu3IsUs1YREEvFUKD1udjAaebnS+27dHOT3g/yxy+nHywM08PJ3KksfXwJ2dA==)

Note

If the initial value of your `v-model` expression does not match any of the options, the `<select>` element will render in an "unselected" state. On iOS this will cause the user not being able to select the first item because iOS does not fire a change event in this case. It is therefore recommended to provide a disabled option with an empty value, as demonstrated in the example above.

Multiple select (bound to array):

template

```
<div>Selected: {{ selected }}</div>

<select v-model="selected" multiple>
  <option>A</option>
  <option>B</option>
  <option>C</option>
</select>
```

Selected: []

ABC

[Try it in the Playground](https://play.vuejs.org/#eNp1kL2OwjAQhF9l5Ya74i7QBhMJeARKTIESIyz5Z5VsAsjyu7NOQEBB5xl/M7vaKNaI/0OvRSlkV7cGCTpNPVbKG4ehJYjQ6hMkOLXBwYzRmfLK18F3GbW6Jt3AKkM/+8Ov8rKYeriBBWmH9kiaFYBszFDtHpkSYnwVpCSL/JtDDE4+DH8uNNqulHiCSoDrLRm0UyWzAckEX61l8Xh9+psv/vbD563HCSxk8bY0y45u47AJ2D/HHyDm4MU0dC5hMZ/jdal8Gg8wJkS6A3nRew4=)

Select options can be dynamically rendered with `v-for`:

js

```
const selected = ref('A')

const options = ref([
  { text: 'One', value: 'A' },
  { text: 'Two', value: 'B' },
  { text: 'Three', value: 'C' }
])
```

template

```
<div>Selected: {{ selected }}</div>

<select v-model="selected">
  <option v-for="option in options" :value="option.value">
    {{ option.text }}
  </option>
</select>
```

Selected: A

OneTwoThree

[Try it in the Playground](https://play.vuejs.org/#eNp9kj9vwjAQxb/KyUtaiYahGwpIgBjaoVSFre6AkguEOnZkOwEpynfv2flDqlZkyt37+fye7ZotiyKsSmQzFplYZ4UFg7YsFlxmeaG0hRo0ptBAqlUOAaEBl1zGShqHCowtJjB30EOwDB5voipsRj+d9skl0CyLVzuDYCsxmEB1ECVStQygmfzS9xc10ld/9ZPG8YQ1EVx+0e7RtI1BAaiwmBfiYNFVNkqyarHrLM+grm/+myaaOtUtAojaPlRPuUpQzDnrQc4IAfqiNh0hqdIEdGUm+9icwcy7G8TQl8MESlN3cOhSkYdu9LTteo7i+K2piKZDGjZh1tApp9kxPBsl6fZqR3MWq7zIBOpt74JytmM5OwihLq++Z3WJ/kT9mhPG3//0z+bqepy9azSoK/I+aPagj2hbebN7I/8jkU6tFETfET/QKFE6jy22KmVCtkecd/vi32Amj3uzuVqUpg/ljDqyfRec0btc34l+s/scPvt1XDas+QENov3B)

## Value Bindings[​](https://vuejs.org/guide/essentials/forms.html#value-bindings)

For radio, checkbox and select options, the `v-model` binding values are usually static strings (or booleans for checkbox):

template

```
<!-- `picked` is a string "a" when checked -->
<input type="radio" v-model="picked" value="a" />

<!-- `toggle` is either true or false -->
<input type="checkbox" v-model="toggle" />

<!-- `selected` is a string "abc" when the first option is selected -->
<select v-model="selected">
  <option value="abc">ABC</option>
</select>
```

But sometimes we may want to bind the value to a dynamic property on the current active instance. We can use `v-bind` to achieve that. In addition, using `v-bind` allows us to bind the input value to non-string values.

### Checkbox[​](https://vuejs.org/guide/essentials/forms.html#checkbox-1)

template

```
<input
  type="checkbox"
  v-model="toggle"
  true-value="yes"
  false-value="no" />
```

`true-value` and `false-value` are Vue-specific attributes that only work with `v-model`. Here the `toggle` property's value will be set to `'yes'` when the box is checked, and set to `'no'` when unchecked. You can also bind them to dynamic values using `v-bind`:

template

```
<input
  type="checkbox"
  v-model="toggle"
  :true-value="dynamicTrueValue"
  :false-value="dynamicFalseValue" />
```

Tip

The `true-value` and `false-value` attributes don't affect the input's `value` attribute, because browsers don't include unchecked boxes in form submissions. To guarantee that one of two values is submitted in a form (e.g. "yes" or "no"), use radio inputs instead.

### Radio[​](https://vuejs.org/guide/essentials/forms.html#radio-1)

template

```
<input type="radio" v-model="pick" :value="first" />
<input type="radio" v-model="pick" :value="second" />
```

`pick` will be set to the value of `first` when the first radio input is checked, and set to the value of `second` when the second one is checked.

### Select Options[​](https://vuejs.org/guide/essentials/forms.html#select-options)

template

```
<select v-model="selected">
  <!-- inline object literal -->
  <option :value="{ number: 123 }">123</option>
</select>
```

`v-model` supports value bindings of non-string values as well! In the above example, when the option is selected, `selected` will be set to the object literal value of `{ number: 123 }`.

## Modifiers[​](https://vuejs.org/guide/essentials/forms.html#modifiers)

### `.lazy`[​](https://vuejs.org/guide/essentials/forms.html#lazy)

By default, `v-model` syncs the input with the data after each `input` event (with the exception of IME composition as [stated above](https://vuejs.org/guide/essentials/forms.html#vmodel-ime-tip)). You can add the `lazy` modifier to instead sync after `change` events:

template

```
<!-- synced after "change" instead of "input" -->
<input v-model.lazy="msg" />
```

### `.number`[​](https://vuejs.org/guide/essentials/forms.html#number)

If you want user input to be automatically typecast as a number, you can add the `number` modifier to your `v-model` managed inputs:

template

```
<input v-model.number="age" />
```

If the value cannot be parsed with `parseFloat()`, then the original (string) value is used instead. In particular, if the input is empty (for instance after the user clearing the input field), an empty string is returned. This behavior differs from the [DOM property `valueAsNumber`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLInputElement#valueasnumber).

The `number` modifier is applied automatically if the input has `type="number"`.

### `.trim`[​](https://vuejs.org/guide/essentials/forms.html#trim)

If you want whitespace from user input to be trimmed automatically, you can add the `trim` modifier to your `v-model`-managed inputs:

template

```
<input v-model.trim="msg" />
```

## `v-model` with Components[​](https://vuejs.org/guide/essentials/forms.html#v-model-with-components)

> If you're not yet familiar with Vue's components, you can skip this for now.

HTML's built-in input types won't always meet your needs. Fortunately, Vue components allow you to build reusable inputs with completely customized behavior. These inputs even work with `v-model`! To learn more, read about [Usage with `v-model`](https://vuejs.org/guide/components/v-model) in the Components guide.