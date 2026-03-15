---
title: "Building a CLI Tool in Rust"
date: 2026-03-14
description: "A walkthrough of building a fast, ergonomic command-line tool using Rust — from argument parsing to colored output."
tags: ["rust", "cli", "systems"]
---

I've been writing more Rust lately. Not because I have to — because it makes me think harder about what my code actually does. Last week I built a small CLI tool for batch-renaming files, and I want to walk through the interesting parts.

## Why Rust for CLI tools

Most of my quick scripts start as Python. But once a tool gets shared with teammates or runs in CI, the "just install Python 3.11 and these 4 packages" conversation gets old fast. A single static binary that works everywhere is worth the upfront cost.

Rust gives you that, plus:

- **Startup time** — no runtime to boot, the binary just runs
- **Error handling** — `Result<T, E>` forces you to think about failure paths
- **Ecosystem** — `clap`, `serde`, `colored` are genuinely excellent

## Project setup

```bash
cargo init renamer
cd renamer
```

The `Cargo.toml` stays minimal:

```toml
[package]
name = "renamer"
version = "0.1.0"
edition = "2021"

[dependencies]
clap = { version = "4", features = ["derive"] }
colored = "2"
regex = "1"
anyhow = "1"
```

Four dependencies. That's it. The compiled binary ends up around 2MB.{{< sidenote >}}Measured with `--release` on x86_64 Linux. Debug builds are significantly larger due to full symbol tables.{{< /sidenote >}}

## Argument parsing with clap

`clap`'s derive API is the sweet spot between ergonomics and control:

```rust
use clap::Parser;

#[derive(Parser, Debug)]
#[command(name = "renamer", about = "Batch rename files using regex")]
struct Args {
    /// Regex pattern to match
    #[arg(short, long)]
    pattern: String,

    /// Replacement string
    #[arg(short, long)]
    replace: String,

    /// Directory to operate on (defaults to cwd)
    #[arg(short, long, default_value = ".")]
    dir: String,

    /// Dry run — show what would change without renaming
    #[arg(long, default_value_t = false)]
    dry_run: bool,
}
```

This gives you `--help`, `--version`, type validation, and shell completions essentially for free. The `dry_run` flag is important — you really don't want a rename tool that doesn't let you preview.

## The core logic

The actual renaming logic is ~30 lines:

```rust
use std::fs;
use anyhow::{Context, Result};
use regex::Regex;
use colored::Colorize;

fn rename_files(args: &Args) -> Result<()> {
    let re = Regex::new(&args.pattern)
        .context("Invalid regex pattern")?;

    let entries = fs::read_dir(&args.dir)
        .context("Failed to read directory")?;

    let mut count = 0;

    for entry in entries {
        let entry = entry?;
        let name = entry.file_name();
        let name_str = name.to_string_lossy();

        if re.is_match(&name_str) {
            let new_name = re.replace_all(&name_str, &args.replace);

            if args.dry_run {
                println!(
                    "  {} → {}",
                    name_str.red(),
                    new_name.green()
                );
            } else {
                let old_path = entry.path();
                let new_path = old_path.with_file_name(new_name.as_ref());
                fs::rename(&old_path, &new_path)?;
                count += 1;
            }
        }
    }

    if !args.dry_run {
        println!("Renamed {} files", count.to_string().green());
    }

    Ok(())
}
```

A few things worth noting:

1. **`anyhow::Context`** lets you add human-readable context to errors without defining custom error types. The `?` operator propagates errors, and `.context()` wraps them.{{< sidenote >}}For libraries, prefer `thiserror` which generates proper `std::error::Error` implementations. `anyhow` is best suited for applications.{{< /sidenote >}}

2. **`to_string_lossy()`** handles non-UTF8 filenames gracefully. On Linux, filenames can be arbitrary bytes — this replaces invalid sequences with the replacement character instead of panicking.

3. **`colored`** makes the dry-run output actually scannable. Red for old names, green for new. Small thing, big UX difference.

## Wiring it up

```rust
fn main() -> Result<()> {
    let args = Args::parse();
    rename_files(&args)
}
```

That's the entire `main`. Parse args, call the function, propagate errors. `anyhow::Result` in main means any error gets printed with its full context chain and the process exits with code 1.

## Testing it

```bash
$ renamer --pattern '\.jpeg$' --replace '.jpg' --dry-run
  photo_001.jpeg → photo_001.jpg
  photo_002.jpeg → photo_002.jpg
  vacation.jpeg → vacation.jpg

$ renamer --pattern '\.jpeg$' --replace '.jpg'
Renamed 3 files
```

The dry run shows exactly what will change. No surprises.

## Things I'd add next

If this were a real tool I'd keep using, I'd add:

- **Undo log** — write a `.renamer-log` file so you can reverse the operation
- **Recursive mode** — walk subdirectories with `walkdir`
- **Interactive mode** — confirm each rename with `dialoguer`

But right now it does one thing well, and that's enough.

---

The thing I keep coming back to with Rust is how the compiler makes you *deal with reality*. Every `?` is an acknowledgment that something can go wrong. Every `Result` is a promise that you've thought about it. It's annoying until it saves you at 2am in production.

> The best code isn't clever. It's code that fails loudly and obviously when something goes wrong.

The full source is on [GitHub](https://github.com/nisarsyed/renamer) if you want to poke around.
