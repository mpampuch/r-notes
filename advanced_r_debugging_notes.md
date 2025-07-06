# Debugging Tools, Techniques, and Packages in Advanced R

*Notes from Hadley Wickham's "Advanced R" (https://adv-r.hadley.nz/)*

## Overview

Advanced R dedicates Chapter 22 to debugging, providing a comprehensive guide to finding and fixing bugs in R code. The book emphasizes that debugging is both an art and science, requiring systematic approaches and the right tools.

## Complete Package Reference: All Debugging & Analysis Tools in Advanced R

### Core Debugging Packages
- **`rlang`**: Enhanced condition handling and error tracing
  - `with_abort()`: Capture errors with better call tree visualization  
  - `last_trace()`: Show hierarchical error traces
  - `trace_back()`: Programmatic trace generation
  - `abort()`: Enhanced error throwing with structured conditions

- **`errorist`**: Automated web search for error messages
  - Automatically opens browser with Google search of error messages
  - Removes code-specific details for better search results

- **`searcher`**: Manual error searching functionality  
  - `search_*()` functions for manual error lookups
  - Supports multiple search engines and R-specific sites

### Memory Analysis & Profiling Packages

#### Current/Modern Tools
- **`lobstr`**: Modern memory profiling and object inspection
  - `obj_size()`: Calculate object sizes accounting for shared references
  - `mem_used()`: Total memory usage by R session
  - `obj_addr()`: Find memory addresses of objects  
  - `ref()`: Display reference trees showing object relationships
  - `ast()`: Display abstract syntax trees
  - `cst()`: Call stack trees

- **`profvis`**: Interactive profiling visualization (current standard)
  - Flame graphs showing call stack over time
  - Source code annotation with timing and memory data
  - Memory allocation and release tracking
  - RStudio integration with interactive exploration
  - HTML output for sharing results

#### Legacy Tools (from older editions)
- **`pryr`**: Memory profiling utilities (largely replaced by lobstr)
  - `object_size()`: Object size calculation
  - `mem_used()`, `mem_change()`: Memory tracking
  - `address()`, `refs()`: Memory location and reference counting
  - `tracemem()` integration

- **`lineprof`**: Line-by-line memory profiling (deprecated, replaced by profvis)
  - Memory allocation per line of code
  - Integration with shiny for interactive exploration

### Performance & Benchmarking Packages

#### Modern Benchmarking
- **`bench`**: Modern microbenchmarking (current standard)
  - `mark()`: High precision timing with memory allocation tracking
  - Automatic garbage collection analysis
  - Built-in statistical analysis of timing results
  - Memory profiling integration

#### Traditional Benchmarking  
- **`microbenchmark`**: Traditional microbenchmarking package
  - `microbenchmark()`: Function timing comparisons
  - Statistical summary of multiple runs
  - Support for different time units

### Parallel Computing & Process Management
- **`parallel`**: Base R parallel computing
  - `mclapply()`: Multicore parallel processing (Unix/Mac)
  - `parLapply()`: Cluster parallel processing (cross-platform)  
  - `detectCores()`: System core detection
  - `makePSOCKcluster()`: Create process clusters

- **`callr`**: Run R processes in isolation
  - `r()`: Execute functions in fresh R sessions
  - Useful for reproducing non-interactive debugging issues
  - Clean environment testing

### Base R Profiling Tools
- **`utils::Rprof()`**: Base R statistical profiler
  - Memory and time profiling
  - Raw profiling data collection
  - Used by other profiling packages as foundation

- **`utils::summaryRprof()`**: Summarize Rprof output
  - Basic profiling summaries
  - Less user-friendly than modern alternatives

### Build & Development Tools
- **`compiler`**: Byte code compilation for performance
  - `cmpfun()`: Compile functions to byte code
  - Automatic compilation in newer R versions

