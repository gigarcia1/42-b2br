# Born2beRoot Complete Guide

## Part 1 — Project Overview

The Born2beRoot project is your introduction to the world of virtualization and system administration. You are building a secure, headless (no graphical user interface) Linux server from scratch.   
PDF+ 1

In the real world, software doesn't run in a vacuum; it runs on servers. Knowing how to write a C program or a Python script is great, but knowing how to securely deploy, monitor, and restrict access to the environment where that code lives is what makes you a complete software engineer. This project solves the problem of "blind deployment" by forcing you to understand disk encryption, logical volume management, network security, and user privilege escalation.

### High-Level Architecture:
```text
[ VirtualBox Hypervisor ]
       |
       v
[ Debian OS (No GUI) ]
       |-- [ UFW Firewall (Port 4242 Only) ]
       |-- [ AppArmor (Kernel Security) ]
       |
       |-- Storage Stack:
            |-- Physical Disk (/dev/sda)
                 |-- /boot (Unencrypted, starts the OS)
                 |-- LUKS Encrypted Container
                      |-- LVM (Logical Volume Manager)
                           |-- root (/)
                           |-- swap
                           |-- home (/home)
```

## Part 2 — Core Concepts Deep Dive

### 1. LVM (Logical Volume Manager)
- **The Metaphor:** Imagine your hard drive is a big block of Lego. Normally, if you build a house (a partition), it's stuck that size. LVM is like melting the Legos down into a liquid pool of plastic. From that pool, you can pour exact molds (Logical Volumes) for your root, swap, and home. If you need a bigger `/home` later, you just pour more plastic into that mold. It's dynamic memory allocation, but for your hard drive.
- **Technical Definition:** A storage virtualization technology that provides a layer of abstraction between the physical disks and the file system.
- **Why It Was Used Here:** The subject requires at least 2 encrypted partitions using LVM. LVM allows us to encrypt one single massive physical partition (the pool) and then safely divide it into smaller logical volumes inside the encrypted safe.
- **Common Pitfalls:** Forgetting that LVM sits inside the LUKS encryption. If you encrypt the logical volumes separately, you'll have to type a password for every single volume on boot.   
PDF

### 2. LUKS (Linux Unified Key Setup)
- **The Metaphor:** A bank vault door. Everything inside the vault (your LVM pool) is completely inaccessible and looks like gibberish data to anyone who steals the hard drive. You need the master combination (your password) to open the vault when the server boots.
- **Technical Definition:** The standard for Linux hard disk encryption. It encrypts block devices at rest.
- **Why It Was Used Here:** To protect data at rest. If someone clones your VirtualBox `.vdi` file, they cannot mount it and read your files without the decryption passphrase.

### 3. SSH (Secure Shell)
- **The Metaphor:** A secure, armored tunnel between two castles. Instead of sending messages in the open where anyone can read them, SSH encrypts the messages.
- **Technical Definition:** A cryptographic network protocol for operating network services securely over an unsecured network.
- **Why It Was Used Here:** You must connect to your VM remotely via port 4242. It prevents sending passwords in plain text.
- **Common Pitfalls:** Leaving `PermitRootLogin yes` enabled. The subject strictly forbids root login via SSH.   
PDF+ 1

### 4. Sudo (Superuser Do)
- **The Metaphor:** Borrowing the CEO's security badge for 5 minutes to unlock a specific door, rather than making yourself the CEO permanently.
- **Technical Definition:** A program that allows users to run programs with the security privileges of another user, by default the superuser.
- **Why It Was Used Here:** It provides granular control over who can do what, and logs every single command.
- **Common Pitfalls:** Breaking the `/etc/sudoers` file by using `nano` or `vim` directly. Always use `visudo`, which checks for syntax errors before saving.   
PDF

## Part 3 — File-by-File Code Walkthrough

### 1. monitoring.sh
- **Purpose:** A Bash script that runs every 10 minutes to broadcast system metrics to all active terminals using `wall`.
- **Architecture role:** It acts as an automated system monitor.   
PDF

