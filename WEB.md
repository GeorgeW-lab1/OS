# web — a web browser for the self-hosted TCC

`web.c` is a small browser written for this OS's own compiler. It opens local
HTML/text files from the FAT32 disk **and** plain `http://` pages over the
network, using a client-side TCP stack added to the kernel.

## Run it

    make BOCHS=1            # rebuild the kernel (needed: adds the network ABI)
    sh mkwebdisk.sh         # compile web/editf/driverkit and put everything on disk.img
    qemu-system-i386 ... -nic user,model=e1000 ...   # a NIC is required (rtl8139 also works)

Inside the OS terminal run `web`. Or build it there: `cc web.c` then `web`.
Try a host web server: `python3 -m http.server 8000` on the host, then in the
browser open `http://10.0.2.2:8000/` (10.0.2.2 is the host under QEMU user networking).

Note: `make` does not track `kernel_parts/*.h`; after editing them `touch kernel.cpp`.

## Using it

Toolbar: Back, Home, URL box, Go. Click a link to follow it. Type `example.com`
or `http://host:port/path` (or a file name like `about` / `web.c`) and press Go.
Keys (URL box not focused): Up/Down, Space / b page down/up, Home/End,
Backspace = back, h = home, g = focus URL box, q = quit.

Supported markup: h1-h6 p br hr div center blockquote pre code b i u a ul ol li
dl/dt/dd table (flattened) img (shown as [alt]) title, entities, comments;
script/style are skipped. Non-HTML text types (and local .txt/.c/.h) show as
plain monospace text. Redirects (301/302/303/307/308, up to 6), chunked
bodies, relative links (`../`, `//host`, `?q`) are handled.

## How the network path works

    web.c  --knet_http_get()-->  ports 0xD0-0xD4  --> bochs_infra.cpp
           (drivers.h)                              --> bochs_glue.cpp: bochs_guest_net_cmd()
                                                    --> kernel.cpp: net_guest_http_get()
                                                    --> 07b_network.h: DNS, tcp_connect, request, read-until-close

Same latch-then-trigger pattern as the disk ABI: the guest writes its
`net_mailbox_t` address to 0xD0-0xD3 and a command byte to 0xD4; that OUT
returns only when the fetch is complete. The guest gets the raw HTTP response.

The kernel TCP is deliberately minimal: one outgoing connection at a time,
active open only, in-order receive (gaps trigger duplicate ACKs), stop-and-wait
transmit, MSS 1460, 14600-byte window, retransmitted SYN/request, RST when the
caller's buffer fills, timeouts: 6 s idle / 25 s total.

## Keyboard, input and performance

Keyboard input was reworked in the kernel; this matters to every program:

* **Key queue.** `poll_input_universal()` used to zero `last_key_press` on entry and
  is called many times per pass (inside the guest time slice too), so while any guest
  ran, most keystrokes were wiped before the main loop saw them - graphics programs got
  none, and a second terminal "lost" keys whenever a browser was open. Decoded keys now
  go into a 64-entry ring (`g_kbd_q`, `kernel_parts/03_input_ps2_mouse.h`) and the main
  loop drains up to 16 per pass.
* **Decoder.** Tables are sized to 128 (function keys / keypad used to read past the end
  of the array and type garbage). Added: 0xE0 prefix tracking (fake shifts around
  arrows ignored), Caps Lock, NumLock keypad digits, keypad `* - +`, Pause swallowed.
  Real arrows always navigate; the keypad navigates only while NumLock is off.
* **Guest side.** `ui_poll_keys()` (comp.h) drains every queued key in one frame;
  `ui_textbox_key()` applies one key to a text box. `web` and `editf` use them.

* **Preemption timer.** A guest used to run until it chose to yield (`cpu_loop()` ignores
  its instruction budget on a single-CPU Bochs build), so during a long frame the kernel
  could not read the keyboard at all. The emulated controller buffers only ~16 bytes
  (like a real PS/2 keyboard), so a burst of typing during a slow frame overflowed it and
  keys vanished. `bochs_glue.cpp` now registers a continuous Bochs timer
  (`PREEMPT_TICKS` = 40000 emulated instructions) whose callback asks `cpu_loop()` to
  return; the slice loop polls input and re-enters the guest at the same EIP. A program
  that never yields no longer freezes the whole OS. Overhead is ~0.1%.
* **Queue.** 256 entries; up to 64 PS/2 bytes are read per poll and up to 32 keys are
  delivered per main-loop pass. `g_kbd_dropped` counts any key lost to a full queue.

Keyboard tests run against this build (QEMU, typing via the monitor):
long frames (~0.7 s each) with 100 keys at ~66/s: 21/100 received before the timer,
100/100 after; 100-key zero-gap burst (13,000 keys/s) into editf: 100/100; 300 random
printable chars at ~66 keys/s: 300/300 exact; all 95 printable ASCII chars: exact;
typing in a second terminal with the browser open: 8/8 (0/8 on the original kernel).

