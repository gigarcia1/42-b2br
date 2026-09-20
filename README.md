*This project has been created as part of the 42 curriculum by gigarcia.*

## Description
**Born2beRoot** is a System Administration project that introduces the concepts of virtualization, operating systems, and basic server setup. The goal is to create a secure, minimal server environment inside a Virtual Machine (VM) using strict guidelines. This includes configuring logical volumes with LVM, implementing a strong firewall, setting up SSH for secure remote access, and enforcing strict password and user policies without relying on any graphical interface.

## Project Description & Technical Choices

### Choice of Operating System
For this project, I chose **Debian**, as it's highly recommended for beginners in system administration due to its stability, extensive community documentation, and user-friendly package management (APT). Rocky Linux is an excellent enterprise-grade OS, but its reliance on SELinux makes the initial learning curve significantly steeper for a foundational project.

### Main Design Choices
*   **Partitioning (LVM):** The disk is encrypted and managed using Logical Volume Manager (LVM). This allows for flexible partition management (resizing, adding drives) without downtime.
*   **Security Policies:** 
    *   Root login via SSH is disabled to prevent brute-force attacks on the superuser.
    *   A strict password policy is enforced (expiration, complexity, and history) via `libpam-pwquality`.
*   **User Management:** The system has a dedicated user (`<login>`) added to the `user42` and `sudo` groups.
*   **Services:** Minimal services are installed. Only SSH (`sshd`) is running to allow remote connections on the custom port `4242`.

### Structure of the partitions when running lsblk
```bash
NAME                     MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sda                        8:0    0     8G  0 disk  
├─sda1                     8:1    0   487M  0 part  /boot
├─sda2                     8:2    0     1K  0 part  
└─sda5                     8:5    0   7.5G  0 part  
  └─sda5_crypt           254:0    0   7.5G  0 crypt 
    ├─<login>--vg-root   254:1    0   2.8G  0 lvm   /
    ├─<login>--vg-swap_1 254:2    0   976M  0 lvm   [SWAP]
    └─<login>--vg-home   254:3    0   3.8G  0 lvm   /home
sr0                       11:0    1  1024M  0 rom
```

### Technical Comparisons

#### Debian vs Rocky Linux
*   **Debian:** A community-driven Linux distribution known for its rock-solid stability and vast software repositories. It uses the `apt` package manager (handling `.deb` packages) and defaults to AppArmor for security.
*   **Rocky Linux:** A downstream, complete binary-compatible release using the Red Hat Enterprise Linux (RHEL) operating system source code. It uses the `dnf`/`yum` package manager (handling `.rpm` packages) and defaults to SELinux. It is geared more towards enterprise server environments.

#### AppArmor vs SELinux
Both are Mandatory Access Control (MAC) systems that provide an extra layer of security by restricting what programs can do.
*   **AppArmor:** Uses a path-based approach (restricts access based on file paths). It is generally easier to configure and learn, which is why it is the default on Debian and Ubuntu.
*   **SELinux:** Uses an inode-based approach with security labels attached to every file, process, and object. It offers extremely fine-grained control but is notoriously complex to master.

#### UFW vs firewalld
*   **UFW (Uncomplicated Firewall):** A front-end for `iptables` (or `nftables`). It is designed to be user-friendly and uses simple commands to allow or deny traffic based on ports or IPs. Default on Debian/Ubuntu.
*   **firewalld:** Uses a "zone" based concept to manage network traffic (e.g., public, home, work) and allows dynamic changes without dropping existing connections. It is more robust for complex networking but slightly more complex to set up. Default on RHEL/Rocky.

#### VirtualBox vs UTM
*   **VirtualBox:** A popular, cross-platform Type-2 hypervisor (runs on top of a host OS) by Oracle. It is highly compatible with x86/amd64 architectures but does not support Apple Silicon (M1/M2/M3) chips effectively.
*   **UTM:** A virtualization and emulation application for macOS and iOS. Under the hood, it uses QEMU. It is essential for students using modern Macs (Apple Silicon) because it can both virtualize ARM operating systems natively and emulate x86 architectures.

## Instructions

### Execution and Installation
1.  Clone this repository (which only contains this README and the `signature.txt`).
2.  The actual project is a Virtual Machine. To run it, you need to import the `.vdi` (VirtualBox) or `.qcow2` (UTM) disk image into your hypervisor.
3.  Start the Virtual Machine.
4.  To connect to the machine via SSH from your host terminal, use the following command (assuming port forwarding is set up from host 4242 to guest 4242):
    ```bash
    ssh <login>@localhost -p 4242
    ```
5.  Enter the password for the user when prompted.

## Further instructions and Useful Commands

### 1. Password Policy Management
The system enforces strict password rules: a 30-day expiration, a minimum of 2 days between changes, a 7-day warning period, and strict character complexity limits.
*   **Package Required:** `sudo apt install libpam-pwquality`
*   **Expiration Configuration:** Edit `/etc/login.defs` (Set `PASS_MAX_DAYS 30`, `PASS_MIN_DAYS 2`, `PASS_WARN_AGE 7`).
*   **Complexity Configuration:** Edit `/etc/pam.d/common-password` (Enforces the 10-character minimum, uppercase, lowercase, numbers, and rejects 3 consecutive identical characters).
*   **Verify User Expiration:** `chage -l <username>`

### 2. Firewall (UFW)
The UFW firewall is active on startup and restricts all incoming connections except for the mandated SSH port.
*   **Check Status & Rules:** `sudo ufw status verbose`
*   **Allow Port:** `sudo ufw allow -port number-`
*   **Delete Rule:** `sudo ufw status numbered` followed by `sudo ufw delete <rule_number>`

### 3. Sudo Configuration
Authentication is limited to 3 attempts, requires a secure TTY session, restricts available executable paths, and logs all inputs/outputs.
*   **Sudoers File Editing:** Execute `sudo visudo` to safely edit permissions and defaults.
*   **Logs Directory:** All actions are systematically stored in `/var/log/sudo/`.
*   **View Sudo Logs:** `sudo cat /var/log/sudo/sudo.log`

### 4. User and Group Management
*   **Create New User:** `sudo adduser <username>`
*   **Create New Group:** `sudo groupadd <groupname>`
*   **Add User to Group:** `sudo usermod -aG <groupname> <username>`
*   **Check User Groups:** `groups <username>`
*   **List Group Members:** `getent group <groupname>`

### 5. Cron & Monitoring Script
A Bash script (`monitoring.sh`) executes every 10 minutes to broadcast real-time system metrics to all active terminals via the `wall` command.
*   **Edit Cron Jobs:** `sudo crontab -u root -e`
*   **Cron Syntax (Every 10 mins):** `*/10 * * * * bash /path/to/monitoring.sh`

## Resources
*   [Debian Official Documentation](https://www.debian.org/doc/)
*   [LVM Administrator's Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_logical_volumes/index)
*   [UFW Manual](https://help.ubuntu.com/community/UFW)
*   **AI Usage:** Artificial Intelligence (like ChatGPT/Gemini) was used strictly as a conceptual tutor to understand the differences between MAC systems (AppArmor vs SELinux) and to brainstorm the structure of this documentation. No configurations or scripts were directly copied from AI tools, adhering to the 42 pedagogy.
