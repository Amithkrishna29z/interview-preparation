# Vim & Nano Editor — Interview Preparation Study Guide

> A complete guide covering both editors: concepts, commands, modes, and interview questions. Essential for Linux/DevOps/Backend roles.

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
   - [Navigation in Vim](#navigation-in-vim)
   - [Editing Operations](#editing-operations)
   - [Search and Replace in Vim](#search-and-replace-in-vim)
   - [Working with Multiple Files](#working-with-multiple-files)
   - [Vim Configuration (.vimrc)](#vim-configuration-vimrc)
4. [Vim vs Nano — Comparison](#vim-vs-nano--comparison)
5. [Common Interview Questions](#common-interview-questions)
6. [Quick Revision Cheat Sheet](#quick-revision-cheat-sheet)

---

## Why Editors Matter in Interviews

In Linux/DevOps/Backend/SRE interviews, you are often expected to:
- Edit configuration files on a remote server (no GUI)
- Fix a bug directly on a production-like system
- Write a quick script in a terminal
- Know the difference between terminal editors and when to use each

**Real-world scenarios where this comes up:**
- SSH into a server and edit `/etc/nginx/nginx.conf`
- Modify a Kubernetes config or Dockerfile on a bastion host
- Fix a cron job entry with `crontab -e` (opens vim/nano)
- Edit environment variables in `/etc/environment`

---

## Nano Editor — Complete Guide

### What is Nano?

**Easy Explanation:** Nano is a simple, beginner-friendly terminal text editor. Unlike Vim, it works the way you'd expect — you open a file, start typing immediately, and keyboard shortcuts are shown at the bottom of the screen.

**Key Points:**
- Comes pre-installed on most Linux/macOS systems
- No modes — you start typing right away
- Shortcut hints displayed at the bottom (`^` means `Ctrl`)
- Ideal for quick edits when you don't need Vim's power
- Not suitable for heavy coding but great for config file edits

```bash
# Check if nano is installed
nano --version

# Install nano (if missing)
sudo apt install nano        # Debian/Ubuntu
sudo yum install nano        # CentOS/RHEL
```

---

### Opening and Saving Files

```bash
# Open an existing file or create a new one
nano filename.txt

# Open a file at a specific line number
nano +15 filename.txt

# Open as read-only (view-only mode)
nano -v filename.txt

# Open with line numbers displayed
nano -l filename.txt

# Open with syntax highlighting (if available)
nano -Y python filename.py
```

**Inside nano — saving and exiting:**

| Action | Shortcut |
|--------|----------|
| Save (Write Out) | `Ctrl + O` then `Enter` |
| Exit nano | `Ctrl + X` |
| Save and exit | `Ctrl + O` → `Enter` → `Ctrl + X` |
| Discard changes and exit | `Ctrl + X` → `N` (when prompted) |

**Real-world analogy:** Nano is like Notepad on Windows — open, type, save with `Ctrl+S` (equivalent), close. Very intuitive.

---

### Navigation in Nano

```
Arrow Keys         → Move cursor up/down/left/right
Ctrl + A           → Go to beginning of line (Home)
Ctrl + E           → Go to end of line (End)
Ctrl + Y           → Page Up (scroll up one screen)
Ctrl + V           → Page Down (scroll down one screen)
Ctrl + _           → Go to specific line number (prompts you)
Alt + \            → Go to beginning of file
Alt + /            → Go to end of file
Alt + A            → Set mark (start selection)
Ctrl + ^           → Set mark (alternative)
```

---

### Editing in Nano

```
Backspace          → Delete character before cursor
Delete             → Delete character after cursor
Ctrl + D           → Delete character under cursor
Ctrl + K           → Cut (delete) current line
Alt + 6            → Copy current line
Ctrl + U           → Paste (uncut) the cut content
Ctrl + I           → Insert a tab
Alt + U            → Undo last action
Alt + E            → Redo last undone action
Ctrl + J           → Justify (reformat) current paragraph
```

**Tip:** `Ctrl + K` cuts the line — press it multiple times to cut multiple lines, then `Ctrl + U` pastes them all together.

---

### Search and Replace (Nano)

```
Ctrl + W           → Open search prompt (Where Is)
Ctrl + W → Enter   → Repeat last search (find next)
Alt + W            → Search backwards
Ctrl + \           → Search and replace
  → Type search term → Enter
  → Type replacement → Enter
  → Y to replace one, A to replace All, N to skip
```

**Example — search and replace in nano:**
1. Press `Ctrl + \`
2. Type: `localhost` → Press `Enter`
3. Type: `0.0.0.0` → Press `Enter`
4. Press `A` to replace all occurrences

---

### Nano Shortcuts Reference

| Category | Shortcut | Action |
|----------|----------|--------|
| **File** | `Ctrl + O` | Save file |
| **File** | `Ctrl + X` | Exit |
| **File** | `Ctrl + R` | Read/insert another file |
| **Navigate** | `Ctrl + A` | Start of line |
| **Navigate** | `Ctrl + E` | End of line |
| **Navigate** | `Ctrl + Y` | Page up |
| **Navigate** | `Ctrl + V` | Page down |
| **Navigate** | `Ctrl + _` | Go to line number |
| **Edit** | `Ctrl + K` | Cut line |
| **Edit** | `Alt + 6` | Copy line |
| **Edit** | `Ctrl + U` | Paste |
| **Edit** | `Alt + U` | Undo |
| **Edit** | `Alt + E` | Redo |
| **Search** | `Ctrl + W` | Search |
| **Search** | `Ctrl + \` | Search and replace |
| **Help** | `Ctrl + G` | Open help |

---

## Vim Editor — Complete Guide

### What is Vim?

**Easy Explanation:** Vim (Vi IMproved) is a powerful, keyboard-driven text editor. It has a steep learning curve but makes you extremely fast once mastered. Professional developers and sysadmins often use it for editing files on servers.

**Key Points:**
- Pre-installed on virtually all Unix/Linux/macOS systems as `vi` or `vim`
- Modal editor — different modes for different tasks
- Entirely keyboard-driven — no mouse needed
- Highly configurable via `.vimrc`
- Used heavily in DevOps, SRE, and backend engineering roles
- `vi` is the original; `vim` is Vi IMproved with more features
- `neovim` (nvim) is the modern fork of vim

```bash
# Check version
vim --version
vi --version

# Install vim
sudo apt install vim          # Debian/Ubuntu
sudo yum install vim          # CentOS/RHEL

# Open vim
vim filename.txt
vi filename.txt
```

---

### Vim Modes — The Core Concept

This is the most important concept. Vim is a **modal editor** — the same key does different things in different modes.

```
┌─────────────────────────────────────────────────────────────┐
│                        VIM MODES                            │
│                                                             │
│   NORMAL MODE  ◄──── Esc ────  INSERT MODE                 │
│   (default)                    (editing text)               │
│       │                              ▲                      │
│       │  i, a, o, I, A, O            │                      │
│       └──────────────────────────────┘                      │
│       │                                                     │
│       │  v, V, Ctrl+V           VISUAL MODE                 │
│       └──────────────────────────────────►                  │
│       │                                                     │
│       │  :                   COMMAND-LINE MODE              │
│       └──────────────────────────────────►  (:w, :q, etc.) │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

| Mode | How to Enter | Purpose |
|------|-------------|---------|
| **Normal** | Press `Esc` (always returns here) | Navigate, delete, copy, paste |
| **Insert** | Press `i`, `a`, `o`, etc. | Type and edit text |
| **Visual** | Press `v`, `V`, or `Ctrl+V` | Select text for operations |
| **Command-Line** | Press `:` from Normal mode | Save, quit, search/replace, settings |
| **Replace** | Press `R` from Normal mode | Overwrite characters |

**Key rule:** Press `Esc` to go back to Normal mode from ANY other mode. When stuck, press `Esc` repeatedly.

---

### Opening and Exiting Vim

```bash
# Open a file
vim filename.txt

# Open multiple files
vim file1.txt file2.txt

# Open vim and go to specific line
vim +25 filename.txt

# Open vim in read-only mode
vim -R filename.txt
view filename.txt
```

**Exiting vim (the most famous problem in programming):**

```vim
:q          → Quit (only if no unsaved changes)
:q!         → Force quit WITHOUT saving (discard all changes)
:w          → Save (write) without quitting
:wq         → Save and quit
:x          → Save and quit (only writes if changes were made)
ZZ          → Save and quit (shortcut, no colon needed)
ZQ          → Quit without saving (shortcut)
:wq!        → Force save and quit (even read-only files, if you have permission)
```

**Common scenario:** You accidentally opened vim and don't know how to exit:
1. Press `Esc` (make sure you're in Normal mode)
2. Type `:q!` and press `Enter`

---

### Normal Mode Commands

Normal mode is the default mode. You spend most of your time here navigating and manipulating text.

**Basic cursor movement:**
```
h       → Move left (← arrow)
j       → Move down (↓ arrow)
k       → Move up (↑ arrow)
l       → Move right (→ arrow)
```

**Word movement:**
```
w       → Jump to start of NEXT word
W       → Jump to start of next WORD (ignores punctuation)
e       → Jump to END of current/next word
b       → Jump BACK to start of previous word
B       → Jump back ignoring punctuation
```

**Line movement:**
```
0       → Go to beginning of line (column 0)
^       → Go to first non-blank character of line
$       → Go to end of line
gg      → Go to first line of file
G       → Go to last line of file
:n      → Go to line number n (e.g., :25)
nG      → Go to line number n (e.g., 25G)
Ctrl+F  → Page forward (scroll down one screen)
Ctrl+B  → Page backward (scroll up one screen)
Ctrl+D  → Scroll down half screen
Ctrl+U  → Scroll up half screen
%       → Jump to matching bracket/parenthesis/brace
```

---

### Insert Mode

Press these keys from Normal mode to enter Insert mode:

```
i       → Insert BEFORE cursor
I       → Insert at BEGINNING of line
a       → Append AFTER cursor
A       → Append at END of line
o       → Open new line BELOW and insert
O       → Open new line ABOVE and insert
s       → Delete character under cursor and insert
S       → Delete entire line and insert
cc      → Delete entire line and insert (same as S)
r       → Replace single character (stays in Normal mode)
R       → Enter Replace mode (overwrite characters)
```

**Real-world analogy:**
- `i` = click before a letter to start typing
- `a` = click after a letter to start typing
- `o` = press Enter to create new line below, then type
- `O` = press Enter to create new line above, then type

---

### Visual Mode

Visual mode lets you select text, then operate on it.

```
v           → Start character-wise visual selection
V           → Start line-wise visual selection (whole lines)
Ctrl+V      → Start block/column-wise visual selection
```

**After selecting, apply operations:**
```
d           → Delete selected text
y           → Yank (copy) selected text
c           → Change (delete and enter Insert mode)
>           → Indent right
<           → Indent left
=           → Auto-indent
~           → Toggle case (upper ↔ lower)
:           → Run command on selected lines
```

**Example — comment out multiple lines:**
1. Press `Ctrl+V` (block visual)
2. Select the lines using `j`
3. Press `I` (capital i)
4. Type `# `
5. Press `Esc` — `# ` is inserted on all selected lines

---

### Command-Line Mode

Enter by pressing `:` from Normal mode.

**File operations:**
```vim
:w              → Save file
:w filename     → Save as new file
:w!             → Force save (overwrite read-only)
:r filename     → Insert contents of another file below cursor
:q              → Quit
:q!             → Quit without saving
:wq             → Save and quit
:e filename     → Edit/open another file
:e!             → Reload file from disk (discard changes)
:ls             → List all open buffers
```

**Editor settings (temporary, reset when vim closes):**
```vim
:set number         → Show line numbers
:set nonumber       → Hide line numbers
:set relativenumber → Show relative line numbers
:set hlsearch       → Highlight search results
:set nohlsearch     → Disable search highlighting
:set ignorecase     → Case-insensitive search
:set autoindent     → Auto-indent new lines
:set tabstop=4      → Set tab width to 4 spaces
:set expandtab      → Use spaces instead of tabs
:set wrap           → Enable line wrapping
:set nowrap         → Disable line wrapping
:syntax on          → Enable syntax highlighting
:syntax off         → Disable syntax highlighting
```

---

### Navigation in Vim

**Marks — bookmarks inside a file:**
```vim
ma          → Set mark 'a' at current position
'a          → Jump to line of mark 'a'
`a          → Jump to exact position of mark 'a'
''          → Jump back to previous position
```

**Jumps and history:**
```vim
Ctrl+O      → Go to previous jump location (jump back)
Ctrl+I      → Go to next jump location (jump forward)
```

**Screen positioning:**
```vim
zt          → Scroll so cursor line is at TOP of screen
zz          → Scroll so cursor line is at MIDDLE of screen
zb          → Scroll so cursor line is at BOTTOM of screen
H           → Jump cursor to top of screen
M           → Jump cursor to middle of screen
L           → Jump cursor to bottom of screen
```

---

### Editing Operations

**Delete (also cuts — goes to clipboard):**
```
x           → Delete character under cursor
X           → Delete character before cursor
dd          → Delete (cut) entire current line
D           → Delete from cursor to end of line
dw          → Delete from cursor to start of next word
d$          → Delete from cursor to end of line (same as D)
d0          → Delete from cursor to beginning of line
dgg         → Delete from current line to start of file
dG          → Delete from current line to end of file
5dd         → Delete 5 lines (number prefix works for most commands)
```

**Yank (copy):**
```
yy          → Copy (yank) current line
Y           → Copy current line (same as yy)
yw          → Copy from cursor to end of word
y$          → Copy from cursor to end of line
ygg         → Copy from current line to start of file
yG          → Copy from current line to end of file
5yy         → Copy 5 lines
```

**Paste:**
```
p           → Paste AFTER cursor (or below current line)
P           → Paste BEFORE cursor (or above current line)
```

**Change (delete and enter Insert mode):**
```
cw          → Change (replace) word from cursor
cc          → Change entire line
c$          → Change from cursor to end of line
C           → Change from cursor to end of line (same as c$)
```

**Undo and Redo:**
```
u           → Undo last action
U           → Undo all changes on current line
Ctrl+R      → Redo (undo the undo)
```

**Repeat:**
```
.           → Repeat last change command (very powerful)
```

**Indentation:**
```
>>          → Indent current line right
<<          → Indent current line left
>5j         → Indent next 5 lines right
=G          → Auto-indent from cursor to end of file
gg=G        → Auto-indent entire file
```

**Case:**
```
~           → Toggle case of character under cursor
gUw         → Uppercase from cursor to end of word
guw         → Lowercase from cursor to end of word
gUU         → Uppercase entire line
guu         → Lowercase entire line
```

---

### Search and Replace in Vim

**Basic search:**
```
/pattern        → Search FORWARD for pattern
?pattern        → Search BACKWARD for pattern
n               → Jump to NEXT match (same direction)
N               → Jump to PREVIOUS match (reverse direction)
*               → Search for word under cursor (forward)
#               → Search for word under cursor (backward)
```

**Search and replace (substitute command):**
```vim
:s/old/new/         → Replace FIRST occurrence on current line
:s/old/new/g        → Replace ALL occurrences on current line
:%s/old/new/g       → Replace all occurrences in ENTIRE FILE
:%s/old/new/gc      → Replace all with CONFIRMATION (y/n/a/q)
:5,10s/old/new/g    → Replace on lines 5 to 10
:'<,'>s/old/new/g   → Replace in visual selection (auto-filled)
```

**Flags for substitute:**
| Flag | Meaning |
|------|---------|
| `g` | Global — replace all on the line |
| `c` | Confirm — ask before each replacement |
| `i` | Case-insensitive |
| `I` | Case-sensitive |

**Example — replace all `localhost` with `0.0.0.0` in file:**
```vim
:%s/localhost/0.0.0.0/g
```

**Clear search highlight:**
```vim
:noh            → Clear search highlights (no highlight)
```

---

### Working with Multiple Files

**Buffers (open files in memory):**
```vim
:ls             → List all buffers
:bn             → Switch to next buffer
:bp             → Switch to previous buffer
:b2             → Switch to buffer number 2
:bd             → Delete (close) current buffer
```

**Split windows:**
```vim
:sp filename    → Horizontal split (open file in split)
:vsp filename   → Vertical split
Ctrl+W h        → Move to left split
Ctrl+W j        → Move to split below
Ctrl+W k        → Move to split above
Ctrl+W l        → Move to right split
Ctrl+W q        → Close current split
Ctrl+W =        → Make all splits equal size
```

**Tabs:**
```vim
:tabnew filename    → Open file in new tab
:tabn               → Next tab
:tabp               → Previous tab
:tabclose           → Close current tab
gt                  → Next tab (Normal mode)
gT                  → Previous tab (Normal mode)
```

---

### Vim Configuration (.vimrc)

The `.vimrc` file (in your home directory) holds permanent settings.

```vim
" ~/.vimrc — basic configuration example

" Display
set number              " Show line numbers
set relativenumber      " Show relative line numbers
set ruler               " Show cursor position in status bar
set showcmd             " Show partial commands
set cursorline          " Highlight current line

" Indentation
set tabstop=4           " Tab width = 4 spaces
set shiftwidth=4        " Indent size for >> and <<
set expandtab           " Use spaces instead of tabs
set autoindent          " Copy indent from current line
set smartindent         " Smart auto-indent

" Search
set hlsearch            " Highlight search results
set incsearch           " Incremental search (as you type)
set ignorecase          " Case-insensitive search
set smartcase           " Case-sensitive if uppercase used

" Usability
set mouse=a             " Enable mouse support
set clipboard=unnamedplus  " Use system clipboard
set undofile            " Persistent undo history
set scrolloff=8         " Keep 8 lines above/below cursor
syntax on               " Syntax highlighting
set wildmenu            " Better command-line completion
```

---

## Vim vs Nano — Comparison

| Feature | Vim | Nano |
|---------|-----|------|
| **Learning curve** | Steep | Very easy |
| **Mode system** | Yes (Normal, Insert, Visual, etc.) | No modes |
| **Speed (once learned)** | Extremely fast | Moderate |
| **Customizability** | Highly configurable (.vimrc) | Limited |
| **Syntax highlighting** | Excellent | Basic |
| **Plugin ecosystem** | Massive (vim-plug, etc.) | Minimal |
| **Available on systems** | Almost all Unix/Linux | Most Linux distros |
| **Default on minimal systems** | `vi` is usually available | Not always |
| **Best for** | Power users, developers, sysadmins | Beginners, quick edits |
| **Startup time** | Fast | Very fast |
| **Config file** | `~/.vimrc` | `~/.nanorc` |
| **Keyboard shortcuts shown** | No | Yes (bottom of screen) |
| **Multiple windows/tabs** | Yes | No |
| **Macro support** | Yes (powerful) | No |
| **Regular expressions** | Full support | Basic support |

**When to use which:**

```
Use NANO when:
  - Quick config file edit (e.g., sudo nano /etc/hosts)
  - You're a beginner
  - Someone unfamiliar with vim might also edit the file
  - You need it done in 30 seconds with no learning

Use VIM when:
  - Writing code on a remote server
  - Making complex edits (multi-file, regex replace, macros)
  - You want speed and efficiency
  - The server has only vi/vim available (minimal containers)
```

---

## Common Interview Questions

---

### Q1: What is the difference between vi and vim?

**Answer:**
- `vi` is the original Unix text editor (Visual editor), created in 1976 by Bill Joy
- `vim` is Vi IMproved — a superset of vi with additional features
- Vim adds syntax highlighting, multi-level undo, plugins, split windows, mouse support, and more
- On most modern Linux systems, `vi` is actually an alias or symlink to `vim`
- You can check with: `which vi` and `readlink -f $(which vi)`

---

### Q2: How do you exit vim without saving changes?

**Answer:**
```vim
:q!     → Force quit, discard all changes
ZQ      → Same as :q! (shortcut without colon)
```
First press `Esc` to ensure you're in Normal mode, then type `:q!` and press `Enter`.

---

### Q3: Explain the different modes in Vim.

**Answer:**

| Mode | Enter With | Purpose |
|------|-----------|---------|
| Normal | `Esc` | Navigate, delete, copy, paste — default mode |
| Insert | `i`, `a`, `o`, `I`, `A`, `O` | Type and edit text |
| Visual | `v`, `V`, `Ctrl+V` | Select text for bulk operations |
| Command-Line | `:` | Save, quit, search/replace, settings |
| Replace | `R` | Overwrite existing text character by character |

---

### Q4: How do you search and replace text in Vim?

**Answer:**
```vim
:%s/search_term/replacement/g
```
- `%` means the entire file
- `s` is substitute
- `g` means globally (all occurrences on each line)
- Add `c` flag for confirmation: `:%s/old/new/gc`

For just the current line: `:s/old/new/g`

---

### Q5: How do you copy and paste in Vim?

**Answer:**
- Vim uses "yank" for copy and "put" for paste
- `yy` — yank (copy) current line
- `y3j` — yank current line and 3 lines below (4 total)
- `p` — paste after cursor
- `P` — paste before cursor
- For system clipboard: `"+y` to yank and `"+p` to paste

---

### Q6: What does the `.` command do in Vim?

**Answer:**
The `.` (dot) command repeats the last change. This is one of vim's most powerful features.

**Example:**
- You delete a word with `dw`
- Press `.` to delete the next word
- Press `.` again to delete another word
- This avoids repeating complex operations manually

---

### Q7: How do you enable line numbers in Vim?

**Answer:**
```vim
:set number         → Enable absolute line numbers
:set relativenumber → Enable relative line numbers (distance from current line)
:set nonumber       → Disable line numbers
```
To make it permanent, add `set number` to `~/.vimrc`.

---

### Q8: How do you open and edit a file as root/sudo in Vim?

**Answer:**
```bash
# Method 1: Open with sudo
sudo vim /etc/nginx/nginx.conf

# Method 2: If you forgot sudo, write from inside vim
:w !sudo tee %
# Explanation: writes current buffer through `sudo tee` to the file (%)
```

---

### Q9: How do you undo and redo in Vim?

**Answer:**
```
u           → Undo last change
U           → Undo all changes on current line
Ctrl+R      → Redo (undo the undo)
```
Vim keeps a full undo tree — you can undo multiple levels and even undo after saving.

---

### Q10: What is the difference between `dd` and `D` in Vim?

**Answer:**
- `dd` — deletes (cuts) the **entire current line**, including the newline character
- `D` — deletes from the **cursor position to the end of the line**, the line itself remains

---

### Q11: How do you navigate between multiple open files in Vim?

**Answer:**
- **Buffers:** `:ls` to list, `:bn`/`:bp` to switch next/previous
- **Split windows:** `:sp file` (horizontal) or `:vsp file` (vertical), `Ctrl+W` + direction to move between splits
- **Tabs:** `:tabnew file` to open, `gt`/`gT` to switch tabs

---

### Q12: How do you record and play a macro in Vim?

**Answer:**
```
qa          → Start recording macro into register 'a'
(do your edits)
q           → Stop recording
@a          → Play back macro 'a'
@@          → Repeat last played macro
10@a        → Run macro 'a' 10 times
```
**Use case:** Apply the same set of edits to 50 lines — record once, run on all.

---

### Q13: What is the difference between `Ctrl+W` and `q` when using split windows?

**Answer:**
- `Ctrl+W q` — closes the current split/window
- `:q` — closes the current buffer; if it's the last window, exits vim
- `Ctrl+W o` — closes ALL other splits (only keeps current one)

---

### Q14: How do you indent multiple lines at once in Vim?

**Answer:**
```
Visual mode → select lines with V → > to indent right, < to indent left
>>          → Indent current line right (Normal mode)
5>>         → Indent next 5 lines right
gg=G        → Auto-indent entire file
```

---

### Q15: How do you find the line number of the current cursor position?

**Answer:**
```vim
Ctrl+G          → Shows file name and current line/total lines
:set number     → Display line numbers in gutter
:.=             → Print current line number in command area
```

---

## Quick Revision Cheat Sheet

### Nano — Essential Shortcuts

```
Open/Exit:     nano file | Ctrl+X
Save:          Ctrl+O → Enter
Cut/Copy/Paste: Ctrl+K | Alt+6 | Ctrl+U
Search:        Ctrl+W
Replace:       Ctrl+\
Go to line:    Ctrl+_
Undo/Redo:     Alt+U | Alt+E
Page Up/Down:  Ctrl+Y | Ctrl+V
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
  /pattern    Search forward      n / N     Next / prev match
  :%s/a/b/g  Replace all in file
  :%s/a/b/gc Replace with confirm

SETTINGS:
  :set number     Line numbers
  :set hlsearch   Highlight search
  :syntax on      Syntax highlight
```

---

## Common Mistakes and Pitfalls

| Mistake | Solution |
|---------|----------|
| Typing in Normal mode and getting weird behavior | Press `Esc`, then enter Insert mode with `i` |
| Can't exit vim | Press `Esc`, then type `:q!` and Enter |
| Changes not saved | Use `:w` to save, or `:wq` to save and quit |
| Accidentally deleted text | Press `u` to undo |
| Pasted wrong content | Press `u` to undo, use the correct yank |
| Search results still highlighted | Type `:noh` to clear |
| Nano shortcut not working | Ensure `^` means `Ctrl`, not caret symbol |
| vim vs vi confusion | `vi` is usually vim on modern systems; check `vi --version` |

---

## Interview Tips

1. **Always start with "Esc"** — if asked to demo vim in an interview, press Esc first to ensure you're in Normal mode
2. **Know the 3 most important exits:** `:q!`, `:wq`, and `ZZ`
3. **Nano is safer for interviews** if you're not confident with vim — no modes to get confused about
4. **Memorize the substitute command:** `:%s/old/new/g` — comes up very often
5. **`.` (dot)** is a frequently asked "hidden gem" of vim — repeat last change
6. **Macros** (`qa`, edit, `q`, `@a`) show advanced knowledge and impress interviewers
7. **Know the difference** between `d` (delete/cut) and nothing going to system clipboard vs `"+y` for system clipboard
8. **vimrc knowledge** shows you use vim regularly — mention `set number`, `set expandtab`, `syntax on`
9. **For Linux/DevOps roles**, knowing `sudo vim` vs `:w !sudo tee %` shows real-world experience
10. **Practice daily** — use vim for even 15 minutes a day and it becomes muscle memory within 2 weeks

---

*Last updated: 2026-06-05 | Focus: Linux/DevOps/Backend Developer Interviews*