**Function breakdown & Tricky lines:**
```bash
#!/bin/bash
# 1. Architecture: 'uname -a' prints all system info.
arc=$(uname -a)

# 2. Physical CPUs: 'nproc' or parsing /proc/cpuinfo.
# Getting only physical sockets.
pcpu=$(grep "physical id" /proc/cpuinfo | sort | uniq | wc -l)

# 3. Virtual CPUs: 
vcpu=$(grep "^processor" /proc/cpuinfo | wc -l)

# 4. Memory Usage: 'free -m' shows memory in Megabytes.
# Tricky line: awk is used to parse the 2nd (total) and 3rd (used) columns.
fram=$(free -m | awk '$1 == "Mem:" {print $2}')
uram=$(free -m | awk '$1 == "Mem:" {print $3}')
pram=$(free | awk '$1 == "Mem:" {printf("%.2f"), $3/$2*100}')

# 5. Disk Usage: 'df -Bm' gets block usage in Megabytes.
fdisk=$(df -Bg | grep '^/dev/' | grep -v '/boot$' | awk '{ft += $2} END {print ft}')
udisk=$(df -Bm | grep '^/dev/' | grep -v '/boot$' | awk '{ut += $3} END {print ut}')
pdisk=$(df -Bm | grep '^/dev/' | grep -v '/boot$' | awk '{ut += $3; ft+= $2} END {printf("%d"), ut/ft*100}')

# 6. CPU Load: 'top' or 'vmstat'
cpul=$(top -bn1 | grep '^%Cpu' | cut -c 9- | xargs | awk '{printf("%.1f%%"), $1 + $3}')

# 7. Last boot:
lb=$(who -b | awk '$1 == "system" {print $3 " " $4}')

# 8. LVM use: check if 'lsblk' contains "lvm"
lvmu=$(if [ $(lsblk | grep "lvm" | wc -l) -eq 0 ]; then echo no; else echo yes; fi)

# 9. TCP Connections: 'ss' checks socket statistics.
tcpc=$(ss -neopt state established | wc -l)

# 10. User log:
ulog=$(users | wc -w)

# 11. Network (IP & MAC):
ip=$(hostname -I)
mac=$(ip link show | grep "ether" | awk '{print $2}')

# 12. Sudo cmd:
cmnd=$(journalctl _COMM=sudo | grep COMMAND | wc -l)

# Broadcast
wall "  #Architecture: $arc
        #Physical CPU: $pcpu
        #vCPU: $vcpu
        #Memory Usage: $uram/${fram}MB ($pram%)
        #Disk Usage: $udisk/${fdisk}Gb ($pdisk%)
        #CPU load: $cpul
        #Last boot: $lb
        #LVM use: $lvmu
        #TCP Connections: $tcpc ESTABLISHED
        #User log: $ulog
        #Network: IP $ip ($mac)
        #Sudo: $cmnd cmd"
```

**What would break if this is wrong?** The evaluator will see empty fields or syntax errors popping up on their terminal.

### 2. /etc/sudoers
- **Purpose:** Enforces the strict sudo rules.

**Tricky lines:**
- `Defaults passwd_tries=3` (Limits password attempts).   
PDF
- `Defaults badpass_message="Wrong password, try again!"` (Custom error message).   
PDF
- `Defaults logfile="/var/log/sudo/sudo.log"` (Logs all actions).   
PDF
- `Defaults requiretty` (Requires a terminal, preventing background scripts from escalating privileges).   
PDF
- `Defaults secure_path="..."` (Limits the directories where sudo looks for executables).   
PDF

### 3. /etc/login.defs & /etc/pam.d/common-password
- **Purpose:** Enforces the password expiration and complexity.

**Tricky lines in common-password:**
```text
password requisite pam_pwquality.so retry=3 minlen=10 ucredit=-1 lcredit=-1 dcredit=-1 maxrepeat=3 reject_username enforce_for_root
```
This single line forces the 10-char length, 1 uppercase (ucredit), 1 lowercase (lcredit), 1 digit (dcredit), stops 3 repeating chars, and enforces it on root.   
PDF+ 1

## Part 4 — System Architecture Analysis

**LVM vs Standard Partitioning**
- **Category:** Storage Architecture.
- **Complexity (Flexibility):** Standard partitions are O(N) to resize (you often have to move adjacent partitions sector-by-sector). LVM resizing is O(1) in effort (just allocate available extents from the Volume Group).
- **Why chosen:** Required by subject. It allows us to put `/`, swap, and `/home` inside one single LUKS encrypted block.   
PDF

