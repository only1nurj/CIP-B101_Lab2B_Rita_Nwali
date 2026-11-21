# Lab 2B: Advanced Linux Commands, Standard Streams and Controlled Evidence Transfer


**Author:** Rita Joan Nwali <br>
**Course:** DFI101: Basic Computer Skills for Digital Forensics <br>
**Lab:** CIP-B101 Lab 2B <br>
**Date:** 11 November 2026 <br>

A hands-on digital forensics training lab. It covers how Linux treats input/output as files, how command output is redirected into evidence logs, and how files are moved between two authorized machines over a controlled network while their integrity is proven with cryptographic hashes.

> **Authorization notice:** Everything here was performed on two machines I own and control in an isolated lab. Never run these techniques against a system you are not explicitly authorized to examine.

---

## Table of Contents

- [Objective](#objective)
- [Environment](#environment)
- [Lab Architecture](#lab-architecture)
- [Evidence Folder Structure](#evidence-folder-structure)
- [Part A: Standard Input, Output and Error](#part-a-standard-input-output-and-error)
- [Part B: Baseline System and Network Documentation](#part-b-baseline-system-and-network-documentation)
- [Part C: Hash the Evidence Before Transfer](#part-c-hash-the-evidence-before-transfer)
- [Part D: Linux to Windows Transfer](#part-d-linux-to-windows-transfer)
- [Part E: Windows to Linux Transfer](#part-e-windows-to-linux-transfer)
- [Part F: Test Image Transfer with dd](#part-f-test-image-transfer-with-dd)
- [Part G: dd Block Size Test (Optional)](#part-g-dd-block-size-test-optional)
- [Hash Verification Summary](#hash-verification-summary)
- [Troubleshooting and Lessons Learned](#troubleshooting-and-lessons-learned)
- [Part H: Security Reflection](#part-h-security-reflection)
- [Evidence Screenshot Index](#evidence-screenshot-index)
- [Key Takeaways](#key-takeaways)

---

## Objective

Acting as a junior digital forensic trainee, the goals were to:

1. Understand how Linux handles `stdin`, `stdout` and `stderr`.
2. Redirect command output and errors into evidence logs.
3. Transfer files between Kali Linux and Windows with `nc` / `ncat`.
4. Verify that no file was altered in transit using SHA-256 and MD5.
5. Document every action, including failures, as a forensic examiner would.

## Environment

| Component | Version |
|---|---|
| Oracle VirtualBox Manager | 7.2.6 r172322 |
| Kali Linux VM | 2026.1 |
| Windows 11 Pro (host) | 26H2 |
| Command Prompt | 10.0.26300.9974 |
| PowerShell | Build 26100, Revision 9818 |
| Tools | `nc` (netcat), `ncat` (Nmap), `dd`, `sha256sum`, `md5sum`, `certutil`, `tee` |

## Lab Architecture

```
┌──────────────────────────┐          TCP          ┌──────────────────────────┐
│  Kali Linux (VirtualBox) │ ───────────────────▶  │   Windows 11 Host        │
│  nc / sha256sum / dd     │   9090 / 9092         │   ncat / certutil        │
│                          │ ◀───────────────────  │                          │
│  ~/CIP-B101_Lab2B_...    │        9091           │  C:\CIP-B101_Lab2B_...   │
└──────────────────────────┘                       └──────────────────────────┘
```

| Port | Direction | Purpose |
|---|---|---|
| 9090 | Kali → Windows | `evidence_note.txt` |
| 9091 | Windows → Kali | `RitaNwaliEvidence.txt` |
| 9092 | Kali → Windows | `test_evidence_image.dd` (5 MB) |

> IP addresses and MAC addresses were redacted in all screenshots. Placeholders such as `<WINDOWS_IP>` and `<KALI_IP>` are used below.

## Evidence Folder Structure

```bash
mkdir -p ~/CIP-B101_Lab2B_Rita_Nwali/{screenshots,logs,evidence,received,hashes,report_notes}
cd ~/CIP-B101_Lab2B_Rita_Nwali
pwd
ls -la
```

```
CIP-B101_Lab2B_Rita_Nwali/
├── evidence/        # files under examination and test images
├── hashes/          # saved SHA-256 / MD5 values
├── logs/            # redirected command output and errors
├── received/
├── report_notes/
└── screenshots/
```

---

## Part A: Standard Input, Output and Error

### A1: Create a test evidence file

```bash
cd ~/CIP-B101_Lab2B_Rita_Nwali/evidence
echo "This is an authorized forensic evidence transfer test for Rita Nwali" > evidence_note.txt
cat evidence_note.txt
ls -l evidence_note.txt
```

`>` creates the file automatically. `ls -l` shows permissions, size (69 bytes) and timestamp.

### A2: Redirect standard output (`>`)

```bash
cd ~/CIP-B101_Lab2B_Rita_Nwali
ls -la evidence > logs/evidence_listing_stdout.txt
cat logs/evidence_listing_stdout.txt
```

The listing is written to a file instead of the screen. `>` overwrites the file if it already exists.

### A3: Redirect standard error (`2>`)

```bash
cat evidence/file_that_does_not_exist.txt 2> logs/error_stderr.txt
cat logs/error_stderr.txt
```

Only the `No such file or directory` error is captured. Normal output would still print to the terminal.

### A4: Redirect stdout and stderr together (`2>&1`)

```bash
(ls evidence && cat evidence/file_that_does_not_exist.txt) > logs/combined_output.txt 2>&1
cat logs/combined_output.txt
```

The parentheses group both commands so the redirect applies to both. The result is the file listing followed by the error message, in one file.

| Operator | Meaning |
|---|---|
| `>` | Redirect stdout (overwrite) |
| `2>` | Redirect stderr only |
| `2>&1` | Send stderr to wherever stdout is going |
| `\|` | Pipe stdout into another command |
| `tee` | Display output and write it to a file simultaneously |

---

## Part B: Baseline System and Network Documentation

### B1: Linux network details

```bash
hostname | tee logs/linux_hostname.txt
ip address | tee logs/linux_ip_address.txt
ip route | tee logs/linux_ip_route.txt
which nc || which ncat || echo "netcat or ncat not found" | tee logs/netcat_check.txt
```

- Hostname: `kali`
- Interfaces: loopback `lo` (127.0.0.1) and `eth0` (lab IP and MAC, redacted)
- `nc` is installed at `/usr/bin/nc`
- `||` runs the next command only if the previous one fails

### B2: Windows network details (Command Prompt)

```cmd
hostname
ipconfig
where ncat
certutil -?
```

- Hostname: `DESKTOP-SBPA495`
- `Ethernet 2` is the VirtualBox host-only network used by Kali; `Wi-Fi` holds the default gateway and is the main internet connection.
- The Wireless LAN and Bluetooth adapters showed `Media disconnected`.
- `ncat` was initially unavailable and was resolved by installing Nmap (see [Part D](#part-d-linux-to-windows-transfer)).

### B3: Connectivity test

```bash
# Kali -> Windows
ping -c 4 <WINDOWS_IP> | tee logs/ping_windows.txt
```
```powershell
# Windows -> Kali
ping <KALI_IP>
```

Both directions returned 4/4 replies with 0% packet loss. The TTL values differ as expected (64 = Linux, 128 = Windows).

---

## Part C: Hash the Evidence Before Transfer

```bash
cd ~/CIP-B101_Lab2B_Rita_Nwali/evidence
sha256sum evidence_note.txt | tee ../hashes/source_evidence_note_sha256.txt
md5sum    evidence_note.txt | tee ../hashes/source_evidence_note_md5.txt
cat ../hashes/source_evidence_note_sha256.txt
```

| Algorithm | Hash |
|---|---|
| SHA-256 | `5505ccb2405afd1e99a6425e744ba7dd96e72efe57b69885e97a45757a25faea` |
| MD5 | `fea0739cbdd2e28095891673bdacae0f` |

> The first attempt failed with `No space left on device` because the Kali disk was full. I enlarged the virtual disk and reran the commands successfully.

---

## Part D: Linux to Windows Transfer

### D1: Start the listener on Windows

`ncat` was not installed on the Windows host. I installed **Nmap** (with the **Ncat** option ticked), closed Command Prompt, and reopened it so the new `PATH` was loaded.

```cmd
cd C:\CIP-B101_Lab2B_Received
ncat -l -p 9090 > received_evidence_note.txt
```

### D2: Send from Kali

```bash
cd ~/CIP-B101_Lab2B_Rita_Nwali/evidence
nc <WINDOWS_IP> 9090 < evidence_note.txt
# Press Ctrl+C once the file has arrived
```

`nc` does not exit on its own, so `Ctrl+C` was used after confirming the file appeared on the Windows side.

### D3: Verify on Windows

```cmd
certutil -hashfile received_evidence_note.txt SHA256
certutil -hashfile received_evidence_note.txt MD5
type received_evidence_note.txt
```

Both hashes **match** those generated on Kali, and the content reads: *This is an authorized forensic evidence transfer test for Rita Nwali*.

---

## Part E: Windows to Linux Transfer

### E1: Create the file on Windows

```cmd
cd C:\CIP-B101_Lab2B_Received
echo hello Rita Nwali! > RitaNwaliEvidence.txt
type RitaNwaliEvidence.txt
certutil -hashfile RitaNwaliEvidence.txt SHA256
```

SHA-256: `e9be431f2636b8fe003fb720ca3b9be90fee45cfdb61f538e31f572c97b75c20`

### E2: Listener on Kali

```bash
nc -l -p 9091 > RitaNwaliEvidence_received.txt
```

### E3: Send from Windows

```cmd
ncat <KALI_IP> 9091 < RitaNwaliEvidence.txt
```

### E4: Verify on Kali

```bash
cat RitaNwaliEvidence_received.txt
sha256sum RitaNwaliEvidence_received.txt | tee ../hashes/received_windows_file_sha256.txt
```

The content reads `hello Rita Nwali!` and the SHA-256 matches the Windows value exactly.

---

## Part F: Test Image Transfer with dd

A small synthetic image is used instead of imaging a real disk, so no real evidence or private data is touched.

### F1: Create the test image

```bash
cd ~/CIP-B101_Lab2B_Rita_Nwali/evidence

# 5 MB file filled with zeros
dd if=/dev/zero of=test_evidence_image.dd bs=1M count=5 status=progress

# Write an identifiable marker at byte offset 1024 without truncating the file
echo "ICDFA test evidence marker - Rita Nwali" | dd of=test_evidence_image.dd bs=1 seek=1024 conv=notrunc

ls -lh test_evidence_image.dd
sha256sum test_evidence_image.dd | tee ../hashes/source_test_image_sha256.txt
```

`conv=notrunc` is essential: without it, `dd` would truncate the file to the marker's length.

SHA-256: `236c4bd6674e825391cc9deebbcabf66259b08d7e44271aba9c43d3d0e4cd20f`

### F2: Transfer to Windows

```cmd
:: Windows listener
ncat -l -p 9092 > test_evidence_image_received.dd
```
```bash
# Kali sender
nc <WINDOWS_IP> 9092 < test_evidence_image.dd
```

### F3: Verify on Windows

```cmd
certutil -hashfile test_evidence_image_received.dd SHA256
```

The hash is identical to the Kali source hash, so the 5 MB image was transferred without change.

---

## Part G: dd Block Size Test (Optional)

I downloaded a `dd_obs_test.sh` benchmarking script with `wget`, made it executable, and ran it to compare write speeds across block sizes, saving results to `dd_block_size_test_result.txt`.

```bash
wget <script-url>
chmod +x dd_obs_test.sh
./dd_obs_test.sh | tee dd_block_size_test_result.txt
```

- **Fastest block size:** 4194304 bytes (4 MiB) at **76.7 MB/s**
- **Caveat:** the script warned that results may be inaccurate because the kernel cache was not cleared (this requires `sudo`).

---

## Hash Verification Summary

| File | Origin | Destination | SHA-256 match |
|---|---|---|---|
| `evidence_note.txt` | Kali | Windows (`received_evidence_note.txt`) | ✅ |
| `RitaNwaliEvidence.txt` | Windows | Kali (`RitaNwaliEvidence_received.txt`) | ✅ |
| `test_evidence_image.dd` | Kali | Windows (`test_evidence_image_received.dd`) | ✅ |

---

## Troubleshooting and Lessons Learned

Failures were documented rather than hidden, as good forensic practice requires.

| Problem | Cause | Resolution |
|---|---|---|
| `tee: No space left on device` | Kali virtual disk was full | Enlarged the virtual disk, reran the commands |
| `ncat` not recognized on Windows | Nmap/Ncat not installed | Installed Nmap with Ncat selected, reopened Command Prompt |
| `nc` did not return after sending | `nc` keeps the connection open by default | Verified arrival, then stopped with `Ctrl+C` |
| Test image hash changed between attempts | File was modified when the marker was rewritten with the real name | Recorded and used the new hash for comparison |
| Network timeouts during transfer | Listener not ready / connection interrupted | Restarted the listener before sending, retried |

---

## Part H: Security Reflection

**1. Standard output vs. standard error?**
`stdout` carries a command's normal results and `stderr` carries its error messages. They can be redirected separately with `>` and `2>`.

**2. Why is redirection useful in forensic documentation?**
It creates a permanent, organized record of exactly what was run and what happened, making the work easy to review, repeat and defend.

**3. Why hash a file before and after transfer?**
A hash is a digital fingerprint. Matching hashes prove the file was not altered or corrupted in transit; differing hashes mean the evidence cannot be trusted.

**4. Why is unauthorized remote shell access unsafe and unethical?**
It is illegal, violates privacy, risks accidental damage or alteration of data, and destroys an examiner's credibility. Examiners should only work on systems they have written permission to examine.

**5. Bind shell vs. reverse shell?**
In a bind shell, the target opens a port and waits for the attacker to connect. In a reverse shell, the target connects back to the attacker's listener, which more easily bypasses firewalls that block inbound connections.

**6. Why might a firewall block one connection but allow another?**
Rules are based on direction, port and program. Many firewalls block inbound connections by default while allowing outbound ones, so an outbound connection on an approved port may pass while an unexpected inbound one is stopped.

**7. Why a small test image instead of a real disk?**
It is safe, fast and fits easily in the lab, allowing practice with hashing and transfer without touching real evidence or raising legal and privacy issues.

---

## Evidence Screenshot Index

Place screenshots in `screenshots/` using the IDs below.

| ID | Description |
|---|---|
| EVID_01 | Evidence folder structure |
| EVID_02 | Test evidence text file |
| EVID_03 | Redirect standard output |
| EVID_04 | Error message saved to stderr file |
| EVID_05 | Redirect stdout and stderr together |
| EVID_06 | Linux IP address and network details |
| EVID_07 | Windows host IP configuration |
| EVID_08 | Ping Kali from Windows |
| EVID_09 | Ping Windows from Kali |
| EVID_10 | SHA-256 and MD5 hash values |
| EVID_11 | Listener on Windows |
| EVID_12 | Send file from Linux |
| EVID_13 | Received file content and hash (Windows) |
| EVID_14 | Evidence file created on Windows |
| EVID_15 | Listener on Linux |
| EVID_16 | Send file from Windows |
| EVID_17 | Received file content and hash (Linux) |
| EVID_18 | Test evidence image creation |
| EVID_19 | Windows listener for image |
| EVID_20 | Linux sender for image |
| EVID_21 | Image hash on Windows |
| EVID_22 | dd block size test results |

## Key Takeaways

- Redirection (`>`, `2>`, `2>&1`) and `tee` turn terminal activity into a reviewable evidence trail.
- Hashing before and after every transfer is what proves integrity.
- Failed attempts (a full disk, a missing tool, a timeout) are part of the record and should be documented and fixed, not hidden.
- Only work on systems you are authorized to examine.

---

*Educational lab project. All activity was performed in a controlled environment on systems owned by the author.*
