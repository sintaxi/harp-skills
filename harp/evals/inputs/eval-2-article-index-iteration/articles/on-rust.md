# <%= title %>

*<%= date %>*

I spent the last month rewriting a small CLI in Rust. It was originally a Node script — maybe 400 lines — that I'd been meaning to harden for a while.

A few things that surprised me:

- The compiler is genuinely a tutor. I learned more about the shape of my own program from fighting `borrowck` than from any book.
- `clap` is delightful. The derive API is the right amount of magic.
- Cross-compilation is still rougher than I expected on macOS.

Would I reach for Rust again on a similar project? Yes, but only if I expected the thing to live for years. For a one-off, the iteration cost is too high.
