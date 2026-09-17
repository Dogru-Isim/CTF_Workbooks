# Flag Hunters

> **Difficulty:** Beginner / Intermediate\
>  **Category:** Reverse Engineering
>  **Skills:** Python code reading, control-flow analysis, input injection, source-code auditing\
>  **Tools:** Python 3, a code editor (VS Code, Vim, Notepad?), `nc` / `netcat``
>  **Source:** https://learn.cylabacademy.org/library/472?page=1&category=3&difficulty=1

---

# Challenge Overview

 The program you're investigating is called `lyric-reader.py`.

 It appears to be a simple Python program that prints the lyrics to a song about CTFs. However, somewhere in the program is a hidden piece of information that the normal execution path never displays.

 Your goal is to understand the program, identify why the hidden information isn't normally printed, and find a way to make the program reveal it.

 The challenge description gives us an important clue:

> Lyrics jump from verses to the refrain kind of like a subroutine call. There's a hidden refrain the program doesn't print by default.

 The challenge also warns us that the program accepts user input and that the input handling may be interesting.

### Learning objectives

 By the end of this challenge, you should be able to:

- Read an unfamiliar Python program and identify its important components.
  - Trace a program's control flow.
  - Identify where user-controlled data enters a program.
  - Recognize unsafe parsing of user input.
  - Understand how a delimiter can change the meaning of input.
  - Construct a small input-manipulation payload.
  - Use `nc` to interact with a remote CTF service.

---

# Challenge 1: Get the program running

 You should have been provided with the challenge source code:

```
lyric-reader.py
```

 If you're working with the original picoCTF challenge, you will also receive a network address and port for the running service.

 You can connect to the service with:

```
nc <HOST> <PORT>
```

 For example:

```
nc example.ctf.server 12345
```

> **Do not use the example host or port above.**
> 
>  Use the connection information provided with your challenge instance. You can find this by navigating to the url given in the **Source** label at the beginning of the file.

---

## First observation

 Run the program and observe what it prints.

 You should see several lines of lyrics followed by something similar to:

```
Crowd:
```

 The program is waiting for you to enter something.

 Try entering an ordinary piece of text:

```
hello
```

### Questions

1. What happens after you enter `hello`? Try giving different answers to the program and answer:
   
   2. Does the program print your input?
   3. Does the program eventually print the flag?
   4. Does the same part of the lyrics appear more than once?
   5. What appears to cause the program to jump back to an earlier part of the song?
   
   Write down your observations before looking at the source code.

---

# Challenge 2: Read the source

 Open `lyric-reader.py` in your editor.

 Start by looking at the imports:

```
import re
import time
```

 Then look for where the flag is read.

 You should find code resembling:

```
flag = open('flag.txt', 'r').read()
```

### Question

 Where does the flag come from?

- [ ] A command-line argument
- [ ] A network request
- [ ] An environment variable
- [ ] A local file
- [ ] The user

---

## Challenge 2.1: Find where the flag goes

 Continue reading the code.

 Look for a variable called:

```
secret_intro
```

 You should find something resembling:

```
secret_intro = \
'''Pico warriors rising, puzzles laid bare,
Solving each challenge with precision and flair.
With unity and skill, flags we deliver,
The ether’s ours to conquer, '''\
+ flag + '\n'
```

### Think about it

 The flag is concatenated directly onto `secret_intro`.

 Then look for:

```
song_flag_hunters = secret_intro + \
'''
...
'''
```

### Questions

1. Is the flag actually part of `song_flag_hunters`?
   
   2. If so, where in the song is it located?
   3. Does the program start reading the song from the beginning?
   
   Don't worry about finding the exact flag yet.
   
   Instead, answer this:
   
   > **If the flag is near the beginning of the song, but the program starts somewhere later, what might we need to do?**

---

# Challenge 3: Understand `reader()`

 Find the function:

```
def reader(song, startLabel):
```

 Read it carefully from top to bottom.

 Start with these variables:

```
lip = 0
start = 0
refrain = 0
refrain_return = 0
finished = False
```

### Question

 What do you think `lip` represents?

 Look at the later code to confirm your hypothesis.

---

## Understand: The song becomes a list of items

 Find:

```
song_lines = song.splitlines()
```

 This is important.

 If the song contains:

```
line zero
line one
line two
line three
```

 then `splitlines()` produces approximately:

```
[
    "line zero",
    "line one",
    "line two",
    "line three"
]
```

 Python indexes lists starting at `0`.

 So:

```
index 0 → line zero
index 1 → line one
index 2 → line two
index 3 → line three
```

### Exercise

 Suppose:

```
song_lines = [
    "SECRET",
    "[REFRAIN]",
    "VERSE1",
    "hello"
]
```

 What is:

```
song_lines[0]
```

 ?

 What is:

```
song_lines[3]
```

 ?

> For beginners: Type python3 (or python) in your terminal to open a python interpreter.
> 
> Type the last the 3 code blocks into the terminal.
> 
> Play around with it and understand what the code is doing.

---

# Challenge 4: Find the starting point

 Look at this section:

```
for i in range(0, len(song_lines)):
    if song_lines[i] == startLabel:
        start = i + 1
