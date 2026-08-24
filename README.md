# File Recovery from Raw Storage

> **TIPE 2024-2025 · Cardot Clément & William Guerin-Garnier · MPI · Lycée Louis Thuillier**

## What is a TIPE?

In France, students in *classes préparatoires* (two-year intensive post-baccalaureate programs preparing for the *grandes écoles* competitive exams) must complete a **TIPE** (*Travail d'Initiative Personnelle Encadré* — supervised personal research project). It is evaluated as a standalone oral exam (~15 min presentation + questions) during the national *concours* and accounts for a significant portion of the final score.

The TIPE must be original, scientifically rigorous, and — for the MPI track (Mathematics, Physics, Computer Science) — rooted in at least one of those three disciplines. Each year a broad theme is announced; students must anchor their work to it.

This project was presented as part of the theme **"Transition, Transformation, Conversion"** (2024-2025): deleted files are not truly gone — they are raw binary data waiting to be transformed back into usable files.

---

## Project overview

**Research question:** Is it theoretically and practically possible to recover supposedly lost files from the raw data of a storage medium?

When a file is deleted on most operating systems, only its metadata (the pointer in the file system table) is erased — the actual data blocks remain on disk until overwritten. This project explores whether those blocks can be found and reconstructed without any metadata.

The work is split into two complementary parts:

- **Clément's part**  — design and implementation of a **simulated storage system** in C with an SQLite backend, acting as a virtual file system
- **William's part**  — implementation of a **file carving** algorithm to recover files from the raw hex dump and testing recovery on **real storage devices**

---

## Repository structure

```
.
├── src/
│   ├── storage_sim.c         # Storage simulator (English)
│   ├── storage_sim_fr.c      # Storage simulator (French — original version)
│   ├── recovery_files.c      # File carving algorithm (English)
│   └── recovery_files_fr.c   # File carving algorithm (French — original version)
├── docs/
│   ├── presentation.pdf
│   └── MCOT.pdf
└── README.md
```

---

## Clément's part — Storage system simulator (`storage_sim.c`)

This program simulates a minimal file system from scratch. Files are stored as a flat hexadecimal dump (`hexadecimal_file.txt`) alongside an SQLite database (`file_manager.db`) that holds the metadata table — mimicking the separation between raw data blocks and the file allocation table found on real drives.

**Simulating deletion** means simply dropping the file's row from the SQL table. The hex data remains untouched in the flat file, exactly as it would on a real disk. The carving algorithm (`recovery_files.c`) can then be run on that raw file without any database access.

### Data model

Each file is stored in the SQLite table `files` with the following fields:

| Field | Type | Description |
|-------|------|-------------|
| `id` | INTEGER | Auto-incremented primary key |
| `name` | TEXT | File name (without extension) |
| `extension` | TEXT | File extension (jpeg, png…) |
| `address` | TEXT | `offset:length` in the hex dump |

The `address` field encodes both where the file starts in the hex dump (`offset`) and how many bytes it occupies (`length`), separated by a colon — a deliberate design choice to keep the schema minimal.

### Available operations

The program exposes an interactive terminal menu:

| Option | Function | Description |
|--------|----------|-------------|
| 1 | `list_database` | List all files in the database |
| 2 | `add_file` | Convert a file to hex, append to dump, insert metadata, delete original |
| 3 | `open_data` | Reconstruct a file from the hex dump using its stored offset/length |
| 4 | `search` | Search files by extension |
| 5 | `rename_file` | Rename a file entry in the database |
| 6 | `delete_file` | Remove DB entry, compact the hex dump, update all offsets |
| 7 | — | Quit |

### Key implementation details

**Adding a file** (`add_file`): the binary file is read byte by byte, each byte written as two hex characters appended to `hexadecimal_file.txt`. The file position before writing gives the offset; the byte count gives the length. Both are stored in the DB as `offset:length`. The original file is then deleted from disk.

**Opening a file** (`open_data`): the offset and length are read from the DB, the hex dump is seeked to that position, and bytes are read two characters at a time and written back as binary — reversing the conversion exactly.

**Deleting a file** (`delete_file`): beyond removing the DB row, the entire hex dump is loaded into memory, the deleted block is excised, the buffer is rewritten to disk, and all subsequent offsets in the DB are decremented accordingly to keep the dump consistent.

**SQL injection mitigation**: the interface uses `snprintf`-built queries with raw user input — a known limitation identified during testing with `DROP TABLE` inputs. Parameterised queries (`sqlite3_bind_*`) would be the correct fix.

### Build & run

```bash
gcc -o storage_sim src/storage_sim.c -lsqlite3
./storage_sim
```

**Dependency:** `sqlite3`

```bash
# Ubuntu/Debian
sudo apt install libsqlite3-dev
```

---

## William's part — File carving algorithm (`recovery_files.c`)

This program takes `hexadecimal_file.txt` as input — typically produced after the database has been deleted — and attempts to reconstruct all files it can find by scanning for known file signatures.

### How it works

The algorithm scans the hex dump sequentially, byte by byte. When it finds a sequence matching a known **header signature**, it enters "inside file" mode and accumulates bytes until it finds the matching **footer signature**. The extracted block is then written to disk as a recovered file.

| Format | Header (hex) | Footer (hex) |
|--------|-------------|-------------|
| JPEG | `FF D8` | `FF D9` |
| PNG | `89 50 4E 47` | `49 45 4E 44 AE 42 60 82` |

File types are defined as an `Extension` struct holding the format name, header bytes, footer bytes, and their respective sizes — making it straightforward to add new formats.

### Build & run

```bash
gcc -o recovery_files src/recovery_files.c
./recovery_files
```

The program reads `hexadecimal_file.txt` from the current directory and writes recovered files as `file1.jpeg`, `file2.png`, etc.

To add a new file format, declare its header and footer byte arrays and pass a new `Extension` to `extract_file()` in `main()`.

### Known limitations

- **Fragmented files** are not handled — the algorithm assumes contiguous storage
- **Video formats (MOV)** were abandoned early due to variable and truncated signatures
- **Text files** were attempted but proved unreliable due to ambiguous signatures (`0x00` / `0x0A`)

---

## Security implications

A key takeaway of this project: **standard deletion is not secure**. Dropping a metadata entry leaves data fully intact on the storage medium. This highlights the importance of secure erase methods (multi-pass overwriting, encryption before deletion) for sensitive data — a point that applies equally to the simulated system and real drives.

---

## Academic context

- **Programme:** MPI, 1st year
- **Theme:** *Transition, Transformation, Conversion* — 2024-2025
- **Thematic positioning:** Computer Science (practical)
- **Keywords:** File carving · File-system simulation · Search algorithm · Sequential reading · Database
- **Group work:** Cardot Clément & William Guerin-Garnier

This project should have been presented as a TIPE oral examination in 2025 at the IUT de Paris — Université Paris Cité, but I personally chose to spend an additional year in preparatory classes (known as a "5/2" in France) to retake the engineering school entrance examinations.

---

## References

1. Steven Alexander — *Understanding Deleted Files and What They Mean*, HG Experts
2. Golden Richard III — *Scalpel: A Frugal, High Performance File Carver*, DFRWS 2005
3. Gary Kessler — *File Signatures Table* — https://www.garykessler.net/library/file_sigs.html
4. Royal E. Frazier Jr. — *All About GIF89a*
5. Kevin Engel — *Comment vérifier l'intégrité d'un fichier ?*, Tech2Tech
6. Thomas Laurenson — *Performance Analysis of File Carving Tools*, INRIA 2017
7. Tanenbaum & Bos — *Modern Operating Systems*, Pearson
8. Nikita Patel et al. — *SQL Injection Attacks: Techniques and Protection Mechanisms*
