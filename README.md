vte
===

[![Build Status](https://travis-ci.org/alacritty/vte.svg?branch=master)](https://travis-ci.org/alacritty/vte)
[![Crates.io Version](https://img.shields.io/crates/v/vte.svg)](https://crates.io/crates/vte/)

Parser for implementing virtual terminal emulators in Rust.

The parser is implemented according to [Paul Williams' ANSI parser state
machine]. The state machine doesn't assign meaning to the parsed data and is
thus not itself sufficient for writing a terminal emulator. Instead, it is
expected that an implementation of the `Perform` trait which does something
useful with the parsed data. The `Parser` handles the book keeping, and the
`Perform` gets to simply handle actions.

See the [docs] for more info.

[Paul Williams' ANSI parser state machine]: https://vt100.net/emu/dec_ansi_parser
[docs]: https://docs.rs/crate/vte/

## OSC limits and extension dispatch

`Perform::osc_max_bytes` lets an embedder bound OSC collection before dispatch.
The parser asks for a limit with `None` when an OSC starts, then with `Some(command)`
at the first semicolon. Both limits count the complete payload, including its
command, separators and ignored control bytes, excluding introducer/terminator.
Use a finite initial limit to bound an unterminated command identifier. A
command-specific limit can permit large clipboard transfers while imposing a
smaller limit on application metadata and unknown commands. The default remains
unlimited with `std`; `no_std` also retains its fixed raw-buffer bound.

A size overflow or more than 16 parameters now discards the entire OSC instead
of dispatching a truncated prefix. Parsing resumes normally after termination.
This changes the previous truncated-dispatch behavior for oversized `no_std`
sequences and parameter overflow. The `Vec` allocation may retain capacity from
previously accepted commands; byte limits bound collection, not total allocator
capacity or the separate synchronized-output buffer. Existing handling of OSC
cancellation, ESC termination and ignored controls is unchanged.

The ANSI `Handler` exposes the same limit callback and `unhandled_osc` for
commands its processor does not recognize. This callback runs at normal parser
dispatch, after preceding printable output and synchronized-output buffering.
It is not an independent raw-byte observer. Recognized commands keep their
existing handlers. Parameters borrow the parser buffer; copy them if needed
beyond the callback.
