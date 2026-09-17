# Shared Secrets

> **Difficulty:** Beginner → Intermediate

> **Category:** Cryptography

> **Estimated time:** 60–90 minutes

> **Tools:** Python 3, Linux terminal, text editor, browser

> **Source:** https://learn.cylabacademy.org/library/715?page=1&category=2&difficulty=1

---

## 0. Challenge overview

You have been given two files:

- `message.txt`

- `encryption.py`

A message was encrypted using a shared secret.

Something has gone wrong with the key exchange, however. One side appears to have leaked information that should have remained secret.

Your goal is to:

1. Understand the key-exchange scheme used by the program.

2. Identify which values are public and which value has leaked.

3. Reproduce the shared secret.

4. Determine how the shared secret becomes an encryption key.

5. Recover the plaintext.

6. Explain **why the cryptographic system failed**.

7. As a bonus, investigate how the implementation could be improved.

> **Important:** Do not search for the flag first. The purpose of this workbook is to practice the investigation process you would use on an unfamiliar CTF crypto challenge.

---

# Challenge 1: Reconnaissance

Before doing any mathematics, inspect what you have been given.

## 1.1 List the challenge files

From the challenge directory, run:

```
ls -lah
```

Then inspect the two files:

```
file message.txt
file encryption.py
```

### Your observations

**What type of file is `message.txt`?**

> Answer:

<br>

**What type of file is `encryption.py`?**

> Answer:

<br>

---

## 1.2 Read the source

Open the encryption script:

```
cat encryption.py
```

Or, if you prefer:

```
less encryption.py
```

Read the entire file before writing any code.

Look especially carefully at:

- random numbers being generated

- mathematical operations

- values written to `message.txt`

- anything named `secret`, `private`, `public`, `key`, or `shared`

- the final encryption operation

### Investigation checklist

Put a check next to each item once you have found it:

- A large prime or modulus

- A generator

- A private value

- A public value

- A shared secret

- The encrypted message

- Code that writes values to `file.txt` (`message.txt` is )

- The operation used to encrypt the plaintext

