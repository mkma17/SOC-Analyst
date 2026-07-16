# SOC Incident Report: Kerberos Pre-Authentication Attack & Password Cracking

## 1. Executive Summary
During a routine network traffic review / alert triaging, a suspicious Kerberos connection was detected. Analysis of the packet capture (PCAP) revealed a Kerberos `AS-REQ` (Authentication Service Request) for the user `william.dupond`. An offline brute-force attack was simulated/conducted against the extracted pre-authentication hash (AES-256), resulting in a successful password recovery within 17 seconds due to a weak password policy.

---

## 2. Incident Details
* **Target User:** `william.dupond`
* **Domain / Realm:** `CATCORP.LOCAL`
* **User Principal Name (UPN):** `william.dupond@catcorp.local`
* **Protocol:** Kerberos v5 (AS-REQ)
* **Encryption Type:** AES256-CTS-HMAC-SHA1-96 (etype 18)
* **Recovered Password:** `kittycat12`

---

## 3. Investigation & Analysis Walkthrough

### Phase 1: Traffic Analysis (Wireshark)
1. **Filtering Traffic:** Filtered the network capture using the display filter `kerberos.msg_type == 10` to isolate Authentication Service Requests (`AS-REQ`).
2. **Identifying the Target:** Located the suspicious request sent by user `william.dupond` under the realm `CATCORP.LOCAL`.
3. **Extracting the Ciphertext:** Inspected the Pre-Authentication data (`PA-ENC-TIMESTAMP`) and extracted the 112-character encrypted `cipher` string.

![Wireshark Packet Details Showing PA-ENC-TIMESTAMP](screenshots/details.png)

### Phase 2: Hash Extraction & Formatting
The extracted metadata and cipher were formatted into a standard Hashcat-compatible format for Kerberos 5 AS-REQ Pre-Auth hashes:

```text
$krb5pa$18$william.dupond$CATCORP.LOCAL$fc8bbe22b2c967b222ed73dd7616ea71b2ae0c1b0c3688bfff7fecffdebd4054471350cb6e36d3b55ba3420be6c0210b2d978d3f51d1eb4f
```

### Phase 3: Offline Password Cracking (Hashcat)
Using **Hashcat (v7.1.2)** with **Mode 19900** (Kerberos 5, etype 18, Pre-Auth) and the standard `rockyou.txt` wordlist, the hash was cracked successfully.

**Command Executed:**
```bash
hashcat -m 19900 hash.txt /usr/share/wordlists/rockyou.txt
```

**Result:**
The attack recovered the plaintext password `kittycat12` in **17 seconds**, exhausting only **0.55%** of the wordlist.

```text
[Insert Hashcat Terminal Output Screenshot Here]
Example: Showing 'Status...........: Cracked' and the recovered password string.
```

---

## 4. Root Cause Analysis
* **Weak Password Complexity:** The user utilized a common dictionary word with a two-digit trailing number (`kittycat12`), making it highly vulnerable to offline dictionary attacks.
* **Lack of Multi-Factor Authentication (MFA):** If MFA was enforced, capturing the password alone would not grant the attacker access to the domain resources.

---

## 5. Remediation & Recommendations

### Immediate Actions
1. **Force Password Reset:** Immediately expire and reset the password for `william.dupond@catcorp.local` in Active Directory.
2. **Revoke Active Tickets:** Perform a rolling reset of the `krbtgt` account password (twice) to invalidate any Golden/Silver tickets or currently active TGTs if a broader compromise is suspected.
3. **Host Isolation:** Identify the source IP address from the Wireshark logs and isolate the affected machine for further forensic investigation.

### Long-Term Hardening
* **Implement Strong Domain Password Policies:** Enforce Fine-Grained Password Policies (FGPP) requiring a minimum length of 14 characters and prohibiting dictionary words.
* **Deploy MFA:** Restrict domain authentications without a secondary multi-factor challenge.
* **Monitor Event Logs:** Configure SIEM alerts for Windows Event ID `4768` (Kerberos Authentication Ticket Requested) where weak encryption algorithms are requested or high-frequency failures occur.
