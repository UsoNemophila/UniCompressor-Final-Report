# UniCompressor — Unified Lossless Data Compression System

A custom lossless compression system capable of compressing both **text and image files** within a single unified pipeline. Developed as a Data Compression course final project, UniCompressor iteratively evolved through **6 algorithm versions**, ultimately achieving a best-in-class result with the **v6 Interval-Optimized** architecture.

---

## Pipeline Architecture

UniCompressor uses a **4-phase serial pipeline**:

```
Raw Data → Delta Encoding → LZ77 Sliding Window → Trinary Adaptive Entropy Coding → Binary Header + Bitstream
```

### Phase 1 — Delta Encoding (Preprocessing)

Instead of storing raw byte values, the encoder stores the **difference between consecutive bytes**:

- **Encoding**: `D_i = (X_i − X_{i-1}) mod 256`
- **Decoding**: `X_i = (D_i + X_{i-1}) mod 256`

Modular arithmetic ensures values stay within the 8-bit range with no overflow. For gradient images and continuous data streams, delta encoding collapses signal variation into large clusters of near-zero bytes, dramatically lowering entropy before LZ77 sees the data.

### Phase 2 — LZ77 Sliding Window Compression

A classic dictionary-based compressor with two key design choices:

- **Triple-hash acceleration**: A hash map keyed on 3-byte tuples (`triple`) reduces search complexity from O(N²) to near-O(1), making LZ77 viable on binary image data.
- **Strict bit-width constraints**: Window size is locked at 3000 bytes (fits in 12 bits ≤ 4095), and max match length is capped at 255 (exactly 8 bits), ensuring clean handoff to the downstream bit-packer.

Tokens output one of two forms:
| Token Type | Format |
|---|---|
| Literal (no match) | `(False, byte_value, 0)` |
| Back-reference (match) | `(True, distance, length)` |

### Phase 3 — Trinary Adaptive Entropy Coding

The most sophisticated stage. After simulating the cost of encoding, the system dynamically selects one of three modes:

| Mode | Trigger Condition | Description |
|---|---|---|
| **Mode 0** — Unified Huffman | Lowest simulated cost | Single canonical Huffman tree covering literals (0–255) + back-reference flag (257). Fixed 12+8 bit blocks for distances/lengths. Best for small files. |
| **Mode 1** — Bi-Tree + Distance Slots ⭐ | Moderate cost | Two decoupled Huffman trees: **Tree A** for literals/lengths (symbols 0–511, with 256 = EOS), **Tree B** for distances (30 exponential slots). Reduces the distance symbol space from 4096 → 30, shrinking the header array by **99.2%**. |
| **Mode 2** — Stored Pass-Through | Compressed ≥ Original + 7 bytes | Skips encoding entirely; raw bytes are stored directly to prevent bloat on already-compressed or incompressible data. |

**Distance Slotting (v6 key innovation)**: Rather than encoding raw distance integers up to 4095, distances are mapped to one of 30 exponential interval slots (similar in spirit to DEFLATE), and only the slot ID + a small number of "extra bits" is stored. This gives Tree B a ceiling of 30 symbols instead of 4096, compressing the header dramatically.

### Phase 4 — 7-Byte Binary Header

Every archive starts with a fixed deterministic header:

| Field | Width | Purpose |
|---|---|---|
| Magic Number | 16-bit | File signature: `b"MY"` |
| Original Size | 32-bit `uint32` (little-endian) | Uncompressed payload size |
| Chosen Mode | 8-bit `uint8` | `0` = Unified, `1` = Bi-Tree, `2` = Stored |

---

## Version History & Results

The algorithm was refined across 6 versions, benchmarked on 5 standard files. The pivot table below shows **space saving (%)** — higher is better; negative means the compressed file is larger than the original.

| File | v1_base | v2_base | v3_two_tree | v4_dynamic_optimized | v5_final | **v6_interval_optimized** |
|---|---|---|---|---|---|---|
| test1.txt | −3962.86% | −717.14% | −728.57% | −720.00% | **−20.00%** | **−20.00%** |
| test2.txt | −201.40% | 24.49% | −49.89% | 24.45% | 24.45% | **32.15%** |
| test3.txt | −103.12% | 27.74% | −16.88% | 27.72% | 27.72% | **35.58%** |
| Lenna.bmp | 14.40% | 18.80% | 30.51% | 30.51% | 30.51% | **31.36%** |
| Cameraman.bmp | −2.51% | 14.77% | 26.97% | 26.97% | 26.97% | **30.91%** |

> **v6** achieves the best compression on all files that compress positively. The small `test1.txt` file expands under all versions — expected behavior for very short files where header overhead dominates.

---

## Repository Structure

```
UniCompressor-Final-Report/
├── main.ipynb          # Full implementation, benchmarking framework, and version history
├── test1.txt           # Short text file (stress-tests header overhead)
├── test2.txt           # Medium text file
├── test3.txt           # Medium text file
├── Lenna.bmp           # Standard 512×512 grayscale image benchmark
├── Cameraman.bmp       # Standard grayscale image benchmark
└── benchmark_history.csv  # Auto-generated cross-version performance log (created on first run)
```

---

## Getting Started

### Prerequisites

- Python 3.10+
- Jupyter Notebook or JupyterLab

```bash
pip install numpy pandas
```

### Running the Notebook

```bash
git clone https://github.com/UsoNemophila/UniCompressor-Final-Report.git
cd UniCompressor-Final-Report
jupyter notebook main.ipynb
```

Run all cells from top to bottom. The benchmark will run each file 5 times (to average out timing noise), print a **Current Run Report**, display a **Historical Version Comparison** pivot table, and save results to `benchmark_history.csv`.

### Running a New Version

To benchmark a modified version of the algorithm:
1. Update `CURRENT_VERSION` and `VERSION_NOTES` at the top of the benchmark cell.
2. Re-run the benchmark cell — the new results are automatically appended to `benchmark_history.csv` and appear in the pivot table.

---

## Key Design Decisions

- **Lossless only**: Delta encoding is applied with modular arithmetic to guarantee perfect round-trip reconstruction (`decompressed == original`), verified in every benchmark run.
- **Adaptive mode selection**: Cost simulation runs before committing to an encoding mode, ensuring the encoder never makes a file larger when storage pass-through is cheaper.
- **Distance Slotting**: The v6 upgrade directly mirrors the DEFLATE/zlib approach of interval-based distance coding, reducing Tree B from 4096 to 30 symbols and yielding consistent gains across all compressible files.
- **No external compression libraries**: The entire compressor — delta encoder, LZ77, canonical Huffman, bit-packer, and binary header — is implemented from scratch in pure Python.

---

## License

This project is licensed under the [MIT License](LICENSE).