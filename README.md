# Forensic-Science-Analysis

### Forensic Science Analysis – Forensic Examination of the Swap File

This article will interest beginner researchers in the field of cybersecurity.
All the experiments are done using basic tools in **GNU/Linux** (like `grep`, `dd`, `cryptsetup`, etc.) without using any specialized forensic software.

### Disclaimer / Warning and Limitation of Liability

**Warning & Disclaimer:**
This article is prepared **for educational and research purposes only**.
All experiments were carried out in an **isolated test system**: Debian GNU/Linux 13.

**Legal aspects:**
The materials in this article are **not an invitation** to use these techniques to access any third-party system without **explicit written authorization** from the owner. Doing so is **against the law**.
Changing system settings can also cause **irreversible data loss**.

**Disclaimer:**
I am **not responsible** for any damage, data loss, system malfunction, or **legal violations** resulting from the use of the information in this article.

**Before you try anything:**
Use a **virtual machine** or a **test system**, back up your data, and make sure you have the required qualifications.
If you continue reading and perform the experiments, **you are fully responsible for the results.**

### Let’s Begin

Many people know that it’s possible to extract memory dumps from **RAM** or storage devices.
But have you ever wondered about the **swap file**?

The **swap file** is rarely mentioned in forensic analysis.
However, unlike **RAM**, which gets cleared after a reboot, the **swap** doesn’t get erased immediately.
Swap works in a **cyclic** manner — old data gets overwritten with new data over time.

### Theory

The **swap file** was created to reduce pressure on **RAM**.
In early systems, entire processes were swapped out to free RAM.
Later, with the introduction of **Virtual Memory (VM)**, only **small sections (pages)** of memory were swapped out, not entire processes.

In systems with **VM**, each program runs using **virtual addresses**, and the **CPU** translates those into **physical addresses**.
When RAM fills up, parts of the data are moved to the swap.
This makes swap management faster and more efficient than the old full-process system.

Since there’s **no strict standard** for swap, it can contain a lot of **valuable forensic artifacts**, such as:

* Screenshots, browser images
* Fragments of cookies or browser history (even if deleted)
* Seed phrases, game tokens
* Passwords, cache data

On Linux, you can control how “aggressive” swap usage is with the **vm.swappiness** parameter (value between 0 and 100, default = 60).
Higher values make swap usage more aggressive, e.g.:

```bash
sudo sysctl vm.swappiness=70
```


### Practical Part

**Tools used:**

* `dd`
* `swap_digger`
* `grep`

**Installation:**

`dd` and `grep` come by default.

For `swap_digger`:

```bash
git clone https://github.com/sevagas/swap_digger.git
cd swap_digger
sudo chmod +x swap_digger.sh
```

**Tool functions:**

* `dd` → creates a dump of the swap
* `swap_digger` → converts the dump into a readable `.txt` file
* `grep` → searches for specific data inside the dump


### Step 1: Increase Swap Aggressiveness

```bash
sudo sysctl vm.swappiness=100
```

Now fill the swap with data you can later extract — for example, create a **VeraCrypt container**, browse the web, etc.


### Step 2: Locate the Swap File

To find where swap is stored:

```bash
sudo swapon --show
# or
sudo cat /proc/swaps
```

Example output: `/dev/dm-2` (this may differ on your system).


### Step 3: Create a Swap Dump

First, temporarily turn off swap:

```bash
sudo swapoff -a
```

Then copy the data using `dd`:

```bash
sudo dd if=/path/to/swap of=/path/to/save.img bs=4M status=progress
```

Turn swap back on:

```bash
sudo swapon -a
```

You’ll now have a `.img` file that looks mostly unreadable (about 80% garbage and artifacts).
To convert it into a readable format:

```bash
cd swap_digger
sudo ./swap_digger.sh -v -a -l -s /path/to/dump.img
```

The output is usually stored at `/tmp/swap_dig/swap_dump.txt`.
Move it to your Desktop for easier access:

```bash
sudo cp /tmp/swap_dig/swap_dump.txt /home/user/Desktop
```

### Step 4: Extract Data with `grep`

Example: searching for browser search queries:

```bash
grep "searchrequest" swap_dump.txt
```

To find user or password info:

```bash
grep "SUDO_USER=" swap_dump.txt
grep -A 5 -B 5 "p@ssw0rd" swap_dump.txt
```

(`-A` shows lines after the match, `-B` before the match.)

This experiment shows that command history and even **superuser passwords** can appear in swap dumps.


### VeraCrypt Case Study

VeraCrypt tries to protect passwords so they don’t appear in RAM or swap.
However, experiments show two main scenarios when creating a VeraCrypt container:

1. **Password entered normally (no interaction):**

Example output from `grep`:

```
dfghjkc
dfghjkc
```

Even though the password appears in the dump, we can’t see how it was entered.
Nearby lines reveal desktop environment components (XFCE), meaning VeraCrypt itself wasn’t responsible for the leak.

2. **Password interacted with (viewed, selected, cursor inside):**

When interaction occurs, the chance of password exposure in swap increases significantly.
Different examples show the password being handled by:

* The XFCE process,
* VeraCrypt GUI,
* Icon catalogs of the desktop environment,
* Or even browser processes running simultaneously.


### Protection / Mitigation Methods

Check where your swap is:

```bash
sudo swapon --show
# or
sudo cat /proc/swaps
```

Example: `/dev/dm-2`


#### Method 1: Regularly Clear the Swap

```bash
sudo swapoff -a
sudo dd if=/dev/zero of=/path/to/swap bs=1M status=progress
sudo mkswap /path/to/swap
sudo swapon /path/to/swap
```


#### Method 2: Disable Swap Entirely

Useful if you have plenty of RAM.
Without swap, low-RAM systems may experience performance drops or crashes.

```bash
sudo swapoff -a
sudo nano /etc/fstab
sudo rm /path/to/swap
sudo swapon --show
```

---

#### Method 3: Use **zram** (compressed swap in RAM)

zram resides only in RAM and is cleared on reboot — a fast, safe alternative to swap.

```bash
sudo apt install zram-tools
sudo nano /etc/default/zramswap
sudo systemctl enable zramswap
sudo systemctl start zramswap
sudo swapon --show
```

Example output:

```
NAME       TYPE      SIZE  USED  PRIO
/dev/dm-2  partition 3.9G   0B    -2
/dev/zram0 partition 1.9G 256K   100
```

zram is used first due to higher priority.
If your RAM is large, you can disable the normal swap entirely.


#### Method 4: Encrypted Swap

This works like zram but with **encryption**.
A random key is generated on every boot, making post-reboot analysis impossible.
This is an advanced setup — incorrect configuration may break your boot.

Steps (example for LVM system):

```bash
sudo apt install cryptsetup
echo "cryptswap /dev/mapper/debian--vg-swap_1 /dev/urandom swap,cipher=aes-xts-plain64,size=256" | sudo tee -a /etc/crypttab
sudo nano /etc/fstab   # replace the old swap line with /dev/mapper/cryptswap
sudo swapoff -a
sudo cryptsetup open --type plain --cipher aes-xts-plain64 --key-size 256 --key-file /dev/urandom /dev/mapper/debian--vg-swap_1 cryptswap
sudo mkswap /dev/mapper/cryptswap
sudo swapon /dev/mapper/cryptswap
sudo update-initramfs -u -k all
sudo swapon --show
free -h
reboot
```

⚠️ **Warning:** This process is risky. Make sure to back up your data and follow the steps carefully.

**Thank you for reading this far!**
I hope this helped you understand the topic.
