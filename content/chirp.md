+++
title = "chirp 🐣, a CHIP-8 interpreter"
date = 2023-02-23

[taxonomies]
tags = ["programming", "rust", "interpreter", "CHIP-8"]
+++

I recently got interested in the idea of emulating hardware and one of my goals
is to emulate a Gameboy some day in the future.
Searching the web for which path to take, I built [`chirp`](https://github.com/luizmugnaini/chirp),
a CHIP-8 interpreter written in [Rust](https://www.rust-lang.org/).

{{ figure(
    src="/img/chirp/maze.png",
    alt="Screenshot of the MAZE game running on chirp",
    position="center",
    caption="MAZE game running on `chirp`",
    caption_style="font-weight: bold; font-style: italic;"
)}}

<!-- more -->

The only external dependencies of `chirp` are the random number generation crate
[`rand`](https://crates.io/crates/rand) and [SDL](https://www.libsdl.org/).
The main reference for the project was [Cowgod's CHIP-8 Technical Reference](http://devernay.free.fr/hacks/chip8/C8TECH10.HTM).

{{ figure(
    src="/img/chirp/invaders.png",
    alt="Screenshot of the Space Invaders game running on chirp",
    position="center",
    caption="Space Invaders game running on `chirp`",
    caption_style="font-weight: bold; font-style: italic;"
)}}
