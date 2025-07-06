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

## Extra Readings

- https://medium.com/accredian/business-analytics-modeling-automated-radiant-r-1942230eae63
- https://renkun.me/2022/03/06/my-recommendations-of-vs-code-extensions-for-r/