### Error Search Integration
The book specifically mentions automated error searching:
- **Web-based error lookup**: Both `errorist` and `searcher` help automate the first debugging step
- **Search engine integration**: Direct browser opening with properly formatted queries
- **R-specific search sites**: Integration with R-focused search engines and Stack Overflow

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

#### Base R Function: `traceback()`

**How it works**: When an error occurs, R maintains a record of all the function calls that led to that error. `traceback()` displays this "call stack" showing exactly how execution reached the point of failure.

**Basic Usage**:
```r
# After an error occurs, immediately run:
traceback()
```

**Reading the Output**:
- Read from **bottom to top** (reverse chronological order)
- Bottom line = initial function call
- Top line = where error actually occurred
- Numbers indicate call depth

**Example**:
```r
f <- function(a) g(a)
g <- function(b) h(b) 
h <- function(c) i(c)
i <- function(d) {
  if (!is.numeric(d)) {
    stop("`d` must be numeric", call. = FALSE)
  }
  d + 10
}

f("a")  # This will error

traceback()
# Output:
# 4: i(c) at debugging.R#3
# 3: h(b) at debugging.R#2  
# 2: g(a) at debugging.R#1
# 1: f("a")
```

**Advanced Features**:
- Shows filename and line numbers for sourced code (`filename.R#linenumber`)
- Line numbers are clickable in RStudio
- Works with both interactive and sourced code

**Setting Up Automatic Traceback**:
```r
# Set traceback to run automatically after any error
options(error = traceback)

# Reset to default behavior
options(error = NULL)
```

**RStudio Integration**: 
- **"Show Traceback" button** appears automatically with errors
- **Clickable links** to source code locations
- **Visual formatting** makes call stack easier to read
- **Integrated with editor** - click to jump to error location

**Limitations**: 
- **Linearizes call tree**: Can be confusing with lazy evaluation
- **Limited context**: Shows where but not why error occurred
- **Call numbering inconsistency**: Different from other tools like `recover()`

**When to Use**:
- First step in debugging any error
- Understanding complex function call chains
- Identifying exactly where in your code the error originated
- Debugging sourced scripts where line numbers help locate issues

### 2. rlang Alternatives for Call Stack

**Package**: `rlang` - Modern condition handling and enhanced error tracing

#### Key Functions:

##### `rlang::with_abort()`
**Purpose**: Captures errors with enhanced call tree visualization

**Usage**:
```r
library(rlang)

# Wrap your code to get better error traces
with_abort(f(j()))
```

**Example with Complex Call Tree**:
```r
j <- function() k()
k <- function() stop("Oops!", call. = FALSE)

# Compare standard traceback vs rlang
f(j())  # Standard error
traceback()  # Linear view

# Enhanced with rlang
with_abort(f(j()))  # Better visualization
```

**Benefits**:
- **Hierarchical display**: Shows true function call relationships
- **Better lazy evaluation handling**: Clearer when arguments are evaluated
- **Enhanced metadata**: More information about the error context

##### `rlang::last_trace()`
**Purpose**: Shows the most recent error trace with hierarchical formatting

**Usage**:
```r
# After an error occurs (must be captured with rlang functions)
last_trace()
```

**Output Format**:
```r
# Example output shows branching structure:
# 1. ├─global::f(j())
# 2. │ └─global::g(a)  
# 3. │   └─global::h(b)
# 4. │     └─global::i(c)
# 5. └─global::j()
# 6.   └─global::k()
```

**Key Differences from `traceback()`**:
- **Opposite ordering**: Same as `recover()` (1 = top level)
- **Tree structure**: Shows branching with visual indicators
- **Lazy evaluation clarity**: Separate branches for lazily evaluated arguments

##### `rlang::trace_back()`
**Purpose**: Generate traces programmatically with more control

**Usage**:
```r
# In error handlers or for custom error reporting
trace_back(bottom = sys.frame(-1))
```

**Advanced Options**:
```r
# Control what's included in the trace
trace_back(
  bottom = sys.frame(-1),    # Where to start the trace
  simplify = "none"          # Level of simplification
)
```

