# logi-m575-sensitivity-app

A small C utility for Linux/X11 that adjusts the pointer acceleration
("sensitivity") of a Logitech ERGO M575 trackball from the command line. The
program is compiled to a binary named `ergosniffer`. It is meant to be bound to
the trackball's extra buttons through
[xbindkeys](https://www.nongnu.org/xbindkeys/), so that pressing a button calls
`ergosniffer up` or `ergosniffer down` and moves the sensitivity one step.

## What it is

`ergosniffer` is a single-file C program (`main.c`) that:

- stores the current acceleration value in `/tmp/sens.txt`,
- reads `up` or `down` as its first argument,
- increases or decreases the value by `0.05`, clamps it to the range
  `-1.0` to `1.0`, and writes it back,
- finds the M575 device id through `xinput`, and
- applies the new value with
  `xinput --set-prop <device-id> 'libinput Accel Speed' <value>`.

It is a personal, Linux-only helper. It has no GUI, no configuration file, and
no background process of its own; the button-to-command binding is provided by
xbindkeys.

## Features

- Adjusts the libinput acceleration speed of the M575 by a fixed step of
  `0.05` per invocation (`main.c`, the `up`/`down` branches).
- Clamps the value so it never leaves `-1.0` .. `1.0` (`main.c`, the clamping
  block).
- Persists the last value between runs in `/tmp/sens.txt`, formatted with two
  decimals (`main.c`, the `fopen`/`fscanf` and `fprintf` calls).
- Creates `/tmp/sens.txt` on first use when the file is missing
  (`main.c`, the `fopen(..., "w+")` branch).
- Detects the M575 automatically by parsing `xinput | grep M575`
  (`main.c`, the two `popen` calls and the `atoi` on the device id).

## Tech stack

- **Language:** C (uses `stdio.h`, `string.h`, `stdlib.h`, `stdbool.h`).
- **Build:** GNU Make and GCC, driven by the repository `makefile`.
- **Runtime dependencies:** `xinput` (Xorg input configuration) and, for the
  button bindings, `xbindkeys`.
- **Platform:** Linux with X11 and libinput.

## Architecture / diagram

There is no server or persistent daemon. Control flows as follows:

```
  M575 extra button (b:8 / b:9)
            |
            v
        xbindkeys
            |
            v
   ergosniffer up | down
            |
            v
   read /tmp/sens.txt  --->  adjust by 0.05, clamp to [-1.0, 1.0]
            |                                   |
            |                                   v
            |                        write /tmp/sens.txt
            v
   xinput | grep M575  ->  device id
            |
            v
   xinput --set-prop <id> 'libinput Accel Speed' <value>
```

All steps above map directly to statements in `main.c`; the `b:8`/`b:9` button
bindings come from the xbindkeys configuration shown below.

## Getting started

Prerequisites:

- `xbindkeys` and `xinput`. The original instructions install xbindkeys on
  Arch Linux with `sudo pacman -S xbindkeys`; `xinput` comes from the
  `xorg-xinput` package on most distributions.
- A C toolchain (`gcc`) and `make`.

Build:

```sh
git clone https://github.com/HyoAkira/logi-m575-sensitivity-app.git
cd logi-m575-sensitivity-app
make
```

This runs `gcc -c main.c` followed by `gcc main.o -o ergosniffer` (the rules in
`makefile`) and produces the `ergosniffer` binary in the repository directory.

Install and configure:

```sh
# Put the binary on PATH.
sudo install -m 755 ergosniffer /usr/bin/

# Create the xbindkeys configuration if it does not exist yet.
touch ~/.xbindkeysrc
```

Add the button bindings to `~/.xbindkeysrc`:

```sh
"ergosniffer up"
  b:9

"ergosniffer down"
  b:8
```

Then (re)start xbindkeys. Running it in the background or from autostart is
recommended so it keeps listening:

```sh
killall xbindkeys
xbindkeys
```

## Usage example

With the bindings above in place, pressing the trackball button mapped to `b:9`
runs:

```sh
ergosniffer up     # raises the acceleration value by 0.05
```

and the button mapped to `b:8` runs:

```sh
ergosniffer down   # lowers the acceleration value by 0.05
```

Each run reads the previous value from `/tmp/sens.txt`, clamps the result to
`-1.0` .. `1.0`, stores it again, and applies it to the M575, for example:

```sh
xinput --set-prop <device-id> 'libinput Accel Speed' 0.55
```

The same commands can be run by hand from a terminal for testing or scripting.

## Status and license

Status: a small personal utility. The repository has two commits and currently
carries no releases or tags. It targets Linux/X11 and the Logitech M575
specifically; the device lookup relies on the string `M575` appearing in
`xinput` output.

License: GNU General Public License v3.0. The full text is in the
[`LICENSE`](LICENSE) file (GNU GPL v3, 29 June 2007).
