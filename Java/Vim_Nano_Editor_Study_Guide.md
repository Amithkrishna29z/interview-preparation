# Vim & Nano Editor — Interview Preparation Study Guide

> A guide covering both editors: concepts, commands, modes, and interview questions. Essential for Linux/DevOps/Backend roles.

---

## Table of Contents
1. [Why Editors Matter in Interviews](#why-editors-matter-in-interviews)
2. [Nano Editor — Complete Guide](#nano-editor--complete-guide)
   - [What is Nano?](#what-is-nano)
   - [Opening and Saving Files](#opening-and-saving-files)
   - [Navigation in Nano](#navigation-in-nano)
   - [Editing in Nano](#editing-in-nano)
   - [Search and Replace](#search-and-replace-nano)
   - [Nano Shortcuts Reference](#nano-shortcuts-reference)
3. [Vim Editor — Complete Guide](#vim-editor--complete-guide)
   - [What is Vim?](#what-is-vim)
   - [Vim Modes — The Core Concept](#vim-modes--the-core-concept)
   - [Opening and Exiting Vim](#opening-and-exiting-vim)
   - [Normal Mode Commands](#normal-mode-commands)
   - [Insert Mode](#insert-mode)
   - [Visual Mode](#visual-mode)
   - [Command-Line Mode](#command-line-mode)
   - [Editing Operations](#editing-operations)
   - [Search and Replace in Vim](#search-and-replace-in-vim)
   - [Working with Multiple Files](#working-with-multiple-files)
   - [Vim Configuration (.vimrc)](#vim-configuration-vimrc)
4. [Vim vs Nano — Comparison](#vim-vs-nano--comparison)
5. [Common Interview Questions](#common-interview-questions)
6. [Quick Revision Cheat Sheet](#quick-revision-cheat-sheet)

---

## Why Editors Matter in Interviews

In Linux/DevOps/Backend/SRE interviews, you are expected to:
- Edit configuration files on a remote server (no GUI)
- Fix a bug directly on a production-like system
- Know the difference between terminal editors and when to use each

**Common scenarios:** SSH into a server to edit `/etc/nginx/nginx.conf`, modify a Dockerfile on a bastion host, or fix a cron job with `crontab -e`.

---

## Nano Editor — Complete Guide

### What is Nano?

Nano is a simple, beginner-friendly terminal text editor. No modes — you open a file and start typing immediately. Shortcut hints are shown at the bottom (`^` means `Ctrl`).

- Comes pre-installed on most Linux/macOS systems
- Ideal for quick config file edits

---

### Opening and Saving Files

```bash
nano filename.txt           # Open or create a file
nano +15 filename.txt       # Open at specific line
nano -v filename.txt        # Read-only mode
nano -l filename.txt        # Show line numbers
```

**Inside nano — saving and exiting:**

| Action | Shortcut |
|--------|----------|
| Save (Write Out) | `Ctrl + O` then `Enter` |
| Exit nano | `Ctrl + X` |
| Save and exit | `Ctrl + O` → `Enter` → `Ctrl + X` |
| Discard changes and exit | `Ctrl + X` → `N` |

---

### Navigation in Nano

```
Arrow Keys         → Move cursor
Ctrl + A           → Beginning of line
Ctrl + E           → End of line
Ctrl + Y           → Page Up
Ctrl + V           → Page Down
Ctrl + _           → Go to specific line number
Alt + \            → Beginning of file
Alt + /            → End of file
```

---

### Editing in Nano

```
Ctrl + K           → Cut current line
Alt + 6            → Copy current line
Ctrl + U           → Paste
Alt + U            → Undo
Alt + E            → Redo
Backspace/Delete   → Delete character
```

**Tip:** `Ctrl + K` cuts the line — press multiple times to cut multiple lines, then `Ctrl + U` pastes them all.

---

### Search and Replace (Nano)

```
Ctrl + W           → Search
Alt + W            → Search backwards
Ctrl + \           → Search and replace
  → Type search term → Enter → type replacement → Enter
  → Y to replace one, A to replace All, N to skip
```

---

### Nano Shortcuts Reference

| Category | Shortcut | Action |
|----------|----------|--------|
| **File** | `Ctrl + O` | Save file |
| **File** | `Ctrl + X` | Exit |
| **Navigate** | `Ctrl + A` | Start of line |
| **Navigate** | `Ctrl + E` | End of line |
| **Navigate** | `Ctrl + Y / V` | Page up / down |
| **Navigate** | `Ctrl + _` | Go to line number |
| **Edit** | `Ctrl + K` | Cut line |
| **Edit** | `Alt + 6` | Copy line |
| **Edit** | `Ctrl + U` | Paste |
| **Edit** | `Alt + U / E` | Undo / Redo |
| **Search** | `Ctrl + W` | Search |
| **Search** | `Ctrl + \` | Search and replace |
| **Help** | `Ctrl + G` | Open help |

---

## Vim Editor — Complete Guide

### What is Vim?

Vim (Vi IMproved) is a powerful, keyboard-driven modal text editor. It has a steep learning curve but is extremely fast once mastered. Pre-installed on virtually all Unix/Linux/macOS systems.

- `vi` is the original; `vim` is Vi IMproved with more features
- `vi` is usually a symlink to `vim` on modern systems

---

### Vim Modes — The Core Concept

Vim is a **modal editor** — the same key does different things in different modes.

| Mode | How to Enter | Purpose |
|------|-------------|---------|
| **Normal** | `Esc` (default mode) | Navigate, delete, copy, paste |
| **Insert** | `i`, `a`, `o`, etc. | Type and edit text |
| **Visual** | `v`, `V`, `Ctrl+V` | Select text for operations |
| **Command-Line** | `:` from Normal mode | Save, quit, search/replace, settings |
| **Replace** | `R` from Normal mode | Overwrite characters |

**Key rule:** Press `Esc` to return to Normal mode from any other mode. When stuck, press `Esc` repeatedly.

---

### Opening and Exiting Vim

```bash
vim filename.txt        # Open a file
vim +25 filename.txt    # Open at specific line
vim -R filename.txt     # Read-only mode
```

**Exiting vim:**

```vim
:q          → Quit (no unsaved changes)
:q!         → Force quit WITHOUT saving
:w          → Save without quitting
:wq         → Save and quit
:x          → Save and quit (only writes if changed)
ZZ          → Save and quit (Normal mode shortcut)
ZQ          → Quit without saving (Normal mode shortcut)
```

**Stuck in vim?** Press `Esc`, then type `:q!` and `Enter`.

---

### Normal Mode Commands

**Cursor movement:**
```
h / l       → Left / Right
j / k       → Down / Up
w / b       → Next / prev word
0 / $       → Start / end of line
gg / G      → First / last line of file
:n          → Go to line number n
Ctrl+F/B    → Page forward / backward
%           → Jump to matching bracket
```

---

### Insert Mode

Press from Normal mode to enter Insert mode:

```
i / I       → Insert before cursor / beginning of line
a / A       → Append after cursor / end of line
o / O       → New line below / above and insert
r           → Replace single character (stays Normal)
R           → Enter Replace mode
```

---

### Visual Mode

```
v           → Character-wise selection
V           → Line-wise selection
Ctrl+V      → Block/column-wise selection
```

**After selecting:**
```
d           → Delete selection
y           → Yank (copy) selection
c           → Change (delete + Insert mode)
> / <       → Indent right / left
~           → Toggle case
```

**Example — comment multiple lines:**
1. Press `Ctrl+V`, select lines with `j`
2. Press `I`, type `# `, press `Esc`

---

### Command-Line Mode

Enter by pressing `:` from Normal mode.

**File operations:**
```vim
:w              → Save
:w filename     → Save as new file
:q / :q!        → Quit / Force quit
:wq             → Save and quit
:e filename     → Open another file
:e!             → Reload from disk (discard changes)
:r filename     → Insert file contents below cursor
```

**Settings (temporary):**
```vim
:set number         → Show line numbers
:set hlsearch       → Highlight search results
:set ignorecase     → Case-insensitive search
:set tabstop=4      → Tab width to 4 spaces
:set expandtab      → Spaces instead of tabs
:syntax on          → Syntax highlighting
```

---

### Editing Operations

**Delete:**
```
x           → Delete character under cursor
dd          → Cut entire line
D           → Delete to end of line
dw          → Delete to start of next word
5dd         → Delete 5 lines
```

**Yank (copy) and Paste:**
```
yy / Y      → Copy current line
yw / y$     → Copy word / to end of line
5yy         → Copy 5 lines
p / P       → Paste after / before cursor
```

**Change:**
```
cw          → Change word
cc / S      → Change entire line
C / c$      → Change to end of line
```

**Undo/Redo/Repeat:**
```
u           → Undo
U           → Undo all changes on current line
Ctrl+R      → Redo
.           → Repeat last change (very powerful)
```

**Indentation:**
```
>> / <<     → Indent line right / left
gg=G        → Auto-indent entire file
```

---

### Search and Replace in Vim

**Search:**
```
/pattern    → Search forward
?pattern    → Search backward
n / N       → Next / prev match
* / #       → Search word under cursor (forward / backward)
:noh        → Clear search highlights
```

**Substitute:**
```vim
:s/old/new/g        → Replace all on current line
:%s/old/new/g       → Replace all in entire file
:%s/old/new/gc      → Replace all with confirmation
:5,10s/old/new/g    → Replace on lines 5–10
```

| Flag | Meaning |
|------|---------|
| `g` | Global — all occurrences on the line |
| `c` | Confirm before each replacement |
| `i` | Case-insensitive |

---

### Working with Multiple Files

**Buffers:**
```vim
:ls             → List all buffers
:bn / :bp       → Next / previous buffer
:bd             → Close current buffer
```

**Split windows:**
```vim
:sp filename    → Horizontal split
:vsp filename   → Vertical split
Ctrl+W h/j/k/l  → Move between splits
Ctrl+W q        → Close current split
```

**Tabs:**
```vim
:tabnew file    → Open in new tab
gt / gT         → Next / previous tab
```

---

### Vim Configuration (.vimrc)

```vim
" ~/.vimrc — recommended junior setup

set number              " Line numbers
set tabstop=4           " Tab = 4 spaces
set shiftwidth=4        " Indent size
set expandtab           " Spaces instead of tabs
set autoindent          " Auto-indent new lines
set hlsearch            " Highlight search results
set incsearch           " Incremental search
set ignorecase          " Case-insensitive search
set smartcase           " Case-sensitive if uppercase used
set scrolloff=8         " Keep 8 lines above/below cursor
syntax on               " Syntax highlighting
```

---

## Vim vs Nano — Comparison

| Feature | Vim | Nano |
|---------|-----|------|
| **Learning curve** | Steep | Very easy |
| **Mode system** | Yes (Normal, Insert, Visual, etc.) | No modes |
| **Speed (once learned)** | Extremely fast | Moderate |
| **Customizability** | Highly configurable (.vimrc) | Limited |
| **Available on systems** | Almost all Unix/Linux | Most Linux distros |
| **Shortcuts shown on screen** | No | Yes (bottom) |
| **Multiple windows/tabs** | Yes | No |
| **Macro support** | Yes | No |
| **Best for** | Power users, developers, sysadmins | Beginners, quick edits |
| **Config file** | `~/.vimrc` | `~/.nanorc` |

**When to use which:**
- **Nano:** Quick config edit, beginner-friendly, need it done in 30 seconds
- **Vim:** Writing code on a remote server, complex edits, minimal container with only `vi`

---

## Common Interview Questions

### Q1: What is the difference between vi and vim?
- `vi` — original Unix text editor (1976)
- `vim` — Vi IMproved; adds syntax highlighting, multi-level undo, plugins, splits, mouse support
- On modern Linux, `vi` is usually an alias/symlink to `vim`

---

### Q2: How do you exit vim without saving?
```vim
:q!     → Force quit, discard all changes
ZQ      → Same as :q! (Normal mode shortcut)
```
Press `Esc` first to ensure Normal mode, then `:q!` + `Enter`.

---

### Q3: Explain the different modes in Vim.

| Mode | Enter With | Purpose |
|------|-----------|---------|
| Normal | `Esc` | Navigate, delete, copy — default |
| Insert | `i`, `a`, `o` | Type and edit text |
| Visual | `v`, `V`, `Ctrl+V` | Select text for bulk operations |
| Command-Line | `:` | Save, quit, search/replace |
| Replace | `R` | Overwrite characters |

---

### Q4: How do you search and replace in Vim?
```vim
:%s/search_term/replacement/g       # Replace all in file
:%s/old/new/gc                      # Replace with confirmation
:s/old/new/g                        # Replace on current line only
```

---

### Q5: How do you copy and paste in Vim?
- `yy` — yank (copy) current line
- `p` / `P` — paste after / before cursor
- System clipboard: `"+y` to yank, `"+p` to paste

---

### Q6: What does the `.` command do?
Repeats the last change. Example: delete a word with `dw`, then press `.` to delete the next word — avoids repeating complex operations.

---

### Q7: How do you open a file as root in Vim?
```bash
sudo vim /etc/nginx/nginx.conf          # Method 1: open with sudo
```
```vim
:w !sudo tee %                          # Method 2: if you forgot sudo
```

---

### Q8: How do you undo/redo in Vim?
```
u           → Undo
U           → Undo all on current line
Ctrl+R      → Redo
```

---

### Q9: What is the difference between `dd` and `D`?
- `dd` — deletes the entire line (including newline)
- `D` — deletes from cursor to end of line; the line itself remains

---

### Q10: How do you indent multiple lines?
```
V → select lines → > (indent right) or < (indent left)
>>      → Indent current line (Normal mode)
gg=G    → Auto-indent entire file
```

---

## Quick Revision Cheat Sheet

### Nano — Essential Shortcuts
```
Open/Exit:      nano file | Ctrl+X
Save:           Ctrl+O → Enter
Cut/Copy/Paste: Ctrl+K | Alt+6 | Ctrl+U
Search:         Ctrl+W
Replace:        Ctrl+\
Go to line:     Ctrl+_
Undo/Redo:      Alt+U | Alt+E
Page Up/Down:   Ctrl+Y | Ctrl+V
Start/End line: Ctrl+A | Ctrl+E
```

### Vim — Essential Commands
```
MODES:
  Normal (default) ← Esc
  Insert           → i / a / o
  Visual           → v / V / Ctrl+V
  Command          → :

EXIT:
  :q!   Quit no save   |   :wq  Save & quit
  ZZ    Save & quit    |   ZQ   Quit no save

NAVIGATE:
  h j k l    ← ↓ ↑ →        gg / G    Start / End of file
  w / b      Next / prev word    0 / $    Start / End of line
  Ctrl+F/B   Page down/up        %        Jump to bracket

EDIT:
  i/a/o      Insert before/after/new line below
  dd / yy    Cut / Copy line
  p / P      Paste after / before
  u / Ctrl+R Undo / Redo
  .          Repeat last change
  x          Delete character
  dw / cw    Delete/Change word
  D / C      Delete/Change to end of line

SEARCH/REPLACE:
  /pattern       Search forward     n / N    Next / prev match
  :%s/a/b/g      Replace all in file
  :%s/a/b/gc     Replace with confirm

SETTINGS:
  :set number     Line numbers
  :set hlsearch   Highlight search
  :syntax on      Syntax highlight
```

### Common Mistakes

| Mistake | Solution |
|---------|----------|
| Typing in Normal mode — weird behavior | Press `Esc`, enter Insert with `i` |
| Can't exit vim | Press `Esc`, type `:q!`, Enter |
| Changes not saved | Use `:w` or `:wq` |
| Accidentally deleted text | Press `u` to undo |
| Search results highlighted | Type `:noh` |
| Nano shortcut not working | `^` means `Ctrl`, not caret |

---

## Interview Tips

1. **Always start with `Esc`** — press it first to ensure Normal mode
2. **Know the 3 exits:** `:q!`, `:wq`, `ZZ`
3. **Memorize:** `:%s/old/new/g` — comes up constantly
4. **`.` (dot)** is a frequently asked "hidden gem" — repeat last change
5. **vimrc knowledge** shows real usage — mention `set number`, `set expandtab`, `syntax on`
6. **For Linux/DevOps roles**, knowing `:w !sudo tee %` shows real-world experience

---

*Last updated: 2026-06-18 | Focus: Linux/DevOps/Backend Developer Interviews*
