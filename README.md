**BuckleScript Bindings Cookbook**

WRITING BuckleScript bindings can be somewhere between an art and a science, taking some learning investment into both the JavaScript and OCaml/Reason type systems to get a proper feel for it.

This cookbook aims to be a quickstart, task-focused guide for writing bindings. The idea is that you have in mind some JavaScript that you want to write, and look up the binding that should (hopefully) produce that output JavaScript.

Along the way, I will try to introduce standard types for modelling various JavaScript data.

**Contents**

- [Globals](#globals)
  - [window // global variable](#window--global-variable)
  - [window? // does global variable exist](#window--does-global-variable-exist)
  - [Math.PI // variable in global module](#mathpi--variable-in-global-module)
  - [console.log // function in global module](#consolelog--function-in-global-module)
- [Modules](#modules)
  - [const path = require('path'); path.join('a', 'b') // function in CJS/ES module](#const-path--requirepath-pathjoina-b--function-in-cjses-module)
  - [const foo = require('foo'); foo(1) // import entire module as a value](#const-foo--requirefoo-foo1--import-entire-module-as-a-value)
  - [import foo from 'foo'; foo(1) // import ES6 module default export](#import-foo-from-foo-foo1--import-es6-module-default-export)
- [Functions](#functions)
  - [const dir = path.join('a', 'b', ...) // function with rest args](#const-dir--pathjoina-b---function-with-rest-args)
  - [const nums = range(start, stop, step) // call a function with named arguments for readability](#const-nums--rangestart-stop-step--call-a-function-with-named-arguments-for-readability)
  - [const nums = range(start, stop, [step]) // optional final argument(s)](#const-nums--rangestart-stop-step--optional-final-arguments)
  - [forEach(start, stop, item => console.log(item)) // model a callback](#foreachstart-stop-item--consolelogitem--model-a-callback)
- [Objects](#objects)
  - [const person = {id: 1, name: 'Bob'} // create an object](#const-person--id-1-name-bob--create-an-object)
  - [person.name // get a prop](#personname--get-a-prop)
  - [person.id = 0 // set a prop](#personid--0--set-a-prop)
- [Classes and OOP](#classes-and-oop)
  - [const foo = new Foo() // call a class constructor](#const-foo--new-foo--call-a-class-constructor)
  - [const bar = foo.bar // get an instance property](#const-bar--foobar--get-an-instance-property)
  - [foo.bar = 1 // set an instance property](#foobar--1--set-an-instance-property)
  - [foo.meth() // call a nullary instance method](#foometh--call-a-nullary-instance-method)
  - [const newStr = str.replace(substr, newSubstr) // non-mutating instance method](#const-newstr--strreplacesubstr-newsubstr--non-mutating-instance-method)
  - [arr.sort(compareFunction) // mutating instance method](#arrsortcomparefunction--mutating-instance-method)
- [Null and undefined](#null-and-undefined)
  - [foo.bar === undefined // check for undefined](#foobar--undefined--check-for-undefined)
  - [foo.bar == null // check for null or undefined](#foobar--null--check-for-null-or-undefined)

## Globals

### window // global variable

```ocaml
[@bs.val] external window: Dom.window = "window";
```

Ref https://reasonml.org/docs/reason-compiler/latest/bind-to-global-values

### window? // does global variable exist

```ocaml
switch ([%external window]) {
| Some(_) => "window exists"
| None => "window does not exist"
};
```

`[%external NAME]` makes `NAME` available as an value of type `option('a)`, meaning its wrapped value is compatible with any type. I recommend that, if you use the value, to cast it safely into a known type first.

Ref https://reasonml.org/docs/reason-compiler/latest/embed-raw-javascript#detect-global-variables

### Math.PI // variable in global module

```ocaml
[@bs.val] [@bs.scope "Math"] external pi: float = "PI";
```

Ref https://reasonml.org/docs/reason-compiler/latest/bind-to-global-values#global-modules

### console.log // function in global module

```ocaml
[@bs.val] [@bs.scope "console"] external log: 'a => unit = "log";
```

Note that in JavaScript, `console.log()`'s return value is `undefined`, which we can model with the `unit` type.

## Modules

### const path = require('path'); path.join('a', 'b') // function in CJS/ES module

```ocaml
[@bs.module "path"] external join: (string, string) => string = "join";
let dir = join("a", "b");
```

Ref: https://reasonml.org/docs/reason-compiler/latest/import-export#import

### const foo = require('foo'); foo(1) // import entire module as a value

```ocaml
[@bs.module] external foo: int => unit = "foo";
let () = foo(1);
```

Ref: https://reasonml.org/docs/reason-compiler/latest/import-export#import-a-default-value

### import foo from 'foo'; foo(1) // import ES6 module default export

```ocaml
[@bs.module "foo"] external foo: int => unit = "default";
let () = foo(1);
```

Ref: https://reasonml.org/docs/reason-compiler/latest/import-export#import-an-es6-default-value

## Functions

### const dir = path.join('a', 'b', ...) // function with rest args

```ocaml
[@bs.module "path"] [@bs.variadic] external join: array(string) => string = "join";
let dir = join([|"a", "b", ...|]);
```

Note that the rest args must all be of the same type for `[@bs.variadic]` to work. If they really have different types, then more advanced techniques are needed.

Ref: https://reasonml.org/docs/reason-compiler/latest/function#variadic-function-arguments

### const nums = range(start, stop, step) // call a function with named arguments for readability

```ocaml
[@bs.val] external range: (~start: int, ~stop: int, ~step: int) => array(int) = "range";
let nums = range(~start=1, ~stop=10, ~step=2);
```

### const nums = range(start, stop, [step]) // optional final argument(s)

```ocaml
[@bs.val] external range: (~start: int, ~stop: int, ~step: int=?, unit) => array(int) = "range";
let nums = range(~start=1, ~stop=10, ());
```

If a Reason function or binding has an optional parameter, it needs a positional parameter at the end of the parameter list to help the compiler understand when function application is finished and the function can actually execute. If this seems tedious, remember that no other language gives you out-of-the-box curried parameters _and_ named parameters _and_ optional parameters.

### forEach(start, stop, item => console.log(item)) // model a callback

```ocaml
[@bs.val] external forEach: (~start: int, ~stop: int, [@bs.uncurry] int => unit) => unit = "forEach";
forEach(1, 10, Js.log);
```

When binding to functions with callbacks, you'll want to ensure that the callbacks are uncurried. `[@bs.uncurry]` is the recommended way of doing that. However, in some circumstances you may be forced to use the static uncurried function syntax. See the docs for details.

Ref: https://reasonml.org/docs/reason-compiler/latest/function#extra-solution

## Objects

### const person = {id: 1, name: 'Bob'} // create an object

```ocaml
let person = {"id": 1, "name": "Bob"};
```

Ref: https://reasonml.org/docs/reason-compiler/latest/object-2#literal

### person.name // get a prop

```ocaml
person##name
```

Ref: https://reasonml.org/docs/reason-compiler/latest/object-2#read

### person.id = 0 // set a prop

```ocaml
person##id #= 0
```

Ref: https://reasonml.org/docs/reason-compiler/latest/object-2#write

## Classes and OOP

In BuckleScript it's idiomatic to bind to class properties and methods as functions which take the instance as just a normal function argument. So e.g., instead of

```javascript
const foo = new Foo();
foo.bar();
```

You will write:

```ocaml
let foo = Foo.make();
let () = Foo.bar(foo);
```

### const foo = new Foo() // call a class constructor

```ocaml
// Foo.re
// or,
// module Foo = {

type t;

// The `Foo` at the end must be the name of the class
[@bs.new] external make: unit => t = "Foo";

//}
...
let foo = Foo.make()
```

Note the abstract type `t`. In BuckleScript you will model any class that's not a [shared data type](https://reasonml.org/docs/reason-compiler/latest/common-data-types#shared-data-types) as an abstract data type. This means you won't expose the internals of the definition of the class, only its interface (accessors, methods), using functions which include the type `t` in their signatures. This is shown in the next few sections.

A BuckleScript function binding doesn't have the context that it's binding to a JavaScript class like `Foo`, so you will want to explicitly put it inside a corresponding module `Foo` to denote the class it belongs to. In other words, model JavaScript classes as BuckleScript modules.

Ref: https://reasonml.org/docs/reason-compiler/latest/class#new

### const bar = foo.bar // get an instance property

```ocaml
// In module Foo:
[@bs.get] external bar: t => int = "bar";
...
let bar = Foo.bar(foo);
```

Ref: https://reasonml.org/docs/reason-compiler/latest/property-access#static-property-access

### foo.bar = 1 // set an instance property

```ocaml
// In module Foo:
[@bs.set] external setBar: (t, int) => unit = "bar"; // note the name
...
let () = Foo.setBar(foo, 1);
```

### foo.meth() // call a nullary instance method

```ocaml
// In module Foo:
[@bs.send] external meth: t => unit = "meth";
...
let () = Foo.meth(foo);
```

Ref: https://reasonml.org/docs/reason-compiler/latest/function#object-method

### const newStr = str.replace(substr, newSubstr) // non-mutating instance method

```ocaml
[@bs.send.pipe: string] external replace: (~substr: string, ~newSubstr: string) => string = "replace";

let newStr = replace(~substr, ~newSubstr, str);
```

`[@bs.send.pipe]` injects a parameter of the given type (in this case `string`) as the final positional parameter of the binding. In other words, it creates the binding with the real signature `(~substr: string, ~newSubstr: string, string) => string`. This is handy for non-mutating functions as they traditionally take the instance as the final parameter.

It wasn't strictly necessary to use named arguments in this binding, but it helps readability with multiple arguments, especially if some have the same type.

Also note that you don't strictly need to use `[@bs.send.pipe]`; if you want you can use `[@bs.send]` everywhere.

### arr.sort(compareFunction) // mutating instance method

```ocaml
[@bs.send] external sort: (array('a), [@bs.uncurry] ('a, 'a) => int) => array('a) = "sort";

let _ = sort(arr, compare);
```

For a mutating method, it's traditional to pass the instance argument first.

Note: `compare` is a function provided by the standard library, which fits the defined interface of JavaScript's comparator function.

## Null and undefined

### foo.bar === undefined // check for undefined

```ocaml
[@bs.get] external bar: t => option(int) = "bar";

switch (Foo.bar(foo)) {
| Some(value) => ...
| None => ...
}
```

If you know some value may be `undefined` (but not `null`, see next section), and if you know its type is monomorphic (i.e. not generic), then you can model it directly as an `option(...)` type.

Ref: https://reasonml.org/docs/reason-compiler/latest/null-undefined-option

### foo.bar == null // check for null or undefined

```ocaml
[@bs.get] [@bs.return nullable] external bar: t => option(t);

switch (Foo.bar(foo)) {
| Some(value) => ...
| None => ...
}
```

If you know the value is 'nullable' (i.e. could be `null` or `undefined`), or if the value could be polymorphic, then `[@bs.return nullable]` is appropriate to use.

Note that this attribute requires the return type of the binding to be an `option(...)` type as well.
