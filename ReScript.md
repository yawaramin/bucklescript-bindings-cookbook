**ReScript - BuckleScript Bindings Cookbook**

WRITING ReScript bindings can be somewhere between an art and a science, taking some learning investment into both the JavaScript and OCaml/ReScript type systems to get a proper feel for it.

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
  - [const foo = require('foo'); foo.bar.baz() // function scoped inside an object in a module](#const-foo--requirefoo-foobarbaz--function-scoped-inside-an-object-in-a-module)
- [Functions](#functions)
  - [const dir = path.join('a', 'b', ...) // function with rest args](#const-dir--pathjoina-b---function-with-rest-args)
  - [const nums = range(start, stop, step) // call a function with named arguments for readability](#const-nums--rangestart-stop-step--call-a-function-with-named-arguments-for-readability)
  - [foo('hello'); foo(true) // overloaded function](#foohello-footrue--overloaded-function)
  - [const nums = range(start, stop, [step]) // optional final argument(s)](#const-nums--rangestart-stop-step--optional-final-arguments)
  - [mkdir('src/main', {recursive: true}) // options object argument](#mkdirsrcmain-recursive-true--options-object-argument)
    - [Alternative way](#alternative-way)
  - [forEach(start, stop, item => console.log(item)) // model a callback](#foreachstart-stop-item--consolelogitem--model-a-callback)
- [Objects](#objects)
  - [const person = {id: 1, name: 'Bob'} // create an object](#const-person--id-1-name-bob--create-an-object)
  - [person.name // get a prop](#personname--get-a-prop)
  - [person.id = 0 // set a prop](#personid--0--set-a-prop)
  - [const {id, name} = person // object with destructuring](#const-id-name--person--object-with-destructuring)
- [Classes and OOP](#classes-and-oop)
  - [I don't see what I need here](#i-dont-see-what-i-need-here)
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

```rescript
@bs.val external window: Dom.window = "window"
```

Ref https://rescript-lang.org/docs/manual/latest/bind-to-global-js-values

### window? // does global variable exist

```rescript
switch %external(window) {
| Some(_) => "window exists"
| None => "window does not exist"
}

```

`%external(NAME)` makes `NAME` available as an value of type `option<'a>`, meaning its wrapped value is compatible with any type. I recommend that, if you use the value, to cast it safely into a known type first.

Ref https://rescript-lang.org/docs/manual/latest/bind-to-global-js-values#special-global-values

### Math.PI // variable in global module

```rescript
@bs.scope("Math") @bs.val external pi: unit => float = "pi"
```

Ref https://rescript-lang.org/docs/manual/latest/bind-to-global-js-values#global-modules

### console.log // function in global module

```rescript
@bs.scope("console") @bs.val external log: 'a => unit = "log"
```

Note that in JavaScript, `console.log()`'s return value is `undefined`, which we can model with the `unit` type.

## Modules

### const path = require('path'); path.join('a', 'b') // function in CJS/ES module

```rescript
@bs.module("path") external join: (string, string) => string = "join"
let dir = join("a", "b")
```

Ref: https://rescript-lang.org/docs/manual/latest/import-from-export-to-js

### const foo = require('foo'); foo(1) // import entire module as a value

```rescript
@bs.module external foo: int => unit = "foo"
let () = foo(1)
```

Ref: https://rescript-lang.org/docs/manual/latest/import-from-export-to-js#import-a-javascript-module-itself-commonjs

### import foo from 'foo'; foo(1) // import ES6 module default export

```rescript
@bs.module("foo") external foo: int => unit = "default"
let () = foo(1)
```

Ref: https://rescript-lang.org/docs/manual/latest/import-from-export-to-js#import-a-javascript-module-itself-es6-module-format

### const foo = require('foo'); foo.bar.baz() // function scoped inside an object in a module

```rescript
module Foo = {
  module Bar = {
    @bs.module("foo") @bs.scope("bar") external baz: unit => unit = "baz"
  }
}

let () = Foo.Bar.baz()
```

It's not necessary to nest the binding inside ReScript modules, but mirroring the structure of the JavaScript module layout does make the binding more discoverable.

Note that `@bs.scope` works not just with `@bs.module`, but also with `@bs.val` (as shown earlier), and with combinations of `@bs.module`, `@bs.new` (covered in the OOP section), etc.

**Tip:** the `@bs.scope(...)` attribute supports an arbitrary level of scoping by passing the scope as a tuple argument, e.g. `@bs.scope(("a", "b", "c"))`.

## Functions

### const dir = path.join('a', 'b', ...) // function with rest args

```rescript
@bs.module("path") @bs.variadic external join: array<string> => string = "join"
let dir = join(["a", "b"])
```

Note that the rest args must all be of the same type for `@bs.variadic` to work. If they really have different types, then more advanced techniques are needed.

Ref: https://rescript-lang.org/docs/manual/latest/interop-cheatsheet#variadic-arguments

### const nums = range(start, stop, step) // call a function with named arguments for readability

```ocaml
@bs.val external range: (~start: int, ~stop: int, ~step: int) => array<int> = "range"
let nums = range(~start=1, ~stop=10, ~step=2)
```

### foo('hello'); foo(true) // overloaded function

```rescript
@bs.val external fooString: string => unit = "foo"
@bs.val external fooBool: bool => unit = "foo"

fooString("")
fooBool(true)
```

Because BuckleScript bindings allow specifying the name on the ReScript side and the name on the JavaScript side (in quotes) separately, it's easy to bind multiple times to the same function with different names and signatures. This allows binding to complex JavaScript functions with polymorphic behaviour.

### const nums = range(start, stop, [step]) // optional final argument(s)

```rescript
@bs.val external range: (~start: int, ~stop: int, ~step: int=?, unit) => array<int> = "range"
let nums = range(~start=1, ~stop=10, ())
```

If a ReScript function or binding has an optional parameter, it needs a positional parameter at the end of the parameter list to help the compiler understand when function application is finished and the function can actually execute. If this seems tedious, remember that no other language gives you out-of-the-box curried parameters _and_ named parameters _and_ optional parameters.

### mkdir('src/main', {recursive: true}) // options object argument

```rescript
type mkdirOptions
@bs.obj external mkdirOptions: (~recursive: bool=?, unit) => mkdirOptions = ""

@bs.val external mkdir: (string, ~options: mkdirOptions=?, unit) => unit = "mkdir"

// Usage:

let () = mkdir("src", ())
let () = mkdir("src/main", ~options=mkdirOptions(~recursive=true, ()), ())
```

The `@bs.obj` attribute allows creating a function that will output a JavaScript object. There are simpler ways to create JavaScript objects (see OOP section), but this is the only way that allows omitting optional fields like `recursive` from the output object. By making the binding parameter optional (`~recursive: bool=?`), you indicate that the field is also optional in the object.

#### Alternative way

Calling a function like `mkdir("src/main", ~options=..., ())` can be syntactically pretty heavy, for the benefit of allowing the optional argument. But there is another way: binding to the same underlying function twice and treating the different invocations as [overloads](#foohello-footrue--overloaded-function).

```rescript
type mkdirOptions
@bs.obj external mkdirOptions: (~recursive: bool=?, unit) => mkdirOptions = ""
@bs.val external mkdir: string => unit = "mkdir"
@bs.val external mkdirWith: (string, mkdirOptions) => unit = "mkdir"

// Usage:

let () = mkdir("src/main")
let () = mkdirWith("src/main", mkdirOptions(~recursive=true, ()))
```

This way you don't need optional arguments, and no final `()` argument for `mkdirWith`.

Ref: https://rescript-lang.org/docs/manual/latest/bind-to-js-object

### forEach(start, stop, item => console.log(item)) // model a callback

```rescript
@bs.val external forEach: (~start: int, ~stop: int, @bs.uncurry (int => unit)) => unit = "forEach"
forEach(~start=1, ~stop=10, Js.log)
```

When binding to functions with callbacks, you'll want to ensure that the callbacks are uncurried. `@bs.uncurry` is the recommended way of doing that. However, in some circumstances you may be forced to use the static uncurried function syntax. See the docs for details.

Ref: https://rescript-lang.org/docs/manual/latest/bind-to-js-function#extra-solution

## Objects

### const person = {id: 1, name: 'Bob'} // create an object

```rescript
let person = {"id": 1, "name": "Bob"};
```

Ref: https://rescript-lang.org/docs/manual/latest/object

### person.name // get a prop

```rescript
person["name"]
```

Ref: https://rescript-lang.org/docs/manual/latest/object#access

### person.id = 0 // set a prop

```rescript
type student = {@bs.set "age": int, @bs.set "name": string}
@bs.module("MyJSFile") external student1: student = "student1"

student1["name"] = "Mary"
```

You can only update props on objects that come from the JavaScript side.

Ref: https://rescript-lang.org/docs/manual/latest/object#update

### const {id, name} = person // object with destructuring

```rescript
type person = {id: int, name: string};

let person = {id: 1, name: "Bob"};
let {id, name} = person;
```

ReScript record types compile to simple JavaScript objects. But you get the added benefits of pattern matching and immutable update syntax on the Reason side. There are a couple of caveats though:

- The object will contain _all_ defined fields; none will be left out, even if they are optional types
- If you are referring to record fields defined in other modules, you must prefix at least one field with the module name, e.g. `let {Person.id, name} = person`

Ref: https://rescript-lang.org/docs/manual/latest/record

## Classes and OOP

In BuckleScript it's idiomatic to bind to class properties and methods as functions which take the instance as just a normal function argument. So e.g., instead of

```javascript
const foo = new Foo();
foo.bar();
```

You will write:

```rescript
let foo = Foo.make()
let () = Foo.bar(foo)
```

Note that many of the techniques shown in the [Functions](#functions) section are applicable to the instance members shown below.

### I don't see what I need here

Try looking in the [Functions](#functions) section; in ReScript, functions and instance methods can share many of the same binding techniques.

### const foo = new Foo() // call a class constructor

```rescript
// Foo.re
// or,
// module Foo = {

type t;

// The `Foo` at the end must be the name of the class
@bs.new external make: unit => t = "Foo";

//}
...
let foo = Foo.make()
```

Note the abstract type `t`. In BuckleScript you will model any class that's not a [shared data type](https://rescript-lang.org/docs/manual/latest/shared-data-types) as an abstract data type. This means you won't expose the internals of the definition of the class, only its interface (accessors, methods), using functions which include the type `t` in their signatures. This is shown in the next few sections.

A ReScript function binding doesn't have the context that it's binding to a JavaScript class like `Foo`, so you will want to explicitly put it inside a corresponding module `Foo` to denote the class it belongs to. In other words, model JavaScript classes as ReScript modules.

Ref: https://rescript-lang.org/docs/manual/latest/bind-to-js-object#bind-to-a-js-object-thats-a-class

### const bar = foo.bar // get an instance property

```rescript
// In module Foo:
@bs.get external bar: t => int = "bar"
...
let bar = Foo.bar(foo)
```

Ref: https://rescript-lang.org/docs/manual/latest/bind-to-js-object#bind-using-special-bs-getters--setters

### foo.bar = 1 // set an instance property

```ocaml
// In module Foo:
@bs.set external setBar: (t, int) => unit = "bar" // note the name
...
let () = Foo.setBar(foo, 1)
```

### foo.meth() // call a nullary instance method

```rescript
// In module Foo:
[@bs.send] external meth: t => unit = "meth"
...
let () = Foo.meth(foo)
```

Ref: https://rescript-lang.org/docs/manual/latest/bind-to-js-object#bind-using-special-bs-getters--setters

### const newStr = str.replace(substr, newSubstr) // non-mutating instance method

```rescript
@bs.send.pipe(: string)
external replace: (~substr: string, ~newSubstr: string) => string = "replace"

let newStr = replace(~substr, ~newSubstr, str)
```

Note: `@bs.send.pipe` is deprecated in favour of `@bs.send`.

`@bs.send.pipe` injects a parameter of the given type (in this case `string`) as the final positional parameter of the binding. In other words, it creates the binding with the real signature `(~substr: string, ~newSubstr: string, string) => string`. This is handy for non-mutating functions as they traditionally take the instance as the final parameter.

It wasn't strictly necessary to use named arguments in this binding, but it helps readability with multiple arguments, especially if some have the same type.

### arr.sort(compareFunction) // mutating instance method

```rescript
@bs.send external sort: (array<'a>, @bs.uncurry ('a, 'a) => int) => array<'a> = "sort"

let _ = sort(arr, compare)
```

For a mutating method, it's traditional to pass the instance argument first.

Note: `compare` is a function provided by the standard library, which fits the defined interface of JavaScript's comparator function.

## Null and undefined

### foo.bar === undefined // check for undefined

```rescript
@bs.get external bar: t => option<int> = "bar"

switch (Foo.bar(foo)) {
| Some(value) => ...
| None => ...
}
```

If you know some value may be `undefined` (but not `null`, see next section), and if you know its type is monomorphic (i.e. not generic), then you can model it directly as an `option<...>` type.

Ref: https://rescript-lang.org/docs/manual/latest/null-undefined-option

### foo.bar == null // check for null or undefined

```rescript
@bs.get @bs.return(nullable) external bar: t => option<t> = "bar"

switch (Foo.bar(foo)) {
| Some(value) => ...
| None => ...
}
```

If you know the value is 'nullable' (i.e. could be `null` or `undefined`), or if the value could be polymorphic, then `@bs.return(nullable)` is appropriate to use.

Note that this attribute requires the return type of the binding to be an `option<...>` type as well.

Ref: https://rescript-lang.org/docs/manual/latest/bind-to-js-function#function-nullable-return-value-wrapping
