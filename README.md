# r-notes

Some tips / things of notes for myself for working with the R programming language

## Setting Up R to be used on VSCode

### 1. Install Required R Packages

First, install the necessary packages in R for VSCode integration:

```r
# Install R Language Server for enhanced IntelliSense and code completion
install.packages("languageserver")

# Install httpgd for interactive plotting in VSCode
# This enables plots to display directly in the VSCode interface
install.packages('httpgd', repos = c('https://community.r-multiverse.org', 'https://cloud.r-project.org'))
```

**Why these packages?**

- `languageserver`: Provides advanced code intelligence features like autocomplete, syntax highlighting, and error detection
- `httpgd`: Enables interactive plotting that displays directly in VSCode instead of opening external plot windows

### 2. Install VSCode Extensions

Install the following VSCode extensions:

- **R Extension for VS Code** (REditorSupport.r)
- **R LSP Client** (REditorSupport.r-lsp)

### 3. Configure VSCode User Settings

Add these settings to your VSCode user settings (File > Preferences > Settings > Open Settings JSON):

```json
{
  "r.rterm.option": [
    "--r-binary=/usr/local/bin/R",
    "--no-save",
    "--no-restore"
  ],
  "[r]": {
    "editor.defaultFormatter": "REditorSupport.r"
  },
  "r.plot.useHttpgd": true
}
```

**Explanation of settings:**

- `r.rterm.option`: Configures R terminal options
  - `--r-binary=/usr/local/bin/R`: Specifies the path to your R installation
  - `--no-save`: Prevents R from saving workspace on exit
  - `--no-restore`: Prevents R from restoring workspace on startup
- `[r].editor.defaultFormatter`: Sets the R extension as the default formatter for R files
- `r.plot.useHttpgd`: Enables httpgd for interactive plotting in VSCode

### 4. Usage Tips

- **Run R Code**: Use `Ctrl+Shift+Enter` (or `Cmd+Shift+Enter` on Mac) to run the current line or selection
- **Run Entire File**: Use `Ctrl+Shift+S` (or `Cmd+Shift+S` on Mac)
- **Interactive Plots**: Plots will now display directly in VSCode's plot viewer
- **Code Completion**: Enjoy enhanced autocomplete and IntelliSense features
- **Error Detection**: Get real-time error highlighting and diagnostics

## Core Principles

To understand computations in R, two slogans are helpful:

- **Everything that exists is an object.**
- **Everything that happens is a function call.**

These principles reflect R's design: all data structures, functions, and even language constructs are objects that can be manipulated, and operations that might look like special syntax (like `+`, `[`, or `<-`) are actually function calls under the hood.

## Inspecting R code:

### Debugging with `browser()`

- **`base::browser()`**: Enters an interactive debugging session at the point where it's called. When execution reaches `browser()`, R pauses and allows you to:
  - Inspect the current environment and variables
  - Execute R code in the current context
  - Step through code line by line
  - Continue execution with `c` (continue), `n` (next line), or `Q` (quit)

You can insert `browser()` directly into your code, or use it with function operators like `auto_browse()` to automatically debug on errors.

- **`purrr::auto_browse()`**: Automatically executes `browser()` inside the function when there's an error. This drops you into an interactive debugging session at the point of failure, allowing you to inspect the function's state and environment.

### `lobstr`

The `lobstr` package provides tools for inspecting R objects and understanding how R manages memory.

#### Object Identifiers

You can access an object's identifier (memory address) using `lobstr::obj_addr()`. This is useful for understanding when two objects point to the same memory location.

For example, if `x` and `y` both reference the same object:

```r
obj_addr(x)
#> [1] "0x55714636e6d8"

obj_addr(y)
#> [1] "0x55714636e6d8"
```

#### Lists and Copy-on-Modify

Lists are more complex than vectors because instead of storing the values themselves, they store references to them.

Like vectors, lists use copy-on-modify behavior; the original list is left unchanged, and R creates a modified copy. However, this is a **shallow copy**: the list object and its bindings are copied, but the values pointed to by the bindings are not. The opposite of a shallow copy is a **deep copy** where the contents of every reference are copied.

To see values that are shared across lists, use `lobstr::ref()`. `ref()` prints the memory address of each object, along with a local ID so that you can easily cross-reference shared components:

```r
ref(l1, l2)
#> █ [1:0x557144725998] <list>
#> ├─[2:0x557144c98240] <dbl>
#> ├─[3:0x557144c98400] <dbl>
#> └─[4:0x557144c985c0] <dbl>
#>
#> █ [5:0x557146b240e8] <list>
#> ├─[2:0x557144c98240]
#> ├─[3:0x557144c98400]
#> └─[6:0x5571439a1540] <dbl>
```

#### Object size

You can find out how much memory an object takes with `lobstr::obj_size()`:

```r
obj_size(letters)
#> 1.71 kB

obj_size(ggplot2::diamonds)
#> 3.46 MB
```

Since the elements of lists are references to values, the size of a list might be much smaller than you expect:

```r
x <- runif(1e6)
obj_size(x)
#> 8.00 MB

y <- list(x, x, x)
obj_size(y)
#> 8.00 MB
```

References also make it challenging to think about the size of individual objects. `obj_size(x) + obj_size(y)` will only equal `obj_size(x, y)` if there are no shared values. Here, the combined size of `x` and `y` is the same as the size of `y`:

```r
obj_size(x, y)
#> 8.00 MB
```

Finally, R 3.5.0 and later versions have **ALTREP** (alternative representation), which are like iterators in Python - they compute values on-the-fly rather than storing them all in memory:

```r
obj_size(1:3)
#> 680 B

obj_size(1:1e9)
#> 680 B
```

#### Garbage Collection and Memory Usage

If you want to find out when the GC runs, call `gcinfo(TRUE)` and GC will print a message to the console every time it runs.

`lobstr::mem_used()` is a wrapper around `gc()` that prints the total number of bytes used:

```r
mem_used()
#> 91.58 MB
```

### `tracemem()`

You can see when an object gets copied using `base::tracemem()`. Once you call this function with an object, you'll get the object's current address:

```r
x <- c(1, 2, 3)
cat(tracemem(x), "\n")
#> <0x7f80c0e0ffc8>
```

From then on, whenever that object is copied, `tracemem()` will print a message telling you which object was copied, its new address, and the sequence of calls that led to the copy:

```r
y <- x
y[[3]] <- 4L
#> tracemem[0x7f80c0e0ffc8 -> 0x7f80c4427f40]:
```

If you modify `y` again, it won't get copied. That's because the new object now only has a single name bound to it, so R applies modify-in-place optimization.

Use `untracemem()` to turn tracing off:

```r
untracemem(x)
```

### `sloop`

The `sloop` package provides tools for understanding R's object-oriented programming systems (S3, S4, R6, etc.).

**`sloop::otype()`**: Determines the object-oriented type of an R object. This is useful for understanding which OOP system an object belongs to:

```r
library(sloop)

# S3 object
otype(mtcars)
#> [1] "S3"

# S4 object
otype(methods::show)
#> [1] "S4"

# Base type (not an OOP object)
otype(1:10)
#> [1] "base"
```

This helps you understand how to work with different types of objects and which methods are available for them.

**`sloop::s3_class()`**: Returns the implicit class that the S3 and S4 systems will use to pick methods. While `base::class()` is safe to apply to S3 and S4 objects, it returns misleading results when applied to base objects. `s3_class()` is safer because it returns the class that method dispatch will actually use.

**`sloop::s3_dispatch()`**: Shows how S3 method dispatch works for a given function call. It displays which methods are considered during dispatch and which one is actually called. This is useful for understanding and debugging S3 method dispatch behavior:

```r
library(sloop)

# Show dispatch for a generic function call
s3_dispatch(print(mtcars))
#> => print.data.frame
#>  * print.default
```

The output shows:

- `=>` indicates the method that will be called
- `*` indicates methods that exist but won't be called (because a more specific method was found)
- Methods are listed in order from most specific to least specific

**`sloop::s3_get_method()`**: Unlike most functions, you can't see the source code for most S3 methods just by typing their names. That's because S3 methods are not usually exported: they live only inside the package, and are not available from the global environment. Instead, you can use `sloop::s3_get_method()`, which will work regardless of where the method lives:

```r
weighted.mean.Date
#> Error: object 'weighted.mean.Date' not found

s3_get_method(weighted.mean.Date)
#> function (x, w, ...)
#> .Date(weighted.mean(unclass(x), w, ...))
#> <bytecode: 0x556c24d30ab8>
#> <environment: namespace:stats>
```

**`sloop::s3_methods_generic()` and `sloop::s3_methods_class()`**: These functions help you discover all S3 methods for a given generic function or class. `s3_methods_generic()` shows all methods defined for a generic function, while `s3_methods_class()` shows all methods defined for a class:

```r
s3_methods_generic("mean")
#> # A tibble: 7 × 4
#>   generic class      visible source
#>   <chr>   <chr>      <lgl>   <chr>
#> 1 mean    Date       TRUE    base
#> 2 mean    default    TRUE    base
#> 3 mean    difftime   TRUE    base
#> 4 mean    POSIXct    TRUE    base
#> 5 mean    POSIXlt    TRUE    base
#> 6 mean    quosure    FALSE   registered S3method
#> 7 mean    vctrs_vctr FALSE   registered S3method

s3_methods_class("ordered")
#> # A tibble: 4 × 4
#>   generic       class   visible source
#>   <chr>         <chr>   <lgl>   <chr>
#> 1 as.data.frame ordered TRUE    base
#> 2 Ops           ordered TRUE    base
#> 3 relevel       ordered FALSE   registered S3method
#> 4 Summary       ordered TRUE    base
```

