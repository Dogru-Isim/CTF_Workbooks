# Workbook: Reverse Engineering Challenges

## Exercise 1: Reverse Engineering Compiled Python Binaries

[Download](https://nhc.ehgn.nl/uploads/5643eb7da1b26c63827fe274a8344830efa9f819d7d00b5bfd96dd7534b5e581/password-panel_handout.zip) the challenge.

### General Information
This challenge gives you a Windows executable called main. When you run it, it asks for a 50 character password
and tells you whether you got it right. The flag is the password.

The program was written in Python and packed with PyInstaller. PyInstaller bundles the Python interpreter together
with the program's bytecode (.pyc files). Bytecode can be unpacked and turned back into readable source with the
right tools.

### Challenge 1: Unpack the Executable

Extract the provided executable (main or main, one for Linux and one for Windows).

<details>
  <summary>Hint 1</summary>
The executables were created with PyInstaller. PyInstaller packages Python scripts, modules, and dependencies into
a single archive format. Tools like pyinstxtractor-ng can unpack this archive to discover compiled Python code (.pyc files).
</details>

<details>
  <summary>Answer</summary>
Run this in the folder that contains main:
  
docker run --rm -v "$PWD":/pwd -w /pwd python:3.12-slim bash -c 'pip install pyinstxtractor-ng && pyinstxtractor-ng main'

  
If you don't have docker you can either install it or run "pip install pyinstxtractor-ng && pyinstxtractor-ng main"

</details>

### Challenge 2: Reading the Entrypoint

You should now have a directory that contains the files extracted from the executable. Find the "main" file.

.pyc files are not human-readable. A decompiler like pylingual or [decompiler.com](https://www.decompiler.com/python) can
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
pylingual main._extracted/PYZ.pyz_extracted/check.pyc
pylingual main._extracted/PYZ.pyz_extracted/chech.pyc
pylingual main._extracted/PYZ.pyz_extracted/chic.pyc
pylingual main._extracted/PYZ.pyz_extracted/czech.pyc
pylingual main._extracted/PYZ.pyz_extracted/cheque.pyc
```
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
