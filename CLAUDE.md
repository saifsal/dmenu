AGENTS.md is the canonical source for this file.
# dmenu

## Overview

dmenu is a keyboard-driven dynamic menu for the X Window System. It reads a newline-separated list of items from standard input. Typing narrows that list to items that contain every space-separated token. When the user confirms a choice, dmenu prints it to standard output and exits.

This repository is Saif Salih's copy of suckless dmenu 5.1 (upstream tarball dated 11 February 2022). `config.mk` sets `VERSION` to 5.1. History on `main` is one commit, `5.1`, dated 18 April 2022. The program builds and runs. The tree has no roadmap, issue templates, or CI configuration.

Differences from that suckless 5.1 tarball:

- `config.h` is checked in. The font is Cozette. Colors are white on black, white on `#005577` for the selection, and black on white for items already confirmed. Upstream ships `config.def.h` instead, with font `monospace:size=10` and colors `#bbbbbb`/`#222222`, `#eeeeee`/`#005577`, and `#000000`/`#00ffff`.
- `CFLAGS` adds `-flto`. The Makefile runs `strip` on every object file and on the `dmenu` and `stest` binaries.
- Item width is clamped (`drw_fontset_getwidth_clamp` and `textw_clamp`). Text that does not fit is drawn with an ellipsis (`...`).
- `cleanup()` frees the item list.
- The C sources are indented with two spaces.

`dmenu_run` is the launcher the man page documents for dwm. It lists executables on `$PATH` and runs the selection in `$SHELL`. `stest` filters files by type and permissions, in the same spirit as `test(1)`. `dmenu_path` uses `stest` to build the program list.

## Repository layout

- `dmenu.c` — menu window, matching, and keyboard handling.
- `drw.c`, `drw.h` — Xft drawing, fonts, colors, and UTF-8 text.
- `util.c`, `util.h` — `die()` and `ecalloc()`.
- `stest.c`, `stest.1` — file filter and its man page.
- `arg.h` — `ARGBEGIN` / `ARGEND` macros, used by `stest`. dmenu walks `argv` in `main`.
- `config.h` — compile-time bar position, font, prompt, colors, line count, and word delimiters.
- `config.mk` — `VERSION`, install prefixes, compiler, and libraries.
- `Makefile` — build, `install`, `uninstall`, `clean`, and `dist`.
- `dmenu.1` — dmenu man page. Install replaces the string `VERSION`.
- `dmenu_path` — shell script that prints executables from `$PATH`, with a cache.
- `dmenu_run` — shell script that pipes `dmenu_path` through `dmenu` into the user's shell.
- `README` — short install note. `make dist` copies this filename into the tarball.
- `README.md` — human-facing guide.
- `LICENSE` — MIT/X Consortium License.
- `AGENTS.md`, `CLAUDE.md`, `.github/copilot-instructions.md` — agent instructions. `AGENTS.md` is canonical.

`config.def.h` is absent. There is no `.gitignore` and no test suite.

## Setup

Building needs a C compiler and development files for X11, Xft, Xinerama, fontconfig, and FreeType.

On Ubuntu 24.04 those headers and libraries came from `build-essential`, `libx11-dev`, `libxft-dev`, `libxinerama-dev`, and `libfreetype-dev`. The verified build used gcc 13. `cc` on that system is clang 18, and plain `make` stops during `strip` (see Gotchas).

`config.mk` sets:

- `X11INC = /usr/X11R6/include`
- `X11LIB = /usr/X11R6/lib`
- `FREETYPEINC = /usr/include/freetype2`
- `PREFIX = /usr/local`
- `MANPREFIX = $(PREFIX)/share/man`

`/usr/X11R6` is absent on Ubuntu 24.04. `make CC=gcc` still linked, because gcc searches `/usr/include` and the default library path. If the compiler cannot find `X11/Xlib.h`, `X11/Xft/Xft.h`, `X11/extensions/Xinerama.h`, or FreeType, edit `X11INC`, `X11LIB`, and `FREETYPEINC` in `config.mk`. The commented FreeType line in that file is the OpenBSD path.

Running the menu needs an X server in `DISPLAY`. The compiled-in font name is Cozette. On the verification machine that font file was not installed, and Xft still opened a font for the name `Cozette`, so the window appeared. When Xft cannot open the configured font, dmenu prints `error, cannot load font from name: 'Cozette'` and exits with `no fonts could be loaded.` Override the font with `-fn`.

## Build, run and test

There is no test suite and no `make test` target. The commands below were run in this repository.

Build both programs with gcc:

```sh
make CC=gcc
```

The gcc 13 build printed its flags and produced `./dmenu` and `./stest` with no warnings. Version:

```sh
./dmenu -v
```

```text
dmenu-5.1
```

`stest` checks that were run:

```sh
./stest -f Makefile          # prints Makefile, exits 0
./stest -d /tmp              # prints /tmp, exits 0
./stest -f /no/such          # prints nothing, exits 1
printf '%s\n' Makefile README /no/such | ./stest -f
                             # prints Makefile and README, exits 0
./stest -q -f Makefile       # prints nothing, exits 0
./stest -l /tmp              # prints passing directory entries; exits 0 when any pass
./stest -z                   # usage on stderr, exits 2
```

`./stest -l /tmp` printed 18 names when it was run. The count follows the directory.

Install into a staging root (verified; binaries and both man pages were installed, with `VERSION` replaced by `5.1`):

```sh
make CC=gcc DESTDIR=/tmp/dmenu-root install
```

That writes `/tmp/dmenu-root/usr/local/bin/{dmenu,dmenu_path,dmenu_run,stest}` and `/tmp/dmenu-root/usr/local/share/man/man1/{dmenu.1,stest.1}`.

