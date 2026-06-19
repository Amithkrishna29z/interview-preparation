# Vim & Nano — Survival Notes

> **Scope note (junior job prep):** Editor mastery isn't an interview topic — but you may need to edit a config/log file over SSH on a server. This file is trimmed to **survival essentials**. The full guide (all Vim modes, `.vimrc` config, multi-file editing, advanced motions) remains in git history.

---

## Nano (the easy one — prefer it if available)

Nano shows shortcuts at the bottom of the screen. `^` means `Ctrl`.

```
nano file.txt      # open (or create) a file
# ...type normally to edit...
Ctrl + O           # save (Write Out) → press Enter to confirm
Ctrl + X           # exit
Ctrl + W           # search
Ctrl + K           # cut current line     Ctrl + U  paste
```

## Vim (survival — the key skill is *getting out*)

Vim has **modes**. You start in **Normal** mode (keys are commands, not text).

```
vim file.txt       # open

i                  # enter INSERT mode (now you can type)
Esc                # back to NORMAL mode

# In NORMAL mode:
:w                 # save (write)
:q                 # quit
:wq   (or  ZZ)     # save and quit
:q!                # quit WITHOUT saving (discard changes)  ← the famous escape hatch

# Basic navigation (Normal mode): h ← j ↓ k ↑ l →   (arrow keys also work)
dd                 # delete current line
u                  # undo          Ctrl+r  redo
/text              # search for "text"  (n = next match)
```

**The #1 thing to remember:** stuck in Vim? Press `Esc`, then type `:q!` and Enter to quit without saving.

| | Nano | Vim |
|---|---|---|
| Learning curve | Easy (shortcuts shown) | Steep (modal) |
| Best for | Quick edits | Power editing once learned |

---

*Trimmed to survival level for junior job prep. Restore the full Vim/Nano guide from version control if you want to go deeper.*
