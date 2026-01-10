# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

PyShare (textarea.my) is a minimalist Python code editor that runs entirely in the browser. It features Python syntax highlighting, in-browser code execution via Pyodide, URL-based code sharing with compression, and file save/load capabilities. The entire application is contained in a single HTML file with embedded CSS and JavaScript.

## Architecture

### Single-File Structure
The entire application lives in `index.html` (~2100 lines) containing:
- HTML structure with semantic markup
- Embedded CSS with dark/light theme support
- Complete JavaScript application logic
- SVG icon definitions

### Core Components

**1. Editor Engine**
- Custom contenteditable-based editor (no dependencies)
- Implements undo/redo history (max 10,000 states)
- Cursor position management and restoration
- Auto-indentation (4 spaces after `:`)
- Auto-completion for Python keywords on Tab
- Auto-closing of brackets, parentheses, and quotes

**2. Python Parser (`parsePython` function, ~line 1042)**
- Tokenizes Python code using regex patterns with sticky flag (`/y`)
- Applies CSS classes for syntax highlighting
- Handles: keywords, built-ins, strings (all quote types, f-strings, raw strings), comments, numbers (int/float/hex/octal/binary), decorators, function/class definitions
- Matching order matters: comments/strings → decorators → function/class defs → keywords → built-ins → numbers

**3. Pyodide Integration (~line 1424-1590)**
- Lazy-loads Pyodide 0.25.0 from CDN on first execution
- `initPyodide()`: Initializes Python runtime, shows loading indicator
- `runPythonCode()`: Executes code, captures stdout/stderr
- `lintPythonCode()`: Real-time syntax checking using Python's `ast.parse()`
- Debounced linting (1 second delay after typing)

**4. Compression & URL Sharing (~line 820-850)**
- `compress()`: Deflate algorithm + base64url encoding
- `decompress()`: Reverse process to restore code from URL hash
- Automatically updates URL hash as content changes
- Warning system: yellow >2000 chars, red >8000 chars

**5. Storage (~line 762-810)**
- Dual persistence: localStorage + URL hash
- `load()`: Reads from hash first, falls back to localStorage
- `save()`: Writes to both localStorage and URL hash
- Stores theme preference separately

**6. File Operations (~line 870-970)**
- `downloadHTML()`: Saves styled HTML file
- `downloadTXT()`: Saves plain text
- `downloadPython()`: Saves .py file
- File loading via file input with FileReader API

**7. UI Management (~line 1236-1420)**
- `initUI()`: Sets up event listeners, menu system
- Floating action button (menu toggle)
- Output console with show/hide/close functionality
- Theme toggle (light/dark/auto)
- Notification system for user feedback

## Testing

### Running Tests
```bash
# Open in browser directly
open tests.html

# Auto-run mode
open tests.html?autorun

# Or serve locally
python3 -m http.server 8080
# Then visit: http://localhost:8080/tests.html
```

### Test Suite
- **Location**: `tests.html` (custom lightweight test framework)
- **22 automated tests** covering syntax highlighting, URL compression, storage, UI elements, and editor functionality
- **Manual tests**: Documented in `TESTING.md` for cross-browser, responsive design, and Pyodide execution
- **Adding tests**: Use `runner.describe()` pattern, see `TEST_SUITE_README.md`

## Development Commands

### Local Development
```bash
# Serve locally (recommended for testing)
python3 -m http.server 8080
# or
npx http-server -p 8080

# Open main application
open http://localhost:8080/index.html

# Open tests
open http://localhost:8080/tests.html?autorun
```

### No Build Process
This project has no build step, bundler, or package manager. All development is done directly in `index.html`.

## Key Implementation Details

### Modifying Python Syntax Highlighting
Located in `parsePython` function (~line 1042). Pattern matching uses sticky regex (`/y` flag) with `lastIndex`:

```javascript
pattern.lastIndex = i
const res = pattern.exec(input)
if (res && res.index === i) {
  // Match found at current position
  const span = document.createElement('span')
  span.className = 'py-tokentype'
  span.textContent = res[0]
  frag.appendChild(span)
  i += res[0].length
  continue
}
```

Token matching order is critical - more specific patterns must be tested before general ones.

### Extending Python Execution
To add Python packages or custom initialization, modify `initPyodide()` function (~line 1441):

```javascript
async function initPyodide() {
  // ... existing initialization ...
  pyodide = await window.loadPyodide()

  // Add custom setup
  await pyodide.loadPackage('numpy')
  await pyodide.runPythonAsync(`
    import sys
    # Custom initialization
  `)

  return pyodide
}
```

### Auto-completion and Auto-indentation
Handled in `handlePythonTyping` function (~line 1647) which listens for Tab (completion) and Enter (indentation) keys. Keywords are hardcoded in an array. Indentation adds 4 spaces after `:` and maintains current indentation level.

### Editor Undo/Redo
The Editor maintains history as `{html, pos}` snapshots. Undo (Ctrl+Z) and redo (Ctrl+Shift+Z) traverse this history. The `save()` and `restore()` methods manage cursor position using `TreeWalker` and character offsets.

## File Organization

```
PyShare/
├── index.html          # Main application (HTML + CSS + JS)
├── qr.html            # QR code generator page
├── tests.html         # Automated test suite
├── sw.js              # Service worker for PWA support
├── manifest.json      # PWA manifest
├── 404.html           # Custom 404 page
├── favicon.ico/png    # Favicons
├── icon.png           # App icon
├── README.md          # User documentation
├── TESTING.md         # Manual testing procedures
└── TEST_SUITE_README.md  # Test suite documentation
```

## Browser Compatibility

Requires modern browser with:
- CompressionStream/DecompressionStream API (for URL compression)
- Pyodide/WebAssembly support (for Python execution)
- localStorage
- contenteditable
- CSS custom properties

Tested on latest Chrome, Firefox, Safari, and Edge.

## Common Pitfalls

1. **Don't use `/g` flag with regex in `parsePython`**: Use `/y` (sticky) flag instead for position-based matching
2. **Match order matters**: In `parsePython`, strings and comments must be matched before keywords to avoid false matches
3. **Pyodide is async**: All Python execution functions must be async and await Pyodide initialization
4. **Editor uses `textContent` not `innerHTML`**: Raw text is extracted via `element.textContent`, syntax highlighting creates new DOM
5. **Cursor position management**: After syntax highlighting, cursor position must be manually restored using `restore(pos)`

## Additional Resources

- Pyodide docs: https://pyodide.org/en/stable/
- Python regex reference: https://docs.python.org/3/library/re.html
- Compression Streams API: https://developer.mozilla.org/en-US/docs/Web/API/Compression_Streams_API
