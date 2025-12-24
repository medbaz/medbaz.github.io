---
title: "File Inclusion Vulnerabilities"
categories:
  - Cybersecurity
tags:
  - file-inclusion-vulnerabilities
  - local-file-inclusion
  - path-traversal-techniques
  - web-application-security
  - vulnerability-exploitation
---

# Exploiting File Inclusion Vulnerabilities: A Practical Guide

## Introduction

File inclusion vulnerabilities remain one of the most critical security flaws in web applications. In this hands-on guide, I will walk you through my journey of discovering and exploiting Local File Inclusion and path traversal vulnerabilities using industry-standard tools and techniques. This article documents real attacks performed in a controlled lab environment against DVWA and OverTheWire Natas challenges.

## Setting Up the Testing Environment

Before diving into exploitation, proper preparation is essential. I started by ensuring my Kali Linux machine had the necessary tools and wordlists.

### Installing SecLists

SecLists is an invaluable collection of wordlists for security testing. To install it, I ran:

```bash
sudo apt update
sudo apt install seclists
```

This gave me access to thousands of carefully crafted payloads organized by attack type, including specialized wordlists for LFI attacks.

## Understanding Authentication Challenges

One of the first hurdles I encountered was working with authenticated web applications. Modern web applications use session cookies to maintain user state, and any security testing must include these credentials.

### Extracting Session Cookies from DVWA

DVWA requires authentication to access vulnerable pages. Here is how I extracted the necessary cookies:

1. Open DVWA in a browser and log in
2. Press Ctrl + Shift + I to open Developer Tools
3. Navigate to Storage tab, then Cookies
4. Copy the PHPSESSID value and security level
![](/assets/images/image1.png)
With these cookies in hand, I could now craft authenticated requests using ffuf.

## Exploiting DVWA File Inclusion

DVWA provides an excellent environment for practicing file inclusion attacks. The vulnerable parameter accepts file paths, making it perfect for testing path traversal techniques.

### Initial Reconnaissance

I started with a basic command to test the waters:

```bash
ffuf -w /usr/share/seclists/Fuzzing/LFI/LFI-gracefulsecurity-linux.txt \
  -u "http://192.168.1.115:8001/DVWA-master/vulnerabilities/fi/?page=FUZZ" \
  -H "Cookie: PHPSESSID=6sikockcg73sqchu9ve1jhd0dn; security=low" \
  -fw 253
```

Let me break down this command:

- w flag specifies the wordlist containing LFI payloads
- u flag defines the target URL with FUZZ as the injection point
- H flag adds the authentication cookie header
- fw 253 filters out responses with exactly 253 words, eliminating false positives
## Understanding Response Filtering

Initially, I tested several invalid paths against the DVWA file inclusion vulnerability. What I discovered was that error responses followed a consistent pattern. Every time the application failed to include a file, it returned the same response characteristics:

- Same number of lines in the HTML response
- Same word count
- Same character length

For example, testing non-existent files like invalid123.php or fakepath.txt all returned responses with exactly 253 words and similar line counts around 90-92 lines. This consistency became my filtering baseline.

Once I identified these patterns, I added filters to my ffuf commands. The fw 253 flag filtered out all responses containing exactly 253 words, which were my error pages. Similarly, the fl 90,92 flag removed responses with 90 or 92 lines.
Notice the fiffffosdifs paths response .
![](/assets/images/image2.png)
### Choosing the Right Wordlist

Since my DVWA instance was hosted on Windows, I switched to a Windows-specific wordlist:

```bash
/usr/share/seclists/Fuzzing/LFI/LFI-gracefulsecurity-windows.txt
```

This wordlist contains payloads optimized for Windows file systems, including variations like:

- Backslash path separators
- Windows-specific file paths
- Drive letter references
- UNC path attempts

The results were immediate. I discovered several accessible file paths that revealed sensitive information about the system configuration.
![](/assets/images/image3.png)
### Advanced Filtering Techniques

As I refined my attacks, I learned to use more sophisticated filtering:

```bash
ffuf -w /home/kali/test/FUZZtest \
  -u "http://192.168.11.131:8001/DVWA-master/vulnerabilities/fi/?page=FUZZ" \
  -H "Cookie: PHPSESSID=cil9mjqrj9ksbhg5o5dq5vj283; security=low" \
  -fl 90,92
```

The fl flag filters by line count, hiding responses with 90 or 92 lines. This technique proved invaluable for cutting through noise and identifying truly vulnerable endpoints.

## Attacking OverTheWire Natas Challenges

The Natas wargames present a different authentication challenge: HTTP Basic Authentication. This required a modified approach.

### HTTP Basic Authentication with ffuf

Instead of cookie-based sessions, Natas uses the standard HTTP authentication mechanism. I embedded the credentials directly in the URL:

```bash
ffuf -w /usr/share/seclists/Fuzzing/LFI/LFI-LFISuite-pathtotest-huge.txt \
  -u "http://natas3:my_password@natas3.natas.labs.overthewire.org/FUZZ" \
  -mc 200
```

The mc 200 flag tells ffuf to match only successful HTTP 200 responses, immediately highlighting accessible resources.

### Discovering Hidden Content

Through systematic fuzzing with multiple wordlists, including:

- /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt
- /usr/share/seclists/Fuzzing/fuzz-Bo0oM.txt
- /usr/share/seclists/Fuzzing/LFI/LFI-LFISuite-pathtotest-huge.txt

I discovered the robots.txt file, which contained clues leading to the solution. 

![](/assets/images/image5.png)
This demonstrates an important principle: sometimes the path to exploitation requires exploring multiple vectors and wordlists.

## Wordlist Selection Strategies

Choosing the right wordlist makes the difference between success and wasted time. I developed a systematic approach to wordlist selection.

### Searching for Relevant Wordlists

To find wordlists containing specific payloads, I used grep:

```bash
grep -r "passwd" /usr/share/seclists 2>/dev/null
```

This command searches all SecLists directories for files containing passwd references, helping me identify the most relevant wordlists for targeting system files.

### Previewing Wordlist Contents

Before launching lengthy fuzzing sessions, I always preview wordlists:

```bash
head /usr/share/seclists/Fuzzing/LFI/LFI-gracefulsecurity-linux.txt
```

This shows the first 10 entries, giving me insight into the payload structure and helping me estimate scan duration.



## Conclusion

File inclusion vulnerabilities remain prevalent in modern web applications, and understanding how to discover and exploit them is essential for any security professional. Through systematic testing against DVWA and Natas challenges, I gained hands-on experience with authentication handling, wordlist selection, response filtering, and platform-specific exploitation techniques.
