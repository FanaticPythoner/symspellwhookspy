# SymSpell Ranker Hook Spec (`symspellwhookspy`)

This document defines **exactly** how the custom ranker hook in `symspellwhookspy` works:

- Types and parameters
- When and how it is invoked
- What guarantees you get
- How to use it correctly (including with embedding/similarity models)

This is based directly on the implementation in `symspellwhookspy/symspellpy.py` and its tests.

## 0. Canonical GitHub / raw documentation links

- **GitHub repository (symspellwhookspy)**  
  <https://github.com/FanaticPythoner/symspellwhookspy>

- **Project overview / README – raw Markdown**  
  <https://raw.githubusercontent.com/FanaticPythoner/symspellwhookspy/master/README.md>

- **`SuggestItem` implementation (source)**  
  <https://raw.githubusercontent.com/FanaticPythoner/symspellwhookspy/master/symspellwhookspy/suggest_item.py>

- **`Verbosity` enum implementation (source)**  
  <https://raw.githubusercontent.com/FanaticPythoner/symspellwhookspy/master/symspellwhookspy/verbosity.py>

- **`SymSpell` + ranker implementation (source)**  
  <https://raw.githubusercontent.com/FanaticPythoner/symspellwhookspy/master/symspellwhookspy/symspellpy.py>

---

## 1. Public surface

### 1.1. Package and imports

```python
from symspellwhookspy import (
    SymSpell,
    SymSpellRanker,
    SuggestItem,
    Verbosity,
)
```

All ranker-related APIs live on `SymSpell` and through the `SymSpellRanker` Protocol.

### 1.2. SymSpellRanker Protocol (conceptual)

```python
class SymSpellRanker(Protocol):
    def __call__(
        self,
        phrase: str,
        suggestions: list[SuggestItem],
        verbosity: Verbosity,
    ) -> list[SuggestItem]:
        ...
```

**Important:** you do **not** need to annotate your ranker. This Protocol exists so type checkers and IDEs infer parameter types automatically for:

```python
def my_ranker(phrase, suggestions, verbosity):
    ...
```

They infer:

- `phrase: str`
- `suggestions: list[SuggestItem]`
- `verbosity: Verbosity`

### 1.3. Attaching a ranker

You can attach a ranker either at construction time:

```python
def my_ranker(phrase, suggestions, verbosity):
    return suggestions

sym = SymSpell(ranker=my_ranker)
```

or later via the property:

```python
sym = SymSpell()

sym.ranker = my_ranker       # enable hook
sym.ranker = None            # disable hook, revert to default ordering
```

If `ranker` is `None`, `symspellwhookspy` behaves like original SymSpell: suggestions are sorted by edit distance ascending, then count descending, using `SuggestItem.__lt__`.

---

## 2. SuggestItem and Verbosity

### 2.1. SuggestItem

Each suggestion is a `SuggestItem` instance with fields:

- `term: str` – the suggested corrected word or phrase
- `distance: int` – edit distance from input phrase (or phrase-part)
- `count: int` – frequency / importance score from the dictionary or derived

Full type and behavior details live in the source here:  
<https://raw.githubusercontent.com/FanaticPythoner/symspellwhookspy/master/symspellwhookspy/suggest_item.py>

Special cases guaranteed by implementation and tests:

- **Exact match (TOP/CLOSEST/ALL)**: distance `0`, count is dictionary frequency.
- **include_unknown=True**: if no candidates, a synthetic suggestion is created:
  - `term = phrase`
  - `distance = max_edit_distance + 1`
  - `count = 0`
- **ignore_token pattern match**: if regex matches:
  - `term = phrase` (unchanged)
  - `distance = 0`
  - `count = 1`

### 2.2. Verbosity values

`Verbosity` is an enum with three values:

- `Verbosity.TOP`
  - Return the single "best" candidate.
  - Ranker usually sees a single-element `suggestions` list for each call.
- `Verbosity.CLOSEST`
  - Return all candidates at the smallest edit distance.
  - Ranker sees **multiple** suggestions, all with the same `distance`.
- `Verbosity.ALL`
  - Return **all** candidates within `max_edit_distance`.
  - Ranker sees a list of all `SuggestItem` instances.

Enum definition and any future extensions are in:  
<https://raw.githubusercontent.com/FanaticPythoner/symspellwhookspy/master/symspellwhookspy/verbosity.py>

For internal calls from `lookup_compound` and `word_segmentation`, `Verbosity.TOP` is used.

---

## 3. Ranker invocation semantics

### 3.1. Central ranking helper

The core helper is `_rank_suggestions` in `symspellpy.SymSpell`:

```python
def _rank_suggestions(self, phrase: str, suggestions: list[SuggestItem], verbosity: Verbosity) -> list[SuggestItem]:
    if not suggestions:
        return suggestions

    if self._ranker is not None:
        return self._ranker(phrase, suggestions, verbosity)

    if len(suggestions) > 1:
        suggestions.sort()  # distance asc, then count desc
    return suggestions
```

**Key points:**