#### Integration with Error Handling:

**Custom Error Functions**:
```r
my_stop <- function(msg) {
  abort(msg, trace = trace_back())
}

my_function <- function() {
  my_stop("Custom error with trace")
}
```

**Conditional Tracing**:
```r
if (debug_mode) {
  with_abort(risky_function())
} else {
  risky_function()
}
```

**Advantages over Base R**: 
- **Better visualization**: Tree structure vs linear list
- **Lazy evaluation clarity**: Separate branches for argument evaluation
- **Consistent numbering**: Same order as `recover()`
- **Enhanced metadata**: More context about error conditions
- **Programmable**: Can capture and manipulate traces programmatically

**When to Use**:
- Complex nested function calls where relationships matter
- Debugging lazy evaluation issues
- When standard `traceback()` is confusing
- Building custom error reporting systems
- Working with modern tidyverse-style code

### 3. Interactive Debugger

The interactive debugger allows you to pause code execution and explore the current state of variables, step through code line by line, and understand exactly what's happening at the point of error.

#### RStudio Tools:

##### "Rerun with Debug"
**Purpose**: Automatically enters debugger exactly where an error occurred

**How to Use**:
1. Run code that produces an error
2. Click **"Rerun with Debug"** button that appears next to error
3. RStudio reruns the command and pauses at the error location

**What You Get**:
- **Editor highlighting**: Shows the exact line about to be executed
- **Environment panel**: Displays all variables in current scope
- **Traceback panel**: Shows the call stack leading to this point
- **Console debugging**: Interactive R prompt within the function

**Example Workflow**:
```r
# This function will error
problematic_function <- function(x) {
  y <- x * 2
  z <- y + invalid_variable  # Error here
  return(z)
}

problematic_function(5)  # Error occurs
# Click "Rerun with Debug" - pauses before the error line
# You can examine y, x, and see that invalid_variable doesn't exist
```

##### Breakpoints
**Purpose**: Pause execution at specific lines you choose

**Setting Breakpoints**:
- **Mouse**: Click to the left of line numbers in the editor
- **Keyboard**: Place cursor on line and press `Shift + F9`
- **Menu**: Debug > Toggle Breakpoint

**Visual Indicators**:
- **Red circle**: Active breakpoint
- **Hollow circle**: Disabled breakpoint
- **Red highlighting**: Current execution point during debugging

**Managing Breakpoints**:
```r
# From console - set breakpoint programmatically
browser()  # Equivalent to a breakpoint at this line

# Conditional breakpoints (manual)
if (suspicious_condition) browser()
```

**Advanced Breakpoint Features**:
- **Enable/Disable**: Right-click breakpoint to toggle
- **Remove All**: Debug > Clear All Breakpoints
- **Breakpoint List**: View all breakpoints in project

**Limitations**:
- Don't work in non-saved files
- Don't work in Shiny applications
- RStudio doesn't support conditional breakpoints (use `if` statements)

##### Debugging Toolbar
**Visual Controls** (appears during debugging):
- **Next** (`n`): Execute current line
- **Step Into** (`s`): Enter function calls  
- **Step Out** (`f`): Finish current function
- **Continue** (`c`): Resume normal execution
- **Stop** (`Q`): Exit debugger

#### Base R Functions:

##### `browser()`
**Purpose**: The most fundamental debugging function - pauses execution anywhere

**Basic Usage**:
```r
my_function <- function(x) {
  y <- x * 2
  browser()  # Execution pauses here
  z <- y + 10
  return(z)
}
```

**Interactive Commands** (in browser prompt):
- **`n` (next)**: Execute the next statement
- **`s` (step into)**: Step into function calls
- **`f` (finish)**: Finish current loop or function
- **`c` (continue)**: Exit browser and continue execution
- **`Q` (quit)**: Stop debugging and return to top level
- **`where`**: Show stack trace (like `traceback()`)
- **Variable names**: Type any variable name to see its value
- **`ls()`**: List all objects in current environment