**Concrete Trace:**
- Physical Disk (`/dev/sda` - 8GB)
  - Boot Partition (`/dev/sda1` - 500MB) -> Unencrypted.
  - LUKS Container (`/dev/sda5` - 7.5GB) -> Encrypted. Password required here.
    - Volume Group (`vg0` - 7.5GB)
      - Logical Volume (`vg0-root` - 3GB) -> Mounted to `/`
      - Logical Volume (`vg0-swap` - 1GB) -> Used as SWAP.

## Part 5 — Design Decisions Log

- **Decision:** Chose Debian over Rocky Linux.
  - **Alternatives:** Rocky Linux.
  - **Why:** Debian's `apt` package manager and AppArmor are more straightforward for beginners compared to Rocky's `dnf` and incredibly strict SELinux.   
  PDF

- **Decision:** Using UFW instead of iptables directly.
  - **Alternatives:** Raw iptables.
  - **Why:** UFW (Uncomplicated Firewall) provides a simple wrapper around iptables, making it much harder to accidentally lock yourself out of the SSH port.

- **Decision:** Scheduling the script with cron.
  - **Alternatives:** Systemd timers or an infinite `while` loop with `sleep`.
  - **Why:** `cron` is the industry standard for time-based job scheduling and is explicitly hinted at in the subject.   
  PDF

## Part 6 — Subject Compliance Checklist

- ✅ **NO GUI:** Verified by base installation without X.org/Wayland.   
PDF
- ✅ **LVM Encrypted Partitions:** `lsblk` shows crypt and lvm layers.   
PDF
- ✅ **SSH Port 4242:** Modified `/etc/ssh/sshd_config` to Port 4242.   
PDF
- ✅ **Root SSH disabled:** `PermitRootLogin no` in `sshd_config`.   
PDF
- ✅ **UFW Active:** `sudo ufw status` shows active, only 4242 allowed.   
PDF
- ✅ **Password Policy:** `login.defs` and `pam_pwquality` configured.   
PDF
- ✅ **Sudo strict rules:** `visudo` configured with TTY, logfile, and limits.   
PDF
- ✅ **monitoring.sh:** Script exists and outputs correctly via `cron`.   
PDF
- ⚠️ **signature.txt:** Needs manual verification that the text file matches the `.vdi` shasum.   
PDF

## Part 7 — Peer Defense Preparation

### 20 Questions the Evaluator Will Ask

**Conceptual:**
1. **What is a virtual machine?**
   Answer: It's a software-based emulation of a physical computer. It runs an OS just like physical hardware, but shares the physical resources of the host machine.
2. **What is the difference between aptitude and apt?**
   Answer: Both are frontends for dpkg. `apt` is newer, more user-friendly, and standard for scripting. `aptitude` has an interactive text-based interface and is sometimes better at resolving complex dependency conflicts.
3. **What is AppArmor?**
   Answer: A Mandatory Access Control (MAC) system in Linux that confines programs to a limited set of resources using profiles loaded into the kernel.   
   PDF
4. **What is LVM?**
   Answer: Logical Volume Manager. It abstracts physical hard drives into volume groups, allowing us to dynamically resize logical partitions without messing with physical disk sectors.
5. **Why do we need a separate, unencrypted /boot partition?**
   Answer: The BIOS/UEFI needs to read the bootloader (GRUB) and the initial ramdisk to prompt for the LUKS password. If `/boot` was encrypted, the computer wouldn't know how to turn on to ask for the password.

**Code & Config-specific:**
6. **How does UFW work under the hood?**
   Answer: It's a user-friendly frontend for iptables, which interacts with the netfilter framework in the Linux kernel to drop or accept packets.
7. **How did you enforce the 3 sudo password attempts?**
   Answer: By adding `Defaults passwd_tries=3` in `/etc/sudoers` using `visudo`.
8. **What is a TTY and why did you require it for sudo?**
   Answer: TTY stands for Teletypewriter (a terminal). Requiring it (`requiretty`) prevents malicious background scripts or cron jobs from executing sudo commands automatically without human interaction.
