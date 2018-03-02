---
author: campoy
date: 2018-03-06
title: "Calling C functions from BigQuery with Web Assembly"
draft: false
image: /post/c-on-bigquery/banner.png
description: "TODO"
categories: ["culture", "why-i-joined"] 
---

Have you ever used [BigQuery](https://cloud.google.com/bigquery/)?
If you have never used it you're missing out on one of my favorite products of Google Cloud Platform.
It has the power to query terabytes of data in seconds while exposing a familar SQL API.

At source{d} we have terabytes of data, so wouldn't it be cool if we could simply store our datasets on BigQuery
and let anyone easily analyze those?
Well, that's a great idea, but it turns out that our datasets become extra useful when you start analyzing
[Universal Abstract Syntax Trees (UASTs)](https://doc.bblf.sh/uast/specification.html) rather than lines of code.

No problem, you say. Let's store those UASTs and use our querying library [libuast](https://github.com/bblfsh/libuast)
to extract the relevant pieces of those trees. But how could you even do that?
`libuast` is written in `C`, and rewriting to ... `SQL`? That's out of the question!

## BigQuery User Defined Functions

BigQuery supports SQL, and as such, it supports [User Defined Functions (UDFs)](https://cloud.google.com/bigquery/docs/reference/standard-sql/user-defined-functions).
UDFs allow you to define new functions in languages other than SQL, one of them is Javascript.
From the BigQuery console you can click on the `UDF` button and you will see the following screen:

![UDF editor](/post/c-on-bigquery/udf-editor.png)

As you can see, you can write some Javascript in here and use it from your SQL statement.
Let's try to write a simple function that adds two numbers is Javascript. That's what Javascript was created for, right?

This is using Legacy SQL, we'll see later why Standard SQL would make it much harder for us.

```javascript
function identity(row, emit) {
  emit({y: row.x});
}

bigquery.defineFunction(
  'identity',
  ['x'],
  [{'name': 'y', 'type': 'float'}],
  identity
);
```

```sql
SELECT y FROM identity(
  SELECT 21 as x
);
```

This is cool, but can we do it over multiple values?

```javascript
function intdiv(row, emit) {
  emit({d: row.x/row.y, m:row.x%row.y});
}

bigquery.defineFunction(
  'intdiv',
  ['x', 'y'],
  [{'name': 'd', 'type': 'integer'},
   {'name': 'm', 'type': 'integer'}],
  intdiv
);
```

```sql
SELECT d, m FROM intdiv(
  SELECT 21 as x, 4 as y
);
```

Great, so we can basically write any Javascript and have it executed as part of our SQL queries.
Unfortunately for us, the library that I would like to call (libuast) is actually written in C!

How to solve this connundrum? Enter wasm.

## Web Assembly

Web Assembly (a.k.a wasm) is a relatively new technology that proposes an assembly language for the web.
Javascript can be compiled to this assembly, but many other languages, such as C, Rust, and hopefully soon Go,
can also be compiled to highly efficient web assembly code.

Could we maybe compile our C libraries to wasm and then load them from Javascript?
Well, it's technically possible and that's how much I need to give it a try.

### Compiling C

Compiling C programs is an incredibly interesting topic.
Lots can be said, but I will keep it to the basics and talk about what objects are,
how to build them, and how to link them either statically or dynamically.

If you already know all of this feel free to skip this part and move directly to how
to compile C to wasm.

`sum.c`
```C
int sum(int a, int b)
{
  return a + b;
}
```

We can compile this piece file into a library.

```bash
> clang -c sum.c
> file sum.o
sum.o: Mach-O 64-bit object x86_64
```

This

In order to compile I wrote a `main` function that predeclares `sum` and calls it.

```C
#include <stdio.h>

extern int sum(int a, int b);

int main() {
    printf("%d\n", sum(39, 3));
    return 0;
}
```

When we compile and execute this program the output is the answer to everything.

```bash
> clang -o main main.c sum.o
> ./main
42
```

When we compile it in this way we are letting the C linker unify the `sum` declared in
`main.c` and realize that it is the same as the one defined in `sum.c`.
This is static linking, but we could also do something similar with dynamic linking,
letting the binary find and identify `sum` at runtime.


```C
#include <stdio.h>
#include <dlfcn.h>

int main()
{
    int (*sum)(int, int);

    void *dl = dlopen("sum.so", RTLD_LAZY);
    if (!dl)
    {
        fprintf(stderr, "%s\n", dlerror());
        return 1;
    }
    sum = dlsym(dl, "sum");
    printf("%d\n", (*sum)(39, 3));
}
```

```bash
> clang sum.o -shared -o sum.so
> file sum.so
sum.so: Mach-O 64-bit dynamically linked shared library x86_64
> clang -o dynamic dynamic.c
> ./dynamic
```

Try it yourself, what happens if you delete `sum.o` and run `main`?
What if you delete `sum.so` and run `dynamic`?

### Compiling C to wasm

Now that we understand how to dynamically load and execute shared objects,
let's talk about wasm.
In wasm we are going to generate a shared object from `sum.c`, but one that
rather than being destined to be read by C it is for wasm.

The easiest way I've seen to use `emcc` (also known as emscripten) is with
the Docker image docker `apiaryio/emcc`.
If you want to learn more detail on how to use the tool I recommend this [Google Codelab on Web Assembly](https://codelabs.developers.google.com/codelabs/web-assembly-intro).

But for now let's just use it! If you run this command from the directory containing `sum.c`
you will create two new files `sum.js` and `sum.wasm`.

```bash
> docker run --rm -v $(pwd):/src apiaryio/emcc emcc -s WASM=1 -s ONLY_MY
_CODE=1 -s EXPORTED_FUNCTIONS="['_sum']" -o sum.js sum.c
```

`sum.js` contains code to load the object in `sum.wasm`, but we will not use that really.
The one we care about is `sum.wasm`.

Let's now write a `main.js` that loads the `wasm` file and uses its `sum` function.

```javascript
const fs = require('fs');

const memory = new WebAssembly.Memory({ initial: 256, maximum: 256 });
const env = {
    'abortStackOverflow': _ => { throw new Error('overflow'); },
    'table': new WebAssembly.Table({ initial: 0, maximum: 0, element: 'anyfunc' }),
    'tableBase': 0,
    'memory': memory,
    'memoryBase': 1024,
    'STACKTOP': 0,
    'STACK_MAX': memory.buffer.byteLength,
};
const imports = { env };

fs.readFile('sum.wasm', (err, bytes) => {
    WebAssembly.instantiate(bytes, imports).then(wa => {
        const exports = wa.instance.exports;
        const sum = exports._sum;
        console.log(sum(39, 3));
    });
});
```

The code might seem complex, but if you ignore all the code before the call to `fs.readFile`
you'll see that we load `sum.wasm`, extract the function `sum`, and simply call it.
This is quite similar to what we did with C in the previous section.

### So ... does this work on BigQuery?

Being able to call C functions from Javascript is obviously cool.
But that doesn't necessarily solve our problem if `WebAssembly` can't be used on BigQuery!

On my first try, I simply check to see if `WebAssembly` was even defined in the environment.

```javascript
function checkAssembly(row, emit) {
  emit({b: WebAssembly != undefined});
}

bigquery.defineFunction(
  'checkAssembly',
  ['x'],
  [{'name': 'b', 'type': 'boolean'}],
  checkAssembly
);
```

```SQL
SELECT b FROM checkAssembly(
  SELECT 21 as x
);
```

Not sure why I need to define an input to `checkAssembly`, but if I don't I get an error.
When I execute this query the result is `true`. Success, `WebAssembly` is available!


### Uploading sum.wasm

How could I upload `sum.wasm` to BigQuery?
Rather than trying to see whether I can upload the file somewhere (probably Google Cloud Storage), or whether I can fetch it from a remote URL, I decided that the easiest way
was to simply embed the bytes of `sum.asm` into the Javascript code.

How? Well, first of all I wrote a little Go program that generates a Javascript declaration containing the bytes in `sum.wasm`.

```go
package main

import (
	"fmt"
	"io/ioutil"
	"log"
	"os"
	"strings"
)

func main() {
	bs, err := ioutil.ReadAll(os.Stdin)
	if err != nil {
		log.Fatal(err)
	}
	b := new(strings.Builder)
	fmt.Fprint(b, bs[0])
	for _, c := range bs[1:] {
		fmt.Fprintf(b, ", %d", c)
	}

	fmt.Printf("const bytes = new Uint8Array([%s]);", b)
}
```

Now I can run the program and use the result inside of our `main.js`.

```bash
$ go run main.go < sum.wasm
const bytes = new Uint8Array([0, 97, 115, 109, 1, 0, 0, 0, 1, 139, 128, 128, 128, 0, 2, 96, 1, 127, ...
```

```javascript
const memory = new WebAssembly.Memory({ initial: 256, maximum: 256 });
const env = {
    'abortStackOverflow': _ => { throw new Error('overflow'); },
    'table': new WebAssembly.Table({ initial: 0, maximum: 0, element: 'anyfunc' }),
    'tableBase': 0,
    'memory': memory,
    'memoryBase': 1024,
    'STACKTOP': 0,
    'STACK_MAX': memory.buffer.byteLength,
};
const imports = { env };

const bytes = new Uint8Array([0, 97, 115, 109, 1, 0, 0, 0, 1, 139, 128, 128, 128, 0, 2, 96, 1, 127, 0, 96, 2, 127, 127, 1, 127, 2, 254, 128, 128, 128, 0, 7, 3, 101, 110, 118, 8, 83, 84, 65, 67, 75, 84, 79, 80, 3, 127, 0, 3, 101, 110, 118, 9, 83, 84, 65, 67, 75, 95, 77, 65, 88, 3, 127, 0, 3, 101, 110, 118, 18, 97, 98, 111, 114, 116, 83, 116, 97, 99, 107, 79, 118, 101, 114, 102, 108, 111, 119, 0, 0, 3, 101, 110, 118, 6, 109, 101, 109, 111, 114, 121, 2, 1, 128, 2, 128, 2, 3, 101, 110, 118, 5, 116, 97, 98, 108, 101, 1, 112, 1, 0, 0, 3, 101, 110, 118, 10, 109, 101, 109, 111, 114, 121, 66, 97, 115, 101, 3, 127, 0, 3, 101, 110, 118, 9, 116, 97, 98, 108, 101, 66, 97, 115, 101, 3, 127, 0, 3, 130, 128, 128, 128, 0, 1, 1, 6, 147, 128, 128, 128, 0, 3, 127, 1, 35, 0, 11, 127, 1, 35, 1, 11, 125, 1, 67, 0, 0, 0, 0, 11, 7, 136, 128, 128, 128, 0, 1, 4, 95, 115, 117, 109, 0, 1, 9, 129, 128, 128, 128, 0, 0, 10, 196, 128, 128, 128, 0, 1, 190, 128, 128, 128, 0, 1, 7, 127, 2, 64, 35, 4, 33, 8, 35, 4, 65, 16, 106, 36, 4, 35, 4, 35, 5, 78, 4, 64, 65, 16, 16, 0, 11, 32, 0, 33, 2, 32, 1, 33, 3, 32, 2, 33, 4, 32, 3, 33, 5, 32, 4, 32, 5, 106, 33, 6, 32, 8, 36, 4, 32, 6, 15, 0, 11, 0, 11]);

WebAssembly.instantiate(bytes, imports).then(wa => {
    const exports = wa.instance.exports;
    const sum = exports._sum;
    console.log(sum(39, 3));
});
```

```bash
> node embed.js
42
```

Amazing! So ... let's run it on BigQuery!

In our UDF editor we're going to adapt the program we just wrote, so it becomes a function BigQuery can call.

```javascript
function sum(row, emit) {
  const memory = new WebAssembly.Memory({ initial: 256, maximum: 256 });
  const env = {
      'abortStackOverflow': _ => { throw new Error('overflow'); },
      'table': new WebAssembly.Table({ initial: 0, maximum: 0, element: 'anyfunc' }),
      'tableBase': 0,
      'memory': memory,
      'memoryBase': 1024,
      'STACKTOP': 0,
      'STACK_MAX': memory.buffer.byteLength,
  };
  const imports = { env };

  const bytes = new Uint8Array([0, 97, 115, 109, 1, 0, 0, 0, 1, 139, 128, 128, 128, 0, 2, 96, 1, 127, 0, 96, 2, 127, 127, 1, 127, 2, 254, 128, 128, 128, 0, 7, 3, 101, 110, 118, 8, 83, 84, 65, 67, 75, 84, 79, 80, 3, 127, 0, 3, 101, 110, 118, 9, 83, 84, 65, 67, 75, 95, 77, 65, 88, 3, 127, 0, 3, 101, 110, 118, 18, 97, 98, 111, 114, 116, 83, 116, 97, 99, 107, 79, 118, 101, 114, 102, 108, 111, 119, 0, 0, 3, 101, 110, 118, 6, 109, 101, 109, 111, 114, 121, 2, 1, 128, 2, 128, 2, 3, 101, 110, 118, 5, 116, 97, 98, 108, 101, 1, 112, 1, 0, 0, 3, 101, 110, 118, 10, 109, 101, 109, 111, 114, 121, 66, 97, 115, 101, 3, 127, 0, 3, 101, 110, 118, 9, 116, 97, 98, 108, 101, 66, 97, 115, 101, 3, 127, 0, 3, 130, 128, 128, 128, 0, 1, 1, 6, 147, 128, 128, 128, 0, 3, 127, 1, 35, 0, 11, 127, 1, 35, 1, 11, 125, 1, 67, 0, 0, 0, 0, 11, 7, 136, 128, 128, 128, 0, 1, 4, 95, 115, 117, 109, 0, 1, 9, 129, 128, 128, 128, 0, 0, 10, 196, 128, 128, 128, 0, 1, 190, 128, 128, 128, 0, 1, 7, 127, 2, 64, 35, 4, 33, 8, 35, 4, 65, 16, 106, 36, 4, 35, 4, 35, 5, 78, 4, 64, 65, 16, 16, 0, 11, 32, 0, 33, 2, 32, 1, 33, 3, 32, 2, 33, 4, 32, 3, 33, 5, 32, 4, 32, 5, 106, 33, 6, 32, 8, 36, 4, 32, 6, 15, 0, 11, 0, 11]);

  WebAssembly.instantiate(bytes, imports).then(wa => {
      const exports = wa.instance.exports;
      const sum = exports._sum;
      emit({s: sum(row.x, row.y)});
  });
}

bigquery.defineFunction(
  'sum',
  ['x', 'y'],
  [{'name': 's', 'type': 'integer'}],
  sum
);
```

And our query will simply add two numbers:

```SQL
SELECT s FROM sum(
  SELECT 39 as x, 3 as y
);
```

We run it and the result is `42`!
How long did it take to process this simple sum? 3.1 seconds!

### Making it go faster