The output shows which methods are visible (exported) and which are registered but not visible. The `source` column indicates where the method is defined (e.g., "base" for base R, "registered S3method" for methods registered but not exported).

**`sloop::ftype()`**: You can quickly determine whether a function is generic by using `sloop::ftype()`; check the output for the word "generic" to identify generic functions.

**Base type vs. class**: While only OO objects have a class attribute, every object has a base type. You can inspect the base type using `base::typeof()`:

```r
typeof(1:10)
#> [1] "integer"

typeof(mtcars)
#> [1] "list"
```

The base type represents the fundamental data structure of the object, while the class attribute (when present) determines how the object behaves with method dispatch in S3 and S4 systems.

**Important note on base types**: Base types do not form an OOP system—they're implemented in C with switch statements, so only R-core can create new types. Adding a new type requires modifying every relevant switch statement, making it very difficult. As a result, new base types are rarely added: the most recent were two exotic types in 2011 (for memory diagnostics), and before that, the S4 type in 2005.

**Summary of all 25 base types**:

There are 25 base types, listed below by category. Since these types are primarily used in C code, they're often referred to by their C type names (shown in parentheses):

**Vectors** (8 types):

- `NULL` (NILSXP)
- `logical` (LGLSXP)
- `integer` (INTSXP)
- `double` (REALSXP)
- `complex` (CPLXSXP)
- `character` (STRSXP)
- `list` (VECSXP)
- `raw` (RAWSXP)

**Functions** (3 types):

- `closure` (CLOSXP) - regular R functions
- `special` (SPECIALSXP) - internal functions
- `builtin` (BUILTINSXP) - primitive functions

**Environments** (1 type):

- `environment` (ENVSXP)

**S4** (1 type):

- `S4` (S4SXP) - used for S4 classes that don't inherit from an existing base type

**Language components** (4 types):

- `symbol` (SYMSXP) - also called "name"
- `language` (LANGSXP) - usually called "calls"
- `pairlist` (LISTSXP) - used for function arguments
- `expression` (EXPRSXP) - special purpose type only returned by `parse()` and `expression()`

**Esoteric types** (8 types, rarely seen in R):

These are important primarily for C code:

- `externalptr` (EXTPTRSXP)
- `weakref` (WEAKREFSXP)
- `bytecode` (BCODESXP)
- `promise` (PROMSXP)
- `...` (DOTSXP)
- `any` (ANYSXP)
- Plus two exotic types added in 2011 for diagnosing memory problems

### S4 Classes

S4 provides a formal approach to functional OOP. The underlying ideas are similar to S3, but implementation is much stricter and makes use of specialised functions for creating classes (`setClass()`), generics (`setGeneric()`), and methods (`setMethod()`).

The Bioconductor community is a long-term user of S4 and has produced much of the best material about its effective use. To learn more about S4 check for a newer version at Bioconductor course materials.
or read anything by Martin Morgan, who is an ex-member of R-core and the project lead of Bioconductor. He's a world expert on the practical use of S4.

**When S4 is worth the investment**: S4 requires more upfront design than S3, and this investment is more likely to pay off on larger projects where greater resources are available. S4 is particularly well-suited for:

