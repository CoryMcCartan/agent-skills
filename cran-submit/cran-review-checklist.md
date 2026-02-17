# CRAN Adversarial Review Checklist

Use this checklist during Phase 3 of the CRAN submission process. Go through
every item systematically. Use grep/search tools to scan all R files, tests,
examples, vignettes, and man pages.

## DESCRIPTION File

- [ ] `Title` is in Title Case (verify with `tools::toTitleCase()`)
- [ ] `Title` does NOT start with "A Package for...", "R Package for...",
      "Tools for...", or similar redundant phrasing
- [ ] `Title` does NOT contain the package name
- [ ] `Title` does NOT end in a period
- [ ] `Description` is a substantive paragraph, at least 2 sentences
- [ ] `Description` does NOT start with "This package...", "Provides...",
      or repeat the package name
- [ ] Software, package, and API names in `Description` are in single quotes
      (e.g., 'Python', 'ggplot2', 'OpenSSL')
- [ ] All acronyms in `Description` are explained
- [ ] References use proper format: `Author (year) <doi:10.prefix/suffix>`
- [ ] DOIs have no space after `doi:` and use angle brackets
- [ ] arXiv references use `<doi:10.48550/arXiv.ID>` (unversioned)
- [ ] URLs in `Description` use angle brackets: `<https://example.com/>`
- [ ] `Authors@R` is used (not separate Author/Maintainer fields)
- [ ] `Authors@R` includes at least one person with 'cre' (maintainer) role
- [ ] `Authors@R` includes a copyright holder ('cph' role)
- [ ] Maintainer email is a real, monitored address (not noreply)
- [ ] ORCID identifiers are included where available
- [ ] `License` is FOSS-compatible and listed in R's license database
- [ ] `License` does NOT include unnecessary `+ file LICENSE` for standard
      licenses (only needed for MIT, BSD, etc. that require attribution)
- [ ] If `+ file LICENSE` is used, the LICENSE file contains ONLY
      year + copyright holder (MIT) or year + holder + org (BSD)
- [ ] Version number follows `x.y.z` format, no leading zeros
- [ ] `Depends: R (>= x.y.z)` version is not unreasonably recent
- [ ] `LazyData: true` is set if the package includes data in `data/`
- [ ] No `LazyData` field if the package has no data
- [ ] `URL` and `BugReports` fields point to valid, accessible URLs

## R Code — Policy Violations

Search ALL files under `R/` for each of these. Use grep to be thorough.

### T/F instead of TRUE/FALSE

Search: `grep -rn '\bT\b' R/` and `grep -rn '\bF\b' R/`

Examine each match. `T` and `F` are NOT reserved words and can be redefined
by users. All logical values must use `TRUE`/`FALSE`. Also check that `T`
and `F` are not used as variable names.

### Writing to user's filespace

Search: `grep -rn 'getwd\|setwd\|write\.csv\|write\.table\|writeLines\|save\b\|saveRDS\|ggsave\|pdf(\|png(\|jpeg(\|tiff(\|bmp(' R/`

No function should write to `getwd()`, `~`, or the package directory by
default. Writing functions should either:
- Have no default path (require the user to specify one), or
- Default to `tempdir()` / `tempfile()`

### Writing to .GlobalEnv

Search: `grep -rn '<<-\|\.GlobalEnv\|globalenv()' R/`

The `<<-` operator writes to parent environments. It's only safe when the
variable is defined in an enclosing function scope. Never assign to
`.GlobalEnv` or `globalenv()`. Also check that `.Random.seed` is not modified.

Exception: Shiny packages may sometimes need to write to `.GlobalEnv`.

### Modifying options/par/wd without reset

Search: `grep -rn 'options(\|par(\|setwd(' R/`

In functions, every `options()`, `par()`, or `setwd()` call MUST be
immediately followed by `on.exit()` to restore the original values:

```r
old <- options(warn = 0)
on.exit(options(old))
```

