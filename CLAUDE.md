# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is Zvec

Zvec is an in-process vector database by Alibaba, built on the Proxima engine. It has a C++17 core with Python bindings (PyBind11) and a Node.js package. It supports dense/sparse vectors, hybrid search, structured filtering, and multiple index types (HNSW, IVF, Flat, Invert).

## Build & Development

**Prerequisites:** Python >= 3.9, CMake >= 3.26 (< 4.0), C++17 compiler (g++11+, clang++, Apple Clang)

```bash
# Clone with submodules (thirdparty deps)
git clone --recursive <repo>
# If already cloned without --recursive:
git submodule update --init --recursive

# Editable install with dev dependencies (builds C++ extension)
pip install -e ".[dev]"

# Debug build with Make instead of Ninja
CMAKE_BUILD_TYPE=Debug CMAKE_GENERATOR="Unix Makefiles" pip install -v .

# Build with tools (benchmarks, etc.)
pip install -v . --config-settings='cmake.define.BUILD_TOOLS="ON"'
```

## Testing

```bash
# Python tests
pytest python/tests/ -v

# Single test file
pytest python/tests/test_collection.py -v

# With coverage
pytest python/tests/ --cov=zvec --cov-report=term-missing

# C++ unit tests (from build/ directory after building with BUILD_TOOLS=ON)
make unittest -j$(nproc)
```

## Linting & Formatting

```bash
# Python lint/format
ruff check .
ruff format --check .

# C++ format (excludes thirdparty/, tests/, scripts/, python/)
clang-format --dry-run --Werror <files>
```

## Commit Conventions

Pre-commit hooks enforce:
- **Conventional commits:** `<type>(<scope>): <message>` where type is feat/fix/docs/style/refactor/test/chore/perf/ci/build/revert
- **Branch names:** `<type>/<lower-case-hyphen-separated>` (e.g., `feat/user-api`, `fix/header-bug`)
- Ruff and clang-format auto-fix on commit
- Gitleaks secret detection

## Architecture

### Layer Structure

```
Python API (python/zvec/)     -- Pythonic wrapper, extensions, query execution
  ↓ PyBind11 (src/binding/)   -- C++/Python bridge
Database (src/db/)             -- Collections, schema, SQL filter engine, segments
Core (src/core/)               -- Search algorithms (HNSW, IVF, Flat), quantizers, metrics
Ailego (src/ailego/)           -- Low-level utilities (I/O, math, containers, memory)
```

### Key Directories

- `src/include/zvec/` - Public C++ API headers
- `src/db/collection.cc` - Main collection implementation (DDL/DML/DQL)
- `src/db/proto/zvec.proto` - Protobuf schema defining vector/index types
- `src/db/sqlengine/` - ANTLR-based SQL parser for filtering expressions
- `src/core/algorithm/` - Index implementations (hnsw, ivf, flat, clustering)
- `src/core/quantizer/` - Vector quantization (int4, int8, fp16, binary)
- `python/zvec/model/collection.py` - Python Collection class wrapping C++ ops
- `python/zvec/executor/query_executor.py` - Multi-vector query with reranking
- `python/zvec/extension/` - Pluggable AI extensions (embeddings, rerankers)

### Build System

CMake with custom Bazel-like macros in `cmake/bazel.cmake`. Python wheels built via scikit-build-core. Key CMake options: `BUILD_PYTHON_BINDINGS` (ON/OFF), `BUILD_TOOLS` (ON/OFF), `ENABLE_SKYLAKE_AVX512` (x86_64 optimization).

### Plugin Architecture

The core uses a plugin/factory pattern: index types, distance metrics, quantizers, and converters are registered as pluggable components via `src/core/interface/` and `src/core/framework/`.

### Python Isort Requirement

All Python files must include `from __future__ import annotations` as the first import (enforced by ruff's isort config).

### Thirdparty Dependencies

All vendored in `thirdparty/` as git submodules: googletest, rocksdb, protobuf, arrow, yaml-cpp, gflags, glog, lz4, CRoaring, sparsehash, magic_enum, antlr.