> Note: The code doesn't touch `message.txt`. Because it doesn't want you to lose the original data. randint(2, p-2Instead, it writes to `file.txt`.

---

## 1.3 Read the output

Now inspect the supplied values:

```
cat message.txt
```

You should see several large numbers and an encrypted message.

Do **not** try to understand every number yet.

Instead, make a table.

| Name              | Value type | Public or secret? | Where did you find it? |
| ----------------- | ---------- | ----------------- | ---------------------- |
| `g`               |            |                   |                        |
| `p`               |            |                   |                        |
| `A`               |            |                   |                        |
| `b`               |            |                   |                        |
| encrypted message |            |                   |                        |

### Question

Which value looks like it should have been secret but was nevertheless included in the supplied data?

> Answer:

<br>

---

# Challenge 2: Recognising Diffie–Hellman

You should now have seen expressions involving modular exponentiation.

The important mathematical pattern is:

```
public_value = g^private_value mod p
```

This is the basic structure of **Diffie–Hellman key exchange**.

You do not need to become a number-theory expert to understand this challenge.

## 2.1 The basic idea

Imagine Alice chooses a private number `a`:

```
a = Alice's private value
```

and Bob chooses a private number `b`:

```
b = Bob's private value
```

Both parties know:

```
g = public generator
p = public prime
```

Alice calculates:

```
A = g^a mod p
```

Bob calculates:

```
B = g^b mod p
```

They exchange `A` and `B`.

Alice can then calculate:

```
shared = B^a mod p
```

Bob can calculate:

```
shared = A^b mod p
```

Both calculations produce the same value:

```
g^(ab) mod p
```

The private values themselves do not need to be transmitted.

---

## 2.2 Work through a tiny example

Use these deliberately small parameters:

```
g = 5
p = 23
a = 6
b = 15
```

Calculate Alice's public value:

```
g = 5
p = 23
a = 6

A = pow(g, a, p)
print(A)
```

Now calculate Bob's public value:

```
b = 15

B = pow(g, b, p)
print(B)
```

Finally calculate the shared secret from both sides:

```
shared_alice = pow(B, a, p)
shared_bob = pow(A, b, p)

print(shared_alice)
print(shared_bob)
```

### Checkpoint

Do the two values match?

> Answer:

<br>

Why is this useful?

> Answer:

<br>

---

# Challenge 3: Find the leak

Now return to `encryption.py`.

Look for the variables corresponding to:

```
g
p
a
A
b
B
shared
```

You do **not** necessarily need all of them.

The challenge description tells us that one side of the exchange leaked something.

## 3.1 Identify the leaked value

Fill in the following:

```
Public generator:       __________________

Public modulus:         __________________

Known public value:     __________________

Leaked private value:   __________________

Encrypted message:      __________________
```

---

## 3.2 Why is the leak important?

In Diffie–Hellman, the shared secret is normally difficult for an attacker to calculate because they don't know the participants' private values.

In this challenge, however, a one of the private values has been leaked.

You already have:

```text
A = <public value>
b = <leaked private value>
p = <public prime>
```

You can use these values directly to calculate the shared secret:

```text
shared_secret = A^b mod p
```

Which is equivelent to the below code in Python:

```python
shared = pow(A, b, p)
```

### How can they both calculate the same shared secret key?

A is Alice's public value: `g^a`

B is Bob's public value: `g^b`

Bob normally calculates the shared secret using Alice's public value:

```text
A^b mod p (shared secret)
```

And Alice calculates the shared secret using Bob's public value:

```text
B^a mod p (shared secret)
```

These 2 produce the same result because:

```text
A^b mod p = (g^a)^b mod p = g^(ab) mod p
B^a mod p = (g^b)^a mod p = g^(ab) mod p
```

They are identical!



which is the same shared secret Alice calculates.

The important part is that **you don't need to know Alice's private value `a`**. The leaked `b` is enough when combined with the public value `A`.

<br>

---

# Challenge 4: Calculate the shared secret

Now we are ready to work with the actual challenge numbers.

Python's three-argument `pow()` is especially useful here:

```
pow(base, exponent, modulus)
```

## 4.1 Enter the challenge parameters

Copy the values from `message.txt` into the following code.

```
# Values copied from message.txt

g = ...
p = ...
A = ...
b = ...

print("g =", g)
print("p =", p)
print("A =", A)
print("b =", b)
```

Run it.

If Python prints the values without errors, continue.

---

## 4.2 Sanity checks

Before performing the calculation, check that the public value is within the expected range.

```python
assert 1 < A < p

print("Public value looks reasonable.")
```

Also inspect the size of the modulus:

```python
print("p has", p.bit_length(), "bits")
```

### Questions

Approximately how large is `p`?

> Answer:

<br>

Why would someone use such a large value instead of a tiny number like `23`?

> Answer:

<br>

---

## 4.3 Calculate the shared secret

Complete the missing line:

```
shared = __________________________

print("Shared secret:")
print(shared)
```

### Hint

You have:

```
A
b
p
```

and the mathematical operation is:

```
A^b mod p
```

---

## 4.4 Verify your result

We calculated the shared secret using:

```text
shared = A^b mod p
```

Because we used `mod p`, the result should be smaller than `p`.

Let's verify that:

```text
assert 0 < shared < p
```

If this runs without an error, our shared secret is in the expected range.

Now let's see how large the number is in bits:

```text
print("Shared secret has", shared.bit_length(), "bits")
```

---

## Challenge 5: Find the Encryption Key

Finding the Diffie–Hellman shared secret is not necessarily the same as finding the value that is actually used for encryption.

Go back to `encryption.py`.

Find the line where `shared` is used in the encryption operation:

```python
enc = bytes([x ^ (shared % 256) for x in flag])
```

Look closely at this part:

```python
shared % 256
```

This takes the shared secret and reduces it to a value between `0` and `255`.

That value is then used as the XOR key.

---

## 5.1 Identify the encryption key

Complete the flow:

```text
shared secret
      |
      v
 shared % 256
      |
      v
  1-byte key
      |
      v
 XOR with flag
```

### Question 1

What operation turns the shared secret into the value used for encryption?

> Answer:
> 
> A) `%`  --> modulus
> 
> B) `^`  --> bitwise XOR
> 
> C) `bytes()`  --> converts values to bytes

<br>

### Question 2

What is the size of the resulting key?

> Answer: ______ bits

<br>

### Hint

A value produced by:

```python
shared % 256
```

can be any integer from `0` through `255`.

How many bits are needed to represent values in that range?

<br>

---

## 5.2 Investigate the `%` operator

If the source contains something similar to:

```
key = shared % 256
```

experiment with it.

