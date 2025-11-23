# symspellwhookspy vs symspellpy

This document explains **exactly** how `symspellwhookspy` differs from the original `symspellpy`, and how to use it.

---

## 1. Package and import names

- **Original library (upstream)**
  - PyPI name: `symspellpy`
  - Import: `from symspellpy import SymSpell, Verbosity, SuggestItem, ...`

- **This fork (with hook)**
  - PyPI name: **`symspellwhookspy`**
  - Import: **`from symspellwhookspy import SymSpell, SymSpellRanker, SuggestItem, Verbosity`**

The public APIs (`SymSpell`, `lookup`, `lookup_compound`, `word_segmentation`, etc.) are kept **source‑compatible** with `symspellpy`. The key addition is the **ranker hook**.

---

## 2. What symspellwhookspy adds

Everything from `symspellpy` **plus**:

- A **ranker hook** on `SymSpell`:
  - Attribute: `sym_spell.ranker`
  - Type: `SymSpellRanker` (a `Protocol` with a typed `__call__`)
- Strong typing & IDE support:
  - `SymSpellRanker` Protocol exposes the parameter types of the ranker function.
  - When you write `def ranker(phrase, suggestions, verbosity): ...` **without type hints**, the IDE can still infer:
    - `phrase: str`
    - `suggestions: list[SuggestItem]`
    - `verbosity: Verbosity`
- Hook coverage:
  - The ranker is invoked for **all non‑empty suggestion sets** produced by:
    - `SymSpell.lookup()`
    - `SymSpell.lookup_compound()`
    - `SymSpell.word_segmentation()` (via internal lookups)
  - It is also called on:
    - Exact matches (TOP)
    - `include_unknown=True` cases
    - `ignore_token` matches
    - `max_edit_distance = 0` paths
- Behavior when `ranker` is **not set** (`None`):
  - Identical to `symspellpy`:
    - Suggestions sorted by `distance` ascending, then `count` descending (via `SuggestItem.__lt__`).

In short: **same SymSpell engine, plus one hook to deterministically rerank/filter suggestions.**

---

## 3. Ranker hook interface

### 3.1. Exported symbols

From `symspellwhookspy` you can import:

```python
from symspellwhookspy import (
    SymSpell,
    SymSpellRanker,
    SuggestItem,
    Verbosity,
)
```

- `SymSpell`: same class as upstream, with an extra `ranker` attribute.
- `SymSpellRanker`: `Protocol` describing the ranker callable.
- `SuggestItem`: suggestion object (`term`, `distance`, `count`).
- `Verbosity`: `TOP`, `CLOSEST`, `ALL`.

### 3.2. Ranker signature

Conceptually:

```python
class SymSpellRanker(Protocol):
    def __call__(
        self,
        phrase: str,
        suggestions: list[SuggestItem],
        verbosity: Verbosity,
    ) -> list[SuggestItem]:
        ...
``

You **do not** need to annotate parameters when implementing it; this is only for static typing and IDEs.

A valid ranker implementation can:

- **Reorder** the list
- **Filter** the list (return a subset)
- **Leave it unchanged** (identity)

It must:

- Return a `list[SuggestItem]`
- Not mutate global SymSpell state (for determinism)

---

## 4. Usage examples

### 4.1. Basic usage (no custom ranker)

This is identical to `symspellpy` except for the import path:

```python
from symspellwhookspy import SymSpell, Verbosity

sym_spell = SymSpell(max_dictionary_edit_distance=2, prefix_length=7)
sym_spell.load_dictionary("frequency_dictionary_en_82_765.txt", 0, 1)

results = sym_spell.lookup("helo", Verbosity.CLOSEST, max_edit_distance=2)
for item in results:
    print(item.term, item.distance, item.count)
```

If you **never** set `sym_spell.ranker`, behavior is equivalent to upstream `symspellpy`.

### 4.2. Attaching a custom ranker

```python
from symspellwhookspy import SymSpell, Verbosity, SuggestItem

sym_spell = SymSpell(max_dictionary_edit_distance=2, prefix_length=7)
sym_spell.load_dictionary("frequency_dictionary_en_82_765.txt", 0, 1)

# No explicit type hints required; IDE can infer from SymSpellRanker Protocol

def my_ranker(phrase, suggestions, verbosity):
    # Example: sort by term length, then alphabetically
    return sorted(suggestions, key=lambda s: (len(s.term), s.term))