**Conditional Debugging**:
```r
my_function <- function(x) {
  for (i in 1:length(x)) {
    if (x[i] < 0) browser()  # Only debug negative values
    result[i] <- sqrt(x[i])
  }
}
```

**Temporary Debugging**:
```r
# Add browser() for debugging, remove when done
if (debugging_mode) browser()
```

**Browser Prompt Features**:
```r
Browse[1]>  # The prompt indicates you're in debugging mode
Browse[1]> x  # View variable x
Browse[1]> str(data)  # Examine data structure  
Browse[1]> n  # Next line
```

##### `debug()` and `undebug()`
**Purpose**: Automatically insert `browser()` at the start of any function

**Basic Usage**:
```r
# Enable debugging for a function
debug(my_function)

# Now every call to my_function will enter debugger
my_function(data)  # Automatically starts in browser

# Disable debugging
undebug(my_function)
```

**Advanced Features**:
```r
# Debug only the next call
debugonce(my_function)
my_function(data)  # Debugs this call
my_function(data)  # This call runs normally

# Debug functions in packages
debug(lm)  # Debug the linear model function
```

**Debugging Package Functions**:
```r
# Debug exported functions
debug(dplyr::filter)

# Debug internal functions (use :::)
debug(dplyr:::filter.data.frame)
```

**Checking Debug Status**:
```r
# See which functions are being debugged
isdebugged(my_function)

# List all debugged functions
for (obj in ls(envir = .GlobalEnv)) {
  if (is.function(get(obj)) && isdebugged(get(obj))) {
    cat(obj, "is debugged\n")
  }
}
```

##### `utils::setBreakpoint()`
**Purpose**: Set breakpoints by filename and line number rather than function name

**Usage**:
```r
# Set breakpoint at specific file location
setBreakpoint("my_script.R", line = 25)

# Clear specific breakpoint
setBreakpoint("my_script.R", line = 25, clear = TRUE)

# List current breakpoints
getBreakpoints()
```

**When Useful**:
- Debugging scripts rather than functions
- When you don't know the function name
- Setting multiple breakpoints in a file

##### `trace()` and `untrace()`
**Purpose**: Most flexible debugging tool - insert arbitrary code at any location

**Basic Function Tracing**:
```r
# Trace function entry/exit
trace(my_function)
my_function(data)  # Prints "trace: my_function" when called

# Remove tracing
untrace(my_function)
```

**Insert Custom Code**:
```r
# Insert browser() at function start
trace(my_function, browser)

# Insert custom debugging code
trace(my_function, quote({
  cat("Entering my_function with args:", deparse(substitute(x)), "\n")
  browser()
}))
```

**Conditional Tracing**:
```r
# Only enter debugger under certain conditions
trace(my_function, quote({
  if (any(x < 0)) {
    cat("Negative values detected!\n")
    browser()
  }
}))
```

**Tracing at Specific Lines**:
```r
# Insert code at specific points in function
trace(my_function, 
      quote(browser()), 
      at = 3)  # Insert at 3rd expression in function

# Find expression numbers
as.list(body(my_function))  # Shows numbered expressions
```

**Advanced Tracing Examples**:
```r
# Trace with custom condition checking
trace(problematic_function, quote({
  if (is.null(data) || nrow(data) == 0) {
    cat("Warning: empty data detected\n")
    browser()
  }
}), print = FALSE)

# Trace function exit to see return values  
trace(my_function, exit = quote({
  cat("Function returning:", str(returnValue()), "\n")
}))
```

**When to Use `trace()`**:
- Debugging functions you can't modify (e.g., package functions)
- Need conditional debugging with complex conditions
- Want to insert logging without modifying source code
- Debugging at specific points within a function
- Need to see function arguments and return values

### 4. Error Handling Options

These functions change R's default behavior when errors occur, automatically launching debugging tools instead of just showing an error message.

