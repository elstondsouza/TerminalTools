# Terminal Tools Quick Start Guide for Bioinformaticians

A practical, example-driven guide to essential terminal tools — this is the shell
environment you *wish* your HPC cluster came with.

---

## 1. iTerm2

macOS terminal emulator with splits, search, and profiles.

### Killer features

| Feature | How |
|---|---|
| Split panes | `Cmd+D` (vertical), `Cmd+Shift+D` (horizontal) |
| Search / regex | `Cmd+F`, then `Tab` to toggle regex |
| Hotkey window | Preferences → Keys → "A single hotkey opens a dedicated window" |
| tmux integration | Preferences → General → tmux — lets you use native windows |
| Profile badges | Preferences → Profiles → Badge — show `hostname` or `$USER` |
| Mark & jump | `Cmd+Shift+M` to mark; `Cmd+Shift+Up/Down` to jump between marks |

### Bioinformatics tip

Send a long job to a hotkey window that you summon with `Ctrl+Space` — never
lose your scrollback buffer.

---

## 2. Zsh (Z shell)

Default macOS shell since Catalina. Better completion, globbing, and plugin
ecosystem.

### Getting started

```zsh
# Make it your default shell (if not already)
chsh -s /bin/zsh

# Install Oh-My-Zsh (popular plugin framework)
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

### Gems for bioinformatics

```zsh
# Recursive glob: match across directories
ls **/*.fastq.gz

# Exclude glob: everything except R2 reads
ls *_R1.fastq.gz  ~(*_R2*)

# Autocorrection: type `samtols` → "did you mean: samtools?"
setopt CORRECT

# Suffix alias: make .bam files clickable from the shell
alias -s bam='samtools view'
```

### Recommended plugins

Add to `~/.zshrc`:
```zsh
plugins=(git zsh-autosuggestions zsh-syntax-highlighting)
```

---

## 3. fzf — Fuzzy Finder

Blazing-fast fuzzy search for files, command history, and anything pipeable.

### Install

```bash
brew install fzf
$(brew --prefix)/opt/fzf/install
```

### Daily use

```bash
# File search — hit Ctrl+T anywhere
vim **<Tab>                  # fzf completes paths inline
Ctrl+T                       # insert selected path at cursor

# History search — much better than raw Ctrl+R
Ctrl+R                       # fuzzy-search every command you've ever run

# Kill processes by name
kill **<Tab>

# cd into any subdirectory — no more `cd ../../..`
Ctrl+Alt+C                   # or use alt-c
```

### Pipe anything into fzf

```bash
# Interactively pick files from a listing
ls *.fastq.gz | fzf --multi --preview 'seqkit stats {}'

# Run a command on the selection
ls *.fastq.gz | fzf --multi | xargs -P 4 fastqc

# SSH into any host from your config
cat ~/.ssh/config | grep -i hostname | awk '{print $2}' | fzf | xargs ssh
```

---

## 4. tmux / byobu — Terminal Multiplexer

Keep sessions alive when your laptop goes to sleep or your SSH drops.

### Install

```bash
brew install tmux byobu     # byobu = tmux with training wheels
```

### tmux essentials

```bash
# Create a named session
tmux new -s rnaseq

# Detach: Ctrl+b then d
# List sessions
tmux ls

# Re-attach
tmux attach -t rnaseq
```

| Shortcut | Action |
|---|---|
| `Ctrl+b` `%` | Split vertical |
| `Ctrl+b` `"` | Split horizontal |
| `Ctrl+b` `c` | New window |
| `Ctrl+b` `1-9` | Switch window |
| `Ctrl+b` `[` | Scroll / copy mode |

### byobu (friendlier)

```bash
byobu                          # start with status bar, key hints
F2                             # new window
F3 / F4                        # navigate windows
Shift+F2                       # split horizontally
Ctrl+F2                        # split vertically
F6                             # detach
byobu -r                       # re-attach
```

### Bioinformatics workflow

```zsh
# Session with 3 windows: jobs, watch, logs
tmux new -s hpc -d
tmux send-keys -t hpc:0 'sbatch align.slurm' Enter
tmux new-window -t hpc -n watch 'watch -n 10 squeue -u $USER'
tmux new-window -t hpc -n logs 'tail -f slurm-*.out'
tmux attach -t hpc
```

---

## 5. GNU Parallel & xargs

Parallelise embarrassingly parallel loops — the bedrock of bioinformatics
pipelines.

### Install

```bash
brew install parallel
```

### The classic loop → parallel transformation

```bash
# Before (serial — wastes CPU on multi-core machines)
for f in *.fastq; do
    fastqc "$f"
done

# After (4 jobs at once)
parallel -j 4 fastqc {} ::: *.fastq
```

### xargs -P (simpler, no install needed)

```bash
ls *.fastq.gz | xargs -P 4 -I {} fastqc {}
```

### GNU Parallel power

```zsh
# Alignment with automatic output naming
parallel -j 8 \
  'bwa mem ref.fa {} > {.}.sam' \
  ::: *_R1.fastq.gz

# {.} = remove extension

# Paired-end: one job per sample
parallel -j 4 \
  'bwa mem ref.fa {1} {2} > {1/.}.sam' \
  ::: *R1.fastq.gz \
  ::: *R2.fastq.gz

# Job log and resume
parallel --joblog align.log --resume -j 8 \
  'bwa mem ref.fa {} > {.}.sam' \
  ::: *_R1.fastq.gz

# Transfer files, run remotely, collect results
parallel -S headnode -j 16 \
  --transfer --return {.}.bam \
  'bwa mem ref.fa {} | samtools sort - > {.}.bam' \
  ::: *_R1.fastq.gz
```

### Quick reference

| Pattern | Meaning |
|---|---|
| `{}` | Full filename |
| `{.}` | Filename without extension |
| `{/}` | Basename only (strip path) |
| `{/.}` | Basename without extension |
| `:::  :::` | Cartesian product (paired ends) |
| `-j N` | Run N jobs in parallel |
| `--joblog` | Log exit status (resumable) |

---

## 6. Dotfiles

One repo to rule your `~/.zshrc`, `~/.tmux.conf`, `~/.gitconfig`, aliases.

### Approach: bare repo (no symlinks needed)

```bash
# Initialise a bare repo in ~/.dotfiles
git init --bare ~/.dotfiles
alias dotfiles='git --git-dir=$HOME/.dotfiles --work-tree=$HOME'

# Track your config files
dotfiles add ~/.zshrc ~/.tmux.conf ~/.gitconfig
dotfiles commit -m "Initial dotfiles"
dotfiles remote add origin <your-repo-url>
dotfiles push -u origin main
```

### Minimal starter `.zshrc`

```zsh
# Aliases bioinformaticians actually use
alias ls='ls -Gh'
alias ll='ls -lh'
alias la='ls -la'
alias grep='grep --color=auto'
alias myjobs='squeue -u $USER'
alias qlogin='srun --pty bash'
alias fastqc='fastqc -t 4'
alias cdl='cd ~/labnotebook'
alias seqfu='seqfu'    # tab completion will reward you

# fzf key bindings
[ -f ~/.fzf.zsh ] && source ~/.fzf.zsh

# Conda (if you use it)
# >>> conda initialize >>>
# __conda_setup="$('/opt/homebrew/anaconda3/bin/conda' 'shell.zsh' 'hook' 2> /dev/null)"
# <<< conda initialize <<<
```

### Starship — Cross-shell prompt

Minimal, fast, informative prompt that shows exactly what you need.

```bash
brew install starship
```

Add to the **end** of `~/.zshrc`:
```zsh
eval "$(starship init zsh)"
```

Default starship shows: current directory, Git branch/status, Python/R
virtualenv, command duration if > 5s, and exit code if non-zero.

Customise with `~/.config/starship.toml`:
```toml
# Show only what matters for bioinformatics
format = """
[](#9A348E)\
$directory\
$git_branch\
$git_status\
$python\
$cmd_duration\
$line_break\
$character\
[](#9A348E)\
"""

[nodejs] disabled = true    # hide what you don't use
[ruby]     disabled = true
[rust]     disabled = true
[golang]   disabled = true
```

### Starter `.tmux.conf`

```tmux
# Mouse mode (tmux ≥ 2.1)
set -g mouse on

# Increase scrollback
set -g history-limit 50000

# More intuitive split keys
bind | split-window -h
bind - split-window -v

# Reload config
bind r source-file ~/.tmux.conf \; display "Reloaded"
```

### Where to borrow ideas

- [mathiasbynens/dotfiles](https://github.com/mathiasbynens/dotfiles)
- [holman/dotfiles](https://github.com/holman/dotfiles)
- [thoughtbot/dotfiles](https://github.com/thoughtbot/dotfiles)
- [jbirnick/dotfiles](https://github.com/jbirnick/dotfiles) (HPC-oriented)

---

## 7. jq, mosh, htop / btop

### jq — JSON processor

Slice, filter, and reshape JSON from the command line.

```bash
brew install jq
```

```bash
# Extract fields from a sample sheet
cat samples.json | jq '.[] | {sample_id, condition, read_path}'

# Filter by condition
cat samples.json | jq '.[] | select(.condition == "treated") | .sample_id'

# NCBI E-utilities → parse JSON accession list
curl -s "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi?db=sra&term=RNA-seq+human&retmode=json" \
  | jq '.esearchresult.idlist'

# Count elements
cat samples.json | jq 'length'

# Pretty-print with colours (also works piped)
tail -n 1 workflow.json | jq .
```

### mosh — Mobile shell

SSH that survives sleep, network switches, and IP changes.

```bash
brew install mosh
```

```bash
# Instead of ssh — connects and stays connected
mosh user@cluster.edu

# Specify port (common on HPC)
mosh user@cluster.edu --ssh="ssh -p 2222"

# Resume after your laptop wakes from sleep — no reconnect needed
# Just start typing — mosh handles the rest
```

Pro tip: run `mosh` inside a tmux session for double resilience — when one
layer dies, the other keeps your process alive.

### htop / btop — Resource monitors

See who's eating your cores and memory at a glance.

```bash
brew install htop btop
```

```bash
htop                # process tree (F5), kill (F9), filter (F4)
btop                # prettier: CPU, GPU, RAM, disk, network in one window
```

Use in a tmux pane alongside your job queue:
```bash
# In pane 1: watch jobs
tmux send-keys -t 0 'watch -n 10 squeue -u $USER' Enter

# In pane 2: watch resources
tmux send-keys -t 1 'htop' Enter

# In pane 3: watch logs
tmux send-keys -t 2 'tail -f slurm-*.out' Enter
```

### pandoc — Universal document converter

Convert between Markdown, LaTeX, Word, PDF, HTML, and more — without opening
a GUI.

```bash
brew install pandoc pandoc-crossref
```

```bash
# Lab notebook (Markdown → PDF)
pandoc notebook.md -o notebook.pdf

# Convert Rmd to a Word doc for collaborators
pandoc report.Rmd -o report.docx

# Batch convert all READMEs in a project
for f in **/README.md; do
  pandoc "$f" -o "${f%.md}.html"
done

# Strip formatting: Word → plain Markdown
pandoc manuscript.docx -t markdown -o manuscript.md --wrap=none
```

Bioinformatics context: write your methods section as Markdown, keep it in
your repo alongside the pipeline, then `pandoc methods.md -o methods.docx`
when the journal demands a Word document.

---

## Appendix: One-shot Install

```bash
# Everything via Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

brew install zsh tmux byobu fzf parallel starship
$(brew --prefix)/opt/fzf/install

# Oh-My-Zsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```