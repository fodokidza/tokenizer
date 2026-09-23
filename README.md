# MSE-GLM Tokenizer (`tokenizer.py`)

A fast, lightweight, two-stage Character/Word tokenizer designed specifically for the **MSE-GLM** architecture.

The tokenizer replaces traditional Byte Pair Encoding (BPE) iterative merge scans with a deterministic two-stage architecture built around a shared ID space.

On the same measured corpus, the current implementation reduces vocabulary-generation time from approximately **25 minutes to under 50 seconds**, while retaining a complete character-level fallback for words that are not included in the Stage 2 vocabulary.

> **Current implementation:** Stage 1 + Stage 2
> **Stage 3:** Planned, but not yet implemented.

---

## Table of Contents

- [Key Features](#key-features)
  - [Two-Stage Architecture](#two-stage-architecture)
  - [Single Shared ID Space](#single-shared-id-space)
  - [Character-Level Fallback](#character-level-fallback)
  - [Explicit Word Boundaries](#explicit-word-boundaries)
  - [Deterministic Vocabulary](#deterministic-vocabulary)
  - [Streaming & Incremental Training](#streaming--incremental-training)
- [Architectural Overview](#architectural-overview)
- [Tokenization Flow](#tokenization-flow)
- [Why This Architecture?](#why-this-architecture)
- [Performance](#performance)
- [Token Representation](#token-representation)
- [Planned Stage 3](#planned-stage-3)
- [Current Status](#current-status)
- [Summary](#summary)

---

## Key Features

### Two-Stage Architecture

The tokenizer separates fundamental character representation from higher-level lexical optimization.

#### Stage 1 — `CharacterVocabulary`

A complete character-level vocabulary containing the unique characters discovered in the training corpus.

```
Characters
    ↓
Character IDs
```

Stage 1 provides the fundamental representation layer and allows words outside the Stage 2 vocabulary to be represented character by character.

> Characters are currently **case-sensitive**.

#### Stage 2 — `TokenVocabulary`

A higher-level vocabulary containing selected multi-character lexical tokens. Each Stage 2 token is built directly from Stage 1 character IDs.

For example:

```
playing
   ↓
[p, l, a, y, i, n, g]
   ↓
Stage 1 character IDs
```

If `"playing"` is selected for Stage 2, it can instead be represented by a single Stage 2 token.

Stage 2 therefore acts as an optimization layer over the complete Stage 1 representation.

---

### Single Shared ID Space

Special tokens, Stage 1 character IDs, and Stage 2 token IDs occupy a single shared integer ID space.

```
<PAD>
<UNK>
<BOS>
<EOS>
<WORD_BOUND>
    ↓
Stage 1 character IDs
    ↓
Stage 2 token IDs
```

This makes every model-visible token ID globally unique and directly usable by MSE-GLM's matrix and graph structures.

---

### Character-Level Fallback

A word does not need to exist in the Stage 2 vocabulary to be represented.

For example, if `elephant` is not present in Stage 2, but all of its characters are known, it can fall back to:

```
<WORD_BOUND>
e l e p h a n t
```

Therefore: **Unknown word ≠ `<UNK>`**

`<UNK>` is only required when the tokenizer encounters a character that was not present in the learned Stage 1 character vocabulary. This provides a complete fallback representation for any word composed entirely of known characters.

---

### Explicit Word Boundaries

The tokenizer uses `<WORD_BOUND>` when a word is represented using lower-level character tokens.

For example, `the elephant runs` can be represented conceptually as:

```
THE
<WORD_BOUND> e l e p h a n t
RUNS
```

The explicit boundary prevents separately encoded character sequences from being incorrectly fused during decoding.

---

### Deterministic Vocabulary

Vocabulary generation is deterministic. Given the same:

- training corpus
- tokenizer configuration
- vocabulary size

...the tokenizer produces the same vocabulary and token IDs.

Stage 2 vocabulary selection is based on word frequency, while Stage 1 character IDs are assigned deterministically.

---

### Streaming & Incremental Training

The tokenizer supports memory-efficient corpus processing through streaming. Large corpora can be processed without requiring the entire corpus to be loaded into memory at once.

It also supports vocabulary extension while preserving previously assigned IDs:

```
Existing vocabulary
        +
New characters / tokens
        ↓
Extended vocabulary
```

Previously assigned IDs are **not** invalidated when the vocabulary is extended.

---

## Architectural Overview

The tokenizer is built as a hierarchy:

```
                    MSE-GLM Tokenizer
                           │
              ┌────────────┴────────────┐
              │                         │
        Stage 1                    Stage 2
   CharacterVocabulary          TokenVocabulary
              │                         │
              │                  Multi-character
              │                  lexical tokens
              │                         │
              └────────────┬────────────┘
                           │
                    Shared ID Space
```

The important architectural property is that **Stage 2 does not replace Stage 1**. Instead, Stage 2 provides a compact representation for selected tokens while Stage 1 remains the underlying fallback representation.

---

## Tokenization Flow

For an input word, the tokenizer follows this conceptual process:

```
                    Input word
                        │
                        ▼
              Is it in Stage 2?
                    /       \
                  YES        NO
                   │          │
                   ▼          ▼
              Stage 2      Stage 1
               token      characters
```

For example:

```
play → [PLAY_ID]                      (if present in Stage 2)
play → [WORD_BOUND, p, l, a, y]        (if not present)
```

Both representations ultimately have a deterministic Stage 1 character interpretation.

---

## Why This Architecture?

Traditional BPE builds its vocabulary through repeated merge operations:

```
Characters → Find frequent pair → Merge → Find another pair → Merge → Repeat → Vocabulary
```

The MSE-GLM tokenizer instead separates the process into explicit representation layers:

```
Corpus
   │
   ▼
Stage 1 — Characters
   │
   ▼
Stage 2 — Frequent multi-character tokens
```

This removes the iterative merge-loop architecture from vocabulary construction. The result is a simpler and more directly auditable tokenization process.

---

## Performance

On the same measured corpus, the current implementation produced approximately:

| Tokenizer Architecture         | Vocabulary Generation |
|---------------------------------|------------------------|
| Previous BPE implementation     | ~25 minutes            |
| MSE-GLM two-stage tokenizer     | <50 seconds            |

> These measurements describe the tokenizer *architectures* being compared. They are not intended as a comparison between programming languages or implementations such as Python versus Rust.

The key architectural difference is the removal of iterative merge scanning.

---

## Token Representation

Every Stage 2 token maintains its Stage 1 recipe. For example:

```
Stage 2: PLAY
    ↓
[p, l, a, y]
    ↓
Stage 1 IDs
```

This gives the tokenizer explicit representation lineage:

```
Stage 2 token → Stage 1 token sequence → Characters
```

This property is particularly useful for MSE-GLM because the model is designed around explicit token relationships and deterministic structures rather than opaque learned embeddings.

---

## Planned Stage 3

Stage 3 is part of the planned tokenizer architecture but is **not yet implemented**.

The intended purpose is to provide a cache of frequently occurring combinations of Stage 2 tokens. For example, given Stage 2 tokens `PLAY`, `ING`, `SAY`, a future Stage 3 could learn:

```
PLAY + ING → PLAYING
SAY  + ING → SAYING
```

The planned hierarchy would therefore become:

```
Stage 3
   ↓
Stage 2
   ↓
Stage 1
   ↓
Characters
```

And lookup would conceptually become:

```
Input
  │
  ▼
Stage 3? ── YES → Stage 3 token
  │
  NO
  │
  ▼
Stage 2? ── YES → Stage 2 token
  │
  NO
  │
  ▼
Stage 1 fallback
```

> Stage 3 should therefore be considered future architecture, not a feature of the current `tokenizer.py`.

---

## Current Status

| Component                                  | Status      |
|---------------------------------------------|-------------|
| Stage 1 — Character vocabulary               | ✅ Implemented |
| Stage 2 — Multi-character vocabulary         | ✅ Implemented |
| Shared ID space                              | ✅ Implemented |
| Character fallback                           | ✅ Implemented |
| `<WORD_BOUND>` handling                      | ✅ Implemented |
| Deterministic IDs                            | ✅ Implemented |
| Streaming corpus processing                  | ✅ Implemented |
| Vocabulary extension                         | ✅ Implemented |
| Save / Load                                  | ✅ Implemented |
| Stage 3 — Long-token cache                   | 🚧 Planned |
| Stage 3 — Stage-2 composition learning       | 🚧 Planned |
| Stage 3 — Lookup hierarchy                   | 🚧 Planned |

---

## Summary

The MSE-GLM tokenizer uses a simple layered representation:

```
             Stage 2
        ┌───────────────┐
        │ Words / Tokens│
        └───────┬───────┘
                │
                ▼
             Stage 1
        ┌───────────────┐
        │  Characters   │
        └───────────────┘
```

Stage 2 provides compact representations for selected multi-character tokens, while Stage 1 guarantees character-level fallback for words outside the Stage 2 vocabulary.

The architecture is designed to provide:

- Deterministic vocabulary construction
- Fast vocabulary generation
- A unified token ID space
- Explicit token lineage
- Character-level fallback
- Efficient representation of frequent words
- Streaming and incremental vocabulary construction
- Compatibility with MSE-GLM's deterministic graph-based architecture

Stage 3 will extend this hierarchy with cached compositions of Stage 2 tokens once implemented.
