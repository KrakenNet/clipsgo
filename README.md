# CLIPSgo Bindings for CLIPS in the Go Language

A Go wrapper for CLIPS, inspired by [clipspy](https://clipspy.readthedocs.io/en/latest/) and implemented using [cgo](https://golang.org/cmd/cgo/). Forked from [mattmsi](https://github.com/mattsmi/clipsgo)

Instructions and examples updated from those found in the original package written by Kris Raney (_Keysight_). All examples are no longer code-snippets but complete Go programs, which you
can run and test at home. Provided by mattmsi

The original repository can be found [here](https://github.com/Keysight/clipsgo).

Updated to include current [documentation](https://github.com/KrakenNet/clipsgo/docs/clips) and to update everything.

## Design

CLIPSgo attempts to follow closely the model defined by clipspy. Anyone familiar with clipspy should find clipsgo fairly straightforward to understand. That said there are a few notable areas of differences.

- Because Go is type safe, some APIs work differently out of necessity
- clipsgo adds a `SendCommand` call inspired by [pyclips](http://pyclips.sourceforge.net/web/?q=node/13), enabling an interactive shell.
- clipsgo includes a `Shell()` API call based on this that opens an interactive shell, and includes readline-style history and syntax highlighting
- clipsgo builds into an executable that simply acts as an interactive CLIPS shell
- clipsgo may also form part of your own Go language project that uses the CLIPS rules engine.

## Interactive use

You may run clipsgo directly in order to use an interactive shell.

![shell](assets/interactive.png)

Also, in your clipsgo-based programs, you may set up an environment that includes Go-based functions and/or is preloaded with data, then open an interactive session within that environment.

```go
func main() {
	env := clips.CreateEnvironment()
	// ... modify to your liking
	env.Shell()
}
```

## Data Types

CLIPS data types are mapped to GO types as follows

| CLIPS            | Go                 |
| ---------------- | ------------------ |
| INTEGER          | int64              |
| FLOAT            | float64            |
| STRING           | string             |
| SYMBOL           | clips.Symbol       |
| MULTIFIELD       | []interface{}      |
| FACT_ADDRESS     | clips.Fact         |
| INSTANCE_NAME    | clips.InstanceName |
| INSTANCE_ADDRESS | clips.Instance     |
| EXTERNAL_ADDRESS | unsafe.Pointer     |

## Basic Data Abstractions

For detailed information about CLIPS see the [CLIPS
documentation](http://www.clipsrules.net/Documentation.html)

While most of the CLIPS documentation starts from _facts_ and _templates_,
you may also build your rules and
"facts" based on classes and instances, instead of templates and facts.
Everything you can do with facts, you can do with instances, but the reverse
is not true. Most programmers are likely to be familiar with the
object-oriented concepts of [CLIPS Object Oriented Language
(COOL)](https://www.csie.ntu.edu.tw/~sylee/courses/clips/bpg/node9.html) and
in fact may find them more familiar and comfortable than the older, more
procedural base language.

### Instances

Instances are instantiations of specific classes. They store values by name, similar to Go maps. They also support inheritance and instance methods using message sending.

```go
package main

import (
	"testing"

	"github.com/mattsmi/clipsgo/pkg/clips"
	"gotest.tools/assert"
)

func main() {

	t := &testing.T{}

	env := clips.CreateEnvironment()
	defer env.Delete()

	err := env.Build(`(defclass Foo (is-a USER)
	        (slot bar
	            (type INTEGER))
	        (multislot baz)
	)`)
	assert.NilError(t, err)
	err = env.Build(`(defmessage-handler Foo handler ()
	        (printout t "bar=" ?self:bar crlf)
	)`)
	assert.NilError(t, err)

	inst, err := env.MakeInstance(`(of Foo (bar 12))`)
	assert.NilError(t, err)

	ret, err := inst.Slot("bar")
	assert.NilError(t, err)
	assert.Equal(t, ret, int64(12))

	ret = inst.Send("handler", "")
}


```

#### Insert

A user-defined struct may be "inserted" as a class and/or instance in
CLIPS. The term "Insert" is taken from DROOLS, although unlike DROOLS
no long-term link between the user struct and the CLIPS instance
is retained. The data is simply copied in.

A nil pointer to a struct type is sufficient to insert a class. Inserting a
class will result in building a defclass construct in CLIPS that represents
the fields of the struct. If the struct contains structs, or pointers to
structs, these will become slots of type INSTANCE-NAME, and the struct
that is referred will also be inserted.

```go
package main

import (
	"fmt"
	"testing"

	"github.com/mattsmi/clipsgo/pkg/clips"
	"gotest.tools/assert"
)

func main() {

	t := &testing.T{}

	env := clips.CreateEnvironment()
	defer env.Delete()

	type ChildClass struct {
		Intval   *int
		Floatval *float64
	}
	type ParentClass struct {
		Str   string
		Child ChildClass
	}
	var template *ParentClass

	cls, err := env.InsertClass(template)
	assert.NilError(t, err)

	/* output should appear as follows.
	   (defclass MAIN::ParentClass
			(is-a USER)
			(slot Str
			   (type STRING))
			(slot Child
			   (type INSTANCE-NAME)
			   (allowed-classes ChildClass)))
	*/
	fmt.Println(cls.String())
}

```

When an instance is inserted, a class for that data type will implicitly be
inserted if no class by that name already exists. If a class already exists,
it will be used as-is (and may not match the fields of the given data,
causing an error, if it was created by some other means.)

```go
package main

import (
	"testing"

	"github.com/KrakenNet/clipsgo/pkg/clips"
	"gotest.tools/assert"
)

func main() {

	t := &testing.T{}

	env := clips.CreateEnvironment()
	defer env.Delete()

	type ChildClass struct {
		Intval   *int
		Floatval *float64
	}
	type ParentClass struct {
		Str   string
		Child ChildClass
	}
	intval := 99
	floatval := 107.0
	template := ParentClass{
		Str: "with actual value",
		Child: ChildClass{
			Intval:   &intval,
			Floatval: &floatval,
		},
	}

	inst, err := env.Insert("", template)
	assert.NilError(t, err)
	assert.Equal(t, inst.String(), `[gen1] of ParentClass (Str "with actual value") (Child [gen2])`)

	subinst, err := env.FindInstance("gen2", "")
	assert.NilError(t, err)
	assert.Equal(t, subinst.String(), `[gen2] of ChildClass (Intval 99) (Floatval 107.0)`)
}



```

#### Extract

An instance can also be "extracted" as either a struct or a map. This
functionality is somewhat analogous to json.Unmarshal. The caller supplies an
object which will be filled in by clipsgo.

Matching the name of the struct field to the slot name is determined by the
following rules:

- If the struct field has a "clips" tag, that tag is used as the slot name.
- Otherwise, if the struct has a "json" tag, that tag is used.
- Otherwise the field name of the struct is used.

Note that instances are extracted recursively; if a slot in the instance is
an INSTANCE-ADDRESS or INSTANCE-NAME, the referred instance will also be
extracted as structured data

```go
package main

import (
	"testing"

	"github.com/KrakenNet/clipsgo/pkg/clips"
	"gotest.tools/assert"
)

func main() {

	t := &testing.T{}

	env := clips.CreateEnvironment()
	defer env.Delete()

	err := env.Build(`(defclass Foo (is-a USER)
    (slot Int (type INTEGER))
    (slot Float (type FLOAT))
    (slot Sym (type SYMBOL))
    (multislot MS))
`)
	assert.NilError(t, err)

	inst, err := env.MakeInstance(`(of Foo (Int 12) (Float 28.0) (Sym bar) (MS a b c))`)
	assert.NilError(t, err)

	type Foo struct {
		IntVal    int     `json:"Int"`
		FloatVal  float64 `clips:"Float"`
		Sym       clips.Symbol
		MultiSlot *[]interface{} `json:"MS,omitempty"`
	}

	var retval Foo
	err = inst.Extract(&retval)

	output := Foo{
		IntVal:   12,
		FloatVal: 28.0,
		Sym:      clips.Symbol("bar"),
		MultiSlot: &[]interface{}{
			clips.Symbol("a"),
			clips.Symbol("b"),
			clips.Symbol("c"),
		},
	}

	assert.DeepEqual(t, retval, output)
}


```

### Facts

A _fact_ is a list of atomic values that are either referenced positionally, for "ordered" or "implied" facts, or by name for "unordered" or "template" facts.

#### Ordered Facts

Ordered Facts represent information as a list of elements. There is no explicit template for an ordered fact, but they do have an implied template. A reference to the implied template of an ordered fact can be obtained, and can be used to programmatically assert ordered facts.

```go
package main

import (
	"fmt"
	"testing"

	"github.com/KrakenNet/clipsgo/pkg/clips"
	"gotest.tools/assert"
)

func main() {

	t := &testing.T{}

	env := clips.CreateEnvironment()
	defer env.Delete()

	fact, err := env.AssertString(`(foo a b c)`)
	if err != nil {
		fmt.Println("Problem asserting fact in CLIPS.")
	}
	defer fact.Drop()

	tmpl := fact.Template()
	assert.Assert(t, tmpl.Implied())
	fact, err = tmpl.NewFact()

	ifact, ok := fact.(*clips.ImpliedFact)
	assert.Assert(t, ok)

	ifact.Append("a")
	ifact.Extend([]interface{}{
		clips.Symbol("b"),
		3,
	})

	ifact.Set(2, "c")
	ifact.Assert()
}

```

Similar to instances, an ordered fact may be extracted to a user-provided structure. Extracted ordered facts will translate to a slice of interface. The user may use a more specific slice, in which case the slice will be automatically converted (if possible - note that an unordered fact need not have all values of the same type, which
would likely lead to errors.)

```go
package main

import (
	"fmt"
	"testing"

	"github.com/KrakenNet/clipsgo/pkg/clips"
	"gotest.tools/assert"
)

func main() {

	t := &testing.T{}

	env := clips.CreateEnvironment()
	defer env.Delete()

	fact, err := env.AssertString(`(foo a b c)`)
	if err != nil {
		fmt.Println("Problem asserting fact in CLIPS.")
	}
	defer fact.Drop()

	tmpl := fact.Template()
	assert.Assert(t, tmpl.Implied())
	fact, err = tmpl.NewFact()

	ifact, ok := fact.(*clips.ImpliedFact)
	assert.Assert(t, ok)

	ifact.Append("a")
	ifact.Extend([]interface{}{
		clips.Symbol("b"),
		3,
	})

	ifact.Set(2, "c")
	ifact.Assert()
}

```

#### Template Facts

Unordered facts represent data similar to Go maps. They require a template to be defined, which provides a formal definition for what data is represented by the fact.

```go
package main

import (
	"testing"

	"github.com/mattsmi/clipsgo/pkg/clips"
	"gotest.tools/assert"
)

func main() {

	t := &testing.T{}

	env := clips.CreateEnvironment()
	defer env.Delete()

	env.Build("(deftemplate foo (slot bar) (multislot baz))")

	tmpl, err := env.FindTemplate("foo")
	assert.NilError(t, err)
	fact, err := tmpl.NewFact()
	assert.NilError(t, err)

	tfact, ok := fact.(*clips.TemplateFact)
	assert.Assert(t, ok)

	tfact.Set("bar", 4)
	tfact.Set("baz", []interface{}{
		clips.Symbol("b"),
		3,
	})
	tfact.Assert()
}

```

Template facts may be extracted to structs or to a map

```go
package main

import (
	"fmt"
	"testing"

	"github.com/mattsmi/clipsgo/pkg/clips"
	"gotest.tools/assert"
)

func main() {

	t := &testing.T{}

	env := clips.CreateEnvironment()
	defer env.Delete()

	env.Build("(deftemplate foo (slot bar) (multislot baz))")

	tmpl, err := env.FindTemplate("foo")
	assert.NilError(t, err)
	fact, err := tmpl.NewFact()
	assert.NilError(t, err)

	tfact, ok := fact.(*clips.TemplateFact)
	assert.Assert(t, ok)

	tfact.Set("bar", 4)
	tfact.Set("baz", []interface{}{
		clips.Symbol("b"),
		3,
	})
	tfact.Assert()

	/* now extract */
	var mapvar map[string]interface{}
	err = fact.Extract(&mapvar)
	assert.NilError(t, err)
	fmt.Println(mapvar)
	/* the contents should look similar to
	map[string]interface{}{
		"bar": int64(4),
		"baz": []interface{}{
			clips.Symbol("b"),
			3,
		},
	}
	*/
}

```

## Evaluating CLIPS code

It is possible to evaluate CLIPS statements, retrieving their results in Go.

### Eval

```go
import (
    "github.com/mattsmi/clipsgo/pkg/clips"
)

env := clips.CreateEnvironment()
defer env.Delete()

ret, err := env.Eval("(create$ foo bar baz)")
```

_Note that this functionality relies on CLIPS `eval` function, which does not accept aribtrary commands. It does not allow CLIPS constructs to be created for example - for that you need `env.Build()`. It also does not allow facts to be asserted - use `env.AssertString()`_

The return from Eval is interface{}, since it is not possible to know in advance what kind of data will be returned.

For cases where the user is able to know an expected return type, an ExtractEval function is provided. This will marshall the returned data into an object provided by the user. Using ExtractEval can reduce the amount of boilerplate type checking required. Type conversions are applied as necessary. An error will be generated if a numeric type conversion results in loss of precision.

```go
package main

import (
	"testing"

	"github.com/mattsmi/clipsgo/pkg/clips"
	"gotest.tools/assert"
)

func main() {

	t := &testing.T{}

	env := clips.CreateEnvironment()
	defer env.Delete()

	intval := 4
	err := env.ExtractEval(&intval, "12")
	assert.NilError(t, err)
	assert.Equal(t, intval, 12)

	sliceval := make([]string, 0)
	err = env.ExtractEval(&sliceval, "(create$ a b c d e f)")
	assert.NilError(t, err)
	assert.DeepEqual(t, sliceval, []string{
		"a",
		"b",
		"c",
		"d",
		"e",
		"f",
	})

	var factval clips.Fact
	err = env.ExtractEval(&factval, "(bind ?ret (assert (foo a b c)))")
	assert.NilError(t, err)
	assert.Equal(t, factval.String(), "(foo a b c)")
}

```

### SendCommand

In order to overcome some of the limitations of the CLIPS `eval` command, clipsgo provides a higher-level function called `SendCommand` which accepts any arbitrary CLIPS command.

```go
package main

import (
	"testing"

	"github.com/KrakenNet/clipsgo/pkg/clips"
	"gotest.tools/assert"
)

func main() {

	t := &testing.T{}

	env := clips.CreateEnvironment()
	defer env.Delete()

	// try some stuff that Eval chokes on
	err := env.SendCommand("(assert (foo a b c))")
	assert.NilError(t, err)
}

```

This avoids the need to know in advance which call to make. On the other hand, no return value is provided; `SendCommand` is primarily intended for more interactive evaluation of unpredictable input.

## Defining CLIPS Constructs

CLIPS constructs must be defined in CLIPS language. Use the `Load()` or `Build()` functions to define them.

```go
package main

import (
	"testing"

	"github.com/KrakenNet/clipsgo/pkg/clips"
	"gotest.tools/assert"
)

func main() {

	t := &testing.T{}

	env := clips.CreateEnvironment()
	defer env.Delete()

	err := env.Build(`
(defrule my-rule
    (my-fact first-slot)
=>
    (printout t "My Rule fired!" crlf)
)`)
	assert.NilError(t, err)
}

```

## Embedding Go

The `DefineFunction()` method allows binding a Go function within the CLIPS environment. It will be callable from within CLIPS using the given name as though it had been defined with the `deffunction` construct.

Clipsgo will attempt to marshall functions passed from CLIPS to the correct types to match the function. If it cannot, CLIPS will error on the attempted function call.

Each argument must be one of types listed as equivalent to one of the CLIPS data types, with one notable exception - it is acceptable to use lower-scale types like int, int8, or float32. Clipsgo will automatically convert, or will return an error if the number is too large for the conversion.

Any number of return values is supported. If the last return value of the funciton is an error type, clipsgo will interpret it as an error and not include it in the function return values. A single non-error value will be returned directly, more than that will return a multifield.

Variadic functions are also supported.

```go
package main

import (
	"testing"

	"github.com/mattsmi/clipsgo/pkg/clips"
	"gotest.tools/assert"
)

func main() {

	t := &testing.T{}

	env := clips.CreateEnvironment()
	defer env.Delete()

	callback := func(foo int, bar float64,
		multifield []interface{},
		vals ...clips.Symbol) (bool, error) {
		return true, nil
	}

	err := env.DefineFunction("test-callback", callback)
	assert.NilError(t, err)

	_, err = env.Eval("(test-callback 1 17.0 (create$ a b c) a b c d e f)")
	assert.NilError(t, err)
}

```

## Go Reference Objects Lifecycle

All of the Go objects created to interact with the CLIPS environment are simple references to the CLIPS data structure. This means that interactions with the CLIPS shell can cause them to become invalid. In most cases, deleting or undefining an object makes any Go reference to it unusable.

```go
package main

import (
	"fmt"

	"github.com/mattsmi/clipsgo/pkg/clips"
)

func main() {

	env := clips.CreateEnvironment()
	defer env.Delete()

	templates := env.Templates()
	env.Clear() // From here, all templates are gone so all references are unusable

	// this will cause an error and/or print rubbish
	for _, tmpl := range templates {
		fmt.Printf("%v\n", tmpl)
	}
}

```

### Building From Sources

The build requires the CLIPS source code to be available, and to be built into a shared library. The  Makefile provided makes this simple. However, because clipsgo requires the CLIPS source code and shared library to be in place to run, we must build these before using clipsgo as part of any Go code.

The following instructions will build a Go executable that provides a CLIPS shell. Along the way,
it will handily place the CLIPS source and shared library (.so) in the places necessary for clipsgo
to use them, when being called from Go code.

1. Copy this repository to disk through either of the following methods.
   1. clone this repository to some location on disk, or
   2. click on the green Code button and choose to copy the ZIP to disk. Unzip it into a directory.
2. Change directory into repository.
   1. If cloned, `cd clipsgo`.
   2. If unzipped, `cd clipsgo-master`.
3. Execute the following commands to build the CLIPS shared library and then to build clipsgo.

```bash
make clips_all
sudo make install-clips
make just_clipsgo
```

There are also targets for `test` and `coverage` in the Makefile to run the test suite 
(i.e. `make test` and `make coverage`).

The `make` will result in an executable that creates a CLIPS shell much like the CLIPS software itself.

To build a Go project that calls or uses clipsgo, you will need to use a command such as the following:
`go build -ldflags "-r /usr/local/lib" .` . Note the location for the shared libraries is that used in the _make_ file. Change it as needed.

For example, assume you have a module resident in a directory called "go_clips_test", the following
will set the environment and then build (i.e. compile) the module to an executable, which
will be called "go_clips_test".

```bash
go mod init go_clips_test
go mod tidy
go get go_clips_test
go build -ldflags "-r /usr/local/lib" .
```

## Reference documentation

The code has [Godoc](https://godoc.org/golang.org/x/tools/cmd/godoc) throughout. To view them, run

```bash
$ godoc -http=:6060
```

Then access http://localhost:6060/ from your browser


# Roadmap
* Bring all updates from CLIPS into consideration
* Update to include http/server bindings for routers