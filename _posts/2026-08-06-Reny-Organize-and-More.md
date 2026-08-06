---
layout: post
title: "The Next Generation of File Organization: Introducing Reny"
date: 2026-08-06
categories: articles
tags: [Reny, CLI Tools, Python, File Management]
---

Do you remember our [previous look into organizing files](https://akpw.github.io/articles/2025/09/22/Print-and-Organize.html)? At the time, it was with the `renamer` component embedded deep within the `batchmp` multimedia suite. It was useful for organizing messy download folders, generating virtual views, and wrangling chronological chaos.

But times change, and tools evolve. 

While `batchmp` continues to be a powerhouse for media processing, the core file management logic inside `renamer` was just too universally useful to keep locked inside a heavy multimedia package. That's why it was completely spun off into a dedicated, standalone tool.

Behold `reny`! 

`reny` takes the existing `renamer` functionality and packages it into a lightweight CLI application. Free from the heavy `ffmpeg` dependencies of its predecessor, this standalone version makes it much easier to install and use for general file management tasks.

Here is a look at what the standalone tool brings to the table.

## Batch Renaming

`reny` keeps all the classic batch renaming commands from the original tool, making bulk modifications straightforward.

### Regex Replace
When dealing with inconsistently named files across directories, `reny` lets you use regular expressions to clean them up:

```bash
$ reny replace -fs 'test' -rs 'tst' -dc
```
This will find all occurrences of 'test' in filenames and replace them with 'tst'. The `-dc` (dry-run) flag lets you preview the exact changes before anything is actually modified.

### Padding Numbers
If files are sorted awkwardly in your filesystem (like `file_1.txt`, `file_10.txt`, `file_2.txt`), the `pad` command ensures numeric sequences sort correctly by padding them with leading zeros:

```bash
$ reny pad -md 2 -dc
```
This forces all numbers to be at least 2 digits (`file_01.txt`, `file_0i2.txt`, etc.), fixing file explorer sorting issues.

### Indexing
To add sequential numbers to a batch of files:

```bash
$ reny index -dc
```
This prefixes your files with a sequential index, which is useful for organizing photo albums or structured datasets.

## File Organization

The signature organization features from the `batchmp` days remain intact.

### Organize by File Type
To organize a mixed directory (photos, videos, music files) by type:

```bash
$ reny organize -b type
```

`reny` detects whether files are images, videos, audio, or other formats and creates the appropriate subdirectories. It always shows what changes it plans to make and asks for confirmation.

### Organize by Date
Chronological sorting works similarly:

```bash
$ reny organize -b date --date-format "%Y/%m"
```
This uses standard Python `strftime` formatting to create nested year/month directories.

## Virtual Views & Dry-Runs

`reny` retains the ability to generate virtual views. 

To see how files would look organized by type, sorted by size descending (`-s sd`), complete with a size summary (`-ss`), *without actually changing anything*:

```bash
$ reny -b type -s sd -ss
```

```bash
Virtual view by type:
~/Downloads
  |->/ 2.1GB video
    |-  531MB movie_part1.mp4
    |-  442MB movie_part2.mp4
    |-  357MB presentation.mp4
  |->/ 1.2GB nonmedia
    |-  200MB installer.dmg
    |-  150MB documentation.pdf
    |-  32MB spreadsheet.xlsx
  |->/ 8.0MB image
    |-  4MB high_res_photo.jpg
    |-  2MB screenshot.png
    |-  1MB thumbnail.jpg
```
The size sorting helps identify which file types are taking up the most space.

## Developers' Quality-of-Life Features

As a standalone CLI tool, `reny` integrates well into standard developer workflows.

### Git Integration
If you're working inside a Git repository, `reny` provides three distinct ways to view and operate on your files:

1. **Show git status (`-g`)**: Visually inspect which files are modified or untracked. `reny` bubbles up file modifications to their parent directories, letting you easily see where changes occurred.
2. **Tracked files only (`-gt`)**: Filters the view to exclusively show files that Git currently tracks. This helps avoid accidentally touching untracked `.tmp` artifacts or build folders when bulk renaming.
3. **Modified files only (`-go`)**: Filters the view to show only files with active git modifications (similar to `git status`), hiding all unmodified clutter.

### Configuration & Exclusions
To avoid typing the same flags repeatedly, you can generate a local config file in the current directory:

```bash
$ reny config -l
```
This generates a `.reny.toml` file and opens it in your default editor. You can configure your default sort orders and formatting rules. Without `-l`, it sets up a global configuration in your home directory.

To ignore certain files or directories:

```bash
$ reny ignore
```
This creates a `.renyignore` file locally in your current directory, which works like a `.gitignore`. Add patterns like `node_modules/` or `*.tmp`, and `reny` will ignore them.

## Getting Started

By shedding its multimedia dependencies, `reny` is now a fast, focused tool for terminal-based file organization. 

Ready to get organized? `reny` is available via Homebrew or PyPI.
