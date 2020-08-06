## How VueJs work


### Data binding

I think most important feature that provided by vue is data binding, two ways and one way, 
which improve speed of development significantly.

Data binding achieved by `Object.defineProperty` which redefined the getter and setter.
**@see src/core/observer/index.js**, the `Observer` class which make properties in the target reactive.
The `Observer` object will save a reference to target object by `Observer.value`, and set `__ob__` property
in target object which prevents duplicated `Observer` object in single target (**@see src/core/observer/index.js#observe**),

``` javascript
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } 
```

The `Observer` object only create by `observe()` function in Vue, 
and `observe()` function only call by `src/core/instance/state.js#initData()` and `src/core/instance/state.js#initState()`.
`initData()` only be called when create Vue instance with `data` option, otherwisw `initData()` would not be called and 
`observe()` will call with `vm._data = {}`.

Constructor fo `Observer` object will call `walk()` method with a non-Array object, the `walk()` method will traverse
all properties of the target and redefine the property with `defineReactive()` function.

``` javascript 
walk (obj: Object) {
  const keys = Object.keys(obj)
  for (let i = 0; i < keys.length; i++) {
    defineReactive(obj, keys[i])
  }
}
```

The `defineReactive()` function will redifine **PropertyDescription** by `Object.defineProperty()`.
`defineReactive()` will preserve origin **getter** and **setter** if they exist, otherwise the value
will resident at the Context of the `defineReactive()` function call. See following source code from **src/core/observer/index.js**
``` javascript
export function defineReactive (
  ...
  val: any,
  ...
) {
  ...
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }
  ...
  Object.defineProperty(obj, key, {
    ...
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      ...
      return value
    },
    set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      ...
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      ...
    }
  })
}
```

**getter** of these reactive property is used to collect watchers that dependent on the reactive property,
``` javascript
get: function reactiveGetter () {
  const value = getter ? getter.call(obj) : val
  if (Dep.target) {
    dep.depend()
    if (childOb) {
      childOb.dep.depend()
      if (Array.isArray(value)) {
        dependArray(value)
      }
    }
  }
  return value
}
```

`Dep.target` is a static member of `Dep` class, which control by `PushTarget()` and `PopTarget()` functions located at 
**src/core/observer/dep.js**. `PushTarget()` and `PopTarget()` primely called from `Watcher.prototype.get()` which evaluate
value getter function to get the value and collect dependencies reactive properties.

**setter** of these reactive property is used to modify value and notify watchers whose value depend on the reactive property.
``` javascript
set: function reactiveSetter (newVal) {
  ...
  dep.notify()
}
```

`dep` variable in above code is resident at context of `defineReactive()` function call, 
type of `dep` is `Dep`. 
Generally, watchers that notified by the **setter** will re-evaluate the getter function and 
re-collect dependencies at nextTick (TODO need confirm).

TODO watcher


### Template


In Vue you can define your Component which can be used in **HTML Template** just same with native HTML tags like `div, span`.
You can define meaningful property which pass data from parent component to child component.
With `v-bind` directive Vue Component can achieve one-way data binding, moreover in `input` html elements Vue using `v-model` 
directive implement two-way data binding.

Templates pass to Vue constructor by template option.

