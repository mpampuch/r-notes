# Debugging Tools, Techniques, and Packages in Advanced R

*Notes from Hadley Wickham's "Advanced R" (https://adv-r.hadley.nz/)*

## Overview

Advanced R dedicates Chapter 22 to debugging, providing a comprehensive guide to finding and fixing bugs in R code. The book emphasizes that debugging is both an art and science, requiring systematic approaches and the right tools.

## General Debugging Strategy (22.2)

The book outlines a four-step debugging process:

1. **Google!** - Search error messages online
   - Remove variable names/values specific to your problem
   - Use packages: `errorist` and `searcher` for automated web searching

2. **Make it repeatable** - Create reproducible examples
   - Use binary search to isolate the problem
   - Create minimal reproducible examples
   - Add automated tests if available

3. **Figure out where it is** - Locate the exact source
   - Use scientific method: generate hypotheses, test them
   - Ask for help with small, shareable examples

4. **Fix it and test it** - Verify the solution works
   - Use automated tests to prevent regressions

## Core Debugging Tools

### 1. Call Stack Analysis (`traceback()`)

**Purpose**: Shows the sequence of function calls leading to an error

**Base R Function**: `traceback()`
- Shows call stack from bottom (initial call) to top (error location)
- Displays filename and line numbers for sourced code
- Read output from bottom to top

**RStudio Integration**: 
- "Show Traceback" button appears with errors
- Clickable links to source code locations

**Limitations**: 
- Linearizes call tree, can be confusing with lazy evaluation

### 2. rlang Alternatives for Call Stack

**Package**: `rlang`

**Functions**:
- `rlang::with_abort()` - Captures errors with better call tree display
- `rlang::last_trace()` - Shows hierarchical call tree (opposite order to `traceback()`)

**Advantages**: 
- Better visualization of complex call hierarchies
- Handles lazy evaluation more clearly

### 3. Interactive Debugger

#### RStudio Tools:
- **"Rerun with Debug"** - Automatically enters debugger at error location
- **Breakpoints** - Click left of line number or press `Shift + F9`
- **Debugging toolbar** with visual controls

#### Base R Functions:

**`browser()`**:
- Insert at any location in code
- Can be conditional: `if (condition) browser()`
- Special commands:
  - `n` (next) - Execute next step
  - `s` (step into) - Step into functions
  - `f` (finish) - Finish current loop/function
  - `c` (continue) - Continue execution
  - `Q` (quit) - Stop debugging
  - `where` - Print stack trace

**`debug()`/`undebug()`**:
- `debug(function_name)` - Insert browser at function start
- `debugonce()` - Debug only next run
- `undebug()` - Remove debugging

**`utils::setBreakpoint()`**:
- Takes filename and line number instead of function name

**`trace()`/`untrace()`**:
- More flexible - insert arbitrary code at any function location
- Useful for debugging code you don't have source for

### 4. Error Handling Options

**`options(error = recover)`**:
- Interactive prompt showing traceback
- Choose which frame to debug
- Reset with `options(error = NULL)`

**`options(error = browser)`**:
- Automatically enters browser at error location

## Non-Interactive Debugging (22.5)

### For Batch/Automated Code:

**`dump.frames()`**:
- Equivalent to `recover()` for non-interactive code
- Saves debugging info to `last.dump.rda`
- Load in interactive session and use `debugger()`

**Example Pattern**:
```r
# In batch process
dump_and_quit <- function() {
  dump.frames(to.file = TRUE)
  q(status = 1)
}
options(error = dump_and_quit)

# Later in interactive session
load("last.dump.rda")
debugger()
```

### Print Debugging:
- Insert `cat()`, `print()`, or `message()` statements
- Start coarse-grained, make progressively finer
- Particularly useful for compiled code

### callr Package:
- `callr::r(f, list(1, 2))` - Run function in fresh session
- Helps reproduce problems that don't occur interactively

## RMarkdown Debugging (22.5.3)

**Key Challenges**: Different execution environment

**Solutions**:
1. Use `rmarkdown::render()` instead of RStudio's "Knit" button
2. Special error handlers need `sink()` call:

```r
options(error = function() {
  sink()
  recover()
})
```

3. For tracebacks, use `rlang::trace_back()` with options:
```r
options(rlang_trace_top_env = rlang::current_env())
options(error = function() {
  sink()
  print(rlang::trace_back(bottom = sys.frame(-1)), simplify = "none")
})
```

## Condition System (Chapter 8)

### Signaling Functions:
- `stop()` / `rlang::abort()` - Errors
- `warning()` / `rlang::warn()` - Warnings  
- `message()` / `rlang::inform()` - Messages

### Handling Functions:
- `try()` - Continue execution after errors
- `tryCatch()` - Handle specific condition types
- `withCallingHandlers()` - Handle without interrupting execution
- `suppressWarnings()` - Ignore warnings
- `suppressMessages()` - Ignore messages

### Custom Conditions:
- Create with `rlang::abort()` using `.subclass` parameter
- Include metadata for programmatic handling
- Better than parsing error message strings

## Debugging Packages and Tools

### Core Packages:
1. **`rlang`** - Enhanced condition handling and tracing
2. **`errorist`** - Automatically searches web for error messages
3. **`searcher`** - Web search functionality for errors

### RStudio Features:
- Interactive debugger with visual interface
- Breakpoint system
- Environment inspection during debugging
- Traceback visualization

### Additional Utilities:
- `lobstr::cst()` - Call stack tree visualization
- `options(browserNLdisabled = TRUE)` - Disable accidental Enter repeats in browser

## Advanced Topics

### Call Stack Consistency Issues:
Different tools show call stacks differently:
- `traceback()` vs `where` vs `recover()` have different numbering
- `rlang` functions use consistent ordering with `recover()`

### Non-Error Failures (22.6):
- **Unexpected warnings**: Convert to errors with `options(warn = 2)`
- **Unexpected messages**: Use `rlang::with_abort()` to convert to errors
- **Infinite loops**: Terminate and examine `traceback()`
- **R crashes**: Indicates compiled code bugs, use print debugging

### Compiled Code Debugging (22.4.3):
- Use interactive C debuggers (gdb, lldb)
- Resources provided for advanced debugging
- Beyond scope of main R debugging

## Best Practices

1. **Start small** - Work interactively on small pieces rather than debugging large functions
2. **Use systematic approach** - Scientific method over intuition
3. **Create tests** - Prevent regressions and verify fixes
4. **Automate searches** - Use `errorist`/`searcher` packages
5. **Document solutions** - Record fixes for future reference
6. **Use appropriate tools** - Choose right debugging method for the situation

## Key Packages Summary

| Package | Purpose | Key Functions |
|---------|---------|---------------|
| `base` | Core debugging | `traceback()`, `browser()`, `debug()`, `recover()` |
| `rlang` | Enhanced conditions | `abort()`, `last_trace()`, `with_abort()`, `catch_cnd()` |
| `errorist` | Auto web search | Automatic error lookup |
| `searcher` | Web search | Manual error searching |
| `utils` | Utilities | `setBreakpoint()`, `dump.frames()` |
| `callr` | Fresh sessions | `r()` for isolated execution |

This comprehensive debugging toolkit makes R one of the better languages for diagnosing and fixing code issues, especially when combined with RStudio's integrated development environment.