9. **Explain the awk command in your memory usage line.**
   Answer: `awk` reads the output of `free -m`. When it finds the line starting with "Mem:", it grabs the 2nd column (Total) and 3rd column (Used) to do the math.
10. **Why did you use wall?**
    Answer: `wall` stands for "write to all". It broadcasts a message to the terminals of all logged-in users, which satisfies the subject's requirement.   
    PDF+ 2

**Command & Task execution:**
11. **Check if AppArmor is running.**
    Answer: Run `sudo aa-status`.
12. **Check your UFW rules.**
    Answer: Run `sudo ufw status numbered`.
13. **Show me that root cannot login via SSH.**
    Answer: Show `/etc/ssh/sshd_config` and point to `PermitRootLogin no`.
14. **Check when the password for your user expires.**
    Answer: Run `sudo chage -l <username>`.
15. **Show me the sudo logs.**
    Answer: `sudo cat /var/log/sudo/sudo.log`.
16. **Create a new user.**
    Answer: `sudo adduser new_evaluator`.
17. **Create a new group called evaluating.**
    Answer: `sudo groupadd evaluating`.
18. **Add the new user to the evaluating group.**
    Answer: `sudo usermod -aG evaluating new_evaluator`.
19. **Modify the hostname from <login>42 to <login>42-eval.**
    Answer: Edit `/etc/hostname` and `/etc/hosts`, change the name, and run `sudo hostnamectl set-hostname <login>42-eval`, then reboot.
20. **Stop the monitoring.sh script from running.**
    Answer: Run `sudo crontab -e` and comment out (`#`) the line executing the script.   
    PDF+ 1

### Live Demo Script

Follow these steps mechanically during the evaluation:
- **The Signature Check:** Before booting, open a terminal on your physical host. Run `shasum path/to/your/machine.vdi`. Show the evaluator that the output exactly matches `signature.txt`.   
  PDF
- **Boot & Unlock:** Start VirtualBox. Type your LUKS decryption password when prompted. Log in as your non-root user.
- **SSH Connection:** Open a terminal on your physical host machine. Connect using: `ssh your_login@127.0.0.1 -p 4242`. Explain that VirtualBox port forwarding routes host 4242 to guest 4242.
- **Sudo Failure Test:** In the SSH session, run `sudo ls`. Intentionally type the wrong password 3 times. Show that it prints your custom error message and locks you out.
- **Show LVM:** Run `lsblk`. Point out the `disk -> part -> crypt -> lvm` hierarchy.
- **Show Cron:** Run `sudo crontab -l` to prove the script runs every 10 minutes.   
  PDF

### Closing Statement
"This project has provided a solid foundation in Linux system administration. By manually configuring LVM, enforcing strict kernel-level access controls with AppArmor, and implementing secure networking and privilege escalation policies, I now deeply understand the architecture underlying modern server infrastructure. I am comfortable defending the technical 'why' behind every configuration decision made here."

## Part 8 — Technical Reference Cheat Sheet

| Term | Kid Explanation | Technical Definition | Syntax Example |
|---|---|---|---|
| **LVM** | Melting and reshaping storage Legos. | Logical Volume Manager; abstracts physical storage. | `lsblk` (shows lvm type) |
| **LUKS** | A digital bank vault for your data. | Linux Unified Key Setup; block device encryption. | `cryptsetup` |
| **UFW** | A bouncer checking VIP passes (ports). | Uncomplicated Firewall; iptables frontend. | `sudo ufw allow 4242` |
| **Cron** | A robotic alarm clock that runs code. | Time-based job scheduler in Unix-like systems. | `crontab -e` |
| **TTY** | A physical or virtual keyboard/screen. | Teletypewriter; the standard terminal interface. | `Defaults requiretty` |
| **PAM** | The ID scanner at the server entrance. | Pluggable Authentication Modules. | `pam_pwquality.so` |

## Resources
*(AI Formatting and Structuring Assistant)*

> This README was formatted, structured, and validated with the assistance of an AI tool to ensure clean Markdown syntax, professional readability, and logical organization of the technical content. All core explanations, scripts, conceptual definitions, and defensive strategies remain entirely original and unedited.