```

 And near the bottom:

```
reader(song_flag_hunters, '[VERSE1]')
```

### Questions

1. What value is passed as `startLabel`?
2. What does the program search for?
3. What happens when it finds `[VERSE1]`?
4. Does it start printing `[VERSE1]` itself? Or does it start with the line immediately after it?

---

## Exercise: Draw the control flow

 Using the source code, draw a simple diagram showing what happens in the last step.

```
Program starts
      |
      v
Find [VERSE1]
      |
      v
Start reading lyrics
      |
      v
Encounter REFRAIN
      |
      v
???
```

 Try to determine what happens when the code encounters `REFRAIN`.

---

# Challenge 5: Investigate `REFRAIN`

 Find this part of the function:

```
if line == 'REFRAIN':
    song_lines[refrain_return] = 'RETURN ' + str(lip + 1)
    lip = refrain
```

 There are two interesting operations here.

 First:

```
song_lines[refrain_return] = 'RETURN ' + str(lip + 1)
```

 Second:

```
lip = refrain
```

### Questions

 What do you think `lip = refrain` does?

 If `refrain` contains the index of the first line of the refrain, where will the program go next?

---

## Challenge 5.1: What is `RETURN`?

 Earlier in the program, look for:

```
elif song_lines[i] == 'RETURN':
    refrain_return = i
```

 So the program remembers where the `RETURN` line is.

 Now find the other occurrence of `RETURN`:

```
elif re.match(r"RETURN [0-9]+", line):
    lip = int(line.split()[1])
```

 This is much more interesting.

 A line such as:

```
RETURN 10
```

 causes:

```
lip = 10
```

### Question

 What would happen if the program processed:

```
RETURN 0
```

 ?

 Where would the next iteration of the loop read from?

---

# Challenge 6: Identify the dangerous input

 We have now discovered that the program understands a special command:

```
RETURN <number>
```

 But where does the input come from?

 Find:

```
elif re.match(r"CROWD.*", line):
    crowd = input('Crowd: ')
    song_lines[lip] = 'Crowd: ' + crowd
    lip += 1
```

 This is the critical section.

 The program asks the user for input:

```
crowd = input('Crowd: ')
```

 Then it modifies the current song line:

```
song_lines[lip] = 'Crowd: ' + crowd
```

### Think carefully

 Suppose the user enters:

```
hello
```

 The program changes the current line into:

```
Crowd: hello
```

 Now suppose the user enters:

```
RETURN 0
```

 The resulting line is:

```
Crowd: RETURN 0
```

 Would that execute the `RETURN` command? (Y/N)

 Look at this code:

```
for line in song_lines[lip].split(';'):
```

 The answer depends on how the semicolon is used.

---

# Challenge 7: Understand the semicolon

 This line is particularly important:

```
for line in song_lines[lip].split(';'):
```

 The program splits a line into multiple pieces whenever it encounters:

```
;
```

 For example:

```
"hello;world".split(';')
```

 produces:

```
[
    "hello",
    "world"
]
```

 And:

```
"hello;RETURN 0".split(';')
```

 produces:

```
[
    "hello",
    "RETURN 0"
]
```

### Exercise

 What does this produce in Python?

```
"ABC;DEF;GHI".split(';')
```

 Write the result below:

```
# Your answer:
```

---

# Challenge 8: Can user input create a command?

 Return to:

```
song_lines[lip] = 'Crowd: ' + crowd
```

 Suppose the user enters:

```
hello;RETURN 0
```

 What would the complete song line become?

```
______________________________________
```

 Now apply:

```
.split(';')
```

 How many pieces are produced?

```
________________________________________
```

 What are those pieces?

```
________________________________________
```

---

## Challenge 8.1: Which branch handles each piece?

 The program checks each resulting piece against several conditions:

```
if line == 'REFRAIN':
    ...
