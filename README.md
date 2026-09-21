# MSE-GLM Tokenizer

**`tokenizer.py` — A fast, lightweight, two-stage Character/Word tokenizer designed for the MSE-GLM architecture.**

The MSE-GLM tokenizer replaces traditional **Byte Pair Encoding (BPE) iterative merge scans** with a deterministic two-stage architecture built around a shared ID space.

On the same measured corpus, the current implementation reduces vocabulary-generation time from approximately **25 minutes to under 50 seconds**, while retaining a complete character-level fallback for words that are not included in the Stage 2 vocabulary.

> **Current implementation:** Stage 1 + Stage 2  
> **Stage 3:** Planned, but not yet implemented.

---

## Features

- **Two-stage architecture**
  - Stage 1: Character vocabulary
  - Stage 2: Multi-character lexical vocabulary
- **Single shared ID space**
- **Character-level fallback for out-of-vocabulary words**
- **`<UNK>` only for genuinely unknown characters**
- **Explicit `<WORD_BOUND>` token**
- **Deterministic vocabulary construction**
- **Streaming corpus processing**
- **Incremental vocabulary extension**
- **Save/load support**
- **Designed for deterministic MSE-GLM graph structures**

---

# Architecture

The tokenizer is built around a layered representation:

```text
                 MSE-GLM Tokenizer
                        │
             ┌──────────┴──────────┐
             │                     │
        Stage 1                Stage 2
   CharacterVocabulary      TokenVocabulary
             │                     │
             │              Multi-character
             │               lexical tokens
             │                     │
             └──────────┬──────────┘
                        │
                 Shared ID Space
