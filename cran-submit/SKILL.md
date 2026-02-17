---
name: cran-submit
description: >
  Prepare and submit an R package to CRAN. Use this skill when asked to submit
  a package to CRAN, prepare for CRAN submission, run CRAN checks, or review a
  package for CRAN compliance. Assumes the working directory is an R package root.
license: MIT
metadata:
  author: Cory McCartan
  version: "0.1"
allowed-tools: shell(curl:*) shell(git:log) shell(git:tag) shell(git:diff) shell(Rscript) shell(grep:*) shell(find:*) read write ask_user
---

# CRAN Package Submission

CRAN submission demands extreme thoroughness. Issues missed now cause expensive
resubmission cycles—CRAN reviewers are overworked volunteers and every round-trip
wastes days. Be adversarial: pretend you are a strict CRAN reviewer looking for
ANY reason to reject the package.

Start by checking if this is a new submission by seeing if the following URL
with the package name substituted exists. If not (404), then it's a new
submission and the checks will be even more strict.
```sh
curl -s -o /dev/null -w "%{http_code}\n" https://cran.r-project.org/web/packages/<PACKAGE>/index.html
```

Execute these phases in order. Phases 3–4 repeat until the package is clean.

General notes:

- Run `devtools::check()` with `R_MAKEVARS_USER=NULL` to avoid local settings
  causing false positives
- Don't duplicatively check things which are checked by `devtools::check()`. 
  These are listed in the output of the check function.
- You can safely ignore the following NOTEs and not report them in `cran-comments.md`:
  - unable to verify current time
  - Skipping checking HTML validation: 'tidy' doesn't look like recent enough HTML Tidy.
- At the end, only summarize changes and action items, NOT tasks completed.

## Phase 1: Automated Checks and Fixes

Run these checks, fixing all issues before proceeding to the next step.

### 1.1 R CMD check

```r
devtools::document()
devtools::check(remote = TRUE, manual = TRUE)
```

Fix ALL errors and warnings (automatic rejection). Fix as many NOTEs as
possible—each requires human review. Common fixes:
- Missing `@returns` or `@examples` on exported functions
- NAMESPACE issues (re-run `devtools::document()`)
- Encoding issues in DESCRIPTION

### 1.2 URL checks

```r
urlchecker::url_check()
```

Fix broken URLs. For 301 redirects, use the target URL. If a legitimate URL
causes a false positive, convert to non-hyperlinked verbatim text.

### 1.3 Rebuild README

```r
if (file.exists("README.Rmd")) devtools::build_readme()
```

### 1.4 Spell check

Often, spell check flags are false positives. But if there are common 
mispellings, fix them yourself. If unsure, flag the next time you report
back to the user.

```r
devtools::spell_check()
```

### 1.5 Win-builder

Submit to win-builder (required by CRAN policy—checks against R-devel on Windows):
ASK the user for confirmation first: they may decline, if the package was checked recently on win-builder.
Continue to the next step regardless of their response.

```r
devtools::check_win_devel()
```

Tell the user results arrive by email in ~30 minutes. Ask them to share any
issues from the report.

### 1.5 Reverse dependency checks

Only if the package is already on CRAN with reverse dependencies:

```r
if (requireNamespace("revdepcheck", quietly = TRUE)) {
  revdepcheck::revdep_check(num_workers = 4)
}
```

Review `revdep/problems.md`. Contact affected maintainers for legitimate
breakages.

## Phase 2: Prepare Submission Files

### 2.1 Version number

Read the version from DESCRIPTION and ask the user whether to update the version
or keep it as is. Edit DESCRIPTION directly.

### 2.2 NEWS.md

Ensure it exists (`usethis::use_news_md()` if not). If it doesn't already
exist, add a section for the new version. Polish it:
- Every user-facing change should be given a bullet in NEWS.md. Do not add
  bullets for small documentation changes or internal refactorings.
- Each bullet should briefly describe the change to the end user and mention
  the related issue in parentheses.
- A bullet can consist of multiple sentences but should not contain any new
  lines (i.e. DO NOT line wrap).
- If the change is related to a function, put the name of the function early in
  the bullet.
- Order bullets alphabetically by function name. Put all bullets that don't
  mention function names at the beginning.


### 2.3 cran-comments.md

Create if missing (`usethis::use_cran_comments()`). 
Edit and update. Contents:
- R CMD check results: `0 errors | 0 warnings | N notes`
- For first submissions: "This is a new release."
- For resubmissions: a "Resubmission" section listing changes made
- Reverse dependency results (paste from `revdep/cran.md`)
- Brief explanation of any unavoidable NOTEs
- Keep it concise—don't over-explain

### 2.4 DESCRIPTION review

Check all of these:
- `Title`: Title Case, no period at end, no "A Package for..." or "R Package
  for...", no package name in title. Verify with `tools::toTitleCase()`.
- `Description`: 2+ sentences, substantive. Software/package names in single
  quotes. Acronyms explained. Does not start with "This package...". References
  in format `Author (year) <doi:10.prefix/suffix>`. URLs in angle brackets.
- `Authors@R`: includes copyright holder (role 'cph'). Maintainer email is
  correct and actively monitored. ORCIDs included where available.
- `Depends`: R version requirement not too recent without good reason

## Phase 3: Adversarial Review (Read-Only)

This is the most critical phase. Read through the package with a CRAN
reviewer's mindset. Do NOT make changes yet—just catalog every issue.
For the detailed checklist of what to look for, see
[cran-review-checklist.md](cran-review-checklist.md).

Perform every check in that checklist systematically. Use grep/search tools
to scan ALL R files, tests, examples, vignettes, and man pages. Read every
exported function's documentation. Read .onLoad/.onAttach hooks. Read
DESCRIPTION and NAMESPACE. Be exhaustive.

Report all findings to the user before proceeding to fixes.

## Phase 4: Fix Issues

Fix every issue found in Phase 3. After fixing:
1. Re-run `devtools::document()` if roxygen comments changed
2. Re-run `devtools::check(remote = TRUE, manual = TRUE)`
3. Update `cran-comments.md` with fresh check results

## Phase 5: Iterate

Return to Phase 3. Repeat Phases 3–4 until the adversarial review finds
ZERO issues. 

Confirm 0 errors, 0 warnings, and only explained NOTEs.

## Phase 6: Submit

Ask the user to commit all changes locally with `git`.

Then tell the user to submit to CRAN with `devtools::submit_cran()`.
Remind them to run the following after acceptance:

```r
usethis::use_github_release()
usethis::use_dev_version()
```