Multiple `on.exit()` calls need `add = TRUE` on all but the first:

```r
on.exit(par(oldpar))
on.exit(options(oldop), add = TRUE)
```

Also check the same pattern in examples and vignettes, where `on.exit()`
doesn't apply — old values must be saved and restored manually:

```r
oldpar <- par(mfrow = c(1, 2))
# ... plotting code ...
par(oldpar)
```

Also check for `Sys.setenv()`, `Sys.setlocale()`, `Sys.setLanguage()` — these
persistently change system state and must also be restored.

### set.seed() in package functions

Search: `grep -rn 'set\.seed' R/`

`set.seed()` must NOT be called with a fixed seed in package functions. If
randomness control is needed, provide a `seed = NULL` argument:

```r
my_function <- function(..., seed = NULL) {
  if (!is.null(seed)) set.seed(seed)
  # ...
}
```

`set.seed()` IS fine in examples, tests, and vignettes for reproducibility.

### Unsuppressable console output

Search: `grep -rn 'cat(\|print(\|writeLines(' R/`

In package functions (NOT print/summary/show methods), `cat()` and `print()`
produce output that users cannot suppress. Replace with:
- `message()` for informational messages (user can use `suppressMessages()`)
- `warning()` for warnings
- `stop()` for errors
- `if (verbose) cat(...)` with a `verbose` argument

Also check for `writeLines()` and logging functions that write to console.

### installed.packages()

Search: `grep -rn 'installed\.packages' R/`

Never use this — it's very slow. Use `requireNamespace("pkg")` or
`system.file(package = "pkg")` instead.

### options(warn = -1)

Search: `grep -rn 'warn.*=.*-1\|warn.*-1' R/`

Setting negative warn values is forbidden. Use `suppressWarnings()` instead.

### More than 2 CPU cores

Search: `grep -rn 'detectCores\|makeCluster\|mclapply\|parLapply\|future::plan\|foreach\|cl.*<-.*makeCluster' R/ tests/ vignettes/ man/`

CRAN check machines run many processes in parallel; each package gets at most
2 cores. Ensure examples, tests, and vignettes never use more than 2 cores.
Functions should accept a `cores`/`workers`/`ncpus` argument with a safe
default (1 or 2).

### Installing packages in code

Search: `grep -rn 'install\.packages\|devtools::install\|remotes::install\|pak::pak\|BiocManager::install' R/ tests/ vignettes/ man/`

Functions should not install packages. Installation functions are only
acceptable if installing software IS the package's purpose, and they should
never be called in examples, tests, or vignettes.

### library()/require() in R code

Search: `grep -rn 'library(\|require(' R/`

Package code should never use `library()` or `require()`. Use `::` notation
or `@importFrom` in roxygen instead. `requireNamespace()` is acceptable for
conditional use of Suggested packages.

### Leftover debugging code

Search: `grep -rn 'browser()\|debug(\|debugonce(\|undebug(' R/ tests/`

Remove all debugging calls before submission.

## Examples (man/ and roxygen @examples)

Check EVERY exported function. Read the man pages or roxygen comments.

- [ ] Every exported function has `@examples` or `\examples{}`
- [ ] Every exported function has `@returns` or `\value{}`
      (even for side-effect-only functions: `\value{No return value, called for side effects}`)
- [ ] Examples are actually runnable — not ALL wrapped in `\dontrun{}`
- [ ] `\dontrun{}` is used ONLY for truly unexecutable examples (missing API
      keys, external software, etc.). Justification comment required.
- [ ] `\donttest{}` is used for examples that work but take > 5 seconds
- [ ] `if (interactive()) {}` is used for Shiny/interactive-only examples
- [ ] `if (requireNamespace("pkg", quietly = TRUE)) {}` guards examples
      that use packages listed in Suggests
- [ ] Examples that create files use `tempdir()`/`tempfile()` and clean up
      (cleanup can go in `\dontshow{}` for aesthetics)