```
key = shared % 256

print("Key:", key)
print("Key in hex:", hex(key))
print("Key size:", key.bit_length(), "bits")
```

### Question

How many different values can a number modulo `256` produce?

> Answer:

<br>

Write that number in binary form, how many ones and zeros does it have?

> Answer:





<br>

---

## 5.3 Why is this interesting?

The Diffie–Hellman calculation may involve a very large secret.

But if the implementation eventually reduces it to 8 bits, what are the biggest and the lowest numbers shared secret can actually be?

> Answer:

<br>

### Think like an attacker

If you knew **nothing** about the Diffie–Hellman calculation but suspected that the encryption key was only one byte long, how many possible keys would you need to test?

> Answer:

<br>

This observation will become useful in the bonus challenge.

---

# Challenge 6: Understand XOR

The final encryption step uses XOR.

XOR operates bit-by-bit.

For a single bit:

| A   | B   | A XOR B |
| --- | --- | ------- |
| 0   | 0   | 0       |
| 0   | 1   | 1       |
| 1   | 0   | 1       |
| 1   | 1   | 0       |

The important property for this challenge is:

```
plaintext XOR key = ciphertext

ciphertext XOR key = plaintext
```

In other words, applying the same XOR operation twice restores the original value.

---

## 6.1 Small Python experiment

Run:

```
plaintext = b"HELLO"
key = 0x42

ciphertext = bytes(byte ^ key for byte in plaintext)

print("Plaintext: ", plaintext)
print("Ciphertext:", ciphertext)

recovered = bytes(byte ^ key for byte in ciphertext)

print("Recovered: ", recovered)
```

### Question

Did the recovered value match the original plaintext?

> Answer:

<br>

---

# Challenge 7: Decode the ciphertext

You should now have:

```
shared
key
```

You also need the encrypted message from `message.txt`.

## 7.1 Convert the encoded ciphertext

First determine how `encryption.py` represents the encrypted message.

Is it:

- hexadecimal?

- Base64?

- raw bytes?

- something else?

Do not guess, read the code.



The representation is used for printing the encrypted value to a human-readable format, it is not the encrypted data itself. How do you revert this operation so that you find the machine-readable format?

> Answer:

---

## 7.2 Write the decryption loop

