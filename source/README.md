# euwren

> The [Eurasian wren](https://en.wikipedia.org/wiki/Wren) has been long
> considered "the king of birds" in Europe.

euwren (pronounced _oyren_, like _euro_ in German) is a high-level
[Wren](https://github.com/wren-lang/wren) wrapper for Nim. Wren is a small,
fast, embedded scripting language.

The main point of euwren is to create a very user-friendly, high-level wrapper:
"The king of Wren wrappers". It leverages Nim's powerful macro system to make
the API as simple as listing all the things you need in Wren. While it may not
be the fastest of wrappers, it's not the primary goal. It's the end user
experience that really counts.

## Features

- Syntactically simple
- Supports proc, object, and enum binding
- Does type checks for procedures
- Supports operator overloading
- Automatically generates Wren glue code with declarations

## Installing

### Adding to your .nimble file
```nim
requires "euwren"
```

### Installing directly from the command line
```bash
$ nimble install euwren
```

## Usage

Because Nim and Wren have different programming paradigms, some work must be
done by the programmer. Fortunately, what needs to be done is very simple, so
don't worry.

### Running code

First, a VM instance must be created.
```nim
import euwren

var wren = newWren()
```
After that, running code is as simple as:
```nim
# run() runs code in the 'main' module
# it's an alias to wren.module("main", <code>)
wren.run("""
  System.print("Hello from Wren!")
""")
```

### Retrieving variables

To get a primitive variable from Wren, use the subscript operator with three
arguments.
```nim
wren.run("""
  var myInt = 2
""")

               # module  name     type
let myInt = wren["main", "myInt", int]
assert myInt == 2
```

Any number/enum type conversions between Nim and Wren are performed
automatically.

To retrieve a Wren object, eg. a class, use the subscript operator with two
arguments.
```nim
wren.run("""
  class Program {
    static run() {
      System.print("Hello from inside the class!")
    }
  }
""")

let classProgram = wren["main", "Program"]
```

### Calling methods

Calling methods on Wren objects is done by first obtaining a call handle, and
then calling the method.

To obtain a call handle, use the curly brace operator. Then, to call the
method, use `call()`.
```nim
# the convention for naming the variable:
# method<name><number of arguments>
# this convention is the preferred naming conventions for variables and fields
# that store Wren call handles, but you're free to use any convention you want
let methodRun0 = wren{"run()"}
# the second parameter is the call handle to the method, the third is the
# receiver of the method, and the rest is the parameters to pass to
# the method.
# when the method is static, the receiver is the class of the method
wren.call(methodRun0, classProgram)
```

### Configuring the VM

The VM can be configured in a few different ways, presented in following
paragraphs.

#### Redirecting output

By default, the VM writes to `stdout`. Sometimes (eg. in a game), you may have
a dedicated GUI console, and you'd like to redirect the VM output there.

To do this, you can set the `onWrite` callback.
```nim
var vmOut = ""

wren.onWrite do (text: string):
  # ``text`` contains the text the VM wants to output.
  # let's redirect it to ``vmOut``.
  vmOut.add(text)

wren.run("""
  System.print("Testing output!")
""")
assert vmOut == "Testing output!\n"
```

#### Controlling imports

By default, `import` cannot be used in scripts. It will throw an error, because
there is no default implementation for imports.

To make imports work, you must set the `onLoadModule` callback in the VM:
```nim
import os

wren.onLoadModule do (name: string) -> string:
  # onLoadModule must return the source code of the module called ``name``.
  # a usual implementation that'd allow for loading from files would look
  # like so:
  result = readFile(name.addFileExt("wren"))
```

Sometimes, you may want to transform the module name before importing. That's
where `onResolveModule` comes in:
```nim
import os

wren.onResolveModule do (importer, name: string) -> string:
  # this is a standard way of implementing relative imports:
  result = importer/name
```
Apart from this, if your callback returns an empty string, an error will be
raised saying that the module does not exist.
```nim
import os

wren.onResolveModule do (importer, name: string) -> string:
  result = importer/name
  if not fileExists(result.addFileExt("wren")):
    result = "" # module does not exist.
```

### Binding procs

Wren is strictly class-based, but Nim is not—that means that any procs passed to
Wren must be nested inside a class. Fortunately, that's pretty simple.

```nim
proc hello() =
  echo "Hello from Nim!"

wren.foreign("nim"):
  # create a namespace 'Nim' that will hold our proc
  [Nim]:
    # bind the proc 'hello'
    hello
# ready() must be called to ready the VM for code execution after any
# foreign() calls. this arms the VM to do code execution with foreign type
# checking. no calls to foreign() should be done after you call this!
wren.ready()
```
```js
import "nim" for Nim
Nim.hello() // Output: Hello from Nim!
```
Here's a more advanced example:
```nim
proc add(a, b: int): int = a + b
proc add(a, b, c: int): int = a.add(b).add(c)
proc subtract(a, b: int): int = a - b

# foreign() accepts the name of the module we want to bind
wren.foreign("math"):
  # we create a namespace 'Math' for our procs
  [Math]:
    # procs can be overloaded by arity, but not by parameter type
    # (this is not enforced, so be careful!)
    add(int, int)
    add(int, int, int)
    # procs can be aliased on the Wren side
    subtract -> sub
wren.ready()
```
```js
import "math" for Math
System.print(Math.add(2, 2)) // Output: 4
```

If a proc has a parameter without a type, but with a default parameter that is
not a literal, you must use `_`:
```nim
var myNumber = 2
proc annoyingProc(a = myNumber) = discard

wren.foreign("wildcard"):
  [Wildcard]:
    annoyingProc(_)
```
This is due to a limitation in Nim. For some reason, default parameters are
untyped, which doesn't let the overload resolution compare the types properly.
This is fine, unless you use a non-literal type for the default parameter, eg.
a variable or `seq`. You don't have to do this if the type is specified
explicitly, like here:
```nim
proc notSoAnnoyingProc(a: int = myNumber) = discard
```

New procedures may be created directly within a class binding. This is referred
to as an *inline* declaration:
```nim
wren.foreign("inline"):
  [Inline]:
    sayHi do ():
      # note that ``do ():`` **must** be used in this case, since ``do:`` is
      # just syntax sugar over a regular ``:`` block. regular blocks are
      # reserved for parameter type-based overloads, which may be implemented
      # in the future.
      echo "Hi!"
    # getters can also be declared this way, and accept no parameters
    ?pi do -> float: 3.14159265
```

Any exceptions raised from Nim procedures will abort the fiber instead of
crashing the program:
```nim
proc oops() =
  raise newException(Defect, "oops! seems you called oops().")

wren.foreign("errors"):
  [Error]:
    oops
wren.ready()
```
```js
import "errors" for Error
// we can use the standard Wren error handling mechanisms to catch our error.
var theError = Fiber.new {
  Error.oops()
}.try()
System.print(theError)
```
```
oops! seems you called oops(). [Defect]
```
When `compileOption("stacktrace") == on`, a stack trace pointing to
where the error was raised will be printed.

Nim procedures can accept `WrenRef` as arguments. This allows Wren objects to
be passed to Nim:
```nim
# this example also demonstrates a way of passing callbacks from Wren to Nim,
# but this works for any Wren type (eg. classes)
var onTickFn: WrenRef

proc onTick(callback: WrenRef) =
  onTickFn = callback

wren.foreign("engine"):
  [Engine]:
    onTick
wren.ready()

wren.run(code)

let methodCall0 = wren{"call()"}
wren.call(methodCall0, onTickFn)
```
```js
import "engine" for Engine
Engine.onTick {
  System.println("Hello from callback!")
}
```

Note that the Wren VM **is not reentrant**, meaning you cannot call Wren in
a foreign method.

### Binding objects

Binding objects is very similar to procs. All *public* object fields are
exported to Wren.

If a proc returns an object, the class for that object must be declared *before*
the proc is declared.

```nim
type
  Foo = object
    name*: string
    count: int

proc initFoo(name: string): Foo =
  result = Foo()
  result.name = name
  result.count = 1

proc more(foo: var Foo) =
  inc(foo.count)

proc count(foo: Foo) = foo.count

wren.foreign("foo"):
  # objects are declared without [] and can be aliased, just like procs
  Foo -> Bar:
    # the * operator makes the bound proc static
    # this is not needed in namespaces, and produces a warning
    *initFoo -> new
    # procs can be bound as usual
    more
    # ? binds the proc for use with getter syntax
    # (``x.count`` instead of ``x.count()``)
    ?count
wren.ready()
```
```js
import "foo" for Bar

var foo = Bar.new("Thing")
foo.more()
System.print(foo.count)
```

*Concrete* generic types are supported:

```nim
import strformat

type
  Vec2[T] = object
    x*, y*: float

proc vec2[T](x, y: T): T =
  result = Vec2[T](x: x, y: y)

proc `+`[T](a, b: Vec2[T]): Vec2[T] =
  result = Vec2[T](x: a.x + b.x, y: a.y + b.y)

proc `$`[T](a: Vec2[T]): string =
  result = fmt"[{a.x} {a.y}]"

wren.foreign("concrete_generic"):
  # concrete generic types *must* be aliased
  Vec2[float] -> Vec2f:
    # you must fill any generic types on procs
    # failing to do so will yield in a compile error, which is not caught
    # by euwren (yet)
    *vec2(float, float) -> new
    `+`(Vec2[float], Vec2[float])
    `$`(Vec2[float])
```
```js
import "concrete_generic" for Vec2f

var a = Vec2f.new(10, 10)
var b = Vec2f.new(20, 30)
var c = a + b

System.print(c) // [30 40]
```

### Binding enums

```nim
type
  Fruit = enum
    fruitApple
    fruitBanana
    fruitGrape
  MenuOpt = enum
    optStart
    optHelp
    optExit
  ProgLanguage = enum
    langNim
    langWren
    langC

wren.foreign("enums"):
  # enums are bound by not specifying a body
  Fruit
  # the conventional prefix can be stripped by using ``-``
  MenuOpt - opt
  # enums can also be aliased
  ProgLanguage - lang -> Lang
wren.ready()
```
The generated class includes all the values of an enum, and additionally,
`low` and `high` for utility purposes. This also means that you should refrain
from naming your enums in `snake_case`, as your names may clash with the
built-in `low` and `high` properties. An option may be added in the future to
automatically convert your enum to `PascalCase`.

Here's an example of a generated module, based on the above input:
```js
class Fruit {
  static fruitApple { 0 }
  static fruitBanana { 1 }
  static fruitGrape { 2 }
  static low { 0 }
  static high { 2 }
}
class MenuOpt {
  static Start { 0 }
  static Help { 1 }
  static Exit { 2 }
  static low { 0 }
  static high { 2 }
}
class Lang {
  static Nim { 0 }
  static Wren { 1 }
  static C { 2 }
  static low { 0 }
  static high { 2 }
}
```
```js
import "enums" for Fruit, MenuOpt, Lang

System.print(Fruit.fruitGrape) // 2
System.print(MenuOpt.Start) // 0
System.print(Lang.Wren) // 1
```

### Compile-time flags
- `-d:euwrenDumpForeignModule` – dumps the Wren module source code generated by
  `foreign()` upon runtime. Useful for debugging code generation.
- `-d:euwrenDumpClasses` – dumps a list of all bound classes, in the order
  they're bound in `foreign()`.

### Gotchas

- Three extra macros called `addProcAux`, `addClassAux`, and `genEnumAux` are
  exposed in the public API. **Do not use them in your code.** They are used
  internally by `foreign()`, and they make the DSL possible by deferring all
  binding to the semantic phase. There are lots of implementation details here,
  feel free to read the source code if you're interested.
- Currently, euwren uses a fork of Wren that fixes an issue related to slots
  in the VM. This fork is not the same as the current stable version of Wren,
  but it will be used until [Wren/#712](https://github.com/wren-lang/wren/pull/712)
  is merged.
- The generated glue code is assembled at run time, which is inefficient and
  possibly slows binding down. This will be fixed in a future release.

