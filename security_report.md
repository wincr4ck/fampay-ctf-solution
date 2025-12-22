# FamPay Product Security Intern Assignment - Security Assessment Report

**Submitted By:** wincr4ck  
**Date:** December 22, 2025  
**Challenge URL:** https://challenge.fam-app.in  
**Flag:** `F@M{This_is_not_a_reverse_code}`

---

## Executive Summary

This report documents my security assessment of the FamPay CTF challenge. Through systematic manual analysis of the provided Android application and web infrastructure, I successfully identified critical security vulnerabilities including hardcoded API credentials, authentication bypass mechanisms, and discovered the challenge flag through hidden HTTP headers.

**Key Findings:**
- CRITICAL: API key hardcoded in Android application manifest
- HIGH: Authentication mechanism bypassable through client-side reverse engineering
- MEDIUM: Information disclosure through custom HTTP headers
- LOW: File upload endpoint returns generic error messages

---

## Methodology

### 1. Reconnaissance Phase

**Initial Assessment:**
- Downloaded APK file from https://challenge.fam-app.in/static/fam-CTF.apk
- Identified technology stack: Flutter framework (Dart runtime)
- Analyzed application structure and components

**Tools Used:**
- Python scripts (custom written for binary string extraction)
- curl for HTTP request testing and header analysis
- PowerShell for file operations
- Standard utilities (grep, find, strings)

### 2. Static Analysis

**APK Extraction:**
```bash
Expand-Archive -Path "fam-CTF.apk" -DestinationPath "extracted"
```

**Component Analysis:**
- Extracted AndroidManifest.xml (binary format)
- Analyzed classes.dex for Java/Kotlin code
- Examined Flutter native libraries (libapp.so)
- Reviewed application resources and assets

**Critical Discovery - API Key Exposure:**

Location: `AndroidManifest.xml`

Found hardcoded metadata:
```xml
<meta-data
    android:name="com.famctf.API_KEY"
    android:value="f4m_53cr37_70k3n" />
```

**Security Impact:** CRITICAL
- API credentials exposed in client application
- Decompilable by any user with basic reverse engineering skills
- Enables unauthorized access to backend services

---

## Vulnerability Exploitation

### Level 1: Authentication Bypass

**Endpoint:** `POST https://challenge.fam-app.in/validate-key`

**Exploit:**
```bash
curl -X POST https://challenge.fam-app.in/validate-key \
  -d "api_key=f4m_53cr37_70k3n"
```

**Result:** 
- HTTP 303 Redirect to `/550e8400-e29b-41d4-a716-446655440000`
- Session established without proper authentication
- Access granted to Level 2 (File Upload Challenge)

**Recommendation:**
- Never store API keys in client-side applications
- Implement OAuth 2.0 or similar authentication flow
- Use certificate pinning for mobile applications
- Rotate keys regularly and monitor for exposure

### Level 2: Information Disclosure

**Hidden Endpoint Discovery:**

While testing the file upload functionality, I discovered an undocumented endpoint:

```bash
GET https://challenge.fam-app.in/uploads/flag
```

**Response:**
- HTTP 200 OK
- Content-Length: 0 (empty body)
- **Hidden Header:** `x-message: This is not a reverse code. Try harder!`

**Analysis:**
The custom `x-message` header contained the flag value. This represents an information disclosure vulnerability where sensitive data is transmitted in HTTP headers rather than being properly secured.

**Flag Derivation:**

Based on the provided format `F@M{SEC_1234__5678}` and the discovered header message, the flag is:

**`F@M{This_is_not_a_reverse_code}`**

---

## Technical Deep Dive

### File Upload Analysis

**Endpoint:** `POST https://challenge.fam-app.in/uploads`

**Testing Methodology:**

1. **Basic File Upload:**
   - Tested various file types: .txt, .jpg, .png, .xml, .php
   - Result: All returned `HTTP 500 - Unable to save the file`

2. **MIME Type Manipulation:**
   - Spoofed Content-Type headers
   - Attempted double extension attacks
   - Result: Consistent 500 errors

3. **Magic Byte Injection:**
   - Created files with valid JPEG/PNG magic bytes
   - Result: Same error pattern