#### `options(error = recover)`
**Purpose**: Automatically launch interactive debugger with frame selection when any error occurs

**Setup**:
```r
# Enable automatic recovery on all errors
options(error = recover)

# Restore default error behavior
options(error = NULL)
```

**How It Works**:
When an error occurs, instead of just printing an error message, R:
1. **Shows the call stack** (like `traceback()`)
2. **Prompts for frame selection** - choose which function environment to debug
3. **Enters browser mode** in the selected frame

**Interactive Session Example**:
```r
options(error = recover)

f <- function(a) g(a)
g <- function(b) h(b)
h <- function(c) stop("Error in h!")

f(1)
# Output:
# Error: Error in h!
# 
# Enter a frame number, or 0 to exit   
# 
# 1: f(1)
# 2: g(a)  
# 3: h(b)
# 
# Selection: 2    # Choose frame 2 to debug g()
# Called from: g(a)
# Browse[1]>      # Now debugging inside g()
```

**Frame Selection**:
- **Frame numbers**: Correspond to call stack levels
- **0**: Exit without debugging
- **1**: Top-level call (usually least useful)
- **Higher numbers**: Deeper in call stack (often more useful)

**Advantages**:
- **Multiple debugging points**: Can examine any level of the call stack
- **Context switching**: Move between different function environments
- **Non-intrusive**: No need to modify code with `browser()` calls
- **Great for package debugging**: Debug functions you don't control

**Use Cases**:
- Debugging deep call stacks where error source is unclear
- Examining intermediate states in complex function chains
- Debugging package functions where you can't add `browser()`
- Understanding data flow through multiple function calls

#### `options(error = browser)`
**Purpose**: Automatically enter `browser()` at the exact location where error occurred

**Setup**:
```r
# Enable automatic browser on all errors
options(error = browser)

# Restore default error behavior  
options(error = NULL)
```

**How It Works**:
- **Immediate debugging**: Enters browser exactly where the error occurred
- **Full context**: Access to all local variables at error point
- **Same interface**: Uses standard `browser()` commands (`n`, `s`, `f`, `c`, `Q`)

**Example**:
```r
options(error = browser)

problematic_function <- function(x) {
  y <- x * 2
  z <- y + undefined_variable  # Error here
  return(z)
}

problematic_function(5)
# Automatically enters browser at the error line
# Browse[1]> y    # Can examine variables
# [1] 10
# Browse[1]> ls() # See all available objects
```

**Comparison with `recover()`**:

| Feature | `browser` | `recover` |
|---------|-----------|-----------|
| **Location** | Exact error point | Choose any frame |
| **Context** | Current function only | Any function in stack |
| **Speed** | Immediate | Requires frame selection |
| **Flexibility** | Less | More |
| **Best for** | Simple debugging | Complex call stacks |

#### Custom Error Handlers

**Combining Approaches**:
```r
# Custom error handler that gives options
my_error_handler <- function() {
  cat("Error occurred! Choose debugging method:\n")
  cat("1. browser() at error location\n") 
  cat("2. recover() with frame selection\n")
  cat("3. Just traceback\n")
  
  choice <- readline("Enter choice (1-3): ")
  
  switch(choice,
    "1" = browser(),
    "2" = recover(),
    "3" = traceback()
  )
}

options(error = my_error_handler)
```

**One-Time Error Handling**:
```r
# Set up temporary error handling
browseOnce <- function() {
  old <- getOption("error")
  function() {
    options(error = old)  # Restore original
    browser()             # Debug this error
  }
}

options(error = browseOnce())
# Next error will be debugged, then behavior returns to normal
```

**Conditional Error Handling**:
```r
# Only use special handling in certain contexts
smart_error_handler <- function() {
  if (interactive() && getOption("debug_mode", FALSE)) {
    recover()
  } else {
    traceback()
  }
}

options(error = smart_error_handler)
options(debug_mode = TRUE)  # Enable debugging
```

#### Error Handler Management

