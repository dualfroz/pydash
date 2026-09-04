# Fix `sorted_last_index_of` raising IndexError on empty array

## Problem

`pydash.sorted_last_index_of(array, value)` raises `IndexError` when `array` is
empty, instead of returning `-1` (the documented "not found" result):

```python
>>> import pydash
>>> pydash.sorted_last_index_of([], 10)
Traceback (most recent call last):
    ...
IndexError: list index out of range
```

The sibling function `sorted_index_of` returns `-1` for the same input, so the
two are inconsistent.

## Root cause

`src/pydash/arrays.py`, in `sorted_last_index_of`:

```python
index = sorted_last_index(array, value) - 1

if index < len(array) and array[index] == value:
    return index
else:
    return -1
```

`sorted_last_index` returns `0` for an empty array (and whenever `value` is
smaller than every element), so `index` becomes `-1`. The guard only checks the
upper bound (`index < len(array)`); since `-1 < 0` is `True` for an empty array,
`array[index]` is evaluated as `array[-1]` on an empty list, raising
`IndexError`. For a non-empty array the same `index == -1` accidentally avoids a
wrong result only because `array[-1]` happens not to equal a value smaller than
`array[0]`; the missing lower-bound check is still a latent correctness issue.

## Fix

Add the lower-bound check so a negative computed index is treated as
"not found":

```python
if 0 <= index < len(array) and array[index] == value:
    return index
else:
    return -1
```

## Test

Extended `test_sorted_last_index_of` in `tests/test_arrays.py` with the
empty-array case and a value-below-range case:

```python
([], 10, -1),
([2, 3, 4], 1, -1),
```

## Gate

- `python -m pytest tests/test_arrays.py -k test_sorted_last_index_of` -> 4 passed
- `python -m pytest tests/ --ignore=tests/pytest_mypy_testing` -> 1917 passed

Counterfactual (fix reverted, test kept): the empty-array case fails with
`IndexError: list index out of range` at `arrays.py` in `sorted_last_index_of`.

## Flags for the account owner

The `tests/pytest_mypy_testing/` suite fails locally with `NameError` (the
`reveal_type` typing plugin is not resolvable in a bare `pip install -e .`
environment). These failures are pre-existing and unrelated to this change.
