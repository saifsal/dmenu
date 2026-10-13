# dmenu

dmenu is a dynamic menu for X. It stays out of the way: a program, or a shell pipeline, feeds it a list of lines, and dmenu prints the line the user picks. That makes it an application launcher and a general picker for anything a script can list.

This repository is [Saif Salih](https://github.com/saifsal)'s copy of [suckless dmenu](https://tools.suckless.org/dmenu/) 5.1, with a Cozette font, a white-on-black color scheme, link-time optimisation, and drawing changes that ellipsize items which do not fit. The version in `config.mk` is 5.1.

The short install note in `README` is the file `make dist` packages. This `README.md` is the longer guide.

## Features

- Reads newline-separated items from stdin and prints the choice to stdout.
- Narrows the list as you type. Every space-separated token must occur in an item. Exact matches come first, then prefixes, then substrings.
- Case-insensitive matching with `-i`.
- A single horizontal bar, or a vertical list with `-l`.
- The bar sits on the top edge, or on the bottom edge with `-b`.
- With Xinerama, follows the focused window, the pointer, or the monitor given with `-m`.
- Prompt (`-p`), font (`-fn`), and colors (`-nb`, `-nf`, `-sb`, `-sf`). `#RGB`, `#RRGGBB`, and X color names work.
- Can embed in another window with `-w`.
- Keyboard control, including word movement, deletion, and paste from the primary selection (`Ctrl-y`) or the clipboard (`Ctrl-Y`).
- Ctrl-Return prints the selection and keeps the menu open. Shift-Return prints the text that was typed. Escape cancels.
- Items that do not fit are drawn with an ellipsis.
- `dmenu_run` lists executables on `$PATH` and runs the one you pick in `$SHELL` (the launcher the man page documents for dwm).
- `stest` filters files by type, age, and permissions.

The menu is keyboard-driven. A full binding list is in `dmenu.1`.

## Examples

Pick from three lines. Typing `b` and pressing Return prints `beta`:

```sh
printf 'alpha\nbeta\ngamma\n' | dmenu -p pick
```

Case-insensitive vertical list on the bottom edge, using a font you know is installed:

```sh
printf 'alpha\nbeta\n' | dmenu -i -l 2 -b -fn 'monospace:size=10'
```

Application launcher (after `dmenu`, `dmenu_path`, and `stest` are on `PATH`):

```sh
dmenu_run
```

`dmenu_run` accepts the same flags as `dmenu` and passes them through.

Ask `stest` for regular files, then for every executable in a directory:

```sh
stest -f Makefile
stest -flx /usr/bin
```

## Prerequisites

- An X11 display for running `dmenu`. `stest` does not need one.
- A C compiler. gcc completes this Makefile. See [Development and testing](#development-and-testing) for clang.
- Development files for X11, Xft, Xinerama, fontconfig, and FreeType.

On Ubuntu 24.04:

```sh
sudo apt install build-essential libx11-dev libxft-dev libxinerama-dev libfreetype-dev
```

The font compiled into this tree is Cozette. Install it if you want that face. Xft will try the name `Cozette` either way; if it cannot open a font, dmenu exits with `no fonts could be loaded.` Use `-fn` to pick another face, for example `monospace:size=10`.

## Install

`config.mk` installs into `/usr/local` (`PREFIX`) and puts man pages in `/usr/local/share/man` (`MANPREFIX`). Edit that file if the X11 headers or libraries live somewhere else. The defaults are `X11INC=/usr/X11R6/include`, `X11LIB=/usr/X11R6/lib`, and `FREETYPEINC=/usr/include/freetype2`. On Ubuntu those `/usr/X11R6` directories are often missing; gcc still finds X11 on its default search path, which is how the build documented below succeeded.

```sh
make CC=gcc clean install
```

That installs `dmenu`, `dmenu_path`, `dmenu_run`, and `stest` to `$PREFIX/bin`, and installs the man pages with the version filled in. Install as root when `/usr/local` is not writable. Remove them later with `make uninstall`.

A staged install, useful when you do not want to write to `/usr/local`:

```sh
make CC=gcc DESTDIR=/tmp/dmenu-root install
```

## Usage

```text
dmenu [-bfiv] [-l lines] [-m monitor] [-p prompt] [-fn font]
      [-nb color] [-nf color] [-sb color] [-sf color] [-w windowid]
```

| Flag | Effect |
| --- | --- |
| `-b` | Place the bar at the bottom of the screen. |
| `-f` | Grab the keyboard before reading stdin when stdin is not a terminal. Faster, and it locks X until stdin reaches end of file. |
| `-i` | Match case-insensitively. |
| `-l lines` | Show a vertical list of `lines` rows. |
| `-m monitor` | Show on that monitor. Numbers start at 0. |
| `-p prompt` | Text drawn to the left of the input, in the selection colors. |
| `-fn font` | Font or font set. Replaces `fonts[0]` from `config.h`. |
| `-nb`, `-nf` | Normal background and foreground. |
| `-sb`, `-sf` | Selected background and foreground. |
| `-w windowid` | Embed in this window. |
| `-v` | Print `dmenu-5.1` and exit. |

Keys you will use constantly:

| Key | Action |
| --- | --- |
| Arrow keys, Page Up, Page Down, Home, End | Move through items. |
| Tab | Copy the selected item into the input. |
| Return | Print the selected item and exit 0. |
| Ctrl-Return | Print the selected item and stay open. |
| Shift-Return | Print the typed text and exit 0. |
| Escape, Ctrl-c, Ctrl-g | Exit 1 with no selection. |
| Ctrl-w | Delete the word to the left. Word boundaries are spaces (`worddelimiters` in `config.h`). |
| Ctrl-Left, Ctrl-Right, Alt-b, Alt-f | Move to the previous or next word edge. |
| Ctrl-y / Ctrl-Y | Paste the primary selection / the clipboard. |

`dmenu_path` caches the executable list in `$XDG_CACHE_HOME/dmenu_run`, or `~/.cache/dmenu_run`. It rebuilds the cache when a directory on `$PATH` is newer than the cache. The first run, before the cache file exists, prints `No such file or directory` from `stest` and still writes the cache.

`stest` reads paths from the command line, or from stdin when none are given. `-l` tests the contents of a directory argument. `-q` prints nothing and only sets the exit status. `-v` inverts the tests. Exit status is 0 when something passed, 1 when nothing passed, and 2 on an error. Details are in `stest.1`.

## Configuration

Two files, both compiled or read at build time:

`config.h` controls what the menu looks like and how words are split.

| Setting | Value in this tree |
| --- | --- |
| `topbar` | `1` (top edge; `-b` turns this off at runtime) |
| `fonts` | `Cozette` |
| `prompt` | `NULL` |
| `colors[SchemeNorm]` | white on black |
| `colors[SchemeSel]` | white on `#005577` |
| `colors[SchemeOut]` | black on white (items confirmed with Ctrl-Return) |
| `lines` | `0` (horizontal bar; `-l` overrides this) |
| `worddelimiters` | a single space |

`config.mk` controls the build: `VERSION`, `PREFIX`, `MANPREFIX`, include and library paths, whether Xinerama is linked, `CFLAGS`, and `CC`. Xinerama is enabled. Comment out `XINERAMALIBS` and `XINERAMAFLAGS` to build without it. `CFLAGS` includes `-flto`.

Edit either file and rebuild. Command-line color, font, prompt, and geometry flags override `config.h` for that run.

## Development and testing

```sh
make CC=gcc
./dmenu -v
./stest -f Makefile
./stest -d /tmp
./stest -f /no/such          # exits 1
printf '%s\n' Makefile README /no/such | ./stest -f
./stest -q -f Makefile       # exits 0, prints nothing
./stest -z                   # usage on stderr, exits 2
make clean
```

These commands were run against this tree. `make CC=gcc` with gcc 13 produced `./dmenu` and `./stest` and printed no warnings. `./dmenu -v` printed `dmenu-5.1`. The `stest` lines behaved as the comments say. `./stest -l /tmp` also ran and listed that directory.

`make` alone was tried while `cc` was clang 18. It fails in the `.c.o` rule:

```text
strip: dmenu.o: file format not recognized
make: *** [Makefile:19: dmenu.o] Error 1
```

Clang's `-flto` objects are LLVM bitcode, and the Makefile strips every object before linking. Use `make CC=gcc`.

`make dist` currently fails:

```text
cp: cannot stat 'config.def.h': No such file or directory
make: *** [Makefile:39: dist] Error 1
```

The Makefile still expects upstream's `config.def.h`. This tree commits `config.h` directly, so ordinary builds do not need `config.def.h`.

There is no automated test suite and no CI. `stest` is a file-filtering utility shipped with dmenu. The man pages `dmenu.1` and `stest.1` are the behaviour reference; `install` substitutes the version number into them.

Build products (`dmenu`, `stest`, `*.o`, and a `dmenu-5.1` dist directory) are not gitignored. `make clean` removes the binaries, the objects, and `dmenu-5.1.tar.gz`.

## Project status

Version 5.1. The git history is the single commit `5.1` from 18 April 2022.

Relative to the suckless dmenu 5.1 tarball, this tree reformats the C sources with two-space indentation, enables `-flto` and `strip`, frees the item list on exit, clamps text width, and draws an ellipsis when an item does not fit. Appearance defaults live in the committed `config.h` described above.

There is no roadmap in the repository.

## Licence

MIT/X Consortium License. See `LICENSE`.

Copyright holders named there:

- © 2006-2019 Anselm R Garbe
- © 2006-2008 Sander van Dijk
- © 2006-2007 Michał Janeczek
- © 2007 Kris Maglione
- © 2009 Gottox
- © 2009 Markus Schnalke
- © 2009 Evan Gates
- © 2010-2012 Connor Lane Smith
- © 2014-2022 Hiltjo Posthuma
- © 2015-2019 Quentin Rameau
