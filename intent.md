# Intent

## Goal
Let a support coordinator preview the tickets in a CSV file before any import is
approved. A future implementation will add one function that takes CSV text and
returns either the parsed tickets, in input order, or a single error code. It never
changes or stores any data. Success looks like
`{"tickets":[{"title":"...","priority":"normal"}]}` and failure looks like
`{"error":"error-code"}`. There is no partial success: if any row is invalid, the
whole preview fails with one error.

## Scope
In scope: a pure preview function in `src/import-tickets.mjs` and its tests in
`tests/import-tickets.test.mjs`. Nothing else should change. The function handles:
- A header row that must contain `title` and may contain `priority`, in either order.
- Trimming surrounding whitespace from every value.
- Priorities `low`, `normal` and `urgent`, with a missing or blank priority becoming
  `normal`.
- Standard CSV quoting, so a quoted title may contain a comma.
- At most 100 data rows.

Out of scope: any UI, database, network calls, file reads or writes, new package
dependencies, and actually importing or saving tickets. If the work seems to need
any of these, or a file outside the two allowed paths, stop and ask first.

## Constraints
- Do not modify `src/tickets.mjs`, its existing tests, anything under `.github/`, or
  `CLAUDE.md`. `src/tickets.mjs` is the stable ticket representation; reuse it rather
  than redefining what a ticket is.
- No side effects: the function only reads its input string and returns a value.
- Use only Node.js 22 built-ins; add no dependencies to `package.json`.
- Errors are returned as `{"error": code}`, never thrown to the caller. The only codes
  are `unknown-column`, `empty-title`, `invalid-priority` and `row-limit`.
- `unknown-column` covers: an empty CSV with no header, a missing `title` column, any
  column other than `title` or `priority`, duplicate column names, and malformed rows
  (wrong number of fields or broken quoting).
- Count data rows before validating any values. More than 100 rows returns `row-limit`
  even if the rows also contain bad values.

## Plan
Ask before starting. Confirm with the student that this intent is approved before
writing any code, and ask again before expanding scope or touching any other file.
Once approved:
1. Write `tests/import-tickets.test.mjs` first, with one test per acceptance case
   below plus the edge cases listed in Constraints.
2. Add a small CSV parser in `src/import-tickets.mjs` that handles quoted fields,
   escaped quotes (`""`) and a trailing newline.
3. Validate the header, then check the row limit, then validate each row and build
   tickets in input order. Return the first error found and no tickets.
4. Run the verification commands, then show the diff and wait for review. Do not
   commit, push or open a PR unless asked.

## Acceptance
The work is done when every case below returns exactly its expected result. Together
the cases cover a valid explicit priority, a default priority, a quoted title with a
comma, an empty title, an unknown column, an invalid priority and too many rows. The
`repeat: 101` case means the one data row is repeated 101 times. The contract also
records the allowed and protected paths, the non-goals and the approval rule.

```json
{
  "schema": 1,
  "change": "ticket-csv-preview",
  "allowed_changes": ["src/import-tickets.mjs", "tests/import-tickets.test.mjs"],
  "protected_paths": ["src/tickets.mjs", ".github/", "CLAUDE.md"],
  "non_goals": ["ui", "database", "network"],
  "requirements": {
    "columns": ["title", "priority"],
    "default_priority": "normal",
    "valid_priorities": ["low", "normal", "urgent"],
    "reject_unknown_columns": true,
    "reject_empty_title": true,
    "max_rows": 100,
    "side_effects": "preview-only"
  },
  "verification": ["npm test", "git diff --check"],
  "approval": "ask-before-implementation",
  "acceptance_cases": [
    {
      "csv": "title,priority\nPrinter issue,urgent\n",
      "expected": {"tickets":[{"title":"Printer issue","priority":"urgent"}]}
    },
    {
      "csv": "title,priority\nReset password,\n",
      "expected": {"tickets":[{"title":"Reset password","priority":"normal"}]}
    },
    {
      "csv": "title,priority\n\"Printer jam, second floor\",low\n",
      "expected": {"tickets":[{"title":"Printer jam, second floor","priority":"low"}]}
    },
    {
      "csv": "title,priority\n   ,urgent\n",
      "expected": {"error":"empty-title"}
    },
    {
      "csv": "title,owner\nPrinter issue,alex\n",
      "expected": {"error":"unknown-column"}
    },
    {
      "csv": "title,priority\nPrinter issue,high\n",
      "expected": {"error":"invalid-priority"}
    },
    {
      "csv": "title,priority\nPrinter issue,urgent\n",
      "repeat": 101,
      "expected": {"error":"row-limit"}
    }
  ]
}
```

## Verification
Run `npm test`. Every existing test and every new import-tickets test must pass.
Run `git diff --check` to confirm there are no whitespace errors. Then run
`git diff --stat` and confirm that only `src/import-tickets.mjs` and
`tests/import-tickets.test.mjs` changed. Report the results, including any failure
output, instead of claiming success.