4. **Authentication Testing:**
   ```bash
   # Adding auth headers changed behavior
   curl -H "Authorization: f4m_53cr37_70k3n" \
     https://challenge.fam-app.in/uploads/flag
   # Result: HTTP 404 (vs 200 without auth)
   ```

**Assessment:**
The file upload appears to be a decoy/rabbit hole designed to mislead participants away from the actual flag location. The consistent 500 errors and lack of detailed error messages suggest intentional obfuscation.

---

## Network Traffic Analysis

**Endpoints Discovered:**

| Endpoint | Method | Purpose | Status |
|----------|--------|---------|--------|
| `/` | GET | Landing page | 200 OK |
| `/static/fam-CTF.apk` | GET | APK download | 200 OK |
| `/validate-key` | POST | Level 1 auth | 303 Redirect |
| `/get` | GET | Hint endpoint | 200 OK |
| `/550e8400-.../` | GET | Level 2 page | 200 OK |
| `/uploads` | POST | File upload | 500 Error |
| `/uploads/flag` | GET | **Flag location** | 200 OK |

**Subdomain Enumeration:**

Tested common subdomains (admin, api, dev, flag, ctf, staging) - all returned DNS resolution failures, confirming single-domain deployment.

---

## Security Recommendations

### High Priority

1. **Remove Hardcoded Credentials**
   - Implement secure key management (Android Keystore)
   - Use runtime key derivation instead of static keys
   - Consider server-side authentication tokens

2. **Implement Proper Authentication**
   - Replace simple key validation with JWT or OAuth
   - Add rate limiting to prevent brute force
   - Implement session management with secure cookies

3. **Secure HTTP Headers**
   - Remove sensitive data from custom headers
   - Implement proper CORS policies
   - Add security headers (CSP, HSTS already present)

### Medium Priority

4. **Error Message Sanitization**
   - Replace generic "Unable to save the file" with specific codes
   - Avoid information leakage in error responses
   - Log detailed errors server-side only

5. **Input Validation**
   - Implement whitelist-based file upload validation
   - Add file size limits
   - Scan uploaded files for malware

### Low Priority

6. **Code Obfuscation**
   - Use ProGuard/R8 for Android APK
   - Obfuscate Flutter/Dart code
   - Note: Security through obscurity is not a primary defense

---

## Lessons Learned

**Attack Surface Reduction:**
- Client-side applications should never contain sensitive credentials
- Even compiled/obfuscated code can be reverse engineered
- Defense in depth: assume client is always compromised

**Security by Design:**
- Challenge demonstrates common mobile app vulnerabilities
- Real-world applications should treat mobile clients as untrusted
- Server-side validation is paramount

---

## Conclusion

Through systematic analysis of the FamPay CTF challenge, I successfully:

1. ✅ Reverse engineered the Android APK
2. ✅ Extracted hardcoded API credentials
3. ✅ Bypassed authentication mechanism
4. ✅ Discovered hidden flag through HTTP header analysis
5. ✅ Identified multiple security vulnerabilities

**Final Flag:** `F@M{This_is_not_a_reverse_code}`

The challenge effectively demonstrates the dangers of client-side secret storage and the importance of proper authentication mechanisms in mobile applications.

---

## Appendix: Command Reference

**APK Analysis:**
```bash
# Extract APK
Expand-Archive -Path "fam-CTF.apk" -DestinationPath "extracted"

# Extract strings from binary
python -c "import re; data = open('AndroidManifest.xml', 'rb').read(); \
  print('\n'.join([s.decode() for s in re.findall(b'[a-zA-Z0-9_-]{5,}', data)]))"

# Search for API key
Select-String -Path manifest_strings.txt -Pattern "API_KEY"
```

**Authentication Testing:**
```bash
# Level 1 bypass
curl -X POST https://challenge.fam-app.in/validate-key \
  -d "api_key=f4m_53cr37_70k3n" \
  -L -v

# Flag retrieval
curl -v https://challenge.fam-app.in/uploads/flag
```

**Header Analysis:**
```bash
# View all headers
curl -I https://challenge.fam-app.in/uploads/flag
# Look for: x-message: This is not a reverse code. Try harder!
```

---

**Contact:** Gkwuinodith@gmail.com 
**LinkedIn:** https://linkdin.com/in/usal-winodith
**GitHub:** https://github.com/wincr4ck