- Ranker is **never** called with an empty `suggestions` list.
- If `ranker` is not set, the default SymSpell ordering is applied.

### 3.2. Where `_rank_suggestions` is used

1. `lookup(...)` – via a local `finalize()` helper
2. `lookup_compound(...)` – on the **final** compound suggestion
3. `word_segmentation(...)` – indirectly, via internal calls to `lookup()`

#### 3.2.1. `lookup(...)`

Signature:

```python
def lookup(
    self,
    phrase: str,
    verbosity: Verbosity,
    max_edit_distance: Optional[int] = None,
    include_unknown: bool = False,
    ignore_token: Optional[Pattern[str]] = None,
    transfer_casing: bool = False,
) -> list[SuggestItem]:
```

All return paths go through `finalize()`:

```python
def finalize() -> list[SuggestItem]:
    if include_unknown and not suggestions:
        suggestions.append(SuggestItem(phrase, max_edit_distance + 1, 0))

    ranked = self._rank_suggestions(phrase, suggestions, verbosity)

    if transfer_casing:
        ranked = [
            SuggestItem(
                helpers.case_transfer_similar(original_phrase, s.term),
                s.distance,
                s.count,
            )
            for s in ranked
        ]
    return ranked
```

**Situations that return via `finalize()` (and thus call the ranker if non-empty):**

- **Too-long phrase early exit**:

  ```python
  if phrase_len - max_edit_distance > self._max_length:
      return finalize()
  ```

- **Exact dictionary match** (`phrase in self._words`):
  - Suggestion appended: `SuggestItem(phrase, 0, count)`
  - For `Verbosity.TOP` and `Verbosity.CLOSEST`, this hits `finalize()` early.

- **`ignore_token` regex match**:

  ```python
  if ignore_token is not None and re.match(ignore_token, phrase) is not None:
      suggestions.append(SuggestItem(phrase, 0, 1))
      if verbosity != Verbosity.ALL:
          return finalize()
  ```

- **`max_edit_distance == 0`**:

  ```python
  if max_edit_distance == 0:
      return finalize()
  ```

- **Normal candidate search completion**:
  - After building all candidates and suggestions from deletes / edit distance search, `return finalize()` at the end of the function.

**What the ranker sees in `lookup`:**

- `phrase`
  - If `transfer_casing=False`: the original input string.
  - If `transfer_casing=True`: the **lowercased** string used for lookup (casing is restored **after** ranking).
- `suggestions`
  - Non-empty, list of `SuggestItem` representing current candidates.
  - For `Verbosity.TOP`: usually a single `SuggestItem`.
  - For `Verbosity.CLOSEST`: all minimal-distance candidates.
  - For `Verbosity.ALL`: all candidates within `max_edit_distance`.
- `verbosity`
  - Exactly the value from the `lookup` call.


#### 3.2.2. `lookup_compound(...)`

`lookup_compound` calls `lookup` many times internally, **all with `Verbosity.TOP`**. Each of those internal calls will trigger the ranker in the same way as above.

At the end, a **single** `SuggestItem` is built for the full corrected phrase:

```python
joined_term = ""  # built from suggestion_parts
joined_count: float = self.N
...
suggestion = SuggestItem(
    joined_term,
    self.distance_comparer.compare(phrase, joined_term, 2**31 - 1),
    int(joined_count),
)

suggestions = [suggestion]
suggestions = self._rank_suggestions(phrase, suggestions, Verbosity.TOP)
return suggestions
```

Final ranker call for `lookup_compound`:

- `phrase`: the **original multi-word input** (string passed to `lookup_compound`).
- `suggestions`: a list with exactly **one** `SuggestItem` (`joined_term`).
- `verbosity`: always `Verbosity.TOP`.

Internal calls to `lookup` (per-token, combined tokens, and splits) also hit the ranker with:

- `phrase`: the token or token-combination string.
- `verbosity`: `Verbosity.TOP`.

Tests confirm this behavior by checking that:

- The ranker sees **at least one call** with `" " in phrase` for the final compound.
- `verbosity` is `Verbosity.TOP` for the final phrase.


#### 3.2.3. `word_segmentation(...)`

`word_segmentation` uses `lookup` internally for each candidate segment:

```python
results = self.lookup(
    part.lower(),
    Verbosity.TOP,
    max_edit_distance,
    ignore_token=ignore_token,
)
```

So the ranker is invoked **indirectly** via these `lookup` calls.

What the ranker sees here:

- `phrase`: `part.lower()` – lowercased segment of the original string (spaces stripped internally before lookup).
- `suggestions`: non-empty list (TOP semantics: usually length 1 when a candidate exists, otherwise the unknown-case synthetic suggestion).
- `verbosity`: always `Verbosity.TOP`.

Tests verify that the ranker is called at least once during segmentation (`test_ranker_called_in_word_segmentation`).

---

## 4. Ranker contract (what you can and cannot do)

### 4.1. What a ranker is allowed to do

Given:

```python
def ranker(phrase, suggestions, verbosity):
    ...
```