`make CC=gcc install` without `DESTDIR` copies the same files under `/usr/local` and needs write access there. `make uninstall` removes them. Remove build output with:

```sh
make clean
```

`make` with no `CC=` was run while `cc` pointed at clang 18. It stops here:

```text
strip: dmenu.o: file format not recognized
make: *** [Makefile:19: dmenu.o] Error 1
```

`make dist` was run. It fails because `config.def.h` is absent:

```text
cp: cannot stat 'config.def.h': No such file or directory
make: *** [Makefile:39: dist] Error 1
```

Menu on an X display (verified on `DISPLAY=:1`). Typing `b` and Return printed `beta`:

```sh
printf 'alpha\nbeta\ngamma\n' | ./dmenu -p pick
```

A vertical list pinned to the bottom of the screen opened a window there, taller than the single-line bar:

```sh
printf 'alpha\nbeta\n' | ./dmenu -i -l 2 -b -fn 'monospace:size=10'
```

`dmenu_path` was run as `PATH="/workspace:$PATH" /workspace/dmenu_path` so it could see this tree's `stest`. With no cache file yet, `stest` printed `<cache>/dmenu_run: No such file or directory` on stderr, the executable list was printed on stdout, and `$HOME/.cache/dmenu_run` was created. Later runs reuse the cache until a directory in `$PATH` is newer than the cache.

## Code style and conventions

C files are compiled as C99 with `-std=c99 -pedantic -Wall -Os -flto`. Indentation in `*.c` and `*.h` is two spaces, and the opening brace of a function sits on the signature line. Lines wrap at 80 columns. Makefile recipes use tabs, which Make requires. `dmenu_path` is a shell script indented with tabs.

Includes run from the C library, to POSIX, then a blank line, then X11, then a blank line, then project headers. `dmenu.c` includes `config.h` after the types that header uses.

`die()` writes to stderr and exits 1. A format that ends in `:` also calls `perror`. `stest` exits 0 when at least one path passed, 1 when none passed, and 2 on a usage error.

Flags for `stest` go through `arg.h`. dmenu parses options in `main`. Appearance is compile-time state in `config.h`; the command line overwrites those defaults in `main`. Prefixes, libraries, and `VERSION` are in `config.mk`.

No formatter config is checked in. Keep a small change in the local style, and leave unrelated lines alone.

## CI and tooling

Continuous integration: Not applicable. There is no `.github/workflows` directory and no other CI configuration.

GNU Make is the build tool. Targets are `all` (the default, which also runs `options` and prints `CFLAGS`, `LDFLAGS`, and `CC`), `clean`, `dist`, `install`, and `uninstall`. `config.mk` supplies `CC`, the flags, `PREFIX`, and `VERSION`. The `.c.o` rule strips each object, and the link rules strip `dmenu` and `stest`. `install` runs `sed` so the installed man pages contain the version string.

## Working rules for agents

- Keep changes small and focused.
- Run the tests before committing. There is no automated suite; run `make CC=gcc`, `./dmenu -v`, and the `stest` commands in Build, run and test.
- Never commit secrets.
- Update the docs when behaviour changes. That covers `README.md`, the man pages when user-facing behaviour changes, and these instructions.
- Use conventional commit messages.
- Keep `AGENTS.md`, `CLAUDE.md`, and `.github/copilot-instructions.md` in sync. Edit `AGENTS.md`, then copy it so the other two stay that text plus their one-line canonical-source header.
- Do not commit build outputs: `dmenu`, `stest`, `*.o`, `dmenu-5.1/`, or `dmenu-5.1.tar.gz`. There is no `.gitignore`.
- Leave the plain `README` in place. `make dist` copies that filename. `README.md` is the long guide.
- Put appearance changes in `config.h` and path or compiler changes in `config.mk`. Both need a rebuild.

## Gotchas

- `config.def.h` is absent. The `config.h` recipe (`cp config.def.h $@`) runs only when `config.h` is missing, so a normal build leaves the checked-in `config.h` alone. `make dist` always tries to copy `config.def.h` and fails. Removing `config.h` makes the next build fail the same way.
- Plain `make` fails when `cc` is clang and `-flto` is on. `strip` rejects the LLVM bitcode object. `make CC=gcc` is the build that completed here.
- `X11INC` and `X11LIB` point at `/usr/X11R6`, which many Linux systems do not have. The Ubuntu gcc build still succeeded through the compiler's default search paths.
- The font compiled into `config.h` is Cozette. Pass `-fn` when that name cannot be opened.
- dmenu grabs the keyboard and maps an override-redirect window. With `-f`, and when stdin is not a tty, it grabs the keyboard before stdin reaches end of file. The man page says this is faster and that it locks X until stdin is closed.
- `dmenu_path` and `dmenu_run` invoke `stest` and `dmenu` from `PATH`. Building in this directory does not put them on `PATH`.
- The first `dmenu_path` run, before `$XDG_CACHE_HOME/dmenu_run` or `~/.cache/dmenu_run` exists, prints `No such file or directory` from `stest -n` and still writes the cache.
- `stest` skips names that begin with `.` unless `-a` is passed. `./stest -d .` fails for that reason: the name `.` is hidden.
- Word movement and word deletion use `worddelimiters` in `config.h`. The value is a single space. The comment above it lists other characters that could be added.
- Xinerama is on by default (`-DXINERAMA` and `-lXinerama`). Comment out `XINERAMALIBS` and `XINERAMAFLAGS` in `config.mk` to build without it.
- On OpenBSD, `main` calls `pledge("stdio rpath", NULL)`.
- Ctrl-Return prints the selected item and leaves the menu open. That item is then drawn with `SchemeOut`. Shift-Return prints the typed text. Escape and Ctrl-c exit with status 1.