- [ ] Examples that change `par()`/`options()`/`setwd()` save and restore
- [ ] Examples use small, fast toy datasets — not large computations
- [ ] No examples are entirely commented out
- [ ] Function names in documentation use `()`: `foo()` not `foo`

## Tests

- [ ] Tests don't write to the user's home directory or package directory.
      Use `tempdir()`, `tempfile()`, `withr::local_tempfile()`, or
      `withr::local_tempdir()` and clean up with `unlink()`
- [ ] Tests clean up all temporary files (check for "detritus in temp dir" NOTE)
- [ ] Tests don't use more than 2 cores
- [ ] Tests don't install packages
- [ ] Tests that require internet access use `skip_on_cran()`
- [ ] Flaky tests (random failures, timing-dependent) use `skip_on_cran()`
- [ ] Tests don't modify global state (options, env vars, wd, par, .GlobalEnv)
      without restoration — use `withr::local_options()`, `withr::local_envvar()`, etc.
- [ ] No `rm(list = ls())` in test files

## Vignettes

- [ ] All vignettes build without errors
- [ ] Total check time (examples + tests + vignettes) < 10 minutes
- [ ] Vignettes don't write to user's filespace
- [ ] Vignettes restore any changed options/par/wd
- [ ] Expensive computations use pre-computed results or `\donttest{}`
- [ ] Packages in Suggests are conditionally loaded with
      `requireNamespace()` checks

## Package Size

- [ ] Package tarball is under 5 MB (strongly preferred)
- [ ] Package tarball is under 10 MB (hard limit; requires CRAN approval)
- [ ] No unnecessarily large files in `inst/`, `data/`, or `extdata/`
- [ ] Large datasets are externalized or in a separate data package
- [ ] If size exceeds 5 MB, explain why in `cran-comments.md`

## Compiled Code (if the package has src/)

- [ ] No compiler warnings with `-Wall -pedantic`
- [ ] Properly registers native routines (`useDynLib` + `R_registerRoutines`)
- [ ] Uses R's RNG (`GetRNGstate()`/`PutRNGstate()`), not `srand()`/`rand()`
- [ ] No memory leaks (test with valgrind/ASAN if possible)
- [ ] Clean address-sanitizer results (no buffer overflows, use-after-free)
- [ ] `Makevars` doesn't set `-O0` or other non-portable flags
- [ ] C++ standard is properly declared if needed (e.g., `CXX_STD = CXX17`)

## Namespace and Dependencies

- [ ] All imported functions are properly declared in NAMESPACE
- [ ] No `library()` or `require()` calls in R/ code
- [ ] Packages in Suggests are used conditionally with `requireNamespace()`
- [ ] No circular dependencies
- [ ] Dependencies are minimal — don't add heavy packages for minor features
- [ ] System requirements are documented in `SystemRequirements` field

## Package Hooks and Load/Attach Behavior

- [ ] `.onAttach()` uses `packageStartupMessage()`, NOT `message()` or `cat()`
- [ ] `.onLoad()` / `.onAttach()` don't modify `.GlobalEnv`
- [ ] `.onLoad()` / `.onAttach()` don't change the search path
      (no `library()` or `attach()`)
- [ ] `.onUnload()` / `.onDetach()` clean up properly if `.onLoad()` sets state
- [ ] No messages printed at load unless essential (citation info is ok)

## Miscellaneous

- [ ] No `rm(list = ls())` in examples, vignettes, or demos
- [ ] No malicious or anti-social code
- [ ] No tracking/analytics/telemetry without explicit user consent
- [ ] All included third-party code has its license properly declared
- [ ] `inst/CITATION` file (if present) uses proper `bibentry()` format
- [ ] No files left over from development (`.DS_Store`, `Thumbs.db`,
      `.Rhistory`, `.RData`, editor backup files)
- [ ] `.Rbuildignore` properly excludes development files
- [ ] Encoding is declared in DESCRIPTION if non-ASCII content is used
