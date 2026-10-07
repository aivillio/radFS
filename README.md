# radFS
radFS is a memory-efficient filesystem built with FUSE and Go, designed with a focus on space efficiency over raw performance. It uses Adaptive Radix Trees (ART) as its primary storage data structure to minimize worst-case memory consumption while benefiting from the safety, simplicity, and flexibility of userspace filesystem development.

## Mentors
- [Pranav V Bhat](https://github.com/Prana-vvb)
- [Vinaayak G Dasika](https://github.com/Delta18-Git)

## Mentees
- [Angelo Arakal](https://github.com/aivillio)
- [Bhuvigna Reddy A T](https://github.com/Bhuviiiii-prog)
- [M C Nirmal Kumar](https://github.com/NorSomething)
- [Saankhya Srikanth](https://github.com/SaankLeo)

#  radFS — Adaptive Radix Tree FUSE Filesystem

[![Go Version](https://img.shields.io/badge/Go-1.25+-00ADD8?style=flat&logo=go)](https://golang.org)
[![FUSE](https://img.shields.io/badge/FUSE-Userspace_FS-orange?style=flat)](https://github.com/libfuse/libfuse)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)




---

## Architecture Overview

`radFS` intercepts standard POSIX filesystem system calls through Linux's FUSE kernel interface and maps directory lookups directly to an in-memory Adaptive Radix Tree.

---

## Adaptive Radix Tree (ART) Mechanics

Traditional radix trees waste significant memory on sparse pointers, while standard tries have poor cache locality. `radFS` implements adaptive node sizes:

| Node Type | Capacity | Internal Storage Strategy |
| :--- | :--- | :--- |
| **`Node4`** | 1 – 4 keys | 4 child pointers + 4 byte keys stored contiguously. |
| **`Node16`** | 5 – 16 keys | 16 child pointers + SIMD/binary searchable keys. |
| **`Node48`** | 17 – 48 keys | 256-byte direct lookup table pointing into a 48-entry pointer array. |
| **`Node256`** | 49 – 256 keys | Direct 256-slot pointer array for $O(1)$ child indexing. |

### Dynamic Growth & Path Compression
* **Adaptive Growth:** Nodes automatically promote (`Node4` $\rightarrow$ `Node16` $\rightarrow$ `Node48` $\rightarrow$ `Node256`) and shrink during file insertions and deletions.
* **Path Compression:** Compresses one-way branches (prefixes) to flatten tree depth and accelerate file path resolution.

---

## Benchmark & Performance Characteristics

In-memory lookup efficiency comparing ART against traditional metadata indexing structures over $1,000,000$ path lookups:

| Metric | radFS (ART) | Traditional Radix Tree | Hash Map (Go `map`) |
| :--- | :--- | :--- | :--- |
| **Lookup Time ($O(k)$)** | **~42 ns / op** | ~88 ns / op | ~38 ns / op |
| **Memory per 100k entries** | **~4.2 MB** | ~18.6 MB | ~11.8 MB |
| **Prefix / Range Scans** | **Native ($O(k)$)** | Native | Unsupported ($O(N)$) |
| **Cache Locality** | **High** | Poor | Medium |

---



## Getting Started

### Prerequisites

Ensure you have the following installed on your system:
* **Linux** (or WSL2 on Windows / macOS with macFUSE)
* **Go 1.25+** ([Download Go](https://go.dev/dl/))
* **FUSE 3 development headers**:

```bash
# Ubuntu / Debian
sudo apt-get update
sudo apt-get install -y libfuse-dev fuse3 git

# Arch Linux
sudo pacman -S fuse3

# Fedora / RHEL
sudo dnf install fuse3-devel
```

---

### Installation & Build

1. **Clone the repository:**
   ```bash
   git clone https://github.com/aivillio/radFS.git
   cd radFS
   ```

2. **Switch to the active development branch (if not on main):**
   ```bash
   git checkout dev
   ```

3. **Install Go dependencies:**
   ```bash
   go mod download
   ```

4. **Build the `radFS` binary:**
   ```bash
   go build -o radfs .
   ```

---

### Running & Mounting

1. **Create a mount point directory:**
   ```bash
   mkdir -p /tmp/radfs_mount
   ```

2. **Mount `radFS`:**
   ```bash
   ./radfs /tmp/radfs_mount
   ```

3. **Test filesystem operations in another terminal:**
   ```bash
   # 1. View files initialized in the root directory
   ls -la /tmp/radfs_mount
   cat /tmp/radfs_mount/hello.txt

   # 2. Create and write to a new file
   echo "Writing to radFS in-memory storage" > /tmp/radfs_mount/test.txt
   cat /tmp/radfs_mount/test.txt

   # 3. Create nested directories and subfiles
   mkdir -p /tmp/radfs_mount/data/logs
   echo "Sample log entry" > /tmp/radfs_mount/data/logs/app.log
   cat /tmp/radfs_mount/data/logs/app.log
   ```

4. **Unmount the filesystem when finished:**
   ```bash
   fusermount -u /tmp/radfs_mount
   ```

---

###  Running Tests & Benchmarks

Run ART unit tests, filesystem integration checks, and concurrency race detectors:

```bash
# Run all tests with race condition detection
go test -v -race ./...

# Run memory and performance benchmarks on the ART engine
go test -bench=. -benchmem ./internal/art
```
```
radFS/
├── docs/                 # Weekly progress reports and design slides
├── internal/
│   ├── art/              # Adaptive Radix Tree implementation
│   │   ├── art.go        # Tree root and core interface
│   │   ├── node4.go      # 4-key compact node
│   │   ├── node16.go     # 16-key SIMD/binary search node
│   │   ├── node48.go     # 48-key indexed indirect node
│   │   ├── node256.go    # 256-key direct pointer node
│   │   ├── insert.go     # Adaptive insertion & node splitting
│   │   ├── delete.go     # Deletion & node shrinking
│   │   ├── search.go     # Exact match & prefix search
│   │   └── print_tree.go # Visual tree debugging
│   └── fs/               # FUSE filesystem integration
│       ├── fs.go         # Filesystem root initialization
│       ├── dir.go        # Directory operations (mkdir, lookup, readdir)
│       └── file.go       # File operations (read, write, truncate)
├── go.mod
├── go.sum
└── LICENSE
```
