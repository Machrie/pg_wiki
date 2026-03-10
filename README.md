# pg_kiwi 🚧

> **Status: Work In Progress** — This project is in the design and early development phase. No functional code is available yet.

A PostgreSQL extension for Korean full-text search, powered by the [Kiwi](https://github.com/bab2min/Kiwi) morphological analyzer.

## Problem

PostgreSQL's built-in full-text search has no meaningful support for Korean. The default parser cannot detect Korean word boundaries, making `tsvector`/`tsquery` essentially useless for Korean text.

Current workarounds all have significant drawbacks:

| Approach | How it works | Limitation |
|---|---|---|
| Default parser | Treats Korean as a single opaque token | No word boundary detection at all |
| `pg_bigm` / n-gram | Splits text into character n-grams | High false positives, no linguistic awareness |
| `pg_cjk_parser` | 2-gram tokenization for CJK | No morphological analysis, same n-gram limitations |
| `PGroonga` | Groonga-based full-text search engine | Heavy external dependency, not native PostgreSQL FTS |
| `textsearch_ja` + MeCab-ko | Japanese parser adapted with Korean MeCab dictionary | Hacky, limited maintenance, fragile setup |

None of these provide **morpheme-level tokenization** — the key to accurate Korean full-text search.

## Proposed Solution

pg_kiwi will integrate Kiwi directly into PostgreSQL as a text search parser extension. Kiwi is a proven Korean morphological analyzer that uses statistical language models (Skip-Bigram + Kneyser-Ney smoothing) to resolve ambiguities with 86.7% accuracy, outperforming deep learning-based alternatives while running significantly faster.

### Reference

> Lee, M. (2024). *Kiwi: Developing a Korean Morphological Analyzer Based on Statistical Language Models and Skip-Bigram.* Korean Journal of Digital Humanities, 1(1), 109-136. https://doi.org/10.23287/KJDH.2024.1.1.6

## Planned Features

- **Morphological tokenization**: Segment Korean text into morphemes for full-text search
- **Custom text search parser**: Register as a native PostgreSQL text search parser (`tsvector`/`tsquery` compatible)
- **POS-aware filtering**: Configure which part-of-speech tags to index (e.g. nouns only, exclude particles)
- **User dictionary**: Add domain-specific terms via Kiwi's custom dictionary API
- **Typo tolerance**: Leverage Kiwi's built-in typo correction model
- **In-process execution**: No external server or engine — runs inside PostgreSQL

## Planned Architecture

```
┌─────────────────────────────────┐
│         PostgreSQL               │
│  ┌───────────────────────────┐  │
│  │   Text Search Framework   │  │
│  │   (tsvector / tsquery)    │  │
│  └─────────┬─────────────────┘  │
│            │                     │
│  ┌─────────▼─────────────────┐  │
│  │      pg_kiwi (Rust/pgrx)  │  │
│  │  ┌─────────────────────┐  │  │
│  │  │  Kiwi C API (FFI)   │  │  │
│  │  └─────────────────────┘  │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

- **Rust + pgrx**: Memory-safe PostgreSQL extension development
- **Kiwi C API via FFI**: Call Kiwi's native C interface from Rust
- **No external process**: Everything runs in-process within PostgreSQL

## Planned API (Draft)

> ⚠️ The following SQL examples represent the **target API design**, not currently working code.

```sql
-- Load extension
CREATE EXTENSION pg_kiwi;

-- Tokenize Korean text
SELECT * FROM kiwi_tokenize('형태소 분석기를 PostgreSQL에서 사용할 수 있습니다');
--  token  |  tag  | start | len
-- --------+-------+-------+-----
--  형태소 | NNG   |     0 |   3
--  분석기 | NNG   |     4 |   3
--  를     | JKO   |     7 |   1
--  ...

-- Full-text search with Korean
SELECT * FROM documents
WHERE to_tsvector('korean', content) @@ to_tsquery('korean', '형태소 & 분석');
```

## Tech Stack

| Component | Choice | Reason |
|---|---|---|
| Language | Rust | Memory safety, no segfault debugging |
| PG Extension Framework | [pgrx](https://github.com/pgcentralfoundation/pgrx) | Idiomatic Rust → PostgreSQL extension |
| Morphological Analyzer | [Kiwi](https://github.com/bab2min/Kiwi) C API | Proven accuracy (86.7%), fast, lightweight |
| License | LGPL-3.0 | Consistent with Kiwi's license |

## Roadmap

### Phase 1: Foundation
- [ ] Project scaffold (`cargo pgrx new`)
- [ ] Kiwi C API FFI bindings in Rust
- [ ] Basic `kiwi_tokenize()` SQL function

### Phase 2: Text Search Integration
- [ ] Custom text search parser (START/GETTOKEN/END/LEXTYPES)
- [ ] Text search configuration for Korean
- [ ] POS-based token filtering

### Phase 3: Production Readiness
- [ ] User dictionary support (GUC or table-based)
- [ ] Typo correction integration
- [ ] Benchmark vs pg_bigm, PGroonga, Elasticsearch Nori
- [ ] PostgreSQL 14–17 compatibility testing
- [ ] PGXN packaging

## Target Compatibility

| Component | Version |
|---|---|
| PostgreSQL | 14, 15, 16, 17 |
| Rust | Latest stable |
| pgrx | 0.12.x+ |
| Kiwi | 0.18.x+ |
| OS | Linux (x86_64, aarch64), macOS |

## Contributing

This project is in its early stages. If you're interested in contributing or have feedback on the design, please open an issue.

## License

[LGPL-3.0](LICENSE)

## Acknowledgements

- [Kiwi](https://github.com/bab2min/Kiwi) by bab2min — the Korean morphological analyzer this extension is built upon
- [pgrx](https://github.com/pgcentralfoundation/pgrx) — Rust framework for PostgreSQL extensions