sym_spell.ranker = my_ranker

results = sym_spell.lookup("helo", Verbosity.ALL, max_edit_distance=2)
```

**Guarantees:**

- `phrase` is the lookup phrase (or sub‑phrase for compound/segmentation).
- `suggestions` is a non‑empty list of `SuggestItem` objects.
- `verbosity` is the same `Verbosity` used in `lookup` (for compound/segmentation, it will typically be `Verbosity.TOP`).

### 4.3. Using a similarity model (e.g., embeddings or VisSimL1)

Pseudocode sketch for ranking by similarity (details of the model omitted):

```python
from symspellwhookspy import SymSpell, Verbosity, SuggestItem

similarity_model = ...  # e.g., VisSimL1 wrapper with compare(phrase, term) -> float


def similarity_ranker(phrase, suggestions, verbosity):
    # Higher similarity score should come first
    return sorted(
        suggestions,
        key=lambda s: similarity_model.compare(phrase, s.term),
        reverse=True,
    )

sym_spell = SymSpell(max_dictionary_edit_distance=2, prefix_length=7)
sym_spell.load_dictionary("frequency_dictionary_en_82_765.txt", 0, 1)

sym_spell.ranker = similarity_ranker

results = sym_spell.lookup("helo", Verbosity.CLOSEST, max_edit_distance=2)
```

You can plug **any deterministic scoring function** into the ranker.

---

## 5. When the ranker is called

For `symspellwhookspy`, the ranker is applied in these situations:

- `SymSpell.lookup(...)`:
  - After candidates are generated and before results are returned.
  - Even if the result is an exact match (TOP) or from `include_unknown` / `ignore_token` paths.
- `SymSpell.lookup_compound(...)`:
  - For internal per‑token lookups.
  - For the **final compound phrase suggestion**.
- `SymSpell.word_segmentation(...)`:
  - For internal lookups used during segmentation.

The **ranker is never called** when there are **no suggestions**.

If `ranker` is `None`, the library behaves like upstream `symspellpy` and simply sorts suggestions by distance then count.

---

## 6. Migration guide (symspellpy → symspellwhookspy)

To migrate an existing project that uses `symspellpy`:

1. **Change dependency**:

   - In `pyproject.toml` or `requirements.txt`:
     - Replace `symspellpy` with `symspellwhookspy` (with appropriate version, e.g. `>=6.9.0+hooks.1`).

2. **Change imports**:

   - Before:

     ```python
     from symspellpy import SymSpell, Verbosity
     ```

   - After:

     ```python
     from symspellwhookspy import SymSpell, Verbosity
     ```

   - If you need the types for ranker:

     ```python
     from symspellwhookspy import SymSpell, SymSpellRanker, SuggestItem, Verbosity
     ```

3. **(Optional) Update resource loading** if you were using `importlib_resources` with package name:

   - Before:

     ```python
     import importlib_resources

     ref = importlib_resources.files("symspellpy") / "frequency_dictionary_en_82_765.txt"
     ```

   - After:

     ```python
     import importlib_resources

     ref = importlib_resources.files("symspellwhookspy") / "frequency_dictionary_en_82_765.txt"
     ```

4. **Attach a ranker** (optional but recommended for custom tie‑breaking):

   - Add something like:

     ```python
     def ranker(phrase, suggestions, verbosity):
         # Custom deterministic ordering/filtering
         return suggestions

     sym_spell.ranker = ranker
     ```

No other API changes are required; existing calls to `lookup`, `lookup_compound`, and `word_segmentation` continue to work.

---

## 7. Facts to remember

- Use **`symspellwhookspy`** (not `symspellpy`) as the PyPI package name.
- Import from **`symspellwhookspy`**:

  ```python
  from symspellwhookspy import SymSpell, SymSpellRanker, SuggestItem, Verbosity
  ```

- The main added feature is `SymSpell.ranker`, a callable matching the `SymSpellRanker` Protocol.
- The ranker receives `(phrase: str, suggestions: list[SuggestItem], verbosity: Verbosity)` and returns a `list[SuggestItem]`.
- The ranker is invoked for **all** non‑empty suggestion sets from `lookup`, `lookup_compound`, and `word_segmentation`, including exact matches and special cases.
- If `ranker` is `None`, behavior is identical to upstream `symspellpy`.
- The library is designed for **deterministic, custom tie‑breaking** (e.g., embedding similarity, VisSimL1) without changing the core SymSpell algorithm.
