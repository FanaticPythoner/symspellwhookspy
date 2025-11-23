symspellwhookspy
================

**SymSpell Python port with custom ranker hooks for deterministic tie-breaking**

This is a fork of [symspellpy](https://github.com/mammothb/symspellpy) that adds a flexible, type-safe **custom ranker hook** system. This allows you to inject your own tie-breaking logic (e.g., embedding-based similarity, VisSimL1, or any custom metric) to achieve 100% deterministic spell correction.

## Key Features

- **Custom Ranker Protocol**: Strongly-typed `SymSpellRanker` Protocol for IDE auto-completion and type inference
- **No Manual Type Hints Required**: Write `def ranker(phrase, suggestions, verbosity):` and your IDE automatically infers parameter types
- **Comprehensive Hook Coverage**: Ranker invoked in all lookup paths: `lookup()`, `lookup_compound()`, and `word_segmentation()`
- **Backward Compatible**: Drop-in replacement for symspellpy; ranker hook is optional
- **Fully Tested**: All 143 tests pass, including new tests for ranker hook invocation in all scenarios

## Installation

```bash
pip install git+https://github.com/FanaticPythoner/symspellwhookspy.git
```

## Usage

```python
from symspellwhookspy import SymSpell, Verbosity

# Create SymSpell instance
sym_spell = SymSpell(max_dictionary_edit_distance=2, prefix_length=7)
sym_spell.load_dictionary("frequency_dictionary.txt", 0, 1)

# Define a custom ranker (no type hints needed - IDE infers automatically!)
def my_ranker(phrase, suggestions, verbosity):
    # Custom tie-breaking logic here
    # For example, rank by similarity to original phrase
    return sorted(suggestions, key=lambda s: my_similarity(phrase, s.term), reverse=True)

# Attach the ranker
sym_spell.ranker = my_ranker

# Use as normal - ranker will be invoked automatically
results = sym_spell.lookup("helo", Verbosity.CLOSEST, max_edit_distance=2)
```

## Original Project

Based on [symspellpy](https://github.com/mammothb/symspellpy) v6.9.0, which is a Python port of [SymSpell](https://github.com/wolfgarbe/SymSpell) v6.7.2.

All original features, performance characteristics, and unit tests are preserved.

