# About
**ImNotepad** is a high-performance text editor with syntax highlighting that can be fully integrated into applications built using [Dear ImGui](https://github.com/ocornut/imgui). It originated as an experiment during the development of another project and was subsequently spun off into a standalone extension. **ImNotepad** relies on two powerful, modern technologies to replace legacy text editor paradigms:

<div align="center">
  
## `FredBuf` (Piece-Tree Text Buffer)

</div>

A significantly modified port of [Cameron DaCamara's `fredbuf`](https://github.com/cdacamar/fredbuf), an immutable piece-tree text buffer implementation inspired by Visual Studio Code's text buffer reimplementation. Key differences from the original repository:

### 1. Removal of Strong Typedefs and Logical Wrappers

- **No more "Strong Typedefs" / "Semantic Types"**: The original library heavily relied on `enum class` aliases (e.g., `enum class Length : size_t`, `enum class CharOffset : size_t`, `enum class Line : size_t`) to prevent mixing up units at compile time. This port completely replaces them with standard, plain `size_t` for all offsets, lengths, line numbers, and indices.
- **Removal of Logical Wrappers**: Utility functions like `rep()`, `retract()`, `extend()` and `implies()` (originally provided by `enum-utils.h` to manipulate these strongly-typed enums) have been entirely removed. Arithmetic and comparisons are now performed directly using standard C++ operators (`+`, `-`, `==`, etc.). This makes the codebase more straightforward, idiomatic, and accessible to general C++ developers, albeit without the compile-time unit safety of the original design.

### 2. Native UTF-8 Support & Custom Line Metrics
- Introduced a comprehensive `LineMetrics` struct that tracks **codepoint counts** (`cp_max_count`), **tab counts** (`tab_count_in_max_line`), and **visual width** (`visual_max_width`), along with prefix/suffix metrics. 
- This allows the buffer to accurately calculate visual line widths considering tab expansion and multi-byte UTF-8 characters, which is critical for rendering in UI frameworks.

### 3. Greatly Enhanced Undo/Redo System
- The original `UndoRedoEntry` only stored the tree root and an operation offset. The port's `UndoRedoEntry` is now a rich structure containing: `op_offset`, `old_len`, `new_len`, `sel_start`, `sel_end`, `version_id`, and `eol_sequence`. This enables precise cursor/selection restoration and robust history grouping (e.g., grouping continuous typing into a single undo step).

### 4. Explicit End-of-Line (EOL) Sequence Management
- Added explicit tracking of `EndOfLineSequence` (`LF`, `CR`, `CRLF`) at the buffer metadata and history entry levels.
- The `TreeBuilder` now includes platform-aware EOL detection logic (Windows, Linux, macOS) when ingesting text chunks.

### 5. Namespace Change
- The root namespace has been changed from `PieceTree` to `FredBuf` to better reflect the library's identity and avoid naming collisions in larger projects.

<div align="center">
  
## `Tree-sitter` (Incremental Parser)

</div>

**ImNotepad** completely abandons traditional Regular Expressions (Regex) for syntax highlighting in favor of [Tree-sitter](https://github.com/tree-sitter/tree-sitter). 
- **True AST-Based Highlighting**: Uses dynamic **Tree-sitter** grammar libraries (`.dll` / `.so` / `.dylib`) and standard `.scm` query files to parse code into a Concrete Syntax Tree.
- **Viewport-Optimized Querying**: To maintain FPS on massive files, the editor does *not* highlight the entire document at once. It restricts the **Tree-sitter** query cursor strictly to the visible viewport plus a small line reserve, making rendering virtually instantaneous.
- **Incremental Updates**: When you type, **Tree-sitter** only re-parses the specific affected region of the AST. The editor seamlessly feeds these byte-level edits directly to the parser without copying the entire text buffer.
- **Custom Predicate Evaluation & Caching**: The core C-library of **Tree-sitter** exposes predicates but doesn't evaluate them. **ImNotepad** implements a basic mechanism for evaluating predicates like `#eq?`, `#not-eq?` and `#any-of?`, that utilizes caching, ensuring fast and accurate filtering of syntactic elements.
- **Zero-Recompile Extensibility**: Add support for any new programming language simply by dropping the compiled grammar binary and its `.scm` query files into the `resources/grammars/` and `_modules/queries/` directories. No editor recompilation is required. **NOTE**: `.scm` query files
### Important notes:
- Support for `#match?` and `#set!` is not provided in this implementation.
- Although the capture names correspond to the standardized **Tree-sitter** captures used in **Neovim**, using **Neovim**’s `.scm` query files directly may lead to unsatisfactory results.

# Color schemes for syntax highlighting
**ImNotepad implements** a syntax highlighting system based on mapping scopes to colors.
- **Scope-to-Color Maps**: Each syntax theme is defined as a flat array of `ImNotepadSyntaxColorMap` entries. Each entry is a pair consisting of a **32-bit FNV-1a** hash of the capture name and a **32-bit ABGR** color value. At render time, the editor hashes the **Tree-sitter** capture name and performs an O(1) lookup in the color map — no string comparisons, no allocations.
- **Compile-Time Evaluation**: Color maps are built using the `CONSTEXPR_KEYWORD_COLOR("scope.name", "#RRGGBB")` macro. Both the hash and the ABGR color value are computed **entirely at compile time** via `constexpr` functions. This means theme initialization has **zero runtime cost** and produces optimal machine code. **NOTE**: your compiler must support **C++14** or a later version.
- **Neovim-Compatible Captures**: Capture names follow the [standardized Tree-sitter highlight groups from Neovim](https://neovim.io/doc/user/treesitter/#treesitter-highlight) (e.g., `@function`, `@keyword.conditional`, `@string.escape`, `@markup.heading.1`). This makes it trivial to port existing Neovim themes to ImNotepad.
- **Built-in Themes**: The library ships with several themes out of the box:
  - **Catppuccin Frappé** — a soft, pastel dark theme
  - **One Dark Pro** (Dark / Light / Visual Studio variants)
  - **Vim Code Dark** — the classic VS Code dark theme

<img width="1602" height="932" alt="base" src="https://github.com/user-attachments/assets/fd9d708e-7d13-4699-9bde-981515a0b620" />
<img width="1602" height="932" alt="base_dark" src="https://github.com/user-attachments/assets/7592999b-5825-4b9d-8877-fef70d7570d0" />
<img width="1602" height="932" alt="base_light" src="https://github.com/user-attachments/assets/b91b1c90-a000-48cb-9e47-7ec69be90cae" />
<img width="1602" height="932" alt="base_classic" src="https://github.com/user-attachments/assets/b9411c1b-9d4c-4289-b2af-3ea8e9be7f3f" />

# Mouse and keyboard controls

```text
 - MOUSE CONTROLS
   - Mouse wheel:                   Scroll vertically.
   - Shift + Mouse wheel:           Scroll horizontally.
   - Ctrl + Mouse wheel:            Adjusts the zoom level of the text editor (font size).
   - Double click:                  Select single word.
   - Triple click:                  Select the entire line.
   - Multiple clicks:               Select All.

 - KEYBOARD CONTROLS
   - Ctrl + Up/Down:                Scrolling the page line by line.
   - Ctrl + Left/Right:             Word jump.
   - Ctrl+Shift + Left/Right:       Select words.
   - Ctrl+A:                        Select All.
   - Ctrl+C, Ctrl+X, Ctrl+V:        Copy, Cut and Paste text.
   - Ctrl+D:                        Duplicate selected word or entire line.
   - Ctrl+Z:                        Undo.
   - Ctrl+Y or Ctrl+Shift+Z:        Redo.
   - Shift + Home/End:              Select to line start/end.
   - Shift + Page Up/Down:          Select viewport text.
   - Shift + Up/Down arrow keys:    Select line up/down.
   - Page Up, Page Down:            Previous page, next page.
   - Home, End:                     Set cursor to line start/end.
   - ESCAPE:                        Deselect text.
```
Hover over any highlighted token to instantly see its exact **Tree-sitter** capture name and scope hierarchy. This is a useful tool for theme authors and language grammar developers.

# License
This work is licensed under the terms of the MIT license. For a copy, see <https://opensource.org/licenses/MIT>.
