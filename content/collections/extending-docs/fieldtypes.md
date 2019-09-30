---
title: Fieldtypes
updated_by: 3a60f79d-8381-4def-a970-5df62f0f5d56
updated_at: 1568643872
id: 83786f60-def6-11e9-aaef-0800200c9a66
intro: Fieldtypes determine the user interface and storage format for your [fields](/fields). Statamic includes 40+ fieldtypes to help you tailor the perfect intuitive experience for your authors, but there's always room for _one more_.
---

## Registering

Any fieldtype classes inside the `App\Fieldtypes` namespace will be automatically registered.

To store them elsewhere, manually register an action in a service provider by calling the static `register` method on your action class.

``` php
public function boot()
{
    Your\Fieldtype::register();
}
```

## Creating

Fieldtypes have two pieces:

  - A PHP class that handles data processing and validation
  - A Vue component that handles the view and data binding

For this example we will create a password field with a "show" toggle control:

<figure>
    <img src="https://docs.statamic.com/assets/examples/password-fieldtype.gif" alt="An example fieldtype that reveals a password field" class="p-4 bg-white">
    <figcaption>Follow along and you could make this!</figcaption>
</figure>

Create a fieldtype PHP class and Vue component by running the following command:

``` bash
php please make:fieldtype TogglePassword
```

``` php
<?php

class TogglePassword extends \Statamic\Fields\Fieldtype
{
    //
}
```

Create a Vue component in [one of your loaded javascript files](/guide/extending/addons.html#assets-css-stylesheets-and-javascript) and register it as `[handle]-fieldtype`.

Your component has two requirements:

- It must expect a `value` prop.
- It must emit an `input` event whenever the value updates.

Statamic provides you with a `Fieldtype` mixin that does this automatically so you can cut away boilerplate code.

### Example Vue component
``` js
import Fieldtype from './TogglePasswordFieldtype.vue';

// Should be named [snake_case_handle]-fieldtype
Statamic.$components.register('toggle_password-fieldtype', Fieldtype);
```

``` vue
<template>
    <div>
        <text-input :type="inputType" :value="value" @input="update">
        <label><input type="checkbox" v-model="show" /> Show Password</label>
    </div>
</template>

<script>
export default {
    mixins: [Fieldtype],
    data() {
        return {
            show: false
        };
    },
    computed: {
        inputType() {
            return this.show ? 'text' : 'password';
        }
    }
};
</script>
```

#### Example walkthrough:
- The `Fieldtype` mixin is providing an `value` prop containing the initial value of the field.
- The `text-input` component emits an `input` event whenever you type into it. Our component is listening for that event and calls the `update` method.
- The `Fieldtype` mixin is providing that `update` method which emits the required `input` event. This is how the parent component is detecting changes in your fieldtype.

**Do not** modify the `value` prop directly. Instead, call `this.update(value)` and let the Vuex store handle the update appropriately.

### Meta Data

Fieldtypes can preload additional "meta" data from PHP into JavaScript. This can be anything you want, from settings to eager loaded data.

``` php
public function preload()
{
    return ['foo' => 'bar'];
}
```

This can be accessed in the Vue component using the `meta` property.

``` js
return this.meta; // { foo: bar }
```

If you have a need to update this meta data on the _JavaScript side_, use the `updateMeta` method. This will persist the value back to Vuex store and communicate the update to the appropriate places.

``` js
this.updateMeta({ foo: 'baz' });
this.meta; // { foo: 'baz' }
```

### Example use cases -

Here are some reasons why you might want to use this feature:

- The assets and relationship fieldtypes only store IDs, so they will fetch item data using AJAX requests. If you have many of these fields in one form, you'd have a bunch of AJAX requests fire off when the page loads. Preload the item data to avoid the initial AJAX requests.
- Grid, Bard, and Replicator fields all preload values for what a new row/set contains, plus the recursive meta values of any nested fields.

## Index Fieldtypes

In listings (collection indexes in the Control Panel, for example), string values will be displayed as a truncated string and arrays will be displayed as JSON.

You can adjust the value before it gets sent to the listing with the `preProcessIndex` method:

``` php
public function preProcessIndex($value)
{
    return str_repeat('*', strlen($value));
}
```

If you need extra control or functionality, fieldtypes may have an additional "index" Vue component.


``` js
import Fieldtype from './TogglePasswordIndexFieldtype.vue';

// Should be named [snake_case_handle]-fieldtype-index
Statamic.$components.register('toggle_password-fieldtype-index', Fieldtype);
```

``` vue
<template>
    <div v-html="bullets" />
</template>

<script>
export default {
    mixins: [IndexFieldtype],
    computed: {
        bullets() {
            return '&bull;'.repeat(this.value.length);
        }
    }
}
</script>
```

The `IndexFieldtype` mixin will provide you with a `value` prop so you can display it however you'd like. Continuing our example above, we will replace the value with bullets.


## Updating from v2

In Statamic v2 we pass a `data` prop that can be directly modified. You might be see something like this:

``` html
<input type="text" v-model="data" />
```

In v3 you need to pass the value _down_ in a prop (call it `value`), and likewise pass the modified value _up_ by emitting an `input` event. This change is the result of architectural changes in [Vue.js 2](https://vuejs.org).

``` html
<!-- Using a standard HTML input field: -->
<input type="text" :value="value" @input="$emit('input', $event.target.value)">

<!-- Using the "Fieldtype" mixin's `update` method to emit the event for you: -->
<input type="text" :value="value" @input="update($event.target.value)">

<!-- Using a Statamic input component to clean it up even further: -->
<text-input :value="value" @input="update">
```

An alternate solution could be to add a `data` property, initialize it from the new `value` prop, then emit the event whenever the `data` changes. By doing this, you won't need to modify your template or the rest of your JavaScript logic. You can just continue to modify `data`.

``` vue
<template>
    <input type="text" v-model="data" />
</template>
<script>
export default {
    mixins: [Fieldtype],
    data() {
        return {
            data: this.value,
        }
    },
    watch: {
        data(data) {
            this.update(data);
        }
    }
}
</script>
```

The PHP file should require no changes.