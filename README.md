<h1 align="center">⠋ nanospinner</h1>

<p align="center">
  <a href="https://github.com/anthonysgro/nanospinner/actions"><img src="https://github.com/anthonysgro/nanospinner/actions/workflows/ci.yml/badge.svg?branch=main" alt="Build Status"></a>
  <a href="https://crates.io/crates/nanospinner"><img src="https://img.shields.io/crates/v/nanospinner" alt="Crates.io"></a>
  <a href="https://docs.rs/nanospinner/latest/nanospinner/"><img src="https://docs.rs/nanospinner/badge.svg" alt="Docs.rs"></a>
  <a href="https://crates.io/crates/nanospinner"><img src="https://img.shields.io/crates/l/nanospinner" alt="License"></a>
  <a href="https://coveralls.io/github/anthonysgro/nanospinner?branch=main"><img src="https://coveralls.io/repos/github/anthonysgro/nanospinner/badge.svg?branch=main" alt="Coverage Status"></a>
  <a href="https://www.codefactor.io/repository/github/anthonysgro/nanospinner"><img src="https://www.codefactor.io/repository/github/anthonysgro/nanospinner/badge" alt="CodeFactor"></a>
</p>

A minimal, zero-dependency terminal spinner for Rust applications. Supports single and multi-spinner modes.

![demo](demo.gif)

Inspired by the Node.js [nanospinner](https://github.com/usmanyunusov/nanospinner) npm package, `nanospinner` gives you a lightweight animated spinner using only the Rust standard library — no heavy crates, no transitive dependencies, builds in .2 seconds.

Part of the [nano](https://github.com/anthonysgro/nano) crate family — zero-dependency building blocks for Rust.

## Comparison

| | `nanospinner` | `spinach` | `spinoff` | `indicatif` |
|---|---|---|---|---|
| Dependencies | 0 | 0 | 3 | 6 |
| Clean Build Time | ~0.2s | ~0.2s | ~1.2s | ~1.4s |
| Customizable Frames | Default Braille set | Yes | Yes (80+ sets) | Yes |
| Multiple Spinners | Yes | No | No | Yes |
| Auto TTY Detection | Yes | No | No | Yes |
| Custom Writer | Yes (io::Write) | No | Stderr only | Yes (custom trait) |
| Thread-Safe Handles | Yes (`Send`) | No | No | Yes (`Send + Sync`) |
| Progress Bars | No | No | No | Yes |
| Async Support | No | No | No | Optional (`tokio` feature) |

Build times measured from a clean `cargo build --release` on macOS aarch64 (Apple Silicon). Your numbers may vary by platform.

## Quick Start

Add `nanospinner` to your project:

```bash
cargo add nanospinner
```

```rust
use nanospinner::Spinner;
use std::thread;
use std::time::Duration;

fn main() {
    let handle = Spinner::new("Loading...").start();
    thread::sleep(Duration::from_secs(2));
    handle.success();
}
```

## Usage

For the full API, see the [docs.rs documentation](https://docs.rs/nanospinner/latest/nanospinner/).

### Single Spinner

`Spinner::new(msg).start()` spawns a background thread that animates the spinner. It returns a `SpinnerHandle` you use to update or finalize the spinner. Calling `success()`, `fail()`, `warn()`, or `info()` stops the thread and prints the final line — no separate `stop()` needed. If you drop the handle without finalizing, the thread is joined and the line is cleared automatically.

```rust
use nanospinner::Spinner;
use std::thread;
use std::time::Duration;

// Basic: start, wait, finalize
let handle = Spinner::new("Downloading...").start();
thread::sleep(Duration::from_secs(2));
handle.success(); // ✔ Downloading...

// Update mid-spin, finalize with a replacement message
let handle = Spinner::new("Step 1...").start();
thread::sleep(Duration::from_secs(1));
handle.update("Step 2...");
thread::sleep(Duration::from_secs(1));
handle.success_with("All steps complete"); // ✔ All steps complete
```

### Multi-Spinner

`MultiSpinner` manages multiple spinner lines with a single background render thread. Finalizing a line (`success`, `fail`, `clear`) only updates that line's status — the render thread keeps running. Call `stop()` on the group handle (or let it drop) to shut down the render thread.

```rust
use nanospinner::MultiSpinner;
use std::thread;
use std::time::Duration;

let handle = MultiSpinner::new().start();

let line1 = handle.add("Downloading...");
let line2 = handle.add("Compiling...");

thread::sleep(Duration::from_secs(2));
line1.success();
line2.fail_with("Compile error");

handle.stop();
```

```rust
// Thread-based: move line handles to worker threads
let handle = MultiSpinner::new().start();

let workers: Vec<_> = (1..=3)
    .map(|i| {
        let line = handle.add(format!("Worker {i} processing..."));
        thread::spawn(move || {
            thread::sleep(Duration::from_secs(i));
            line.success_with(format!("Worker {i} done"));
        })
    })
    .collect();

for w in workers {
    w.join().unwrap();
}

handle.stop();
```

### Custom Writers and TTY Detection

Both `Spinner` and `MultiSpinner` auto-detect whether stdout is a terminal. When it isn't (piped, redirected), animation and ANSI codes are skipped — only plain text is printed:

```text
$ my_tool | cat
✔ Done!
```

For custom output targets, both offer `with_writer` and `with_writer_tty` constructors:

```rust
// Custom writer (defaults to non-TTY — no ANSI codes)
let handle = Spinner::with_writer("Processing...", std::io::stderr()).start();
let handle = MultiSpinner::with_writer(my_writer).start();

// Custom writer with explicit TTY control
let handle = Spinner::with_writer_tty("Building...", my_writer, true).start();
let handle = MultiSpinner::with_writer_tty(my_writer, true).start();
```

## Contributing

Contributions are welcome. To get started:

1. Fork the repository
2. Create a feature branch (`git checkout -b my-feature`)
3. Make your changes
4. Run the tests: `cargo test`
5. Submit a pull request

Please keep changes minimal and focused. This crate's goal is to stay small and as dependency-free as possible.

## License

This project is licensed under the [MIT License](LICENSE).