**Saving and Restoring**:
```r
# Save current error handler
old_error_handler <- getOption("error")

# Set temporary handler
options(error = recover)

# Restore original handler later
options(error = old_error_handler)
```

**Checking Current Handler**:
```r
# See what error handler is currently set
getOption("error")
```

**Default Handlers**:
```r
# Reset to completely default behavior
options(error = NULL)

# Set to print traceback automatically
options(error = traceback)
```

#### Best Practices

**When to Use Each**:
- **`recover`**: Complex debugging, multiple functions, unclear error source
- **`browser`**: Simple debugging, single function issues, examining local state
- **Custom handlers**: Specific workflows, conditional debugging, team environments

**Temporary Usage**:
```r
# Good practice: temporarily enable, then restore
{
  old_handler <- getOption("error")
  on.exit(options(error = old_handler))
  
  options(error = recover)
  # Your risky code here
}
# Automatically restored when block exits
```

**Integration with Development Workflow**:
```r
# In .Rprofile for development
if (interactive() && Sys.getenv("R_DEBUG") == "TRUE") {
  options(error = recover)
  cat("Debug mode enabled - errors will launch recover()\n")
}
```

## Non-Interactive Debugging (22.5)

Debugging becomes challenging when you can't run code interactively - typically in batch jobs, automated scripts, remote servers, or CI/CD pipelines. This section covers tools and techniques for these scenarios.

### Common Non-Interactive Scenarios:
- **Batch scripts**: R CMD BATCH, Rscript
- **Automated pipelines**: cron jobs, CI/CD systems
- **Remote execution**: Code running on servers without interactive access
- **Reproducible research**: Scripts that must run without user intervention
- **Different environments**: Code that works interactively but fails in batch

### Key Challenges:
- **No interactive prompt**: Can't use `browser()` or `recover()`
- **Different environments**: Variables, packages, working directories may differ
- **Limited output**: Restricted logging and debugging information
- **Delayed feedback**: Errors discovered long after code execution

### 1. `dump.frames()` - Post-Mortem Debugging

#### Purpose:
Save the complete debugging environment when an error occurs, then debug it later in an interactive session.

#### Basic Setup:
```r
# In your batch script - set up error handling
dump_and_quit <- function() {
  # Save all debugging information
  dump.frames(to.file = TRUE)  # Creates last.dump.rda
  # Exit with error status
  q(status = 1)
}

# Set as error handler
options(error = dump_and_quit)
```

#### Advanced Setup:
```r
# More sophisticated error handler
dump_with_info <- function() {
  # Create timestamped dump file
  timestamp <- format(Sys.time(), "%Y%m%d_%H%M%S")
  dump_file <- paste0("error_dump_", timestamp, ".rda")
  
  # Save debugging info
  dump.frames(to.file = TRUE, dumpto = dump_file)
  
  # Also save session info and environment
  save(list = ls(envir = .GlobalEnv), 
       file = paste0("session_", timestamp, ".rda"))
  
  # Log error details
  cat("Error occurred at:", as.character(Sys.time()), "\n",
      file = paste0("error_log_", timestamp, ".txt"))
  cat("Working directory:", getwd(), "\n", 
      file = paste0("error_log_", timestamp, ".txt"), append = TRUE)
  cat("R version:", R.version.string, "\n",
      file = paste0("error_log_", timestamp, ".txt"), append = TRUE)
  
  q(status = 1)
}

options(error = dump_with_info)
```

#### Interactive Debugging Session:
```r
# Later, in an interactive R session
load("last.dump.rda")        # or specific dump file
debugger()                   # Launch interactive debugger

# In debugger, you can:
# 1. Select frame to examine
# 2. Inspect variables at time of error
# 3. Understand the exact state when error occurred
```

