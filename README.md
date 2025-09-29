## TL;DR

It’s a classic “overflow your username to hijack the return address” challenge—just on a tiny custom CPU in an emulator. We smash the stack, point execution to a built-in `win` routine, and it prints the flag.

Final one-liner that wins:

`python3 - <<'PY' | nc -N sunshinectf.games 25701 import sys def p16(x): return bytes((x & 0xff, (x>>8)&0xff))  # little-endian 16-bit PAD = 54            # bytes to reach saved return address PC  = 0x0236        # address of @win DPC = 0x0000        # paired return value this CPU expects sys.stdout.buffer.write(b"A"*PAD + p16(PC) + p16(DPC) + b"\n") PY`

You’ll see:

`Welcome, AAAAA...6! sun{th1s_i5_ju57_7h3_t1p_0f_7h3_spEAR}`

---

## What the service looks like

Connecting with netcat:

`nc sunshinectf.games 25701  EAR-SAT-7 Terminal v2.3.1 Connected to: GEO-SAT-042 ... ... Enter username:`

- If you type a normal name (e.g., `Bob`), it prints `Welcome, Bob!`.
    
- If you paste a _very_ long name, you sometimes get:
    
    `PANIC! Segmentation fault Halted: Kernel panic`
    
    That’s our “the cup overflowed” signal.
    

This tells us the program is reading your input into a fixed-size buffer and not checking length—i.e., the classic buffer overflow.

---

## The plan in plain English

Think of the program’s memory like a stack of papers. Your username goes onto that stack. If your username is longer than the space reserved for it, it starts writing over the next papers—eventually covering the “where to go back to after finishing this function.” If we can overwrite that “go-back address” (the return address), we can send the program to **our** chosen function, namely `@win`, which prints the flag.

Two things we need:

1. **How far** to type before we hit the return address (the _offset_).
    
2. The **address** of the `@win` function (where to jump).
    

---

## Peeking under the hood (lightly)

The emulator prints debug info like this while running, and we’ve got snippets of the disassembly. Two key crumbs:

1. The function sets up a local buffer with:
    

`025B: SUB SP, 0x32`

That’s a 0x32-byte (50-byte) local area for your username.

2. The function epilogue looks like:
    

`0292: POP {S0, FP, PC-DPC}`

Translation: when returning, it pops **four** 2-byte values from the stack in this order:

- `S0` (2 bytes)
    
- `FP` (2 bytes)
    
- `PC` (2 bytes) ← the return address
    
- `DPC` (2 bytes) ← a partner value this CPU uses
    

So the stack layout after that 50-byte buffer is:

`[ 50 bytes of username ][ S0 ][ FP ][ PC ][ DPC ]`

That means the saved **PC** sits at offset **50 + 2 + 2 = 54**.

> Bonus quirk: this CPU stores addresses little-endian (low byte first), and it expects both `PC` _and_ `DPC` to be present. If you only overwrite `PC`, you often just crash. Set `DPC` to `0x0000` and life is good.

---

## Finding the right jump target (`@win`)

From the trace, there’s a label:

`@win:   0236: RDB A0, (15)   0238: BRR.GE  @win+0xa   ...`

You don’t need the details; the important bit is the address: **`0x0236`** (the start of `@win`). That’s where we want the program to “return” to.

---

## Putting it together

- **Padding** to the saved PC: 54 bytes
    
- **PC** (return address): `0x0236` → `b"\x36\x02"` (little-endian)
    
- **DPC**: `0x0000` → `b"\x00\x00"`
    
- **Newline** at the end (the program reads a line)
    

So our payload is:

`"A" * 54 + "\x36\x02" + "\x00\x00" + "\n"`

When you send that, the service echoes your username. The `\x36` (`'6'`) shows up as a literal `6` in the echoed “Welcome, …” line—don’t worry, that’s just the low byte of the address passing through the “printable char” filter. After echoing, control returns using our faked `PC/DPC`, lands in `@win`, and prints the flag.

---

## Reproduce it quickly

Disable terminal software flow control so `^S` doesn’t freeze you:

`stty -ixon -ixoff`

Then send the exploit:

`python3 - <<'PY' | nc -N sunshinectf.games 25701 import sys def p16(x): return bytes((x & 0xff, (x>>8)&0xff)) PAD = 54 PC  = 0x0236 DPC = 0x0000 sys.stdout.buffer.write(b"A"*PAD + p16(PC) + p16(DPC) + b"\n") PY`

Expected result:

`Enter username: Welcome, AAAAA...6! sun{th1s_i5_ju57_7h3_t1p_0f_7h3_spEAR}`

---

## How we confirmed the numbers (optional)

- Short names: program returns normally → too small to touch the return address.
    
- Super long names: crash with “PANIC!” → we’re overwriting _past_ the return address with junk.
    
- Sweep lengths: around **54** bytes is where things get interesting.
    
- Append the target address: `b"\x36\x02"` (0x0236 little-endian).
    
- Add the extra required word for this CPU’s calling convention: `DPC = 0x0000`.
    

---

## Common pitfalls & tips

- **No newline?** The program waits for end-of-line. Always end your payload with `\n`.
    
- **Order matters.** It’s little-endian: `0x0236` → `\x36\x02`.
    
- **DPC is required.** Because the epilogue pops `{PC-DPC}`, omit `DPC` and you’ll likely crash.
    
- **Seeing a stray `6` in your name?** That’s just the `\x36` byte being printed back. Totally fine.
    

---

## Takeaways

- Even on a “weird” CPU, stack overflows are the same story: find the buffer, find the return address, point it to a win path.
    
- Understand the function epilogue/calling convention. Here, the extra `DPC` word was the key to turning crashes into clean wins.
    
- A tiny amount of disassembly goes a long way: `SUB SP, 0x32` (buffer size) and `POP {…, PC-DPC}` (what we must overwrite) basically gave us the whole exploit plan.
    

Have fun, and congrats on poking the satellite just right 🚀
