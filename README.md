# tree-sitter-languages

Prebuilt [tree-sitter](https://tree-sitter.github.io/tree-sitter/) parsers for Android, together with the
highlight queries, file type mappings and themes that go with them. The language list, pinned revisions
and queries are kept in sync with [nvim-treesitter](https://github.com/nvim-treesitter/nvim-treesitter),
and everything is built and published automatically by GitHub Actions.

## Downloads

Everything is published on one release, **`android-tree-sitter`**. Its assets are overwritten in place on each
update, so the download links stay stable.

| Asset | Contents |
| --- | --- |
| `tree-sitter-languages-aarch64.zip` | Parsers for `arm64-v8a` |
| `tree-sitter-languages-arm.zip` | Parsers for `armeabi-v7a` |
| `tree-sitter-languages-i686.zip` | Parsers for `x86` |
| `tree-sitter-languages-x86_64.zip` | Parsers for `x86_64` |
| `tree-sitter-queries.zip` | The `queries/` directory |
| `tree-sitter-configs.zip` | `filetypes.json` and the `themes/` directory |

Each parser zip contains one shared library per language, named `tree_sitter_<language>.so`
(for example `tree_sitter_python.so`, `tree_sitter_c_sharp.so`). The libraries are built with the latest stable
Android NDK for API level 24 and stripped. A few grammars may fail to build; the build log of the
*Build tree-sitter parsers* workflow lists them.

## Repository layout

| Path | Description |
| --- | --- |
| `parsers.json` | Languages to build: `name`, pinned `revision` and upstream `url`. Generated, do not edit by hand |
| `queries/` | Highlight and other queries, mirrored from nvim-treesitter. Generated, do not edit by hand |
| `filetypes.json` | Maps each library (`tree_sitter_<language>`) to file extensions and file names |
| `themes/` | Color themes |
| `build_parsers.py` | Clones the grammars and compiles them into shared libraries |
| `.github/workflows/` | The three workflows described below |

## How it works

| Workflow | Trigger | What it does |
| --- | --- | --- |
| `sync-parsers.yml` | Every Monday 03:00 UTC, or manually | Converts nvim-treesitter's `parsers.lua` into `parsers.json` and mirrors `runtime/queries` into `queries/`. Commits only when something really changed, uploads `tree-sitter-queries.zip` when the queries changed, and starts the build when `parsers.json` changed |
| `build-parsers.yml` | `parsers.json` or `build_parsers.py` changes, or manually | Builds every parser for each Android architecture and uploads the four `tree-sitter-languages-*.zip` files |
| `publish-configs.yml` | `filetypes.json` or `themes/` changes, or manually | Packs both into `tree-sitter-configs.zip` and uploads it |

The release and its tag are created once. Later runs only overwrite the assets, and the release notes always show
the latest commit.

## Using the libraries

Every library exports a single entry point, `tree_sitter_<language>`, which returns the language definition:

```c
#include <dlfcn.h>
#include <tree_sitter/api.h>

void *handle = dlopen("tree_sitter_python.so", RTLD_NOW);
const TSLanguage *(*get_language)(void) = dlsym(handle, "tree_sitter_python");

TSParser *parser = ts_parser_new();
ts_parser_set_language(parser, get_language());
```

Parsers generated from `grammar.js` use the language ABI of the tree-sitter CLI used by the build. Make sure the
tree-sitter runtime you link against supports that ABI.

### File types

`filetypes.json` maps a library name to the file extensions and file names it handles:

```json
{
  "tree_sitter_c": [".c", ".h"],
  "tree_sitter_cmake": ["CMakeLists.txt", ".cmake", ".cmake.in"],
  "tree_sitter_comment": []
}
```

Languages that are only used for injections (for example `comment`, `jsdoc` or `markdown_inline`) have an
empty list. Keys are sorted alphabetically.

### Queries

The queries are taken unchanged from nvim-treesitter, so they use its conventions:

- `; inherits: <language>` and `; extends` comments at the top of a file have to be resolved by the consumer.
- Some queries use Neovim-specific predicates and directives such as `#lua-match?` or `#set!`.

## Building locally

Requirements: Python 3.9 or newer, git, a C/C++ compiler, and for grammars that ship without a pre-generated
`parser.c` the [tree-sitter CLI](https://github.com/tree-sitter/tree-sitter) plus Node.js and npm.

```bash
# Build on the host machine
python3 build_parsers.py --json parsers.json --cache ./ts-cache --output ./libs

# Build only some languages
python3 build_parsers.py --only python,lua,typescript,tsx

# Cross-compile for Android with the NDK clang (here: arm64, API 24)
NDK_BIN=/path/to/ndk/toolchains/llvm/prebuilt/linux-x86_64/bin
python3 build_parsers.py \
    --cc  "$NDK_BIN/aarch64-linux-android24-clang" \
    --cxx "$NDK_BIN/aarch64-linux-android24-clang++" \
    --output ./libs-arm64
```

Run `python3 build_parsers.py --help` for the remaining options (`--jobs`, `--abi`, `--no-generate`, `--strict`, ...).

## Licenses

This repository is licensed under the [Apache License 2.0](LICENSE). Each parser is built from its own upstream
repository (listed in `parsers.json`) and stays under that repository's license. The queries come from
nvim-treesitter, which is also licensed under Apache 2.0.
