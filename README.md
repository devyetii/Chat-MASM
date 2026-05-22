# Chat-MASM — 8086 Serial Communication Chat

A **split-screen terminal chat application** written entirely in **MASM 8086 Assembly**. It enables real-time text communication between two PCs connected via a **serial (RS-232) port** using the on-board **UART chip**.

The screen is divided into two independent panes: the **top half** shows what you type, and the **bottom half** shows what the other person sends — all running at **9600 baud** with full support for `Enter`, `Backspace`, and `Esc` to quit.

---

## Features

- **Dual-Pane UI** — Top half = local input (sender), bottom half = remote input (receiver)
- **Real-Time Serial I/O** — Polls the UART Line Status Register for non-blocking send/receive
- **Full Keyboard Support** — Normal typing, `Enter` for new lines, `Backspace` for deletion, `Esc` to exit
- **Auto-Scrolling** — Both panes scroll independently when text reaches the edge
- **Intelligent Backspace** — Handles line wrap: deletes character, scrolls down if at start of line
- **Baud Rate** — 9600 bps, 8 data bits, 1 stop bit, even parity
- **Cross-Platform DOS** — Runs on real 8086 hardware or DOSBox with virtual serial passthrough

---

## Architecture

### `chat.asm` — Main Program
- **`CONFIG`** — Initializes the **UART** (COM1 / `0x3F8`):
  - Sets divisor latch for **9600 baud**
  - Configures line control: 8-bit, 1 stop bit, even parity
- **`MAIN_LP`** — The core event loop:
  1. Checks if a local key is pressed (`INT 16h / AH=01`)
  2. Reads the key, handles `Esc`, `Enter`, `Backspace`, or prints the character
  3. Sends the character over serial (`OUT 0x3F8`)
  4. Polls the serial port for incoming data (`IN 0x3FD`)
  5. Receives and prints the remote character to the bottom pane
- **`SENDMSG`** — Waits for UART transmitter-empty, then sends `CHAR1`
- **`RECMSG`** — Checks UART data-ready bit; if set, reads `CHAR2` from receiver buffer
- **`PRTCHAR1/2`** — Prints a character to the correct pane and advances cursor; wraps at column 80
- **`HANDLEBS1/2`** — Backspace logic: deletes character, handles scroll-down at line start
- **`DRWLINECENTER`** — Draws a horizontal separator line (`=`) between the two panes

### `lib.inc` — Macro Library
| Macro | Purpose |
|-------|---------|
| `SETCURSOR X,Y` | Move text cursor via BIOS `INT 10h` |
| `GETCURSOR` | Read current cursor position (returns X in `DL`, Y in `DH`) |
| `CHECKKEY` | Non-blocking keyboard check (`INT 16h / AH=01`) |
| `GETKEY` / `GETKEY1` / `GETKEY2` | Blocking keyboard read; stores ASCII in `CHAR1` or `CHAR2` |
| `PRTCHAR` | Prints a colored character via BIOS `INT 10h / AH=09h` |
| `PRINTMSG` | Prints a `$`-terminated string via DOS `INT 21h / AH=09h` |
| `SCROLL` / `SCROLLDOWN` | Scroll a text window up or down via BIOS |
| `SCROLLUP1/2` | Scroll the top/bottom pane up by one line |
| `SCROLLDOWN1/2` | Scroll the top/bottom pane down by one line |
| `CLRS` | Clear the entire screen |
| `HALT` | Exit to DOS |
| `PAUSE` | Infinite loop (debugging) |
| `COMPARE_KEY` | Compare ASCII code of last key |
| `clearkeyboardbuffer` | Directly clears the BIOS keyboard buffer in memory |

---

## Serial Port Configuration (UART)

The app uses **COM1** (`I/O base 0x3F8`):

| Register | Address | Setting |
|----------|---------|---------|
| Divisor Latch (LSB) | `0x3F8` | `0x0C` → 9600 baud |
| Divisor Latch (MSB) | `0x3F9` | `0x00` |
| Line Control | `0x3FB` | `0x80` (DLAB on) → then `0x1B` = 8 bits, even parity, 1 stop bit |
| Line Status | `0x3FD` | Bit 5 = TX empty, Bit 0 = RX ready |
| Transmitter | `0x3F8` | Send byte when TX empty |
| Receiver | `0x3F8` | Read byte when RX ready |

---

## How to Build & Run

### Prerequisites
- DOSBox or real DOS/8086 system
- Two PCs with physical serial ports **OR** DOSBox virtual serial bridge

### Build
```batch
masm chat.asm;
link chat.obj;
```

### Run
```batch
chat.exe
```

### DOSBox Virtual Serial Setup
Use the included `virtSerial.txt` commands to bridge two DOSBox instances via a physical COM port or a virtual null-modem:

```batch
:: Instance 1 (Sender / Top Pane)
dosbox -c "serial1 = directserial realport:COM1" -c "mount c C:\chatlab" -c "c:"

:: Instance 2 (Receiver / Bottom Pane) — connect to COM2 or same COM1 with a null-modem
dosbox -c "serial1 = directserial realport:COM2" -c "mount c C:\chatlab" -c "c:"
```

> A **null-modem cable** (or virtual null-modem driver) is required to connect COM1 on one PC to COM1 on another.

---

## Controls

| Key | Action |
|-----|--------|
| `A-Z`, `0-9`, `Space`, `Symbol` | Type a character |
| `Enter` | New line in your pane |
| `Backspace` | Delete previous character (wraps to previous line if at start) |
| `Esc` | Send exit signal and quit the program |

---

## Project Structure

```
.
├── chat.asm          # Main chat program (serial I/O, dual-pane UI, event loop)
├── lib.inc           # BIOS/DOS macro library (cursor, print, scroll, keyboard)
├── virtSerial.txt    # DOSBox virtual serial setup commands
├── masm.exe          # MASM assembler (included for convenience)
├── LINK.EXE          # DOS linker (included for convenience)
├── AFDEBUG.EXE       # DOS debugger (included for convenience)
├── debug.exe         # DOS DEBUG utility (included for convenience)
└── .vscode/          # VS Code settings
```

---

## How It Works

1. **Screen Split** — The display is divided by a line of `=` characters at row 13. Rows `0–12` are the local sender pane; rows `14–24` are the remote receiver pane.
2. **Polling Loop** — The main loop runs continuously:
   - Checks the keyboard buffer. If a key is pressed, it reads it, updates the top pane, and **immediately transmits** it over UART.
   - Checks the UART Line Status Register. If data has arrived, it reads one byte and **immediately prints** it in the bottom pane.
3. **Cursor Tracking** — Two byte variables (`CURSOR1_CUR_X`, `CURSOR2_CUR_X`) track the current column in each pane so the app knows where to print next.
4. **Line Wrapping** — When a pane reaches column 80, the screen scrolls up by one line and the cursor resets to column 0.
5. **Backspace Handling** — If the cursor is at column 0, the pane scrolls down by one line and the cursor jumps to column 79, mimicking standard terminal behavior.
6. **Exit Protocol** — Pressing `Esc` sends `0x1B` over serial and halts locally. Receiving `0x1B` also halts locally, so both sides exit cleanly.

---

## Tech Stack

- **Language**: MASM 8086 Assembly
- **Platform**: DOS / 8086 real mode
- **I/O**: BIOS `INT 10h` (video), `INT 16h` (keyboard), direct `IN`/`OUT` to UART (`0x3F8`–`0x3FF`)
- **Serial**: RS-232 COM1 via 8250/16450/16550 UART

---

## License

Provided for educational and retro-computing purposes.
