# vexRecon
<p align="center">
<pre>
____    ____  __________   ___ .______       _______   ______   ______   .__   __. 
\   \  /   / |   ____\  \ /  / |   _  \     |   ____| /      | /  __  \  |  \ |  | 
 \   \/   /  |  |__   \  V  /  |  |_)  |    |  |__   |  ,----'|  |  |  | |   \|  | 
  \      /   |   __|   >   <   |      /     |   __|  |  |     |  |  |  | |  . `  | 
   \    /    |  |____ /  .  \  |  |\  \----.|  |____ |  `----.|  `--'  | |  |\   | 
    \__/     |_______/__/ \__\ | _| `._____||_______| \______| \______/  |__| \__| 
                                                                                   
</pre>
</p>

<p align="center">
  <b>Domain Reconnaissance & OSINT Framework</b><br/>
  <sub>For authorized penetration testing and bug bounty programs only.</sub>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/python-3.10%2B-blue?style=flat-square"/>
  <img src="https://img.shields.io/badge/license-MIT-green?style=flat-square"/>
  <img src="https://img.shields.io/badge/platform-linux-lightgrey?style=flat-square"/>
  <img src="https://img.shields.io/badge/purpose-bug%20bounty-orange?style=flat-square"/>
</p>

---

## Overview

vexRecon is an automated domain reconnaissance tool that chains together the best open-source security tools into a single workflow. Everything runs silently in the background while a live progress bar keeps you updated. At the end you get one clean `results.txt` file and a per-IP folder structure.

### What it does

| Phase | Tools used | Output |
|---|---|---|
| OSINT | requests (website.informer, whoxy) | Emails, registrar, IPs |
| TLD Fuzzing | dnsx | Related domains (basalam.net, .io, etc.) |
| DNS Records | dig | NS, A, MX, TXT, SPF analysis |
| Subdomain Enum | subfinder | Passive subdomain discovery |
| Subdomain Bruteforce | shuffledns | Active bruteforce with wordlist |
| DNS Resolution | dnsx | Resolves subdomains, filters unwanted IPs |
| HTTP Probing | httpx | Status codes, titles, tech stack, CDN detection |
| Reverse DNS | dig | PTR records for every discovered IP |
| SSL Probing | httpx | CN and SAN from TLS certificates |
| Virtual Hosts | ffuf | Hidden vhosts on non-CDN IPs |
| Port Scan | naabu | Top 1000 ports (optional, requires consent) |

---

## Output structure

```
vexRecon_example.com_20260101_120000/
├── results.txt          ← single clean report (subdomains, IPs, DNS, OSINT)
├── log.txt              ← timestamped run log
└── ips/
    ├── 1.2.3.4/
    │   ├── ptr.txt      ← reverse DNS results
    │   ├── ssl.txt      ← SSL certificate CN + SANs
    │   ├── ports.txt    ← open ports (if port scan was run)
    │   └── vhosts.txt   ← virtual hosts discovered on this IP
    └── 5.6.7.8/
        └── ...
```

`results.txt` groups subdomains by HTTP status code:

```
──────────────────────────────────────────────────────────────
  [200] Subdomains
──────────────────────────────────────────────────────────────
  www.example.com  [Nginx, React]  Home Page
  api.example.com

──────────────────────────────────────────────────────────────
  [301] Subdomains
──────────────────────────────────────────────────────────────
  old.example.com

──────────────────────────────────────────────────────────────
  IPs
──────────────────────────────────────────────────────────────
  1.2.3.4   PTR=host.provider.com   SSL=example.com   ports=80,443
  5.6.7.8   [Cloudflare]
```

---

## Requirements

### Python

Python 3.10 or higher is required.

```bash
pip install rich requests
```

### Go tools

All tools must be in your `$PATH`. vexRecon checks automatically on startup and tells you exactly what is missing and where to get it.

First make sure Go is installed and your PATH includes the Go bin directory:

```bash
# Install Go (if not already installed)
sudo apt install golang-go      # Debian/Ubuntu
# or download from https://go.dev/dl/

# Add Go binaries to PATH (add this to your ~/.bashrc or ~/.zshrc)
export PATH=$PATH:$(go env GOPATH)/bin
```

Install all required tools:

```bash
go install github.com/projectdiscovery/dnsx/cmd/dnsx@latest
go install github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
go install github.com/projectdiscovery/shuffledns/cmd/shuffledns@latest
go install github.com/projectdiscovery/httpx/cmd/httpx@latest
go install github.com/projectdiscovery/naabu/v2/cmd/naabu@latest
go install github.com/ffuf/ffuf/v2@latest
```

Install system tools:

```bash
sudo apt install dnsutils wget     # Debian/Ubuntu
# or
sudo dnf install bind-utils wget   # Fedora/RHEL
```

---

## Installation

```bash
git clone https://github.com/YOUR_USERNAME/vexRecon.git
cd vexRecon
pip install rich requests
```

---

## Usage

```bash
python3 vexRecon.py
```

You will be prompted for:

1. **Target domain** — e.g. `example.com`
2. **Path to tlds.txt** — the TLD list for fuzzing (default: `tlds.txt`, included in repo)
3. **Wordlist size** for shuffledns bruteforce:
   - `1` → 5,000 entries
   - `2` → 20,000 entries
   - `3` → 110,000 entries
   - `4` → Four-character subdomains *(recommended)*
4. **Port scan** — you must explicitly agree before naabu runs

### Example session

```
Target domain: example.com
Path to tlds.txt: tlds.txt
Wordlist: [4] Four-char

▶ Starting recon: example.com

 ████████████████████████ 100% • Writing report     0:04:12

✓ Done!  Output: vexRecon_example.com_20260101_120000/

  vexRecon_example.com_20260101_120000/results.txt
```

---

## tlds.txt format

Place this file next to `vexRecon.py`. One TLD per line, with the dot:

```
.net
.org
.co
.io
.app
.dev
.uk
.de
.ir
```

A sample `tlds.txt` is included in this repo.

---

## Notes

- All discovered IPs in the range `10.10.x.x` are automatically skipped (ISP-level filtering, not useful for recon).
- CDN IPs (Cloudflare, Akamai, Fastly, CloudFront) are detected and flagged separately. The report includes Shodan and Censys links to help you find the real IP behind the CDN.
- If the SPF record is missing `-all`, vexRecon warns you that email spoofing may be possible from that domain.
- Subdomains that return 404/403 without CDN are flagged as potential subdomain takeover candidates.
- Wordlist download requires internet access. If it fails, a built-in minimal wordlist is used automatically.

---

## Legal

> **This tool is for authorized security testing only.**
>
> Only use vexRecon against domains you own or have **explicit written permission** to test.
> Port scanning, subdomain enumeration, and probing systems without authorization may be
> illegal in your jurisdiction. The author is not responsible for any misuse of this tool.

---

## License

MIT — see [LICENSE](LICENSE)
