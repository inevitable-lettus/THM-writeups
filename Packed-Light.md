# Packed Light

**Name:** Garv Gurwara \
**Platform:** TryHackMe \
**Category:** Network Forensics \
**Difficulty:** Easy \
**Date completed:** 4th October 2026

## Summary
Analyzed a PCAP of guest network traffic to identify a covert data exfiltration channel. Traced a suspicious script download, recovered the exfiltration mechanism and its encryption key from the retrieved artifact, then reconstructed the exfiltrated data by decoding and decrypting a sequence of HTTP cookie values.

## Investigation

### 1. Initial Triage
- Opened the capture in Wireshark and filtered on `http` to get a readable overview of web traffic first.
- The first GET request stood out: a request for `/temp/updates.py` — a Python source file served over plain HTTP, unusual for guest-facing hotel traffic.
- Followed the TCP stream for that request/response pair and retrieved the full script.

### 2. Artifact Analysis
The retrieved script turned out to be a keylogger: it hooked keyboard input, encrypted each captured character, and beaconed it out one request at a time by encoding the ciphertext into an HTTP `Cookie` header sent to a fixed host (`byte-lotus-hotel.thm:8080`) — matching the "pinging some random :8080 address every single second" behavior flagged in the room's own briefing.

Two details recovered from the script were critical for decoding the exfiltrated data:
- The encryption key was stored in cleartext inside the script itself (split across two variables and concatenated), rather than derived or obfuscated.
- Each cookie value was built as: encrypt one character → base64-encode the result → send as `hotel_sess_state=<value>`.

*(The script itself isn't reproduced here — it's a functioning keystroke-exfiltration tool, and a portfolio writeup doesn't need to double as a working copy of it. The methodology above is enough to show the approach.)*

### 3. Extracting the Exfiltrated Data
- Filtered the capture further with `http.cookie contains "hotel_sess_state"` to isolate every beacon request.
- Each request's cookie value corresponds to one captured keystroke, in order. Sample (first 3 of ~30 tokens):
  `HA==  AA==  BQ==`
- Base64-decoded each token to recover the raw XOR ciphertext bytes, then decrypted using the key recovered from the script to recover the original character.

### 4. Correction: Transcription Error
On the first pass, the reconstructed message came out garbled in two places. Comparing against a recount: two base64 tokens had been skipped while manually scrolling and selecting values out of Wireshark's packet list (several rows look near-identical at a glance, making it easy to miss one), and one token was misread. Because the cipher encrypts each character independently rather than as a chained block, the error stayed localized to those three characters instead of cascading through the rest of the message — which made it straightforward to catch by rechecking the token count against the number of filtered packets.

**Takeaway:** when manually transcribing many similar-looking short values from a GUI list, count entries against the expected total before decoding, rather than trusting a manual scroll-and-copy.

## Findings / Indicators

| Indicator | Value |
|---|---|
| C2 host | `byte-lotus-hotel.thm:8080` |
| Delivery path | `/temp/updates.py`, served over plaintext HTTP |
| Exfil channel | HTTP `Cookie` header, parameter `hotel_sess_state` |
| Beacon pattern | ~1 request per keystroke |
| Encoding | Base64 (transport) over XOR (confidentiality) |
| User-Agent | Spoofed to look like a legitimate browser/client |

Flag recovered but omitted here per this repo's policy (see README).

## Detection & Mitigation
- **Beacon regularity:** a host making near-constant, fixed-interval requests to the same external endpoint — tied to keystrokes rather than normal browsing — is a strong anomaly signal on its own, independent of payload content. This is the kind of behavioral signature a frequency/baseline check should catch without needing to inspect the payload at all.
- **Plaintext key storage:** the exfiltration key was recoverable directly from the delivered script — "encrypted" C2 traffic is only as strong as how the key is protected, so catching the script before it runs (host-based detection) is more reliable than relying on the channel looking encrypted enough to pass unnoticed.
- **Cookie-based exfiltration:** data hidden in a `Cookie` header blends into normal traffic at a glance. Egress monitoring that flags unusually high-entropy or unusually long cookie values, or cookies sent with no prior corresponding `Set-Cookie`, is a reasonable detective control.
- **Practical mitigations:** egress filtering/allow-listing of outbound destinations, blocking scripts from being served or executed from unexpected paths, and endpoint monitoring for processes using keyboard-hook libraries with no legitimate business reason.

## Takeaways
- First network-forensics writeup in this repo — a good complement to the exploitation-chain writeups, and directly relevant to the detection-first approach NetForensics is built around.
- Reinforced the difference between encoding (Base64 — reversible, not secret) and encryption (XOR — meant to be secret, but only as strong as key handling allows).
- CyberChef is worth having set up for exactly this kind of quick decode/decrypt chaining instead of writing one-off scripts.
- Manual transcription from a GUI is a real source of error at scale — cross-check counts before trusting a decoded result.
