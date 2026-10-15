# Workbook: Reverse Engineering Challenges

## Exercise 1: Reverse Engineering Compiled Python Binaries

[Download](https://nhc.ehgn.nl/uploads/5643eb7da1b26c63827fe274a8344830efa9f819d7d00b5bfd96dd7534b5e581/password-panel_handout.zip) the challenge.

### General Information
This challenge gives you a Windows executable called main. When you run it, it asks for a 50 character password
and tells you whether you got it right. The flag is the password.

The program was written in Python and packed with PyInstaller. PyInstaller bundles the Python interpreter together
with the program's bytecode (.pyc files). Bytecode can be unpacked and turned back into readable source with the
right tools.

### Challenge 0: Identify the executable type

Find how this executable was created. What was the original programming language that the challenge maker use and which software did they use
to create an executable?

<details><summary>Hint 1</summary>
Try running `strings -n 10 name_of_the_file` to see if any information is leaked by the executable strings.

Alternatively, use Detect It Easy.
</details>

<details><summary>Answer</summary>
The executable was written in Python and compiled with PyInstaller.
</details>

### Challenge 1: Unpack the Executable

Extract the provided executable (main or main, one for Linux and one for Windows).

<details>
  <summary>Hint 1</summary>
The executables were created with PyInstaller. PyInstaller packages Python scripts, modules, and dependencies into
a single archive format. Tools like pyinstxtractor-ng or [its web version](https://pyinstxtractor-web.netlify.app/) can unpack this archive to discover compiled Python code (.pyc files).
</details>

<details>
  <summary>Answer</summary>
Run this in the folder that contains main:
  
docker run --rm -v "$PWD":/pwd -w /pwd python:3.12-slim bash -c 'pip install pyinstxtractor-ng && pyinstxtractor-ng main'

  
If you don't have docker you can use the web version as well.

</details>

### Challenge 2: Reading the Entrypoint

You should now have a directory that contains the files extracted from the executable. Find the "main" file.

.pyc files are not human-readable. A decompiler like [pylingual](https://pylingual.io/) or [decompiler.com](https://www.decompiler.com/python) can
decompile Python bytecode (.pyc files) back into its original form. The result is not always identical to the original code
(variable names and comments may be lost), but it is usually easy to read.

Decompile the main file and read its contents

<details> <summary>Answer</summary>
Use <code>pylingual main_extracted/main.pyc</code>

Or drop main.pyc into [decompiler.com](https://www.decompiler.com/python)

The contents are:
```
from colorama import Fore, Style, init
from check import check
from cheque import cheque
from czech import czech
from chic import chic
from chech import chech
init()
# ... banner printing ...
guess = input(Fore.CYAN + Style.BRIGHT + '  Password> ' + Style.RESET_ALL)
a, b, c, d, e = (guess[0:10], guess[10:20], guess[20:30], guess[30:40], guess[40:50])
if len(guess) == 50 and check(a, 'hC_Zyp{CHN') and cheque(b, b'N>y1!G&EF1M@2MMPgyfdNlHjYQ8ijMNL4*OJv}`=') and czech(c, b'\x00\xd3\x0cV5\x86\xc0;a\xf1W+P\xae@W') and chic(d, b'tiZ_HuNeat') and chech(e, 834116172812670004082183):
    print('[+] ACCESS GRANTED')
else:
    print('[-] ACCESS DENIED')
```
</details>

> Understand the contents of the entrypoint before going further.

### Challenge 3: Find the files where the imported functions are defined

You have a directory that contains the files extracted from the executable. Some of these
files are standard Python libraries like libz.so.1 and some of them are custom. By finding the code
that the challenge creator wrote themselves, you can find the custom functions that main.py imports.
(i.e. check, cheque, czech, chic, chech)

Find and decompile the files that define the functions that main.py imports.

<details>
  <summary>Hint 1</summary>
Look into the directories with scary names too :)
You can also run commands like "find . -name '*.pyc'" on Linux. This way you don't have to do
look into every directory manually.
</details>

<details>
  <summary>Hint 2</summary>
"from check import check" means that check is being imported from a file called check.py (in our case check.py would be check.pyc)
</details>

<details><summary>Answer</summary>
Run
  
```bash
pylingual main_extracted/PYZ.pyz_extracted/check.pyc
pylingual main_extracted/PYZ.pyz_extracted/chech.pyc
pylingual main_extracted/PYZ.pyz_extracted/chic.pyc
pylingual main_extracted/PYZ.pyz_extracted/czech.pyc
pylingual main_extracted/PYZ.pyz_extracted/cheque.pyc
```

Or use the website version (pylingual.io) to decompile the aforementioned files.
</details>

### Challenge 4: Find the first piece  

Reverse the `check` function and discover the first piece.

Piece 1: check
def check(piece, target):
    return piece[::-1] == target


<details> <summary>Hint</summary>
  <code>[::-1]</code> is a Python slice. What does it do to a string?
</details>
<details> <summary>Answer</summary>
[::-1] reverses a string, so the piece is the target reversed.

Open a Python interpreter an run `'hC_Zyp{CHN'[::-1]`:
```
>>> 'hC_Zyp{CHN'[::-1]
'NHC{pyZ_Ch'
```
</details>

### Challenge 5: Find the second piece  
Reverse the `cheque` function and recover the second piece.

<details> <summary>Hint 1</summary>
  Look at the order of the encodings. Which one is applied first,and which one last?
</details>

<details> <summary>Hint 2</summary>

Every <code>encode</code> in the <code>base64</code> module has a matching <code>decode</code>.

To undo a chain, apply the decodes in the opposite order.
</details>

<details> <summary>Answer</summary>
The piece is first encoded with base64, then base32, then base85. To get it back we decode with base85 first, then base32, then base64:

```
>>> import base64
>>> base64.b64decode(base64.b32decode(base64.b85decode(b'N>y1!G&EF1M@2MMPgyfdNlHjYQ8ijMNL4*OJv}`=')))
b'3Cks_Fr0M_'
```
</details>


### Challenge 6: Find the third piece
Reverse the `czech` function and recover the second piece.

<details>
  <summary>Hint 1</summary> AES is a symmetric cipher. The same key that encrypts can decrypt.
</details>

<details> <summary>Hint 2</summary>
  Where is the key? Where is the initialization vector (IV)? Are they secret?
</details>

<details> <summary>Answer</summary>

The key (The fun has only) and the IV (just begun. glhf) are both hardcoded in the
program, so we can just decrypt the target. The piece was padded to a full 16 byte
block before encrypting, so we remove the padding afterwards.
```
>>> from Crypto.Cipher import AES
>>> from Crypto.Util.Padding import unpad
>>> cipher = AES.new(b'The fun has only', AES.MODE_CBC, b'just begun. glhf')
>>> unpad(cipher.decrypt(b'\x00\xd3\x0cV5\x86\xc0;a\xf1W+P\xae@W'), AES.block_size)
b'1Nv3Rs3_T0'
```

> Note: If the Crypto import fails, install it with `pip install pycryptodome`.
</details>



### Challenge 7: Find the fourth piece: 
Reverse the `chic` function and recover the second piece.

This is a Vigenère cipher on a custom alphabet. A Caesar cipher (e.g. ROT13)
shifts every letter by the same amount. A Vigenère cipher shifts each letter by a
different amount, taken from a repeating key. With the key KEY, the first character
is shifted by K, the second by E, the third by Y, the fourth by K again, and so on.

Here the shift is the position of the key letter in ALPHABET, and the result
wraps around at the end of the alphabet (that is the % len(ALPHABET)).

<details> <summary>Hint 1</summary>
  Encryption adds the key position. What do you do to undo an addition?
</details>

<details><summary>Hint 2</summary>
Subtracting can give a negative number. Adding <code>len(ALPHABET)</code> before taking the remainder keeps it positive.
</details>

<details> <summary>Answer</summary>
Instead of c + k we use c - k, written as c + (len(ALPHABET) - k) so the number never goes negative:

```
>>> from itertools import cycle
>>> KEY = 'KEY'
>>> ALPHABET = '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ_abcdefghijklmnopqrstuvwxyz{}'
>>> out = []
>>> for c, k in zip('tiZ_HuNeat', cycle(KEY)):
...     c = ALPHABET.index(c)
...     k = ALPHABET.index(k)
...     out.append(ord(ALPHABET[(c + (len(ALPHABET) - k)) % len(ALPHABET)]))
>>> bytes(out)
b'_V1G3N3R3_'
```
</details>

### Challenge 8: Find the fifth piece
Reverse the `chech` function and recover the second piece.

<details><summary>Hint 1</summary>
This check turns the piece into one big number and then does some arithmetic on it:

- int.from_bytes(raw, 'big') reads the 10 bytes as a single 80 bit integer x.
- It computes 1337 * x + 424242.
- It keeps only the last 10 bytes with % 256**10. This is the same as saying "the result wraps around at 80 bits".

The result has to equal the target. So we are solving the equation 1337 * x + 424242 = target for x, in an 80 bit world.

A solver like z3 is good at this kind of equation. It takes a set of constraints and finds values that satisfy them.
</details>

<details>
  <summary>Hint 1</summary>
  How long is the piece? Look at how <code>e</code> is created in the entrypoint.
</details>

<details> <summary>Hint 2</summary>
Use <code>z3</code> with a bitvector of the right size (<code>pip install z3-solver</code>), then turn the number back
  into bytes with <code>long_to_bytes</code> from <code>Crypto.Util.number</code>.
</details>

<details> <summary>Answer</summary>
The slice guess[40:50] is 10 bytes, so x is an 80 bit number.
  
```
>>> from z3 import *
>>> from Crypto.Util.number import long_to_bytes
>>> s = Solver()
>>> x = BitVec("x", 10*8)  # 10 bytes
>>> s.add((1337 * x + 424242) % 256 ** 10 == 834116172812670004082183)
>>> s.check()
sat
>>> s.model()
[x = 397571092732711202467965]
>>> long_to_bytes(s.model()[x].as_long())
b'T0_Crypt0}'
```

> Can you solve this one without z3?

### Finally

Build the full password and get the flag
<details><summary>Answer</summary>
NHC{pyZ_Ch3Cks_Fr0M_1Nv3Rs3_T0_V1G3N3R3_T0_Crypt0}
</details>


## Exercise 2: Reversing a Self-Decrypting Binary

### General Information

Tools used for this exercise:

| Tool | What it does |
|---|---|
| `strings`, `nm` or `objdump -t` | Quick look at text and symbols in the binary |
| `objdump` | Disassembler. Ghidra, IDA or Binary Ninja work too |
| Python 3 with `pycryptodome` | Decrypting the code, re-implementing the hash |
| `gdb` (optional) | For the dynamic route |

---

### Challenge 1: Run the program and understand its behavior

<details>
<summary>Answer</summary>
Option 3 only shows up on one specific dream level.

Each level shows a menu. "Go deeper" (1) takes you one level down and "Kick" (2) takes you one level up. The extra option "Implant" (3) only appears at level 3. Type `1`, `1`, `3` to get there. The program then asks:

```
> Enter idea:
```

If you pick 3 on any other level you get "Mission failed" and the program exits. Kicking at level 1 also ends the program.
</details>

We can now start solving the challenge.

## Step 1: Type a few different ideas. What does the program print?

<details>
<summary>Hint</summary>
Try <code>AAAAAAAA</code>, then something with only a single letter changed.
</details>

<details>
<summary>Answer</summary>

The program prints a `> Hash:` line with hex bytes. The hash always has the same number of bytes as your input, and `AAAAAAAA` gives:

```
> Hash: 8c3f9fde8c3f9fde
```

Keep this value. It is a test vector you can use later to check your own re-implementation of the hash function.

Note that the hash has the same length as the input and that some characters repeat in this example. Both are hints that the function is some kind of simple scrambling and not a real cryptographic hash.
</details>

> Why is it useful that the output length equals the input length?

<details>
<summary>Answer</summary>
It means the function is almost certainly a reversible transformation (a permutation of bits plus some xors), and not a one-way hash. It also tells us the flag length: if we find the target hash and it has 37 bytes, the flag has 37 characters.
</details>

---

## Step 2: First Look at the Binary

### Challenge 2: List the readable strings and function names. Can you identify the encryption algorithm and the functions the program uses?

<details>
<summary>Hint 1</summary>
<code>strings -n 6 main</code> shows text. <code>objdump -t main</code> shows the symbol table (things like variable and function names used by the executable).
</details>

<details>
<summary>Hint 2</summary>
Symbols that are in the `.text` section are function names. For more information, see what the .text section does in an executable
</details>

<details>
<summary>Answer</summary>

```sh
strings -n 6 main'
```

```
secure-p
...
mmap@GLIBC_2.2.5
...
AES_ECB_decrypt
...
AES_init_ctx
...
inception
...
AES_ECB_encrypt
```

```sh
objdump -t main
```

```
0000000000001229 g     F .text    0000000000000072              print_hash
...
000000000000129b g     F .text    00000000000002d1              inception
000000000000156c g     F .text    0000000000000108              main
0000000000001933 g     F .text    000000000000002a              AES_init_ctx
...
00000000000027a1 g     F .text    000000000000002a              AES_ECB_encrypt
00000000000027cb g     F .text    000000000000002a              AES_ECB_decrypt
```

<details>
<summary>Some more information that will help with understanding the challenge</summary>

Things that stand out:

- AES functions in a program that has nothing to do with encryption on the surface.
- `mmap` (and, if you look at the imports, `mprotect`), which are used to create memory and change its permissions.
- A function named `inception`.
- The start of a string, `secure-p`. It is cut off because the compiler loads the rest of it in a different way (more on that below).
</details>

> What do `mmap` and `mprotect` together suggest about what the program does with that memory?
  
Normal data is not executable. If a program creates memory, fills it with something and then makes it executable with <code>mprotect</code>, it is preparing to run code that did not exist in the file as plain code. Combined with the AES functions, the likely story is: decrypt code, put it in new memory, make it executable, call it.
</details>

---

## Step 3: Reading `inception()`

### General Information

Disassemble the one function that does the work:

```sh
objdump -d -M intel main --disassemble=inception
```

You do not have to read every line. Look for calls to functions you recognize (`AES_init_ctx`, `AES_ECB_decrypt`, `mmap`, `mprotect`, `scanf`, `print_hash`) and read the instructions around them.

### Challenge 3: Find the AES key

<details>
<summary>Hint 1</summary>
Look right before the call to <code>AES_init_ctx</code>. Two big constants are loaded with <code>movabs</code>.
</details>

<details>
<summary>Hint 2</summary>
x86 is little endian, so the bytes of each constant come out in reverse order when you read the hex from left to right.
</details>

<details>
<summary>Answer</summary>

```
movabs rax,0x702d657275636573
movabs rdx,0x64726f77737361
mov    QWORD PTR [rbp-0x4a0],rax
mov    QWORD PTR [rbp-0x498],rdx
...
call   1933 <AES_init_ctx>
```

Reverse the bytes of each constant:

```py
>>> bytes.fromhex('702d657275636573')[::-1]
b'secure-p'
>>> bytes.fromhex('0064726f77737361')[::-1]
b'assword\x00'
```

The key is `secure-password` plus a zero byte, which makes exactly 16 bytes (AES-128).
</details>

### Challenge 4: Find where the encrypted code is stored and how long it is

<details>
<summary>Hint 1</summary>
Before the key is set up there is a block copy (<code>rep movs</code>). Look at the <code>lea</code> that loads the source address and the <code>mov ecx, ...</code> that sets the count.
</details>

<details>
<summary>Hint 2</summary>
<code>rep movs QWORD</code> copies 8 bytes per count.
</details>

<details>
<summary>Answer</summary>

```
12b9: lea    rax,[rbp-0x3e0]          ; destination, on the stack
12c0: lea    rdx,[rip+0x1e99]         ; source: address 0x3160 (in .rodata)
12c7: mov    ecx,0x7a                 ; 0x7a = 122 qwords
...
12d2: rep movs QWORD PTR es:[rdi],QWORD PTR ds:[rsi]
```

- Source address: `0x3160`. The `.rodata` section starts at `0x3000` and its file offset is also `0x3000` (check with `readelf -S main`), so the data sits at file offset `0x3160`.
- Length: 122 × 8 = 976 bytes.

The decryption loop confirms this. It runs in steps of `0x10` (16 bytes, one AES block) while the counter is at most `0x3cf`:

```
1340: add    DWORD PTR [rbp-0x56c],0x10
...
134d: cmp    eax,0x3cf
1352: jbe    131c
```

That is 61 blocks of 16 bytes, which is 976 bytes again.
</details>

> The loop calls `AES_ECB_decrypt` once per 16 byte block with no IV and no chaining. What does that tell you about the mode, and what is the practical difference for us compared with CBC?

<details>
<summary>Answer</summary>
It is ECB mode: every block is decrypted on its own. We do not need an IV, and we can decrypt the blob with one call to any AES library using ECB mode.
</details>

### Challenge 5: What happens after decryption? How is the hidden function called?

<details>
<summary>Hint</summary>
Follow the <code>mmap</code> call, then the copy, then <code>mprotect</code>. Finally look at what is loaded into registers before the <code>call r8</code>.
</details>

<details>
<summary>Answer</summary>

1. `mmap(NULL, 0x1000, 3, 0x22, -1, 0)` creates one page that is readable and writable (3 = `PROT_READ | PROT_WRITE`, `0x22` = private and anonymous).
2. The decrypted bytes are copied into that page (second `rep movs`).
3. `mprotect(page, 0x1000, 5)` changes it to readable and executable (5 = `PROT_READ | PROT_EXEC`).
4. The page address is called as a function with four arguments:

| Register | Value |
|---|---|
| `rdi` | pointer to the user input |
| `esi` | length of the input (from `strlen`) |
| `rdx` | address of `malloc` |
| `rcx` | address of `free` |

5. After the call, `print_hash` prints the input buffer. So the function changes the input in place.

The reason `malloc` and `free` are passed in as arguments is that the hidden code is raw machine code with no way to link to libc, so the caller hands it the functions it needs.
</details>

---

## Step 4: Decrypting the Hidden Function

### Challenge 6: Get the decrypted machine code (static route)

<details>
<summary>Hint 1</summary>
Read 976 bytes from file offset <code>0x3160</code> of <code>main</code>.
</details>

<details>
<summary>Hint 2</summary>
<code>pip install pycryptodome</code>, then use <code>AES.new(key, AES.MODE_ECB)</code>.
</details>

<details>
<summary>Answer</summary>

```py
from Crypto.Cipher import AES

data = open('main', 'rb').read()
blob = data[0x3160:0x3160 + 0x3d0]        # 976 bytes

key = b'secure-password\x00'
code = AES.new(key, AES.MODE_ECB).decrypt(blob)

open('hash_func.bin', 'wb').write(code)
print(code[:16].hex())
```

Output of the last line:

```
f30f1efa554889e54883c48048897d98
```

`f3 0f 1e fa` is `endbr64` and `55 48 89 e5` is `push rbp; mov rbp, rsp`. That is how a normal compiled function starts, so the decryption worked.
</details>

> How can you tell that you have the right key without running anything?

<details>
<summary>Answer</summary>
With the wrong key, ECB decryption gives random-looking bytes. With the right key the output starts with a typical function prologue (<code>endbr64</code>, <code>push rbp</code>, <code>mov rbp, rsp</code>) and the disassembly makes sense all the way through.
</details>

### Alternative: Get the decrypted code with gdb (dynamic route)

<details>
<summary>Hint</summary>
The decrypted code is in memory right before <code>mprotect</code> is called. The first argument of <code>mprotect</code> is the address of the page that holds the code.
</details>

<details>
<summary>Answer</summary>

```sh
gdb ./main
(gdb) break mprotect
(gdb) run
```

Type `1`, `1`, `3` in the program to reach the point where `inception()` runs. When gdb stops at `mprotect`, the page address is in `rdi`:

```
(gdb) dump binary memory hash_func.bin $rdi $rdi+976
```

You end up with the same `hash_func.bin` as the static route.
</details>

---

## Step 5: Reading the Hash Function

### Challenge 7: Disassemble `hash_func.bin`

<details>
<summary>Hint</summary>
The file is raw machine code with no headers. Tell the tool what architecture it is. In Ghidra, import it as a raw binary with language x86 64-bit little endian.
</details>

<details>
<summary>Answer</summary>

```sh
objdump -D -b binary -mi386:x86-64 -M intel hash_func.bin
```

The function is about 0x3c6 bytes long. The rest of the 976 bytes is zero padding (it shows up as `...` at the end).
</details>

### Challenge 8: Work out what the function does, step by step

<details>
<summary>Hint 1</summary>
Start with the function prologue. Four arguments are saved to the stack, then two constants and the number 4 are stored. After that it calls <code>rdx</code> (malloc) and copies the input into the new buffer.
</details>

<details>
<summary>Hint 2</summary>
The function consists of five loops that each walk over all bytes (<code>cmp ..., [rbp-0x6c]</code> is the comparison with the length). Look at what each loop reads and writes.
</details>

<details>
<summary>Hint 3</summary>
Three loops read bytes at an offset and combine two neighbouring bytes with <code>shl</code>, <code>sar</code> and <code>or</code>. Two loops use <code>xor</code> with a 4 byte constant. Read the offsets from the <code>mov DWORD PTR [rbp-...], N</code> lines at the start of each loop.
</details>

<details>
<summary>Answer</summary>

**Setup (offsets 0x00 to 0x85)**

| Instruction | Meaning |
|---|---|
| `mov DWORD PTR [rbp-0x10],0xb43016ae` | 4 byte constant K1 |
| `mov DWORD PTR [rbp-0xc],0xa0175f75` | 4 byte constant K2 |
| `mov DWORD PTR [rbp-0x34],0x4` | key length for the xor loops is 4 |
| `call rdx` | `tmp = malloc(len)` |
| loop at 0x5a | `tmp[i] = input[i]` for all i |

Since x86 is little endian, K1 as bytes is `ae 16 30 b4` and K2 is `75 5f 17 a0`.

**The five loops** (n is the input length, indexes wrap around with a modulo)

| Loop at | Offset constants | What it does |
|---|---|---|
| 0x91 | `2` and `1` | `input[i] = (tmp[i-3] << 7) \| (tmp[i-2] >> 1)` |
| 0x162 | uses K1 | `tmp[i] = input[i] ^ K1[i % 4]` |
| 0x1ab | `5` and `1` | `input[i] = (tmp[i+5] << 1) \| (tmp[i+6] >> 7)` |
| 0x289 | uses K2 | `tmp[i] = input[i] ^ K2[i % 4]` |
| 0x2d2 | `0xc` and `1` | `input[i] = (tmp[i-13] << 7) \| (tmp[i-12] >> 1)` |

Then `free(tmp)` is called and the function returns.

All indexes are taken modulo n. The assembly has extra code for this (the `test eax,eax`, `jns` and `idiv` parts) because the C modulo operator gives a negative result for negative numbers and the compiler has to correct that. In Python `%` already does the right thing.

**What the three shift loops really do**

Each output byte takes the top 7 bits of one byte and the lowest bit of its neighbour. If you glue all bytes together into one long bit string, that is just a rotation of the whole string:

| Loop | Bit string operation |
|---|---|
| 0x91 | rotate right by 2 × 8 + 1 = 17 bits |
| 0x1ab | rotate left by 5 × 8 + 1 = 41 bits |
| 0x2d2 | rotate right by 12 × 8 + 1 = 97 bits |

So the whole "hash" is:

```
rotate right 17  ->  xor K1  ->  rotate left 41  ->  xor K2  ->  rotate right 97
```

where "xor K" means xor with the 4 byte constant repeated over the whole string.
</details>

> The task description calls this an "advanced proprietary cryptographic hash function". Name two properties of a real cryptographic hash that this function does not have.

<details>
<summary>Answer</summary>
It is reversible (a hash must be one way), and the output length depends on the input length (a hash has a fixed output size). It also has no avalanche effect: changing one input bit changes exactly one output bit, because rotations and xors just move bits around.
</details>

### Challenge 9: Re-implement the function in Python and check it against the real program

<details>
<summary>Hint 1</summary>
Follow the five loops exactly, using <code>tmp</code> and <code>input</code> as two separate byte arrays. Mind which loop reads from which array.
</details>

<details>
<summary>Hint 2</summary>
Test vector from Step 1: the input <code>AAAAAAAA</code> must give <code>8c3f9fde8c3f9fde</code>.
</details>

<details>
<summary>Answer</summary>

```py
K1 = bytes.fromhex('ae1630b4')
K2 = bytes.fromhex('755f17a0')

def hash_bytes(data):
    n = len(data)
    inp = bytearray(data)
    tmp = bytearray(data)

    # loop 1: shift 2 bytes, then 1 bit
    for i in range(n):
        a = tmp[(i - 2) % n]
        b = tmp[(i - 3) % n]
        inp[i] = ((b << 7) | (a >> 1)) & 0xff
    # loop 2: xor with K1
    for i in range(n):
        tmp[i] = inp[i] ^ K1[i % 4]
    # loop 3: shift 5 bytes, the other direction
    for i in range(n):
        a = tmp[(i + 5) % n]
        b = tmp[(i + 6) % n]
        inp[i] = ((a << 1) | (b >> 7)) & 0xff
    # loop 4: xor with K2
    for i in range(n):
        tmp[i] = inp[i] ^ K2[i % 4]
    # loop 5: shift 12 bytes, then 1 bit
    for i in range(n):
        a = tmp[(i - 12) % n]
        b = tmp[(i - 13) % n]
        inp[i] = ((b << 7) | (a >> 1)) & 0xff
    return bytes(inp)
```

```py
>>> hash_bytes(b'AAAAAAAA').hex()
'8c3f9fde8c3f9fde'
```

It matches the program. If yours does not, look at loop 3 first. It is the only one that rotates to the left, and it is easy to copy loop 1 by mistake.
</details>

---

## Step 6: Finding the Target Hash

### Challenge 10: Find the bytes that the hash is compared against

<details>
<summary>Hint 1</summary>
Look at the end of <code>inception()</code>, just after the call to <code>print_hash</code>. There is a loop that compares two arrays byte by byte.
</details>

<details>
<summary>Hint 2</summary>
Four <code>movabs</code>/<code>mov</code> pairs and two smaller moves fill the array on the stack. Reverse the bytes of each immediate, just like with the key.
</details>

<details>
<summary>Answer</summary>

```
14a0: movabs rax,0xa8c4272696cf06ae        ; ae 06 cf 96 26 27 c4 a8
14a9: movabs rdx,0x4388af19a35d1107        ; 07 11 5d a3 19 af 88 43
14c1: movabs rax,0xc7852eb5c710879b        ; 9b 87 10 c7 b5 2e 85 c7
14cb: movabs rdx,0x6606ab96d1032534        ; 34 25 03 d1 96 ab 06 66
14e3: mov    DWORD PTR [rbp-0x470],0x518b2c16   ; 16 2c 8b 51
14ed: mov    BYTE PTR  [rbp-0x46c],0xb4         ; b4
```

Put together in order, this is 37 bytes:

```
ae06cf962627c4a807115da319af88439b8710c7b52e85c7342503d196ab0666162c8b51b4
```

The loop right after compares `input[i]` with this array for each of the 37 positions, where `input` has already been replaced by its hash. So the input we want is the one whose hash is this array.
</details>

> Does the program check the length of the input in any other way?

<details>
<summary>Answer</summary>
No. It only compares the first 37 bytes. The hash function keeps the length, so a 37 byte flag is what we expect, and it matches the flag format length.
</details>

---

## Step 7: Reversing the Hash

### Challenge 11: Recover the flag from the target bytes

<details>
<summary>Hint 1</summary>
Every step of the hash can be undone. Go through the five steps in reverse order.
</details>

<details>
<summary>Hint 2</summary>
Xor undoes itself. A rotation to the right by k bits is undone by a rotation to the left by k bits, and the other way around.
</details>

<details>
<summary>Hint 3</summary>
Treat the 37 bytes as one 296 bit big endian integer. Rotations are then simple shifts with a mask.
</details>

<details>
<summary>Answer</summary>

The hash is: rotate right 17, xor K1, rotate left 41, xor K2, rotate right 97.

The inverse is: rotate left 97, xor K2, rotate right 41, xor K1, rotate left 17.

```py
K1 = bytes.fromhex('ae1630b4')
K2 = bytes.fromhex('755f17a0')
target = bytes.fromhex(
    'ae06cf962627c4a807115da319af88439b8710c7b52e85c7342503d196ab0666162c8b51b4'
)

n = len(target)          # 37
bits = n * 8             # 296
mask = (1 << bits) - 1

def rotr(x, k):
    k %= bits
    return ((x >> k) | (x << (bits - k))) & mask

def rotl(x, k):
    return rotr(x, bits - (k % bits))

def xor_key(x, key):
    return x ^ int.from_bytes((key * n)[:n], 'big')

def backward(b):
    x = int.from_bytes(b, 'big')
    x = rotl(x, 97)
    x = xor_key(x, K2)
    x = rotr(x, 41)
    x = xor_key(x, K1)
    x = rotl(x, 17)
    return x.to_bytes(n, 'big')

print(backward(target).decode())
```

Output:

```
Trojan{n0_s3cur1ty_thr0ugh_0bscur1ty}
```

Check it against the real program:

```sh
printf '1\n1\n3\nTrojan{n0_s3cur1ty_thr0ugh_0bscur1ty}\n' | ./main
```

The program prints the same hash as the target bytes, followed by:

```
Mission completed successfully.
You have unlocked Roberts mind, you are a free man now!
```

You can also feed `hash_bytes` from Step 5 the flag and compare the result with the target. If you did the byte-by-byte version, this is a good way to check both directions at once.
</details>