You are allowed to:

- **Reorder** suggestions:

  ```python
  suggestions_sorted = sorted(suggestions, key=lambda s: s.term)
  return suggestions_sorted
  ```

- **Filter** suggestions:

  ```python
  filtered = [s for s in suggestions if s.term.isalpha()]
  return filtered
  ```

- **Transform** suggestions by constructing new `SuggestItem` objects, if needed:

  ```python
  from symspellwhookspy import SuggestItem

  def ranker(phrase, suggestions, verbosity):
      return [
          SuggestItem(s.term.lower(), s.distance, s.count)
          for s in suggestions
      ]
  ```

(However, if you change `term`, be aware that other code will see the new text.)

### 4.2. Recommended constraints

For deterministic, sensible behavior, a ranker **should**:

- Return only `SuggestItem` objects that correspond to the **same logical phrase** it was given.
- Not introduce suggestions with arbitrary `distance` or `count` values unless you know exactly what you are doing.
- Be **pure and deterministic**: same `(phrase, suggestions, verbosity)` → same output, no hidden state.

### 4.3. Must-haves

Your ranker **must**:

- Return a `list[SuggestItem]`.
- Not return an empty list if you intend to preserve at least one candidate (empty is allowed but means "drop everything").

The library does **not** re-check the returned list; it trusts your ranker.

---

## 5. Practical examples

### 5.1. Simple lexicographic ranker

```python
from symspellwhookspy import SymSpell, Verbosity

sym = SymSpell()

sym.create_dictionary_entry("xbc", 3)
sym.create_dictionary_entry("axc", 2)
sym.create_dictionary_entry("abx", 1)


def ranker(phrase, suggestions, verbosity):
    # Sort candidates by term alphabetically
    return sorted(suggestions, key=lambda s: s.term)

sym.ranker = ranker

results = sym.lookup("abc", Verbosity.ALL, 1)
terms = [s.term for s in results]
# terms are now in lexicographic order
```

### 5.2. Filtering non-alphabetic suggestions

```python
from symspellwhookspy import SymSpell, Verbosity

sym = SymSpell()

sym.create_dictionary_entry("hello", 10)
sym.create_dictionary_entry("hello1", 5)
sym.create_dictionary_entry("hello2", 1)


def ranker(phrase, suggestions, verbosity):
    # Keep only alphabetic terms
    return [s for s in suggestions if s.term.isalpha()]

sym.ranker = ranker

results = sym.lookup("hello", Verbosity.ALL, 1)
# Only "hello" remains
```

### 5.3. Using a similarity model (e.g., VisSimL1)

Pseudocode sketch:

```python
from symspellwhookspy import SymSpell, Verbosity

similarity_model = ...  # must implement compare(a: str, b: str) -> float in [0, 1]


def similarity_ranker(phrase, suggestions, verbosity):
    # Higher similarity score comes first
    return sorted(
        suggestions,
        key=lambda s: similarity_model.compare(phrase, s.term),
        reverse=True,
    )

sym = SymSpell()
sym.load_dictionary("frequency_dictionary_en_82_765.txt", 0, 1)

sym.ranker = similarity_ranker

results = sym.lookup("helo", Verbosity.CLOSEST, 2)
```

The similarity model becomes the **tie-breaker** or even the primary score, while SymSpell still enforces edit-distance gating.

---

## 6. When exactly the ranker runs (summary)

For a single `SymSpell` instance:

- **lookup()**
  - Ranker runs once per `lookup` call, if any suggestions exist **after**:
    - exact match handling
    - ignore_token handling
    - include_unknown logic
    - full edit-distance search

- **lookup_compound()**
  - Ranker runs:
    - Multiple times for internal token lookups (all with `Verbosity.TOP`).
    - Once at the end on the **final joined phrase** (`Verbosity.TOP`, single-element list).

- **word_segmentation()**
  - Ranker runs indirectly via `lookup(part.lower(), Verbosity.TOP, ...)` calls inside segmentation.

In all cases, if your ranker is `None`, the library falls back to default SymSpell ordering.

---

## 7. Facts to remember

- Use this import for ranker-related types:

  ```python
  from symspellwhookspy import SymSpell, SymSpellRanker, SuggestItem, Verbosity
  ```

- Ranker callable shape (conceptual):

  ```python
  (phrase: str, suggestions: list[SuggestItem], verbosity: Verbosity) -> list[SuggestItem]
  ```

- Ranker is attached via `SymSpell(ranker=...)` or `sym.ranker = ...`.
- Ranker is **never** called with an empty `suggestions` list.
- Ranker may reorder or filter; it must return `list[SuggestItem]`.
- `Verbosity.TOP` typically gives 1 suggestion; `CLOSEST` gives all min-distance ones; `ALL` gives all candidates within `max_edit_distance`.
- `lookup_compound` and `word_segmentation` rely on `lookup` internally, so the ranker sees both final phrases and internal token-level lookups.
- When `ranker` is `None`, behavior is equivalent to traditional SymSpell sorting by distance then count.
