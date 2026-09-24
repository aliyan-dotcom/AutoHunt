# AutoHunt — Automated Honeypot-Based Attacker TTP Profiling System

AutoHunt is a Final Year Project that combines a real SSH/Telnet honeypot,
a Python log-parsing engine that maps attacker behaviour to the MITRE
ATT&CK framework, and a live React dashboard for visualising attacks
as they happen.

**Team:** Aliyan Khan & Syed Usman Ahmed
**Supervisor:** Iqbal Uddin Khan
**Department:** Digital Forensics & Cyber Security, Hamdard University

---

## Architecture

There are three main pieces working together:

1. **The trap — Cowrie honeypot.** A medium-interaction SSH/Telnet
   honeypot that looks like a real, poorly-secured server. It fools
   attackers into logging in, running commands, and downloading tools,
   while recording every single action into a structured JSON log.

2. **The brain — Python parsing engine + Flask API.** Reads the Cowrie
   JSON log file, classifies each event against the MITRE ATT&CK
   framework (e.g. brute force login attempts become technique T1110),
   and serves a clean JSON summary over an HTTPS API.

3. **The face — React dashboard.** Polls the Flask API every 10
   seconds and renders live charts, attacker IPs, and a MITRE
   technique breakdown. Falls back to demo/sample data automatically
   if the API is unreachable, so the interface never shows a broken
   screen.

```
Attacker  --->  Cowrie Honeypot  --->  cowrie.json log
                                              |
                                              v
                                   Python parser (attack_api.py)
                                              |
                                     MITRE ATT&CK mapping
                                              |
                                              v
                                     Flask HTTPS API (/api/summary)
                                              |
                                              v
                                   React Dashboard (polls every 10s)
```

---

## Part 1 — Setting up the honeypot server (DigitalOcean droplet)

1. Create a droplet: Ubuntu 22.04, Basic plan (this project used the
   $8/month tier), any region.
2. SSH into it as root:
   ```
   ssh root@YOUR_DROPLET_IP
   ```
3. Create a dedicated non-root user to run the honeypot:
   ```
   adduser cowrie
   su - cowrie
   ```
4. Install Cowrie inside its own Python virtual environment:
   ```
   sudo apt update
   sudo apt install git python3-venv python3-dev libssl-dev libffi-dev build-essential -y
   git clone http://github.com/cowrie/cowrie
   cd cowrie
   python3 -m venv cowrie-env
   source cowrie-env/bin/activate
   pip install --upgrade pip
   pip install -r requirements.txt
   ```
5. Start Cowrie:
   ```
   bin/cowrie start
   ```
   By default Cowrie listens on port 2222 (SSH) so it doesn't clash
   with your droplet's real SSH port 22.
6. Logs will appear at:
   ```
   ~/cowrie/var/log/cowrie/cowrie.json
   ```

---

## Part 2 — Setting up the Flask API

This is the "brain" that reads Cowrie's logs and maps them to MITRE
ATT&CK techniques.

1. Copy `backend/attack_api.py` and `backend/requirements.txt` onto
   the droplet, into the cowrie user's home folder (e.g. via `scp`
   or by pasting the content directly with `nano`).
2. Still inside the `cowrie-env` virtual environment, install the
   API's dependencies:
   ```
   pip install -r requirements.txt
   ```
3. Double check the `COWRIE_LOG_PATH` variable at the top of
   `attack_api.py` matches where your Cowrie logs actually are.