Drawing cost (guest code is interpreted, so cost = guest instructions per frame):

* `gfx_clear()` and `ui_fill_rect()` use one `rep stosl` per row (`gfx_fill_u32()` in
  drivers.h) instead of a per-pixel `gfx_set_pixel()` loop; glyphs have an on-canvas
  fast path.
* `gfx_yield()` (new, GFX_CMD_YIELD): hands the CPU back without copying the 256 KB
  canvas. A loop with nothing new to draw calls it instead of `gfx_present()`.
* The kernel now repaints the desktop only when a guest actually presented a new canvas
  (`g_bochs_gfx_serial`); it used to repaint everything at 60 Hz while any guest was
  alive. A guest's slice also ends as soon as it yields.
* `web` redraws the toolbar only when hover/focus/text changes and the page only when
  scrolled/navigated; `editf` stops drawing text below the last visible line.

Measured under QEMU without KVM (nested emulation, so absolute numbers are low):
browser-like frame 1.2 -> 18.4 fps (~15x); idle UI loop ~700 -> ~6500 passes/s (~9x).

## Memory: the "out of memory" while compiling, and `meminfo`

Root cause (kernel bug, not the compiler): `kernel_main` initialises the 64 MiB free-list
heap, then ~150 lines later `kernel_run_global_ctors_once()` runs EVERY static constructor
in the kernel - including `FreeListAllocator()`'s, which reset `freeListHead` to null. The
whole heap was forgotten, and every later allocation silently fell through to the 48 MiB
Bochs bump pool, whose `free()` does nothing. From then on each `cc web.c` leaked ~1.6 MB
and each launched program leaked its memory image, until the pool ran dry.

Fixes: the allocator constructor is `constexpr` (nothing to re-run), and `kernel_main`
also saves/restores the allocator around the constructor pass. The ~16 MB the emulator
needs now comes from the freeable heap.

New shell command `meminfo [-v]` shows heap free / largest free block / free-block
count and bump-pool usage; `-v` adds allocation counters and a log of large allocations.
Healthy output: pool "0 KB used", heap free stays flat across repeated compiles and
program launches. It warns if more than 1 MB ever spills into the pool.

Measured (QEMU): before - heap 0 KB free at boot, pool +1.6 MB per `cc web.c`; after - heap
65,428 KB free after 8 consecutive compiles, pool 0 KB, allocations balanced. Four
launch/quit cycles of `web`, file operations, compiles and all desktop apps ran without
faults. This also made the old "`ls` then a graphics program hangs" problem disappear.

## Text-mode program output order

A guest's text sits in a per-program output ring and is printed to the terminal when the
main loop drains it, but the exit callback printed `[guest exit, code=N]` immediately, so
the banner always appeared BEFORE the program's own output (`hello` showed the banner, then
HELLO WORLD). `elf_io_exit` now drains the ring first (`drain_slot_output`), prints the
banner to the program's own terminal, and skips the extra blank line when the program
already ended with a newline.

## Limits

* **http:// only.** No TLS, so https sites cannot be opened (and many sites
  redirect http to https).
* The desktop **freezes while a page downloads** (blocking call, worst case ~25 s).
* Pages are cut at 48000 bytes ("[page truncated]"). No images, CSS or scripts.
* The URL box holds 63 characters (comp.h limit); the page keeps the full URL.
* Needs a NIC: `ifconfig` in the OS terminal must show an IP address.

## Testing notes

* `knet_http_get` returns `KNET_ERR_UNREACHABLE` on a kernel without the ABI;
  `web` then says to rebuild the kernel.
* `-DWEB_START_URL='"http://10.0.2.2:8000/"'` builds a copy that opens that URL
  at startup (useful where keyboard input isn't available).
* Known pre-existing issue, not fixed: `busybox <applet> ...` prints no output (on the
  original kernel it does not even start the process). Unrelated to the memory/ordering fixes.
* A kernel built WITHOUT the gfx_yield change ignores GFX_CMD_YIELD, so a program
  that only yields on idle frames would never hand control back. Rebuild the kernel
  (and use the matching drivers.h) together.
* FAT names (and Windows/macOS checkouts) are case-insensitive. The repo used to carry
  stale uppercase twins `COMP.H` / `EDITF.C` / `GUI_DEMO.C` that collided with `comp.h` /
  `editf.c` / `gui_demo.c`, silently replacing them with older versions (symptom:
  `implicit declaration of function 'ui_poll_keys'` / `'ui_textbox_key'` while building
  `web`). The twins are removed, and `mkwebdisk.sh` now refuses to build if `comp.h` or
  `editf.c` is a stale copy.