Which of the 2 variables should you swap here to decrypt the data (remember XOR's previously described property):

```python
enc = bytes([x ^ (shared % 256) for x in flag])
print(enc)
```

### Hint

Make sure the encrypted data is back in its machine-readable format, not human-readable.



---

# Challenge 8: Build a complete solver

Now combine everything you have learned into a small standalone script.

Create:

```
solve.py
```

Paste the template into solve.py:

```
#!/usr/bin/env python3

# Challenge parameters
g = ...
p = ...
A = ...
b = ...

# Encrypted message
enc = "..."

# Recover the shared secret
shared = ...

decrypted = <your modified decryption loop here>

print(decrypted.decode())
```

Run it:

```
python3 solve.py
```



---

# Challenge 9: Validate the cryptographic reasoning

Getting a flag is not the end of the exercise.

You should be able to explain **why** your solution works.

Answer the following without looking at your code.

## 9.1 Diffie–Hellman

What are `g` and `p`?

> Answer:

<br>

What is a private value?

> Answer:

<br>

What is a public value?

> Answer:

<br>

What is the shared secret?

> Answer:

<br>

---

## 9.2 The vulnerability

What information was leaked?

> Answer:

<br>

Why does knowing that value allow an attacker to calculate the shared secret?

> Answer:

<br>

Why did we not need to recover the other party's private value?

> Answer:

<br>

---

## 9.3 Encryption

What operation was used to encrypt the message?

> Answer:

<br>

What value was used as the XOR key?

> Answer:

<br>

Why can the same operation decrypt the ciphertext?

> Answer:

<br>

---

# Challenge 11: Attack the weak key derivation

This is the first bonus challenge.

Suppose you did **not** notice the leaked secret.

Suppose instead you noticed that the encryption key is only one byte.

How many possible keys are there?

Write a brute-force script.

```
ciphertext = bytes.fromhex("...")

for key in range(256):
    plaintext = bytes(byte ^ key for byte in ciphertext)

    # What condition could you use to recognise the flag?
    if ______________________________:
        print("Possible key:", key)
        print(plaintext)
```

### Hint

CTF flags have a predictable prefix.

Don't hard-code the complete flag.

---

## 11.1 Why does this work?

The challenge may use a very large Diffie–Hellman modulus.

Yet the final encryption key has only a tiny number of possible values.

Explain the difference between:

```
strength of the shared secret
```

and:

```
strength of the derived encryption key
```

> Answer:

<br>

---

# Challenge 12: What would a secure implementation look like?

The challenge contains two separate lessons.

## Problem 1 — Secret leakage

A private Diffie–Hellman value must remain secret.

Imagine an application contains:

```
logger.info("client private key = %s", b)
```

Why is this dangerous?

> Answer:

<br>

Where else might accidental key leakage occur?

- Debug logs

- Error messages

- Crash reports

- Source repositories

- Configuration files

- Backups

- Command-line arguments

- Monitoring systems

- Other: __________________

---

## Problem 2 — Weak key derivation

Reducing a large shared secret to:

```
shared % 256
```

throws away almost all of the shared secret.

A production protocol should use an established key-derivation mechanism rather than inventing a custom one. HKDF, for example, is designed to derive fixed-size cryptographic keys from key material. Cryptography

### Discussion

What would be better than:

```
key = shared % 256
```

?

> Answer:

<br>

Why is using a standard KDF preferable to inventing your own?

> Answer:

<br>

---

# Bonus Challenge 13: Implement a stronger KDF

If you have the Python `cryptography` package installed, experiment with HKDF.

Install it if necessary:

```
python3 -m pip install cryptography
```

Then investigate:

```
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.hkdf import HKDF
```

The general idea is:

```
shared_bytes = ...

kdf = HKDF(
    algorithm=hashes.SHA256(),
    length=32,
    salt=None,
    info=b"shared-secrets-training",
)

key = kdf.derive(shared_bytes)
```

The documentation describes HKDF as an extract-and-expand mechanism for deriving fixed-size keys from input key material. Cryptography

### Your task

Modify the challenge's design so that:

```
Diffie–Hellman shared secret
            |
            v
           HKDF
            |
            v
       32-byte key
            |
            v
       authenticated
       encryption
```



---

### Reflection

Which attack is conceptually more interesting?

- Recovering the shared secret using the leaked private value

- Brute-forcing the one-byte XOR key

Why?

> Answer:

<br>

---

# Final checklist

Before considering the challenge complete, make sure you can check every box:

- I inspected `encryption.py`.

- I inspected `message.txt`.

- I identified the Diffie–Hellman parameters.

- I identified the leaked private value.

- I understand the difference between a private and public value.

- I reproduced a small Diffie–Hellman exchange.

- I calculated the real shared secret with Python.

- I understand why `pow(A, b, p)` works.

- I identified how the shared secret becomes an encryption key.

- I understand XOR.

- I decoded the ciphertext.

- I wrote my own solver.

- I can explain the vulnerability without looking at my code.

- I understand why reducing the shared secret to one byte is weak.

- I can describe how the implementation should be improved.

---

# Key concepts learned

By the end of this challenge, you should be comfortable with:

| Concept                      | What you should understand                                 |
| ---------------------------- | ---------------------------------------------------------- |
| Modular arithmetic           | Calculations involving a modulus                           |
| Modular exponentiation       | Computing `a^b mod p` efficiently                          |
| Diffie–Hellman               | Establishing a shared secret using public values           |
| Private/public keys          | Which values must remain secret                            |
| Key leakage                  | Why exposing private material defeats cryptography         |
| XOR                          | A reversible operation when the key is known               |
| Key derivation               | Turning shared key material into an encryption key         |
| Brute force                  | Testing a small keyspace systematically                    |
| Cryptographic implementation | Why correct algorithms can still be implemented insecurely |

---

# CTF takeaway

When you encounter a cryptography challenge, don't immediately reach for advanced mathematics.

Start with:

```
1. What files did I receive?
2. What does the source code actually do?
3. Which values are public?
4. Which values should be secret?
5. Did anything secret leak?
6. How is the key actually derived?
7. How is the ciphertext actually encoded?
8. Is the resulting key really large enough?
9. Can I reproduce each step independently?
10. Can I explain why my solution works?
```

In real-world security work, **reading the implementation carefully is often more valuable than knowing a sophisticated attack**.

---

# Optional further reading

- HKDF documentation: Python `cryptography` HKDF documentation

- Diffie–Hellman background: search for the original Diffie–Hellman key-exchange construction and modern DHE/ECDHE deployments.