elif re.match(r"CROWD.*", line):
    ...
elif re.match(r"RETURN [0-9]+", line):
    ...
elif line == 'END':
    ...
else:
    print(line, flush=True)
```

 Consider:

```
Crowd: hello
```

 Which branch handles it?

```
________________________________________
```

 Now consider:

```
RETURN 0
```

 Which branch handles it?

```
________________________________________
```

### Key observation

 The semicolon allows us to make the program interpret part of our input as a **new instruction**.

 This is a form of **input injection**.

 The program expected the user to provide ordinary text, but because it parses the input as part of its own instruction language, the user can potentially inject a command.

---

# Challenge 9: Find the target line

 We now know that:

```
RETURN <number>
```

 changes `lip`.

 We also know that the flag is stored in `secret_intro`, which is placed at the beginning of `song_flag_hunters`.

 The next question is:

> **Which line number contains the flag?**

 Look at the beginning of the source.

 Count the lines carefully.

 Remember:

> Python uses zero-based indexing.

 The first line has index:

```
0
```

 The second line has index:

```
1
```

 and so on.

---

## Exercise

 Create a small table:

| Index | Line |
| ----- | ---- |
| 0     | ?    |
| 1     | ?    |
| 2     | ?    |
| 3     | ?    |
| 4     | ?    |

Use the actual source code to fill it in.

### Hint

 The flag is appended to this line:

```
The ether’s ours to conquer, <FLAG>
```

 How does that affect the line number you need to jump to?

---

# Challenge 10: Build the payload

 At this point we have identified three important facts:

1. User input is inserted into a song line.
   
   2. The program splits song lines on `;`.
   3. `RETURN <number>` changes the instruction pointer.
   
   Your task is to combine those facts.
   
   You want your input to contain:

```
<ordinary text>;<RETURN command>
```

 The ordinary text is useful because the program expects the first part to look like normal crowd input.

 The second part should be interpreted as a `RETURN` instruction.

### Your task

 Construct the payload that causes execution to jump to the beginning of the song.

 Do **not** look at the solution yet.

 Write your payload here:

```
________________________________________
```

---

# Challenge 11: Test your hypothesis

 Connect to the remote service:

```
nc <HOST> <PORT>
```

 When the program asks:

```
Crowd:
```

 enter the payload you constructed.

### Observe the output

 Ask yourself:

- Did the program jump somewhere unexpected?
  
  - Did it begin printing lines from earlier in the song?
  - Did it reach the hidden section?
  - Did the flag appear?
  
  If it doesn't work, go back through these questions:
1. Did I use a semicolon?
   2. Did I use the correct `RETURN` syntax?
   3. Did I calculate the correct zero-based line number?
   4. Did I enter the payload at the `Crowd:` prompt?
   5. Did I accidentally add extra spaces or characters?

---

# Challenge 12: Explain the exploit

 Before moving on, explain the vulnerability in your own words.

 Complete the following:

> The program is vulnerable because it takes user input from `input()`, inserts that input into a line of its internal song representation, and then \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_.

---

## Explain the attack chain

 Fill in the missing steps:

```
User input
    |
    v
input('Crowd: ')
    |
    v
Inserted into song_lines[lip]
    |
    v
Split using __________________
    |
    v
User-controlled text becomes __________________
    |
    v
Program changes __________________
    |
    v
