---
layout: post
title: "The Next Chapter in File Organization: Introducing Reny"
date: 2026-08-06
categories: articles
tags: [Reny, CLI Tools, Python, File Management]
---

Remember our [previous look into organizing files](https://akpw.github.io/articles/2025/09/22/Print-and-Organize.html)? Back then, the work was done by `renamer`, a component buried deep inside the `batchmp` multimedia suite. It was useful for tidying up messy download folders, generating virtual views, and wrangling chronological chaos.

But times change, and tools evolve.

`batchmp` remains a powerhouse for media processing, yet the core file management logic inside `renamer` proved far too universally useful to stay locked inside a heavy multimedia package. So it was spun off into a dedicated, standalone tool.

Behold `reny`!

`reny` takes the familiar `renamer` functionality and repackages it as a lightweight CLI application. Free from the `ffmpeg` dependencies of its predecessor, it is easier to install and far better suited to everyday file management.

The original feature set is all still there, along with a number of additions: global and local configs, contextual ignore files that cut down on long strings of command line options, Git integration, and a handful of quality-of-life touches earned from daily use.

Here is a brief tour of what the standalone tool brings to the table.

## Files Visualization

Before renaming or moving anything, it helps to see exactly what you are dealing with. Run `reny` with no arguments and it prints the current directory:

```bash
$ reny
```

```text
~/Dev/reny/demo_data
  |-/downloads
  |-/episodes
  |-/nested
  |-/photos
0 files, 4 folders
```

### Recursion Control
`reny` stays at the top level unless told otherwise. `-r` displays recursively, `-el` (end level) sets how deep to descend, and `-sl` (start level) skips levels at the top:

```bash
$ reny -el 1
```

```text
~/Dev/reny/demo_data
  |->/downloads
    |- receipt.pdf
    |- screen_recording.mov
    |- screenshot.png
    |- tax_return.pdf
    |- vacation_movie.mp4
  |->/episodes
    |- ep1.mp4
    |- ep10.mp4
    |- ep2.mp4
  |->/nested
    |-/folder1
    |-/folder2
  |->/photos
    |- IMG_2023.jpg
    |- IMG_2024.jpg
10 files, 6 folders
```

### Sizes & Sorting
`-ss` adds sizes, aggregated per folder and totalled at the end, and `-s` picks the sort order: `na` / `nd` by name, `sa` / `sd` by size:

```bash
$ reny -el 1 -ss -s sd
```

```text
~/Dev/reny
  |-  9KB README.md
  |-  1KB pyproject.toml
  |-  1KB LICENSE
  |->/ 19.3MB tests
    |-  0KB conftest.py
    |-  0KB __init__.py
    |-/ 19.2MB fs
    |-/ 45KB commons
    |-/ 20KB base
    |-/ 1KB __pycache__
  |->/ 688KB reny
    |-  0KB __init__.py
    |-/ 363KB fstools
    |-/ 192KB cli
    |-/ 126KB commons
    |-/ 0KB __pycache__
  |->/ 97KB dist
    |-  53KB reny-1.0.10-py3-none-any.whl
    |-  44KB reny-1.0.10.tar.gz
14 files, 17 folders
Total selected entries size: 20.1MB
```

Because folder sizes roll up, a single glance is usually enough to find whatever is quietly eating your disk.

### Filtering
Filters are what turn a wall of output into something readable. `-in` and `-ex` take Unix-style name patterns separated by `;`:

```bash
$ reny -d reny -el 2 -in '*.py'
```

```text
~/Dev/reny/reny
  |- __init__.py
  |../cli
    |- __init__.py
  |../commons
    |- __init__.py
    |- chainedhandler.py
    |- descriptors.py
    |- progressbar.py
    |- taskprocessor.py
    |- utils.py
  |../fstools
    |- __init__.py
    |- dirtools.py
    |- fsutils.py
    |- rename.py
    |- virtual_organizer.py
    |- walker.py
14 files, 0 folders
```

For media libraries, `-ft` filters by media class rather than by name — `image`, `video`, `audio`, `media`, `nonmedia`, `playable`, `nonplayable`, or `any`:

```bash
$ reny -d demo_data -el 1 -ft image -ad
```

```text
~/Dev/reny/demo_data
  |->/downloads
    |- screenshot.png
  |->/episodes
  |->/nested
    |-/folder1
    |-/folder2
  |->/photos
    |- IMG_2023.jpg
    |- IMG_2024.jpg
3 files, 6 folders
```

Finally, `-ig` reads patterns from an ignore file and appends them to the exclusions, so a repository can keep its noise permanently out of the way:

```bash
$ reny -el 2 -ig .renyignore
```

Any `.gitignore`-style file works here. `reny` also picks up `./.renyignore` from the target directory automatically, falling back to `~/.renyignore`.

## Virtual Views

Organizing usually starts with a question: what would this folder look like if it were tidy? A virtual view answers it by rendering the reorganized structure — here by type, sorted by size descending (`-s sd`), with sizes shown (`-ss`) — *without changing anything on disk*:

```bash
$ reny -b type -s sd -ss
```

```text
Virtual view by type:
~/Downloads
  |->/ 1.3GB mp4
    |-  531.0MB movie_part1.mp4
    |-  442.0MB movie_part2.mp4
    |-  357.0MB presentation.mp4
  |->/ 200.0MB dmg
    |-  200.0MB installer.dmg
  |->/ 150.0MB pdf
    |-  150.0MB documentation.pdf
  |->/ 32.0MB xlsx
    |-  32.0MB spreadsheet.xlsx
  |->/ 5.0MB jpg
    |-  4.0MB high_res_photo.jpg
    |-  1.0MB thumbnail.jpg
  |->/ 2.0MB png
    |-  2.0MB screenshot.png
9 files, 6 folders
Total selected entries size: 1.7GB
```
Sorting by size makes it obvious which file types are eating up the most space. Once the preview looks right, the very same flags handed to `organize` turn it into reality.

## File Organization

### Organize by File Type
To commit the view above and actually sort a mixed directory by type:

```bash
$ reny organize -b type
```

`reny` groups files by extension, creating one subdirectory per type and dropping anything extensionless into `unknown`. As always, it shows the changes it intends to make and asks for confirmation first.

### Organize by Date
Chronological sorting works the same way:

```bash
$ reny organize -b date --date-format "%Y/%m"
```
This relies on standard Python `strftime` formatting to build nested year/month directories.

## Batch Renaming

With everything in its place, the names themselves are usually next. `reny` keeps all the classic batch renaming commands from the original tool, so bulk modifications stay straightforward. Every command is a dry run by default: `reny` visualizes the targeted changes and asks for confirmation before touching anything. Adding `-dc` (display current) puts the current names into that confirmation prompt too, giving you a before-and-after comparison.

### Regex Replace
When filenames are inconsistent across directories, regular expressions clean them up in one pass:

```bash
$ reny replace -fs 'test' -rs 'tst' -dc
```
This finds every occurrence of 'test' in filenames and replaces it with 'tst'.

### Padding Numbers
When files sort awkwardly (`file_1.txt`, `file_10.txt`, `file_2.txt`), the `pad` command restores the expected order by padding numeric sequences with leading zeros:

```bash
$ reny pad -md 2 -dc
```
This forces every number to at least two digits (`file_01.txt`, `file_02.txt`, and so on), fixing file explorer sorting.

### Indexing
To add sequential numbers to a batch of files:

```bash
$ reny index -dc
```
This prefixes each file with a sequential index, which comes in handy for photo albums and structured datasets. Indexing is multi-level by default, restarting the count inside each directory; `-sq` numbers files sequentially across the whole tree instead.

## Developers' Quality-of-Life Features

As a standalone CLI tool, `reny` slots neatly into everyday developer workflows.

### Git Integration
Inside a Git repository, `-g` annotates the view with each entry's status and bubbles modifications up to their parent directories, so it is immediately clear where the changes live:

```bash
$ reny -el 1 -ig .renyignore -g
```

```text
~/Dev/reny
  |- LICENSE
  |- pyproject.toml
  |- README.md [ M]
  |->/demo_data [??]
    |-/downloads
    |-/episodes
    |-/nested
    |-/photos
  |->/reny [* ]
    |-/cli [* ]
    |-/commons
    |-/fstools [* ]
  |->/tests
    |- conftest.py
    |-/base
    |-/commons
    |-/fs
4 files, 13 folders
```

The markers will look familiar: `[ M]` for modified, `[??]` for untracked, and `[* ]` for a directory holding changes somewhere below it. `-go` goes further and drops everything unmodified, which is ideal for scoping a rename to work in progress:

```bash
$ reny -el 2 -ig .renyignore -go
```

```text
  |- README.md [ M]
  |->/demo_data [??]
  |->/reny [* ]
    |->/cli [* ]
      |-/base [* ]
    |->/fstools [* ]
      |- dirtools.py [ M]
2 files, 5 folders
```

Similarly, `-gt` or `--git-tracked` filters the view to show only files already tracked by Git, completely ignoring untracked clutter.

### Configuration & Exclusions
Rather than retyping the same flags, generate a local config file in the current directory:

```bash
$ reny config -l
```
This creates a fully commented `.reny.toml` and opens it in your default editor, where you can set defaults for recursion depth, filtering, sizes, sort order, and more. Without `-l`, the same command manages the global configuration at `~/.config/reny/config.toml`. A local config takes precedence over the global one, and command line flags override both.

To exclude certain files or directories:

```bash
$ reny ignore
```
This creates a local `.renyignore` that behaves like a `.gitignore`; `reny ignore -gl` does the same globally at `~/.renyignore`. Add patterns such as `node_modules/` or `*.tmp`, and `reny` will skip them from then on.

## Getting Started

By shedding its multimedia dependencies, `reny` has become a fast, focused tool for terminal-based file organization.

Ready to get organized? `reny` is available via Homebrew:

```bash
$ brew tap akpw/tap
$ brew install reny
```

Or from PyPI:

```bash
$ pip install reny
```
