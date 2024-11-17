+++
title = "exploring emulation with chirp 🐣: a CHIP-8 interpreter"
date = 2023-02-23

[taxonomies]
tags = ["programming", "rust", "interpreter", "emulation"]
+++

I always wondered how hardware emulation works under the hood and one of my
long-term aspirations is to forge an emulator for the legendary [Game Boy](https://en.wikipedia.org/wiki/Game_Boy)
console. In order to gather some knowledge and skills for this future project,
I've decided to create a [CHIP-8](https://en.wikipedia.org/wiki/CHIP-8)
interpreter.

{{ figure(
    src="/img/chirp/maze.png",
    alt="Screenshot of the MAZE game running on chirp",
    position="center",
    caption="MAZE game running on `chirp`",
    caption_style="font-weight: bold; font-style: italic;"
)}}

<!-- more -->

The heart of [`chirp`](https://github.com/luizmugnaini/chirp) is an interpreter
of the CHIP-8 programming language, with roots in the early days of game
development. Built entirely in Rust, this project can serve as a learning
material for someone interested in dipping their toes in the world of
emulation. The goal of this project was to lay the fundamental ground knowledge
of emulation.

{{ figure(
    src="/img/chirp/invaders.png",
    alt="Screenshot of the Space Invaders game running on chirp",
    position="center",
    caption="Space Invaders game running on `chirp`",
    caption_style="font-weight: bold; font-style: italic;"
)}}

I tried to make the external dependencies as minimal as possible (being Rust,
this is quite the challenge):

- [`rand`](https://crates.io/crates/rand): most famous random number generation
  crate in the Rust ecosystem.
- [SDL (Simple DirectMedia Layer)](https://www.libsdl.org/): cross-platform
  development library providing low-level access to audio, keyboard, mouse,
  and graphics. SDL serves as the backbone for `chirp`, allowing you to
  interact with CHIP-8 games.

Throughout the creation of `chirp`, I found
[Cowgod's CHIP-8 Technical Reference](http://devernay.free.fr/hacks/chip8/C8TECH10.HTM)
to be an invaluable resource, serving as the main reference for the project.

If you find any issues with the project, or even suggestions, I would be
pleased to hear them. You can open an issue or a pull request in the
github repository, or contact me directly.