- **Large collaborative projects**: Bioconductor (~1,300 packages) uses S4 extensively because its key data structures (e.g., `SummarizedExperiment`, `IRanges`, `DNAStringSet`) are built with S4, and the stricter review process encourages formal design.
- **Complex systems of interrelated objects**: The `Matrix` package exemplifies this—it defines 108 classes, 23 generic functions, and 1,780 methods to efficiently handle many different types of sparse and dense matrices. S4 makes it easy to provide general methods that work for all inputs, then provide specialized methods where specific combinations allow more efficient implementations. This requires careful planning to avoid method dispatch ambiguity, but the planning pays off with higher performance.
  ![S4 Matrix class diagram](https://adv-r.hadley.nz/diagrams/s4/Matrix.png)

**S4 dispatch complexity**: S4 dispatch is complicated because S4 has two important features:

- **Multiple inheritance**: A class can have multiple parents
- **Multiple dispatch**: A generic can use multiple arguments to pick a method

These features make S4 very powerful, but can also make it hard to understand which method will get selected for a given combination of inputs. In practice, keep method dispatch as simple as possible by avoiding multiple inheritance, and reserving multiple dispatch only for where it is absolutely necessary.

An important new component of S4 is the slot, a named component of the object that is accessed using the specialised subsetting operator `@`.

**Defining S4 classes with `setClass()`**: Use `setClass()` to define new S4 classes. The `contains` argument specifies inheritance (parent classes), `slots` defines named components with their types, and `prototype` provides default values for slots:

```r
setClass("Employee",
  contains = "Person",
  slots = c(
    boss = "Person"
  ),
  prototype = list(
    boss = new("Person")
  )
)

str(new("Employee"))
#> Formal class 'Employee' [package ".GlobalEnv"] with 3 slots
#>   ..@ boss:Formal class 'Person' [package ".GlobalEnv"] with 2 slots
#>   .. .. ..@ name: chr NA
#>   .. .. ..@ age : num NA
#>   ..@ name: chr NA
#>   ..@ age : num NA
```

This example creates an `Employee` class that inherits from `Person` and has a `boss` slot of type `Person` with a default value.

`setClass()` has 9 other arguments but they are either deprecated or not recommended.

**Inspecting S4 objects**: Given an S4 object, you can see its class with `is()` and access slots with `@` (equivalent to `$`) and `slot()` (equivalent to `[[`).

**Introspection with `is()`**: To determine what classes an object inherits from, use `is()`. It returns a character vector of all classes in the inheritance hierarchy:

```r
is(new("Person"))
#> [1] "Person"
is(new("Employee"))
#> [1] "Employee" "Person"
```

To test if an object inherits from a specific class, use the second argument of `is()`:

```r
is(john, "Person")
#> [1] TRUE
```

**Getting help for S4 classes and methods**: If you're using an S4 class defined in a package, you can get help on it with `class?Person`. To get help for a method, put `?` in front of a call (e.g. `?age(john)`) and `?` will use the class of the arguments to figure out which help file you need.

**Identifying S4 objects and generics**: You can use `sloop` functions to identify S4 objects and generics found in the wild:

```r
sloop::otype(john)
#> [1] "S4"

sloop::ftype(age)
#> [1] "S4"      "generic"
```

All functions related to S4 live in the methods package. This package is always available when you're running R interactively, but may not be available when running R in batch mode, i.e. from Rscript. For this reason, it's a good idea to call `library(methods)` whenever you use S4. This also signals to the reader that you'll be using the S4 object system.

#### Inspecting Methods

To list all methods for a generic or class, use `methods("generic")` or `methods(class = "class")`; to find a specific method's implementation, use `selectMethod("generic", "class")`. Determine arguments from documentation or `args(generic)`.

```r
library(methods)
```

**The `.Data` virtual slot**: If an S4 object inherits from an S3 class or a base type, it will have a special virtual slot called `.Data`. This contains the underlying base type or S3 object. This allows S4 classes to extend existing S3 classes or base types while maintaining compatibility with the underlying data structure.

```r
RangedNumeric <- setClass(
  "RangedNumeric",
  contains = "numeric",
  slots = c(min = "numeric", max = "numeric"),
  prototype = structure(numeric(), min = NA_real_, max = NA_real_)
)
rn <- RangedNumeric(1:10, min = 1, max = 10)
rn@min
#> [1] 1
rn@.Data
#>  [1]  1  2  3  4  5  6  7  8  9 10
```

### R6 Classes

**`R6::R6Class()`**: Creates R6 classes, which provide a more traditional object-oriented programming interface compared to S3 and S4. R6 classes support:

- Encapsulation (public and private methods/fields)
- Reference semantics (objects are mutable and passed by reference)
- Inheritance
- Active bindings

R6 objects are different from S3/S4 objects in that they use reference semantics—modifying an R6 object affects all references to it, similar to environments.

R6 objects have reference semantics which means that they are modified in-place, not copied-on-modify.

```r
library(R6)

Person <- R6Class("Person",
  public = list(
    name = NULL,
    initialize = function(name) {
      self$name <- name
    },
    greet = function() {
      cat("Hello, I'm", self$name, "\n")
    }
  )
)

alice <- Person$new("Alice")
alice$greet()
#> Hello, I'm Alice
```

Out of all the Object types in R, R6 objects are the most similar to objects in Python.

### The Search Path

The search path is the chain of environments that R searches through when looking for objects. You can inspect it with `base::search()` or `rlang::search_envs()`:

```r
search()
#>  [1] ".GlobalEnv"        "package:rlang"     "package:stats"
#>  [4] "package:graphics"  "package:grDevices" "package:utils"
#>  [7] "package:datasets"  "package:methods"   "Autoloads"
#> [10] "package:base"

search_envs()
#>  [[1]] $ <env: global>
#>  [[2]] $ <env: package:rlang>
#>  [[3]] $ <env: package:stats>
#>  [[4]] $ <env: package:graphics>
#>  [[5]] $ <env: package:grDevices>
#>  [[6]] $ <env: package:utils>
#>  [[7]] $ <env: package:datasets>
#>  [[8]] $ <env: package:methods>
#>  [[9]] $ <env: Autoloads>
#> [[10]] $ <env: package:base>
```

`search()` returns a character vector of environment names, while `search_envs()` returns a list of the actual environment objects. The search path shows the order in which R searches for objects: starting with the global environment, then through attached packages, and finally ending with the base package.

### The Call Stack and Caller Environments

The **caller environment** (also called the call stack) represents the sequence of function calls that led to the current execution. Each function call creates a new execution environment, and these environments form a chain showing how you got to where you are.

**Accessing caller environments**: You can access the caller environment (the environment of the function that called the current function) using `rlang::caller_env()` or `base::parent.frame()`:

```r
f <- function() {
  g()
}

g <- function() {
  caller_env()  # Returns the execution environment of f()
  parent.frame()  # Base R equivalent
}
```

**Inspecting the call stack**: When an error occurs, you can inspect the call stack to see the sequence of function calls:

```r
f <- function(x) {
  g(x = 2)
}

g <- function(x) {
  h(x = 3)
}

h <- function(x) {
  stop()
}

f(x = 1)
#> Error:

traceback()
#> 4: stop()
#> 3: h(x = 3)
#> 2: g(x = 2)
#> 1: f(x = 1)
```

`base::traceback()` shows the call stack from the error backwards to the initial call. For a more visual representation, use `lobstr::cst()` (call stack tree):

```r
h <- function(x) {
  lobstr::cst()
}

f(x = 1)
#> █
#> └─f(x = 1)
#>   └─g(x = 2)
#>     └─h(x = 3)
#>       └─lobstr::cst()
```

Note that `cst()` shows the order from the beginning (top) to the current call (bottom), which is the opposite of `traceback()`. As call stacks get more complicated, it's easier to understand the sequence of calls if you start from the beginning rather than the end.

### Inspecting Environments

**Creating environments**: `rlang::env()` is a function for creating new environments. In base R, you can use `new.env()` to create a new environment:

```r
# Using rlang
e1 <- env()

# Using base R
e1 <- new.env()
e1$a <- TRUE
e1$b <- "hello"
```

**The nature of environments**: The job of an environment is to associate, or bind, a set of names to a set of values. You can think of an environment as a bag of names, with no implied order (i.e. it doesn't make sense to ask which is the first element in an environment). Environments are unordered collections of name-value bindings.

Printing an environment directly just displays its memory address, which is not terribly useful:

```r
e1
#> <environment: 0x5597361f0aa8>
```

Instead, use `rlang::env_print()` (or `lobstr::env_print()`) which provides much more informative output:

```r
env_print(e1)
#> <environment: 0x5597361f0aa8>
#> Parent: <environment: global>
#> Bindings:
#> • a: <lgl>
#> • b: <chr>
#> • c: <dbl>
#> • d: <env>
```

`env_print()` shows the environment's memory address, its parent environment, and a list of all bindings (variables) in the environment along with their types. This makes it much easier to understand the structure and contents of an environment.

**Current and global environments**: Two important environments to be aware of are:

- **`environment()`** (base R) or **`current_env()`** (rlang): The environment in which code is currently executing. When you're experimenting interactively, that's usually the global environment.
- **`globalenv()`** (base R) or **`global_env()`** (rlang): Also called your "workspace", this is where all interactive (i.e. outside of a function) computation takes place. The global environment is printed as `R_GlobalEnv` and `.GlobalEnv`.
- **`rlang::base_env()`**: You can access the base environment, which contains all the base R functions and objects. This is typically at the end of the search path.

**Parent environments**: Every environment has a parent, another environment. The parent is what's used to implement lexical scoping: **if a name is not found in an environment, then R will look in its parent** (and so on, recursively up the environment chain). This is how R searches for variables and functions when they're not found in the current environment. The ancestors of every environment eventually terminate with the empty environment, forming a complete path from the current environment up to the empty environment.

You can find the parent of an environment with `rlang::env_parent()`:

```r
env_parent(e2b)
#> <environment: 0x559735c4a248>

env_parent(e2a)
#> <environment: R_GlobalEnv>
```

You can see all ancestors in the environment chain with `rlang::env_parents()`:

```r
env_parents(e2b)
#> [[1]]   <env: 0x559735c4a248>
#> [[2]] $ <env: global>

env_parents(e2d)
#> [[1]]   <env: 0x5597366d1bf8>
#> [[2]] $ <env: empty>
```

**Types of environments**: There are several important types of environments to understand:

- **Package environment**: Each package has its own namespace environment that contains all the functions and objects exported by that package. When you load a package with `library()`, its namespace is attached to the search path.

- **Function environment**: The environment where a function was created. This is stored in the function's `environment()` attribute and determines where the function looks for values (lexical scoping). The function environment is also called the "enclosing environment" because it encloses the function definition.

- **Execution environment**: A temporary environment created each time a function is called. This is where the function's arguments and local variables are stored during execution. The execution environment's parent is the function's environment, allowing the function to access variables from where it was defined. Execution environments are ephemeral—they're created when the function starts and typically destroyed when it finishes.

**Capturing execution environments**: An execution environment is usually ephemeral; once the function has completed, the environment will be garbage collected. However, you can capture it by explicitly returning it using `current_env()` (or `environment()` in base R):

```r
h2 <- function(x) {
  a <- x * 2
  current_env()
}

e <- h2(x = 10)
env_print(e)
#> <environment: 0x559733944920>
#> Parent: <environment: global>
#> Bindings:
#> • a: <dbl>
#> • x: <dbl>

fn_env(h2)
#> <environment: R_GlobalEnv>
```

**Note on `fn_env()`**: `rlang::fn_env()` (or `base::environment()` in base R) returns the function's enclosing environment—the environment where the function was created. This is different from the execution environment: the function's environment is where it looks for values through lexical scoping, while the execution environment is created each time the function is called.

By returning the execution environment, you keep it alive and can inspect its contents, including the function's arguments and local variables. Notice that the execution environment's parent is the function's environment (the global environment in this case), not the execution environment itself.

**Namespaces**: There's an important distinction between package environments and namespaces:

- **Package environment**: Controls how **we find the function** (i.e., how R searches for functions when you call them). When you load a package with `library()`, its package environment is attached to the search path.

- **Namespace**: Controls how **the function finds its variables** (i.e., where the function looks for objects it uses internally). Each package has a namespace environment that contains all the functions and objects in that package, including non-exported ones.

The package environment and namespace are different environments, but they're related: the package environment typically contains only exported functions, while the namespace contains everything. This separation ensures that functions in a package can access their internal dependencies even if those aren't exported to users.

**Superassignments**: The superassignment operator `<<-` assigns to a variable in the parent environment (or further up the environment chain) rather than creating a new binding in the current environment. This is useful when you want to modify a variable that exists in an enclosing environment:

```r
x <- 1
f <- function() {
  x <<- 2  # Modifies x in the parent environment
}
f()
x  # x is now 2
```

Unlike regular assignment (`<-`), which creates a new binding in the current environment, `<<-` searches up the environment chain to find an existing binding and modifies it. If no binding is found, it creates one in the global environment.

**Working with environment bindings**: The `rlang` package provides several functions for working with environment bindings:

- **`rlang::env_poke()`**: Assigns a value to a name in an environment. This is similar to using `$<-` but is more explicit:

```r
e1 <- env()
env_poke(e1, "x", 10)
e1$x
#> [1] 10
```

- **`rlang::env_bind()`**: Binds multiple name-value pairs to an environment at once. This is useful for setting up multiple bindings:

```r
e1 <- env()
env_bind(e1, a = 1, b = 2, c = 3)
env_print(e1)
```

- **`rlang::env_has()`**: **An important inspection tool** that checks whether an environment has bindings with the given names. It returns a logical vector indicating which names exist in the environment:

```r
e1 <- env()
env_bind(e1, a = 1, b = 2)
env_has(e1, c("a", "b", "c"))
#>    a    b    c
#> TRUE TRUE FALSE
```

- **`rlang::env_unbind()`**: Removes bindings from an environment. This is useful for cleaning up environments or removing specific variables:

```r
e1 <- env()
env_bind(e1, a = 1, b = 2, c = 3)
env_unbind(e1, "b")
env_has(e1, c("a", "b", "c"))
#>    a    b    c
#> TRUE FALSE TRUE
```

**Comparing environments**: To compare environments, you need to use `identical()` and not `==`. This is because `==` is a vectorised operator, and environments are not vectors:

```r
# Correct way to compare environments
identical(environment(), globalenv())
# Using == will not work as expected for environments
```

### Function Structure and Inspection

A function has three parts:

- **`formals()`**: The list of arguments that control how you call the function
- **`body()`**: The code inside the function
- **`environment()`**: The data structure that determines how the function finds the values associated with the names

While the formals and body are specified explicitly when you create a function, the environment is specified implicitly, based on where you defined the function. The function environment always exists, but it is only printed when the function isn't defined in the global environment.

You can inspect these parts of a function using the corresponding functions:

```r
f02 <- function(x, y) {
  x + y
}

formals(f02)
#> $x
#>
#>
#> $y

body(f02)
#> {
#>     x + y
#> }

environment(f02)
#> <environment: R_GlobalEnv>
```

These functions are useful tools for inspecting and understanding function definitions in R.

**Accessing a function's environment**: You can access a function's environment (where it was created) using `base::environment(fn)` or `rlang::fn_env(fn)`. Both return the environment that the function encloses, which determines where it looks for values through lexical scoping:

```r
f <- function(x) x + y
environment(f)  # base R
fn_env(f)       # rlang
```

The function's environment is also called its "enclosing environment" because it encloses the function definition.

**Function attributes**: Like all objects in R, functions can also possess any number of additional attributes. One attribute used by base R is `srcref`, short for source reference. It points to the source code used to create the function. The `srcref` is used for printing because, unlike `body()`, it contains code comments and other formatting:

```r
attr(f02, "srcref")
#> NULL
```

You can access any attribute of a function (or any R object) using `attr()`.

**Inspecting purrr formula transformations**: **`purrr::as_mapper()`** is an important tool for understanding how purrr translates formulas (created with `~`) into functions. You can see what's happening behind the scenes when purrr converts a formula to a function:

```r
as_mapper(~ length(unique(.x)))
#> <lambda>
#> function (..., .x = ..1, .y = ..2, . = ..1)
#> length(unique(.x))
#> attr(,"class")
#> [1] "rlang_lambda_function" "function"
```

The function arguments look a little quirky but allow you to refer to `.` for one argument functions, `.x` and `.y` for two argument functions, and `..1`, `..2`, `..3`, etc. for functions with an arbitrary number of arguments. This is useful for debugging and understanding how purrr's formula syntax works internally.

**Primitive functions**: There is one exception to the rule that a function has three components. Primitive functions, like `sum()` and `[`, call C code directly:

```r
sum
#> function (..., na.rm = FALSE)  .Primitive("sum")

`[`
#> .Primitive("[")
```

They have either type `builtin` or type `special`:

```r
typeof(sum)
#> [1] "builtin"

typeof(`[`)
#> [1] "special"
```

These functions exist primarily in C, not R, so their `formals()`, `body()`, and `environment()` are all `NULL`. These functions are only found in the base package.

**First-class functions**: R functions are objects in their own right, a language property often called "first-class functions". Unlike in many other languages, there is no special syntax for defining and naming a function: you simply create a function object (with `function`) and bind it to a name with `<-`:

```r
f01 <- function(x) {
  sin(1 / x ^ 2)
}
```

![First-class functions diagram](https://adv-r.hadley.nz/diagrams/functions/first-class.png)

While you almost always create a function and then bind it to a name, the binding step is not compulsory. If you choose not to give a function a name, you get an **anonymous function**. This is useful when it's not worth the effort to figure out a name:

```r
lapply(mtcars, function(x) length(unique(x)))
Filter(function(x) !is.numeric(x), mtcars)
integrate(function(x) sin(x) ^ 2, 0, pi)
```

In R, you'll often see functions called **closures**. This name reflects the fact that R functions capture, or enclose, their environments.

### Metaprogramming

**Metaprogramming** is the practice of writing code that manipulates code as data. In R, this means working with expressions (code trees) rather than evaluating them immediately.

**Constants, symbols, and calls**: Expressions in R are composed of three fundamental components:

- **Constants**: Test for constants with `rlang::is_syntactic_literal()`. These are literal values like numbers, strings, etc.

- **Symbols**: Symbols represent variable names and are printed without quotes. You can identify them because `str()` shows "symbol", and `is.symbol()` returns `TRUE`. Symbols are not vectorized (always length 1). For multiple symbols, use `rlang::syms()` to create a list:

```r
str(expr(x))
#>  symbol x
is.symbol(expr(x))
#> [1] TRUE
```

- **Calls**: Call objects represent function calls and look like function calls when printed. Note that `typeof()` and `str()` print "language" for call objects, but `is.call()` returns `TRUE`:

```r
x <- expr(read.table("important.csv", row.names = FALSE))
typeof(x)
#> [1] "language"
is.call(x)
#> [1] TRUE
```

**Base R vs rlang type testing functions**: Both base R and rlang provide functions for testing input types, but with different naming conventions and coverage:

- **Base R functions** start with `is.` (e.g., `is.symbol()`, `is.call()`, `is.pairlist()`, `is.expression()`)
- **rlang functions** start with `is_` (e.g., `is_symbol()`, `is_call()`, `is_pairlist()`, `is_syntactic_literal()`)

Coverage differs slightly: base R has `is.expression()` (no rlang equivalent), while rlang has `is_syntactic_literal()` (no base R equivalent). For symbols, calls, and pairlists, both provide equivalent functionality.

**`rlang::expr()` and `rlang::enexpr()`**: These functions capture expressions (code) without evaluating them. `expr()` captures a literal expression, while `enexpr()` captures an expression from a function argument:

```r
# expr() captures the literal expression
x <- expr(x + y)
x
#> x + y

# enexpr() captures an expression from an argument
f <- function(arg) enexpr(arg)
f(x + y)
#> x + y
```

**The `!!` (bang-bang) unquote operator**: The `!!` operator (pronounced "bang-bang") inserts a code tree stored in a variable into an expression. This makes it easy to build complex expression trees from simple fragments:

```r
xx <- expr(x + x)
yy <- expr(y + y)

expr(!!xx / !!yy)
#> (x + x)/(y + y)
```

The `!!` operator essentially "splices" the stored expression into the new expression, allowing you to programmatically construct complex code structures from simpler components.

**Dynamic lookup**: An important aspect of R's scoping is that value lookup happens at runtime, not when the function is defined. While lexical scoping determines where R searches for values (following the environment chain), the actual lookup occurs each time the function executes. This means a function's behavior can change based on what exists in the environment when it runs:

```r
g12 <- function() x + 1
x <- 15
g12()
#> [1] 16

x <- 20
g12()
#> [1] 21
```

This runtime lookup can lead to subtle bugs. Typos in variable names won't be caught when you define the function, and the function might work in some contexts but fail in others depending on what's available in the global environment.

**Debugging tools for detecting external dependencies**:

- **`codetools::findGlobals()`**: This function identifies all symbols used within a function that aren't defined locally. It's particularly useful for finding functions that accidentally depend on variables from the global environment:

```r
codetools::findGlobals(g12)
#> [1] "+" "x"
```

- **`emptyenv()`**: This creates an environment with no bindings. By setting a function's environment to `emptyenv()`, you can test whether the function truly stands alone or has hidden dependencies. Any missing symbols will cause an error, revealing the dependency:

```r
environment(g12) <- emptyenv()
g12()
#> Error in x + 1: could not find function "+"
```

This behavior exists because R uses lexical scoping to resolve all symbols—not just function calls like `mean()`, but also operators like `+` and even control structures like `{`. Everything must be found through the environment chain.

**Function return visibility**: Most functions return visibly, meaning that calling the function in an interactive context automatically prints the result:

```r
j03 <- function() 1
j03()
#> [1] 1
```

However, you can prevent automatic printing by applying `invisible()` to the last value:

```r
j04 <- function() invisible(1)
j04()
```

To verify that an invisible value does indeed exist, you can explicitly print it or wrap it in parentheses:

```r
print(j04())
#> [1] 1

(j04())
#> [1] 1
```

Alternatively, you can use `withVisible()` to return both the value and a visibility flag:

```r
str(withVisible(j04()))
#> List of 2
#>  $ value  : num 1
#>  $ visible: logi FALSE
```

This is useful for functions that perform side effects (like plotting or writing files) where you don't want the return value to clutter the console output.

**Cleanup and resource management**: The `on.exit()` function registers expressions to be executed when a function exits, whether normally or due to an error. This is useful for cleanup tasks like closing file connections or restoring options:

```r
my_function <- function() {
  old_options <- options(digits = 2)
  on.exit(options(old_options), add = TRUE)
  # function code here
}
```

The `withr` package provides a more convenient interface for temporarily modifying settings and automatically restoring them. It offers functions like `with_options()`, `with_dir()`, `with_par()`, and many others that handle the setup and cleanup automatically:

```r
library(withr)
with_options(list(digits = 2), {
  # code that runs with digits = 2
  # digits is automatically restored afterward
})
```

The `withr` package is particularly useful for managing temporary changes to options, working directories, graphics parameters, and other global settings.

### Condition Objects

When conditions are signalled (using functions like `stop()`, `warning()`, or `message()`), R creates condition objects behind the scenes. To inspect these objects, you can catch them using `rlang::catch_cnd()`:

```r
cnd <- catch_cnd(stop("An error"))

str(cnd)
#> List of 2
#>  $ message: chr "An error"
#>  $ call   : language force(expr)
#>  - attr(*, "class")= chr [1:3] "simpleError" "error" "condition"
```

Built-in conditions are lists with two standard elements:

- **`message`**: A length-1 character vector containing the text to display to a user. Extract it using `conditionMessage(cnd)`.
- **`call`**: The call which triggered the condition. As described above, we don't use the call, so it will often be `NULL`. Extract it using `conditionCall(cnd)`.

Custom conditions may contain other components beyond these two standard elements.

Conditions also have a **class attribute**, which makes them S3 objects. The class hierarchy determines how the condition is handled and displayed.

**Muffling conditions**: When using calling handlers (established with `withCallingHandlers()`), you can use `rlang::cnd_muffle()` to signal that a condition has been handled and prevent it from propagating further. This is useful for suppressing specific messages or warnings without affecting the rest of your code execution:

```r
fn <- function() {
  message("First message")
  message("Second message")
  "Function output"
}

withCallingHandlers(
  fn(),
  message = function(cnd) {
    if (conditionMessage(cnd) == "First message") {
      cnd_muffle(cnd)
    }
  }
)
```

In this example, the handler checks if the message is "First message" and muffles it, preventing it from being displayed. The second message and the function's output are unaffected.

Not all conditions can be muffled. `cnd_muffle()` works with `warning` and `message` conditions, as well as bare conditions signalled with `signal()` or `cnd_signal()`. If you attempt to muffle a condition that is not mufflable, `cnd_muffle()` will return `FALSE`.

**Proper error signaling**: When creating custom error conditions, use `rlang::abort()` in conjunction with `glue::glue()` to create structured errors with metadata. This pattern creates a user-friendly error message while storing additional metadata for developers:

```r
abort_bad_argument <- function(arg, must, not = NULL) {
  msg <- glue::glue("`{arg}` must {must}")

  if (!is.null(not)) {
    not <- typeof(not)
    msg <- glue::glue("{msg}; not {not}.")
  }

  abort("error_bad_argument",
    message = msg,
    arg = arg,
    must = must,
    not = not
  )
}
```

This approach provides:

- **User-friendly messages**: Clear, informative error messages using `glue::glue()` for string interpolation
- **Structured metadata**: Additional fields (`arg`, `must`, `not`) stored in the condition object for programmatic access
- **Custom error classes**: A specific error class (`"error_bad_argument"`) that can be caught by handlers

The metadata stored in the condition can be accessed programmatically, making it easier to write handlers that respond to specific error types or extract useful debugging information.

**Success and failure values**: You can extend the `tryCatch()` pattern to return one value if code evaluates successfully and another if it fails. The trick is to evaluate the user-supplied code, then the success value. If the code throws an error, execution never reaches the success value and instead returns the error value:

```r
foo <- function(expr) {
  tryCatch(
    error = function(cnd) error_val,
    {
      expr
      success_val
    }
  )
}
```

This pattern is useful for creating helper functions that test whether expressions succeed or fail. For example, to determine if an expression fails:

```r
does_error <- function(expr) {
  tryCatch(
    error = function(cnd) TRUE,
    {
      expr
      FALSE
    }
  )
}
```

Or to capture any condition (similar to `rlang::catch_cnd()`):

```r
catch_cnd <- function(expr) {
  tryCatch(
    condition = function(cnd) cnd,
    {
      expr
      NULL
    }
  )
}
```

The key insight is that if `expr` throws an error, the handler catches it and returns the error value. If `expr` succeeds, execution continues to the success value, which becomes the return value.

**`purrr::safely()`**: When using functionals like `map()`, errors can be frustrating because they stop execution and provide limited information about which input caused the problem. For example, if you're mapping `sum()` over a list and one element fails, you might only see:

```r
map_dbl(x, sum)
#> Error in map_dbl(x, sum): ℹ In index: 4.
#> Caused by error:
#> ! invalid 'type' (character) of argument
```

This tells you something failed at index 4, but you don't get the successful results from the other elements, making it harder to understand the full picture.

The `safely()` function operator solves this by transforming a function to turn errors into data rather than stopping execution. When you apply `safely()` to a function, it returns a wrapped function that always returns a list with two elements:

- **`result`**: Contains the output of the original function if it executes successfully; otherwise, `NULL`
- **`error`**: Contains the error object (a condition object) if an error occurs; otherwise, `NULL`

Here's how it works when called directly:

```r
safe_sum <- safely(sum)

# Successful call
str(safe_sum(c(1, 2, 3)))
#> List of 2
#>  $ result: num 6
#>  $ error : NULL

# Failed call
str(safe_sum("oops"))
#> List of 2
#>  $ result: NULL
#>  $ error :List of 2
#>   ..$ message: chr "invalid 'type' (character) of argument"
#>   ..$ call   : language .Primitive("sum")(...)
#>   ..- attr(*, "class")= chr [1:3] "simpleError" "error" "condition"
```

When used with functionals like `map()`, you get a list of these result/error pairs:

```r
x <- list(c(1, 2), c(3, 4), c(5, 6), "oops")
out <- map(x, safely(sum))
```

The output structure can be a bit unwieldy since you have nested lists. You can reorganize it using `purrr::transpose()` to separate results from errors:

```r
out <- transpose(map(x, safely(sum)))
str(out)
#> List of 2
#>  $ result:List of 4
#>   ..$ : num 3
#>   ..$ : num 7
#>   ..$ : num 11
#>   ..$ : NULL
#>  $ error :List of 4
#>   ..$ : NULL
#>   ..$ : NULL
#>   ..$ : NULL
#>   ..$ :List of 2
#>   .. ..$ message: chr "invalid 'type' (character) of argument"
#>   .. ..$ call   : language .Primitive("sum")(...)
```

Now you can easily identify which inputs succeeded or failed, and extract the successful results:

```r
# Find which elements succeeded
ok <- map_lgl(out$error, is.null)
ok
#> [1]  TRUE  TRUE  TRUE FALSE

# See which inputs failed
x[!ok]
#> [[1]]
#> [1] "oops"

# Extract only the successful results
out$result[ok]
#> [[1]]
#> [1] 3
#>
#> [[2]]
#> [1] 7
#>
#> [[3]]
#> [1] 11
```

The `safely()` function accepts:

- **`.f`**: The function to be modified
- **`otherwise`**: A default value to return when an error occurs (optional)
- **`quiet`**: A logical value indicating whether to suppress error messages (`TRUE` by default)

This pattern is particularly useful when processing large datasets or performing operations where some inputs might cause errors. It allows you to continue processing all elements, capture errors for later analysis, and still work with the successful results.

## Data Frames

Data frames are lists of vectors, which has important performance implications for copy-on-modify behavior:

- **Column modifications**: When you modify a single column, only that specific column vector needs to be copied. Other columns can continue pointing to their original references, making column-wise operations relatively efficient.

- **Row modifications**: When you modify a row, every column must be copied because each column vector contains elements from that row. This makes row-wise operations significantly more expensive than column-wise operations.

**Performance takeaway**: Prefer column-wise operations over row-wise operations when working with data frames to minimize unnecessary copying and improve performance.

## Vectors and Factors

### Types of Vectors

Vectors come in two flavours: **atomic vectors** and **lists**. They differ in terms of their elements' types:

- **Atomic vectors**: All elements must have the same type
- **Lists**: Elements can have different types

There are four primary types of atomic vectors:

- **Logical**: Boolean values (`TRUE`, `FALSE`)
- **Integer**: Whole numbers
- **Double**: Floating-point numbers
- **Character**: Strings

Collectively, integer and double vectors are known as **numeric vectors**. There are two rare types: **complex** (for complex numbers, rarely needed in statistics) and **raw** (for binary data).

![Vector type hierarchy](https://adv-r.hadley.nz/diagrams/vectors/summary-tree-s3-1.png)

### Factors

A **factor** is a vector that can contain only predefined values. It is used to store categorical data. Factors are built on top of an integer vector with two attributes:

- **Class**: `"factor"`, which makes it behave differently from regular integer vectors
- **Levels**: Defines the set of allowed values

Factors are useful when you know the set of possible values but they're not all present in a given dataset. In contrast to a character vector, when you tabulate a factor you'll get counts of all categories, even unobserved ones:

![Factor structure](https://adv-r.hadley.nz/diagrams/vectors/factor.png)

**Important note on base R behavior**: In base R, you tend to encounter factors very frequently because many base R functions (like `read.csv()` and `data.frame()`) automatically convert character vectors to factors. This is suboptimal because there's no way for those functions to know the set of all possible levels or their correct order: the levels are a property of theory or experimental design, not of the data. Instead, use the argument `stringsAsFactors = FALSE` to suppress this behaviour:

```r
# Prevent automatic conversion to factors
df <- read.csv("data.csv", stringsAsFactors = FALSE)
df <- data.frame(x = c("a", "b", "c"), stringsAsFactors = FALSE)
```

**Important note on subsetting factors**: Factors are not treated specially when subsetting. This means that subsetting will use the underlying integer vector, not the character levels. This is typically unexpected, so you should avoid subsetting with factors:

```r
y <- c(a = 1.1, b = 2.1, c = 3.1)
y[factor("b")]
#>   a
#> 2.1
```

In the example above, `factor("b")` is converted to its underlying integer value (which is 2), so `y[2]` is returned instead of `y["b"]`, resulting in the value for "a" (the second element) rather than "b".

**Factor subsetting and the `drop` argument**: Factor subsetting also has a `drop` argument, but its meaning is rather different. It controls whether or not levels (rather than dimensions) are preserved, and it defaults to `FALSE`. If you find you're using `drop = TRUE` a lot it's often a sign that you should be using a character vector instead of a factor.

### Inspecting Vectors

You can determine the type of a vector with `typeof()` and its length with `length()`:

```r
x <- c(1, 2, 3)
typeof(x)
#> [1] "double"

length(x)
#> [1] 3
```

### Ordering Vectors

`order()` takes a vector as its input and returns an integer vector describing how to order the subsetted vector:

```r
x <- c("b", "c", "a")
order(x)
#> [1] 3 1 2

x[order(x)]
#> [1] "a" "b" "c"
```

The returned integer vector represents the indices needed to sort the original vector in ascending order.

### Boolean Operators

R has two sets of boolean operators with different behaviors:

- **Vector boolean operators** (`&` and `|`): Perform element-wise operations on vectors, returning a vector of the same length. These evaluate all elements regardless of the result.
- **Short-circuiting scalar operators** (`&&` and `||`): Only evaluate the first element of each vector and short-circuit (stop evaluating) once the result is determined. These are typically used in `if` statements and return a single logical value.

**When to use which:**

- Use `&` and `|` when working with vectors and you need element-wise boolean operations
- Use `&&` and `||` in conditional statements (`if`, `while`) where you only need to check a single condition and want short-circuiting behavior

### Converting Boolean to Integer Indices

`which()` allows you to convert a Boolean representation to an integer representation. It returns the indices of elements that are `TRUE`:

```r
x <- sample(10) < 4
which(x)
#> [1] 2 3 4
```

### NULL

`NULL` is a special object in R that represents an empty or undefined value:

```r
typeof(NULL)
#> [1] "NULL"

length(NULL)
#> [1] 0
```

Unlike other objects, `NULL` cannot have attributes set on it:

```r
x <- NULL
attr(x, "y") <- 1
#> Error in attr(x, "y") <- 1: attempt to set an attribute on NULL
```

You can test for `NULL` values with `is.null()`:

```r
is.null(NULL)
#> [1] TRUE
```

### Data Frames and Tibbles

The two most important S3 vectors built on top of lists are **data frames** and **tibbles**.

![S3 vector hierarchy](https://adv-r.hadley.nz/diagrams/vectors/summary-tree-s3-2.png)

**Key differences between data frames and tibbles:**

- **Printing**: Tibbles have a more informative print method that shows the first 10 rows and column types, while base R data frames print all rows by default (which can be overwhelming for large datasets)
- **Subsetting**: Tibbles are stricter about subsetting behavior and will always return a tibble, whereas data frames can sometimes return vectors
- **Column creation**: Tibbles allow you to refer to columns that were just created in the same expression, while data frames do not
- **Row names**: Tibbles do not support row names (they are converted to a regular column), while data frames do
- **Partial matching**: Tibbles never do partial matching when subsetting columns, while data frames do (which can lead to unexpected behavior)
- **Recycling**: Tibbles are stricter about recycling rules and will warn or error when values are recycled, while data frames silently recycle

**Subsetting behavior:**

Data frames have the characteristics of both lists and matrices:

- **Single index** (like lists): When subsetting with a single index, they behave like lists and index the columns, so `df[1:2]` selects the first two columns.
- **Two indices** (like matrices): When subsetting with two indices, they behave like matrices, so `df[1:3, ]` selects the first three rows (and all the columns).

Subsetting a tibble with `[` always returns a tibble, providing more consistent and predictable behavior.

**The `drop` parameter:**

Data frames with a single column will return just the content of that column (a vector) by default:

```r
df <- data.frame(a = 1:2, b = 1:2)
str(df[, "a"])
#>  int [1:2] 1 2

str(df[, "a", drop = FALSE])
#> 'data.frame':    2 obs. of  1 variable:
#>  $ a: int  1 2
```

The default `drop = TRUE` behaviour is a common source of bugs in functions: you check your code with a data frame or matrix with multiple columns, and it works. Six months later, you (or someone else) uses it with a single column data frame and it fails with a mystifying error. When writing functions, get in the habit of always using `drop = FALSE` when subsetting a 2D object. For this reason, tibbles default to `drop = FALSE`, and `[` always returns another tibble.

Tibbles are part of the **tidyverse** ecosystem and are generally preferred for modern R data analysis workflows due to their more predictable and user-friendly behavior.

### Why Use `subset()` or `dplyr::filter()` Instead of `[`

While `[` can be used for subsetting data frames, `subset()` and `dplyr::filter()` are generally preferred for several reasons:

- **Programmability**: `subset()` and `filter()` can be used inside functions, while `[` with non-standard evaluation can be problematic in non-interactive contexts.

- **Better defaults**: `subset()` sets `drop = FALSE` by default, guaranteeing a data frame is returned (avoiding the common bug where single-column subsetting returns a vector).

- **NA handling**: `subset(df, x == y)` automatically drops rows where the condition evaluates to `NA`, whereas `df[x == y,]` includes those rows. To achieve the same behavior with `[`, you'd need to write `df[x == y & !is.na(x == y), , drop = FALSE]`, which is much more verbose.

- **Additional features**: Modern alternatives like `dplyr::filter()` provide even more capabilities, such as translating R expressions to SQL for database queries, making them more powerful for programming workflows.

## Missing and Out-of-Bounds Indices (and the `purrr` package)

The behavior of `[[` with invalid indices (zero-length objects, out-of-bounds values, or missing values) is inconsistent across different data structures:

| `row[[col]]` | Zero-length | OOB (int) | OOB (chr) | Missing |
| ------------ | ----------- | --------- | --------- | ------- |
| Atomic       | Error       | Error     | Error     | Error   |
| List         | Error       | Error     | NULL      | NULL    |
| NULL         | NULL        | NULL      | NULL      | NULL    |

**Key points:**

- Atomic vectors always throw errors with invalid indices
- Lists return `NULL` for out-of-bounds character indices and missing values, but throw errors for zero-length and out-of-bounds integer indices
- `NULL` always returns `NULL` regardless of the index type
- If the vector is named, OOB/missing/NULL components will have names of `<NA>`

**The `purrr` package:**

Because of these differences in how various R objects handle subsetting and missing elements, the `purrr` package provides two helpful tools: `purrr::pluck()` and `purrr::chuck()`.

- **`pluck()`**: Always returns `NULL` (or the value of `.default`) when an element is missing - well-suited for deeply nested structures where components may not exist (common with JSON from web APIs)
- **`chuck()`**: Always throws an error when an element is missing - useful when you want to ensure an element exists

`pluck()` also allows mixing integer and character indices and provides a default value:

```r
x <- list(
  a = list(1, 2, 3),
  b = list(3, 4, 5)
)

purrr::pluck(x, "a", 1)
#> [1] 1

purrr::pluck(x, "c", 1)
#> NULL

purrr::pluck(x, "c", 1, .default = NA)
#> [1] NA
```

## Piping

The `magrittr` package provides a binary operator `%>%`, which is called the **pipe** and is pronounced as "and then". The pipe allows you to chain function calls together in a readable, left-to-right flow:

```r
library(magrittr)

x %>%
  deviation() %>%
  square() %>%
  mean() %>%
  sqrt()
#> [1] 0.274
```

This is equivalent to `sqrt(mean(square(deviation(x))))`, but the piped version is often more readable, especially with longer chains of operations. The pipe takes the result of the left-hand side and passes it as the first argument to the function on the right-hand side.

**Note**: As of R 4.1.0, base R includes a native pipe operator `|>` that provides similar functionality without requiring the `magrittr` package.

## Higher-Order Functions

Collectively, functionals, function factories, and function operators are called **higher-order functions** because they work with functions as their inputs or outputs. They fill out a two-by-two table based on whether they take functions as input and whether they return functions as output:

![Higher-order functions diagram](https://adv-r.hadley.nz/diagrams/fp.png)

### Functionals

- **Functionals** are functions (like `lapply()`) that take another function as an argument. Functionals allow you to take a function that solves the problem for a single input and generalise it to handle any number of inputs. Functionals are by far and away the most important technique and you'll use them all the time in data analysis.

**Base R equivalents to `purrr::map()`:**

- The base equivalent to `purrr::map()` is `lapply()`. The only difference is that `lapply()` does not support the helpers that you'll learn about below, so if you're only using `map()` from purrr, you can skip the additional dependency and use `lapply()` directly.

- Base R has two apply functions that can return atomic vectors: `sapply()` and `vapply()`. I recommend that you avoid `sapply()` because it tries to simplify the result, so it can return a list, a vector, or a matrix. This makes it difficult to program with, and it should be avoided in non-interactive settings. `vapply()` is safer because it allows you to provide a template, `FUN.VALUE`, that describes the output shape. If you don't want to use purrr, I recommend you always use `vapply()` in your functions, not `sapply()`. The primary downside of `vapply()` is its verbosity: for example, the equivalent to `map_dbl(x, mean, na.rm = TRUE)` is `vapply(x, mean, na.rm = TRUE, FUN.VALUE = double(1))`.

**Purrr formula shortcut:**

- Purrr supports a special shortcut using formulas created by `~` (pronounced "twiddle"). All purrr functions translate formulas into functions. You can use `.x` for one argument functions, `.x` and `.y` for two argument functions, and `..1`, `..2`, `..3`, etc. for functions with an arbitrary number of arguments. `.` remains for backward compatibility but is not recommended because it's easily confused with the `.` used by magrittr's pipe. This shortcut is particularly useful for generating random data:

```r
map_dbl(mtcars, ~ length(unique(.x)))
#>  mpg  cyl disp   hp drat   wt qsec   vs   am gear carb
#>   25    3   27   22   22   29   30    2    2    3    6

x <- map(1:3, ~ runif(2))
str(x)
#> List of 3
#>  $ : num [1:2] 0.281 0.53
#>  $ : num [1:2] 0.433 0.917
#>  $ : num [1:2] 0.0275 0.8249
```

**Common purrr functionals:**

- **`map()`**: Applies a function to each element of a vector or list and returns a list. This is the most fundamental purrr functional and the equivalent of `lapply()`.

- **`map2()`**: Applies a function to pairs of elements from two vectors or lists. Useful when you need to iterate over two inputs simultaneously. The function receives `.x` (from the first input) and `.y` (from the second input) as arguments.

- **`walk()`**: Similar to `map()`, but used for functions that perform side effects (like printing, plotting, or writing files) rather than returning values. Returns the input invisibly, making it useful in pipelines.

- **`walk2()`**: Similar to `map2()`, but for side effects. Applies a function to pairs of elements from two inputs and returns the inputs invisibly.

- **`pmap()`**: Applies a function to multiple arguments stored in a list or data frame. It's often convenient to call `pmap()` with a data frame. A handy way to create that data frame is with `tibble::tribble()`, which allows you to describe a data frame row-by-row (rather than column-by-column, as usual). Thinking about the parameters to a function as a data frame is a very powerful pattern. The following example shows how you might draw random uniform numbers with varying parameters:

```r
params <- tibble::tribble(
  ~ n, ~ min, ~ max,
   1L,     0,     1,
   2L,    10,   100,
   3L,   100,  1000
)

pmap(params, runif)
#> [[1]]
#> [1] 0.332
#>
#> [[2]]
#> [1] 53.5 47.6
#>
#> [[3]]
#> [1] 231 715 515
```

### Function Factories

- **Function factories** are functions that create functions. Function factories are less commonly used than functionals, but can allow you to elegantly partition work between different parts of your code.

**Forcing evaluation in function factories**: There's a subtle bug that can occur in function factories due to lazy evaluation. When a factory function uses an argument that is only referenced in the manufactured function, the argument is not evaluated until the manufactured function is called. This can lead to unexpected behavior if the binding changes between creating the function and calling it:

```r
power1 <- function(exp) {
  function(x) {
    x ^ exp
  }
}

x <- 2
square <- power1(x)
x <- 3
square(2)
#> [1] 8  # Unexpected! Should be 4
```

The problem occurs because `exp` is only evaluated lazily when `square()` is run, not when `power1()` is run. We can fix this by forcing evaluation with `force()`:

```r
power2 <- function(exp) {
  force(exp)  # Force evaluation of exp
  function(x) {
    x ^ exp
  }
}

x <- 2
square <- power2(x)
x <- 3
square(2)
#> [1] 4  # Correct!
```

**Important**: Whenever you create a function factory, make sure every argument is evaluated, using `force()` as necessary if the argument is only used by the manufactured function.

**Using function factories with `optimise()`**: Function factories are particularly useful when working with optimization functions like `optimise()`. Instead of manually trying different parameter values, `optimise()` can automatically find the optimal value by evaluating your function many times and using optimization algorithms to converge on the best solution.

A common use case is maximum likelihood estimation (MLE), where you want to find parameter values that maximize the likelihood of your data. You can create a function factory that captures your data and returns a log-likelihood function ready for optimization:

```r
# Function factory that captures data and returns a log-likelihood function
ll1 <- function(x) {
  function(lambda) {
    -sum(dpois(x, lambda, log = TRUE))
  }
}

# Use optimise() to find the maximum
optimise(ll1(x1), c(0, 100), maximum = TRUE)
#> $maximum
#> [1] 32.1
#>
#> $objective
#> [1] -30.3
```

Note that you don't strictly need a function factory here—`optimise()` passes `...` arguments through, so you could call the log-probability function directly:

```r
optimise(lprob_poisson, c(0, 100), x = x1, maximum = TRUE)
#> $maximum
#> [1] 32.1
#>
#> $objective
#> [1] -30.3
```

However, function factories really shine when you have more complex optimization problems with multiple parameters and data vectors. They provide a cleaner interface by encapsulating the data within the function, making it easier to work with optimization algorithms that expect a single-argument function.

### Function Operators

- **Function operators** are functions that take functions as input and produce functions as output. They are like adverbs, because they typically modify the operation of a function.

**Common function operators:**

- **`purrr::safely()`**: Transforms a function into a safe version that captures errors without halting execution. Returns a list with `result` and `error` elements. Useful for processing data where some inputs might fail. (See detailed notes above.)

- **`purrr::possibly()`**: Returns a default value when there's an error. It provides no way to tell if an error occurred or not, so it's best reserved for cases when there's some obvious sentinel value (like `NA`).

- **`purrr::quietly()`**: Turns output, messages, and warning side-effects into `output`, `message`, and `warning` components of the output. Useful for capturing and inspecting side effects programmatically.

- **`memoise::memoise()`**: Caches function results so that repeated calls with the same arguments return the cached value instead of recomputing. Useful for expensive computations or functions that make external API calls. The cache persists for the duration of the R session unless explicitly cleared with `memoise::forget()`.

**Note on memoization and pure functions**: Memoization is a key technique in dynamic programming where overlapping subproblems are solved once and cached, because complex problems can be broken down into many overlapping subproblems, and remembering the results of a subproblem considerably improves performance. But it only works correctly with pure functions whose output depends solely on their input—memoizing impure functions will produce misleading and confusing results.

## For Loops

`for` loops allow you to iterate over a sequence and execute code for each element. The basic syntax is:

```r
for (variable in sequence) {
  # code to execute
}
```

The loop variable takes on each value in the sequence, one at a time, and the code block is executed for each iteration.

### Variable Assignment

**Important**: `for` assigns the loop variable to the current environment, overwriting any existing variable with the same name:

```r
i <- 100
for (i in 1:3) {}
i
#> [1] 3
```

### Loop Control

There are two ways to terminate a `for` loop early:

- **`next`**: Exits the current iteration and continues to the next one
- **`break`**: Exits the entire `for` loop

```r
for (i in 1:10) {
  if (i < 3)
    next

  print(i)

  if (i >= 5)
    break
}
#> [1] 3
#> [1] 4
#> [1] 5
```

### Common Pitfalls

#### 1. Preallocate Output Containers

If you're generating data in a loop, always preallocate the output container. Otherwise the loop **will be very slow** because R has to copy and grow the vector/list on each iteration:

```r
means <- c(1, 50, 20)
out <- vector("list", length(means))
for (i in 1:length(means)) {
  out[[i]] <- rnorm(10, means[[i]])
}
```

The `vector()` function is helpful for preallocation.

#### 2. Avoid `1:length(x)` with Empty Vectors

Beware of iterating over `1:length(x)`, which will fail in unhelpful ways if `x` has length 0:

```r
means <- c()
out <- vector("list", length(means))
for (i in 1:length(means)) {
  out[[i]] <- rnorm(10, means[[i]])
}
#> Error in rnorm(10, means[[i]]): invalid arguments
```

This occurs because `:` works with both increasing and decreasing sequences:

```r
1:length(means)
#> [1] 1 0
```

**Solution**: Use `seq_along(x)` instead. It always returns a value the same length as `x`:

```r
seq_along(means)
#> integer(0)

out <- vector("list", length(means))
for (i in seq_along(means)) {
  out[[i]] <- rnorm(10, means[[i]])
}
```

#### 3. S3 Vectors Lose Attributes When Iterating

You might encounter problems when iterating over S3 vectors (like `Date` objects), as loops typically strip the attributes:

```r
xs <- as.Date(c("2020-01-01", "2010-01-01"))
for (x in xs) {
  print(x)
}
#> [1] 18262
#> [1] 14610
```

**Workaround**: Call `[[` yourself to preserve attributes:

```r
for (i in seq_along(xs)) {
  print(xs[[i]])
}
#> [1] "2020-01-01"
#> [1] "2010-01-01"
```

### Related Tools

`for` loops are useful when you know in advance the set of values to iterate over. If you don't, there are two related tools with more flexible specifications:

- **`while(condition) action`**: Performs action while condition is `TRUE`
- **`repeat(action)`**: Repeats action forever (until it encounters `break`)

**When to avoid loops**: Generally speaking, you shouldn't need to use `for` loops for data analysis tasks, as `map()` and `apply()` functions already provide less flexible (and often more efficient) solutions to most problems.

---

## Debugging Tools in R

More information on R debugging can be found here: https://adv-r.hadley.nz/debugging.html

### Locating Errors & Call Stacks

- **`traceback()`**  
  Shows the sequence of function calls leading to an error (read bottom → top).

- **RStudio “Show Traceback”**  
  GUI version of `traceback()` with clickable source lines.

- **`rlang::last_trace()`**  
  Displays a structured call tree; helpful with lazy evaluation.

- **`rlang::with_abort()`**  
  Turns conditions (warnings/messages) into errors for easier tracing.

---

### Interactive Debugging

- **`browser()`**  
  Pauses execution inside a function and opens an interactive prompt.

- **Browser Commands**

  - `n` — next line
  - `s` — step into function
  - `f` — finish current function
  - `c` — continue execution
  - `Q` — quit debugging
  - `where` — show call stack

- **RStudio “Rerun with Debug”**  
  Automatically reruns failing code and enters the debugger at the error.

---

### Breakpoints & Automatic Debugging

- **RStudio Breakpoints**

  - Click next to a line number or press `Shift + F9`.
  - Easier than inserting `browser()` manually.

- **`options(error = recover)`**

  - Opens an interactive menu after an error.
  - Allows debugging in any call frame.

- **`recover()`**
  - Manually invoked interactive debugger for inspecting call frames.

---

### Function-Level Debugging

- **`debug()` / `undebug()`**

  - Inserts/removes `browser()` at the start of a function.

- **`debugonce()`**

  - Debugs only the next function call.

- **`utils::setBreakpoint()`**

  - Sets a breakpoint using file name and line number.

- **`trace()` / `untrace()`**
  - Injects custom code into an existing function.
  - Useful for debugging package or base R code.

---

### Non-Interactive Debugging (Batch Jobs, Pipelines)

- **`dump.frames()`**

  - Saves the call stack to `last.dump.rda` for later debugging.

- **`debugger()`**

  - Loads dumped frames and provides a `recover()`-like interface.

- **Print Debugging (`print()`, `cat()`)**

  - Manual logging of execution and variable values.
  - Slow but always reliable.

- **`callr::r()`**
  - Runs code in a clean R session to reproduce environment issues.

You might also want to double check for these common issues:

- Is the global environment different? Have you loaded different packages? Are objects left from previous sessions causing differences?

- Is the working directory different?

- Is the `PATH` environment variable, which determines where external commands (like git) are found, different?

- Is the `R_LIBS` environment variable, which determines where `library()` looks for packages, different?

---

### Debugging RMarkdown

- **`rmarkdown::render()`**

  - Runs RMarkdown in the current session for easier debugging.

- **`sink()`**

  - Restores console output when debugging knitr errors.

- **`rlang::trace_back()`**
  - Produces cleaner tracebacks for RMarkdown errors.

---

### Handling Non-Error Failures

- **`options(warn = 2)`**

  - Converts warnings into errors.

- **Message-to-error conversion**

  - Use `rlang::with_abort(..., "message")` to trace messages.

- **Infinite loops**
  - Interrupt execution and inspect `traceback()` or use print debugging.

---

### Compiled Code Debugging

- **External debuggers (gdb, lldb, valgrind)**
  - Required for crashes caused by C/C++ code.
  - Use minimal reproducible examples when reporting bugs.

---

## Profiling tools in R

- Use **profiling** to identify bottlenecks in real code.
- Use **microbenchmarking** to compare small alternative implementations.
- Always measure performance using **realistic inputs**.
- Avoid premature optimization—optimize only after you know what matters.

---

R uses a **sampling (statistical) profiler**:

- Execution is paused every few milliseconds.
- The current call stack is recorded (i.e., which function is currently executing, and the function that called it, and so on).
- Results are approximate but low-overhead.

Profiling tells you:

- Which functions run the most
- Where execution time is spent
- Whether time is lost to garbage collection

---

### Profiling Tools — Notes

#### `utils::Rprof()`

- Base R profiling tool.
- Records call stacks at regular intervals.
- Produces raw output that can be hard to interpret directly.
- Results vary slightly between runs due to randomness.

---

#### `profvis`

- **Main recommended profiling tool.**
- Visualizes profiling results interactively.
- Connects performance data to source code.
- Displays:
  - **Source view** (time & memory per line)
  - **Flame graph** (call stack visualization)
  - **Data tree** (interactive exploration)

Usage:

```r
profvis::profvis(f())
```

After profiling is complete, profvis will open an interactive HTML document that allows you to explore the results.

**Flame Graph**

- Width = time spent
- Height = call depth

Helps identify:

- Repeated function calls
- Expensive call paths

![Flame graph example from Advanced R](https://adv-r.hadley.nz/screenshots/performance/info.png)

**`<GC>` (Garbage Collector)**

- Appears in flame graphs when memory is heavily allocated/freed.
- Indicates creation of many short-lived objects.
- Often caused by:

  - Growing objects in loops
  - Copy-on-modify behavior

**Memory Profiling (via profvis)**

- Memory bars show allocation (right) and freeing (left).
- Helps identify memory inefficiencies.
- Especially useful when `<GC>` dominates runtime.

---

### Other Profiling Tools

**`utils::summaryRprof()`**

- Aggregates profiling data from `Rprof()`.

**`proftools` package**

- Alternative profiling visualizations.

---

### Profiling Limitations

- Does not profile inside C/C++ code.
- Anonymous functions are hard to distinguish.
- Lazy evaluation can distort call stacks.
- Use `force()` to simplify lazy evaluation effects.

---

### Microbenchmarking (Comparing Small Code Snippets)

#### What Microbenchmarking Is

- Measures extremely fast operations (ns, µs, ms).
- Used to compare small alternatives for a specific task.
- Results should not be overgeneralized to full programs.

#### `bench::mark()`

- High-precision timing for small expressions.
- Automatically repeats expressions many times.
- Verifies identical results by default.

Example:

```r
bench::mark(
  sqrt(x),
  x ^ 0.5
)
```

---

#### `bench::mark()` Output Columns

- **min** – fastest observed time (best case)
- **median** – typical execution time
- **max / mean** – less reliable due to skew
- **itr/sec** – iterations per second
- **mem_alloc** – memory allocated
- **gc/sec** – garbage collections per second
- **n_itr** – total iterations
- **total_time** – total elapsed time

Focus on:

- `min` and `median`
- Memory allocation and garbage collection

---

#### `plot()` (for bench results)

- Visualizes timing distributions.
- Distributions are often right-skewed.
- Avoid comparing means.

Example:

```r
plot(results)
```

---

### Interpreting Performance Results

#### Time Scale Awareness

- **1 ms** → 1,000 calls per second
- **1 µs** → 1,000,000 calls per second
- **1 ns** → 1,000,000,000 calls per second

Small differences (nanoseconds) often do not matter in real programs unless  
the code runs millions of times.

---

## Improving Performance in R

### 1. Organize code for safe optimization

- Write **separate functions** for each optimization idea.
- Use **representative test data** (large enough to matter, small enough to be fast).
- Benchmark alternatives with tools like `bench::mark()`.
- Keep records of attempts (e.g., with R Markdown).
- Verify results are equivalent (ideally with unit tests).

---

### 2. Look for existing solutions

- Many performance problems are already solved:
  - Check **CRAN task views**
  - Explore packages using **C++ (e.g., Rcpp)**
  - Search StackOverflow and R-specific resources
- Even slower solutions may be easier to optimize later.

---

### 3. Do as little work as possible

- Use **more specific functions**:
  - `rowSums()`, `colMeans()` > `apply()`
  - `vapply()` > `sapply()`
  - `any(x == v)` > `v %in% x` (single-value checks)
- Avoid unnecessary coercions (e.g., `apply()` on data frames).
- Provide extra information when possible:
  - `colClasses` in `read.csv()`
  - `levels` in `factor()`
- Avoid generic dispatch in tight loops:
  - `mean.default()` > `mean()`
  - `.Internal()` calls are fastest but unsafe

**Trade-off:** Speed often comes at the cost of safety and generality.

---

### 4. Rewrite for known input types

- Base R functions prioritize flexibility over speed.
- Specialized versions can be orders of magnitude faster.
- Requires reading and understanding source code.
- Risky if assumptions about inputs are violated.

---

### 5. Vectorize your code

- Vectorization means operating on **entire vectors or matrices**, not scalars.
- Benefits:
  - Simpler code
  - Loops executed in **C**, not R
- Key tools:
  - `rowSums()`, `rowMeans()`, `cumsum()`, `diff()`
  - `cut()`, `findInterval()`
  - Matrix algebra (`%*%`, `crossprod()`)
- Scaling can be non-linear—benchmark carefully.

---

### 6. Avoid unnecessary copying

- Growing objects in loops is slow:
  - `c()`, `append()`, `rbind()`, `paste()` inside loops
- Prefer preallocation or vectorized alternatives.
- Object modification may silently trigger copies.

---

### 7. Case study: speeding up t-tests

Performance gains achieved by:

1. Avoiding slow interfaces (formula vs vectors)
2. Computing only what’s needed (t-statistic only)
3. Vectorizing across rows with matrix operations

**Result:** Up to **1000× speedup** over naïve `t.test()` usage.

---

### 8. Broader performance skills

- Read performance-focused R blogs and source code.
- Learn algorithms and data structures.
- Use parallel computing when appropriate.
- Ask good questions with minimal reproducible examples.

---

## Extra Readings

- [Making functions less of a "black box" with `debug()` in R Studio](https://www.youtube.com/watch?v=x7BdImJ6loA)
- [An introduction to R7 (aka S7)](https://www.youtube.com/watch?v=P3FxCvSueag)
- [Cracking open ggplot internals with {ggtrace}](https://www.youtube.com/watch?v=dUBnitXf5mk)
- https://medium.com/accredian/business-analytics-modeling-automated-radiant-r-1942230eae63
- https://renkun.me/2022/03/06/my-recommendations-of-vs-code-extensions-for-r/
