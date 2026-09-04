# Fix `truncate` dropping a character when a string separator is absent

## Problem

`pydash.truncate(text, length, omission, separator)` is documented to truncate
`text` at the last occurrence of `separator`. When `separator` is a **string
that does not occur** in the truncated text, `truncate` incorrectly removes an
extra trailing character instead of falling back to a plain cut.

```python
>>> import pydash
>>> pydash.truncate("hello world", 10, "...", "xyz")
'hello ...'      # expected 'hello w...'
>>> pydash.truncate("hello world", 8, "...", "q")
'hell...'        # expected 'hello...'
```

This is inconsistent with the equivalent lodash behaviour (the API pydash
mirrors), where a separator that is not found leaves the plain truncation
untouched:

```js
_.truncate('hello world', {length: 10, separator: 'xyz'}) // 'hello w...'
_.truncate('hello world', {length: 8,  separator: 'q'})   // 'hello...'
```

It is also internally inconsistent: the regular-expression `separator` branch
already handles "no match" correctly (it leaves `trunc_len` at the full
truncated length), while the string branch does not.

## Root cause

`src/pydash/strings.py`, in `truncate`:

```python
if pyd.is_string(separator):
    trunc_len = text.rfind(separator)
```

`str.rfind` returns `-1` when the separator is not present. That `-1` is then
used as a slice end (`text[:trunc_len]` == `text[:-1]`), which silently drops
the last character of the already-truncated text before appending `omission`.

## Fix

Only move the truncation point when the separator is actually found:

```python
if pyd.is_string(separator):
    separator_index = text.rfind(separator)
    if separator_index >= 0:
        trunc_len = separator_index
```

This mirrors the existing regex branch, which only updates `trunc_len` when a
match exists.

## Test

Added two cases to the existing `test_truncate` parametrization in
`tests/test_strings.py` covering a string separator that does not occur in the
truncated text:

```python
(("hello world", 10, "...", "xyz"), "hello w..."),
(("hello world", 8, "...", "q"), "hello..."),
```

## Gate

- `python -m pytest tests/test_strings.py -k test_truncate` -> 13 passed
- `python -m pytest tests/ --ignore=tests/pytest_mypy_testing` -> 1917 passed

Counterfactual (fix reverted, test kept): the new cases fail with
`AssertionError: assert 'hello ...' == 'hello w...'` and
`AssertionError: assert 'hell...' == 'hello...'`.

## Flags for the account owner

The `tests/pytest_mypy_testing/` suite fails locally with `NameError` (the
`reveal_type` typing plugin is not resolvable in a bare `pip install -e .`
environment). These failures are pre-existing and unrelated to this change,
which touches only `truncate`.