4. Run it:
   ```
   python attack_api.py
   ```
   This starts an HTTPS server on port 5000 using a self-signed
   certificate (via Flask's `ssl_context="adhoc"`), so the dashboard
   can talk to it securely.
5. **Important — keep it running permanently.** If you just run
   `python attack_api.py` directly, it dies the moment your SSH
   session disconnects. To keep it alive in the background even
   after closing the terminal, use:
   ```
   nohup python attack_api.py > flask.log 2>&1 &
   ```
   This detaches the process from your SSH session entirely. Check
   it's alive later with:
   ```
   curl -k https://127.0.0.1:5000/api/summary
   ```
   (For a production-grade setup, you'd eventually convert this into
   a proper `systemd` service — noted under Future Work.)

---

## Part 3 — Setting up the React dashboard

1. On your own computer, install Node.js if you don't already have
   it, then create the Vite project (skip this step if you're using
   the `dashboard/` folder provided in this repo):
   ```
   npm create vite@latest autohunt-dashboard -- --template react
   cd autohunt-dashboard
   ```
2. Copy the provided `dashboard/src/App.jsx` into your project's
   `src/App.jsx`, overwriting the default file completely (don't
   paste it in twice — that causes duplicate-import errors).
3. Install dependencies, including the two extra packages the
   dashboard needs beyond the Vite defaults:
   ```
   npm install
   npm install lucide-react recharts
   ```
4. Open `src/App.jsx` and find the `API_URL` constant near the top.
   Make sure it points at your droplet's IP address, for example:
   ```
   https://YOUR_DROPLET_IP:5000/api/summary
   ```
5. Run the dev server:
   ```
   npm run dev
   ```
6. Open the local address it gives you (usually
   `http://localhost:5173`) in your browser.

---

## Part 4 — Trusting the self-signed certificate (important!)

Because the Flask API uses a self-signed HTTPS certificate, browsers
will block the dashboard's background requests to it until you
manually tell the browser to trust that address, once per browser
session.

1. In the **same browser** where your dashboard is open, open a new
   tab and go directly to:
   ```
   https://YOUR_DROPLET_IP:5000/api/summary
   ```
2. You'll see a warning like "Your connection is not private." Click
   **Advanced**, then **Proceed to YOUR_DROPLET_IP (unsafe)**.
3. Once that page loads and shows raw JSON text, go back to the
   dashboard tab and refresh it. It should switch from demo mode to
   live mode.

If you ever see `net::ERR_CERT_AUTHORITY_INVALID` in the browser
console, this is exactly what's happening — just repeat the steps
above in that browser.

---

## Live vs Demo mode

The dashboard is designed to never show a broken screen. If it can
reach the Flask API, it shows real live attacker data (**live mode**).
If the API is unreachable for any reason (server restarted, SSH
session dropped, certificate not yet trusted), it automatically shows
realistic sample data instead (**demo mode**), so the interface always
looks functional.

---

## Real-world validation

During testing, this honeypot was found and attacked by a genuine,
unsolicited attacker at IP `205.210.31.195`, with no advertising or
linking of the server anywhere — proof that it is genuinely
internet-facing and credible as a real security tool, not just a
local simulation. Manual test sessions were also run from
`221.132.115.71` to confirm the classification pipeline worked
end-to-end.

---

## MITRE ATT&CK techniques currently mapped

| Technique | Name |
|---|---|
| T1110 | Brute Force |
| T1078 | Valid Accounts |
| T1105 | Ingress Tool Transfer |
| T1059 | Command and Scripting Interpreter |
| T1595 | Active Scanning |
| T1082 | System Information Discovery |
| T1003 | OS Credential Dumping |

---

## Known limitations

- Currently only covers SSH and Telnet based attacks, since that's
  what Cowrie emulates.
- The Flask API is not yet running as a proper `systemd` service, so
  after a full droplet reboot it needs to be manually started again.
- This is a single honeypot instance rather than a distributed
  network of honeypots across multiple network segments.

## Future work

- Convert the Flask API into a persistent `systemd` service so it
  survives droplet reboots automatically.
- Add more honeypot instances (e.g. web/HTTP honeypots, database
  honeypots) feeding into the same dashboard.
- Add authentication to the dashboard/API for safe multi-user access.
- Replace the self-signed certificate with a real one (e.g. via
  Let's Encrypt) for a cleaner browser experience.

---

## Jarvis voice assistant (optional add-on)

There's also a voice-controlled companion app, Jarvis, that knows the
whole project and can answer questions, troubleshoot, and quiz you for
your defense. See `jarvis-assistant/README.md` for details, or open
the already-hosted version directly:
https://claude.ai/artifact/V7Y4u2VDZaDNbwL9emn4NS

## Repository contents

```
AutoHunt-project/
  backend/
    attack_api.py       - Flask API + MITRE mapping engine
    requirements.txt    - Python dependencies
  dashboard/
    src/App.jsx          - React dashboard source
    package.json         - Node dependencies
  jarvis-assistant/
    jarvis.html          - voice assistant app source
    README.md            - how to use / run it
  docs/
    (full report, proposal, presentation, defense document, etc.)
  README.md              - this file
```