#### Complete Workflow Example:
```r
# File: batch_script.R
options(error = function() {
  dump.frames(to.file = TRUE)
  q(status = 1)
})

# Your actual code here
process_data <- function(filename) {
  data <- read.csv(filename)
  result <- complex_analysis(data)
  return(result)
}

# This might error in batch mode
result <- process_data("data.csv")

# ---- Later, debugging interactively ----
# R> load("last.dump.rda")
# R> debugger()
# Selection: 1    # Choose frame to debug
# Browse[1]> ls()  # See what variables were available
# Browse[1]> str(data)  # Examine data state
```

### 2. Print Debugging - Strategic Logging

#### Purpose:
Insert diagnostic output throughout your code to trace execution and variable states.

#### Basic Principles:
- **Start coarse-grained**: Add prints at major function boundaries
- **Progressively refine**: Add more detailed prints near suspected problem areas
- **Include context**: Print variable names, values, and states
- **Use timestamps**: Help understand timing and sequence

#### Effective Print Debugging:
```r
analyze_data <- function(data) {
  cat("=== Starting analyze_data ===\n")
  cat("Input data dimensions:", dim(data), "\n")
  cat("Input data classes:", sapply(data, class), "\n")
  
  # Check for issues early
  if (any(is.na(data))) {
    cat("WARNING: NA values detected in", sum(is.na(data)), "positions\n")
  }
  
  # Step 1: Data cleaning
  cat("Step 1: Cleaning data...\n")
  clean_data <- clean_function(data)
  cat("After cleaning: dimensions =", dim(clean_data), "\n")
  
  # Step 2: Analysis
  cat("Step 2: Running analysis...\n")
  tryCatch({
    result <- complex_calculation(clean_data)
    cat("Analysis completed successfully\n")
  }, error = function(e) {
    cat("ERROR in analysis step:\n")
    cat("Data state at error:\n")
    cat("  - Dimensions:", dim(clean_data), "\n")
    cat("  - Column types:", sapply(clean_data, class), "\n")
    cat("  - Sample values:\n")
    print(head(clean_data, 3))
    stop(e)  # Re-throw the error
  })
  
  cat("=== Finished analyze_data ===\n")
  return(result)
}
```

#### Conditional Debugging Output:
```r
# Use environment variable or option to control debug output
DEBUG <- as.logical(Sys.getenv("R_DEBUG", "FALSE"))

debug_cat <- function(...) {
  if (DEBUG) {
    cat("[DEBUG]", ..., "\n")
  }
}

# Use throughout your code
my_function <- function(x) {
  debug_cat("Entering my_function with x =", deparse(substitute(x)))
  debug_cat("x value:", x)
  
  result <- expensive_operation(x)
  debug_cat("Operation result:", result)
  
  return(result)
}

# Enable debugging: Sys.setenv(R_DEBUG = "TRUE")
```

#### Structured Logging:
```r
# More sophisticated logging
log_message <- function(level, message, ...) {
  timestamp <- format(Sys.time(), "%Y-%m-%d %H:%M:%S")
  formatted_msg <- sprintf(message, ...)
  cat(sprintf("[%s] %s: %s\n", timestamp, level, formatted_msg))
}

log_debug <- function(message, ...) log_message("DEBUG", message, ...)
log_info <- function(message, ...) log_message("INFO", message, ...)
log_warn <- function(message, ...) log_message("WARN", message, ...)
log_error <- function(message, ...) log_message("ERROR", message, ...)

# Usage in functions
process_file <- function(filename) {
  log_info("Starting to process file: %s", filename)
  
  if (!file.exists(filename)) {
    log_error("File not found: %s", filename)
    stop("File not found")
  }
  
  log_debug("File size: %d bytes", file.size(filename))
  # ... rest of processing
}
```

### 3. `callr` Package - Fresh Session Debugging

#### Purpose:
Run code in a completely fresh R session to reproduce issues that only occur in clean environments.

#### Basic Usage:
```r
library(callr)

# Run function in fresh session
result <- r(function(x) {
  # This runs in a completely new R process
  return(x^2 + 1)
}, args = list(x = 5))

# Run with different arguments
result <- r(my_function, args = list(arg1 = "value1", arg2 = 10))
```