Execution reaches the hidden flag
```

---

# Challenge 13: Source-code debugging

 Modify the code so that you can see the value of `lip`.

 For example, temporarily add:

```
print("[DEBUG] lip is: ", lip)
```

 at an appropriate location.

 Run the program again.

### Questions

1. What is `lip` when the program first starts printing?
2. What happens to `lip` when `REFRAIN` is encountered?
3. What happens to `lip` after your injected `RETURN` instruction?
4. Why does changing `lip` change what gets printed?

---

# Challenge 14: Why does the exploit work?

 This challenge is a good example of a small interpreter implemented inside an ordinary Python program.

 The program has its own tiny language containing instructions such as:

```
REFRAIN
RETURN <number>
END
```

 The program also accepts user-controlled text.

 The problem is that the boundary between:

```
DATA
```

 and:

```
INSTRUCTIONS
```

 is not properly enforced.

### Question

 What would a safer design look like?

 For example, how could the program allow a user to enter:

```
hello;RETURN 0
```

 as ordinary text without interpreting it as a command?

 Write down at least two ideas.

```
1. _______________________________________________

2. _______________________________________________
```

---

# Bonus Challenge: Break the parser in a different way

 Now that you understand the parser, investigate what happens if you provide other special strings.

 Try experimenting with:

```
REFRAIN
```

```
END
```

```
hello;END
```

```
hello;REFRAIN
```

```
hello;RETURN 1
```

### Questions

- Which inputs change the program's control flow?

- Which inputs terminate the program?

- Which inputs cause it to jump?

- Can you create a loop?

- What happens if you jump to an invalid line number?

> **Hint:** The challenge itself warns that the program can easily enter undefined or undesirable states. Don't be afraid to stop it with `Ctrl-C` while experimenting.

---

# Bonus Challenge 2: Understand the regular expression

 The program uses:

```
re.match(r"RETURN [0-9]+", line)
```

 Break this regular expression into its components.

### `RETURN`

 What does it require?

```
________________________________________
```

### A space

 Why is there a literal space after `RETURN`?

```
________________________________________
```

### `[0-9]+`

 What does this mean?

```
________________________________________
```

### Exercise

 Which of these would match?

| Input        | Matches? |
| ------------ | -------- |
| `RETURN 0`   | ?        |
| `RETURN 123` | ?        |
| `RETURN abc` | ?        |
| `RETURN`     | ?        |
| `RETURN 1 2` | ?        |

---

# Bonus Challenge 3: Make the program safer

 Imagine you are reviewing this code as a developer rather than an attacker.

 The problematic design is roughly:

```
crowd = input('Crowd: ')
song_lines[lip] = 'Crowd: ' + crowd

for line in song_lines[lip].split(';'):
    ...
```

 The program should treat the user's input purely as text.

### Task

 Rewrite this part conceptually so that user input cannot become a command.

 You don't necessarily need to produce a complete working program.

 Describe your preferred fix:

```
____________________________________________________

____________________________________________________

____________________________________________________
```

---

# Final Questions

 Before considering the challenge solved, make sure you can answer all of these without looking back at the source.

### 1\. Where is the flag stored?

```
________________________________________
```

### 2\. Why isn't it printed during normal execution?

```
________________________________________
```

### 3\. What variable controls which song line is processed?

```
________________________________________
```

### 4\. What command changes that variable?

```
________________________________________
```

### 5\. Where does user input enter the program?

```
________________________________________
```

### 6\. What delimiter lets you inject an additional parser instruction?

```
________________________________________
```

### 7\. Why does zero-based indexing matter?

```
________________________________________
```

### 8\. What is the final payload you used?

```
________________________________________
```

---

# Takeaways

 This challenge demonstrates several concepts that appear repeatedly in real security work:

- **Source-code review:** You don't always need a debugger or disassembler to reverse engineer a program.
  
  - **Control-flow analysis:** Following variables such as `lip` can reveal how execution moves through a program.
  - **Input injection:** User-controlled data becomes dangerous when it is subsequently interpreted as syntax or commands.
  - **Delimiter abuse:** Characters such as `;` can sometimes change how an application parses input.
  - **Zero-based indexing:** Small indexing mistakes can make an otherwise-correct exploit fail.
  - **Trust boundaries:** Data supplied by a user should not silently cross from a data context into an instruction context.
  
  The important lesson isn't just the final payload.
  
  The important lesson is the reasoning chain:

```
Find the secret
      ↓
Find where it is stored
      ↓
Determine why normal execution doesn't reach it
      ↓
Find what controls execution
      ↓
Find where user input enters
      ↓
Understand how that input is parsed
      ↓
Turn input into an unintended instruction
      ↓
Redirect execution
      ↓
Recover the flag
```