#### Debugging with `callr`:
```r
# Reproduce batch-only errors
debug_in_fresh_session <- function() {
  # This mimics the clean environment of batch execution
  result <- callr::r(function() {
    # Your problematic code here
    data <- read.csv("problematic_file.csv")
    analysis_result <- analyze(data)
    return(analysis_result)
  })
  
  return(result)
}

# If this succeeds, the problem is environment-related
# If this fails, you've reproduced the issue
```

#### Advanced `callr` Debugging:
```r
# Run with specific package loading
result <- r(
  function(packages_to_load) {
    # Load only specific packages
    lapply(packages_to_load, library, character.only = TRUE)
    
    # Your code here
    return(some_analysis())
  },
  args = list(packages_to_load = c("dplyr", "ggplot2"))
)

# Test different working directories
result <- r(
  function(wd) {
    setwd(wd)
    # Your code that depends on working directory
    return(list.files())
  },
  args = list(wd = "/path/to/test/directory")
)
```

### 4. Environment Diagnosis

#### Common Environment Issues:
When non-interactive code fails but interactive code succeeds, check:

**Working Directory**:
```r
# Add to your script
cat("Current working directory:", getwd(), "\n")
cat("Files in working directory:", paste(list.files(), collapse = ", "), "\n")
```

**Package Loading**:
```r
# Check loaded packages
cat("Loaded packages:\n")
print((.packages()))

# Check package versions
cat("Package versions:\n")
print(sessionInfo())
```

**Environment Variables**:
```r
# Check PATH and R_LIBS
cat("R_LIBS:", Sys.getenv("R_LIBS"), "\n")
cat("PATH:", Sys.getenv("PATH"), "\n")

# Check all environment variables
all_env <- Sys.getenv()
cat("Environment variables:\n")
str(all_env)
```

**Global Environment**:
```r
# Check what's in global environment
cat("Objects in global environment:\n")
print(ls(envir = .GlobalEnv))

# Check for conflicts
cat("Conflicted functions:\n")
conflicts()
```

### 5. Advanced Non-Interactive Techniques

#### Capturing All Output:
```r
# Capture both stdout and stderr
capture_all_output <- function(expr) {
  output_file <- "debug_output.txt"
  
  # Redirect all output
  sink(output_file, split = TRUE)  # Also show on console
  sink(output_file, type = "message", append = TRUE)
  
  tryCatch({
    result <- expr
    return(result)
  }, finally = {
    # Always restore output
    sink()
    sink(type = "message")
  })
}

# Usage
result <- capture_all_output({
  cat("Debug: starting analysis\n")
  data <- read.csv("file.csv")
  cat("Debug: data loaded, nrow =", nrow(data), "\n")
  analyze(data)
})
```

#### Checkpoint-Based Debugging:
```r
# Save intermediate results for inspection
checkpoint_dir <- "debug_checkpoints"
dir.create(checkpoint_dir, showWarnings = FALSE)

save_checkpoint <- function(obj, name) {
  file_path <- file.path(checkpoint_dir, paste0(name, ".rds"))
  saveRDS(obj, file_path)
  cat("Checkpoint saved:", file_path, "\n")
}

# In your analysis
data <- read.csv("input.csv")
save_checkpoint(data, "01_raw_data")

cleaned_data <- clean_data(data)
save_checkpoint(cleaned_data, "02_cleaned_data")

result <- analyze(cleaned_data)
save_checkpoint(result, "03_final_result")
```

### Best Practices for Non-Interactive Debugging:

1. **Always set up error handling** with `dump.frames()` or similar
2. **Use systematic logging** throughout your code
3. **Test in fresh sessions** to catch environment dependencies  
4. **Save intermediate results** for post-mortem analysis
5. **Document environment assumptions** in your code
6. **Use version control** to track changes that introduce bugs
7. **Create minimal reproducible examples** that fail in both interactive and batch modes

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