# TryHackMe — Pickle Rick Writeup

A guided walkthrough of the TryHackMe *Pickle Rick* room, covering web recon, hidden-file discovery, gaining shell access through a command panel, and escalating to root to collect all three ingredients.

| Item | Details |
|---|---|
| Platform | TryHackMe |
| Room | Pickle Rick |
| Difficulty | Easy |
| Target | `<MACHINE_IP>` |
| Room URL | https://tryhackme.com/room/picklerick |

---

## 1. Initial Scan

I kicked things off with a standard Nmap scan to map out which TCP services were exposed on the box:

```bash
nmap <MACHINE_IP>
```

![nmap scan results](https://github.com/user-attachments/assets/77c894c8-b238-4c46-81e1-d3fdd49610dd)

Two ports came back open: **22/tcp (SSH)** and **80/tcp (HTTP)**. Since a web server was running, that became the obvious next stop.

---

## 2. Web Recon

### 2.1 Checking the Page Source

Loading the site and digging through its HTML turned up something useful — a leftover developer comment containing a username.

![HTML source comment](https://github.com/user-attachments/assets/996f33a8-9272-4251-8e9c-a22123fb2d32)

**Username found:** `R1ckRul3s`

### 2.2 Brute-Forcing Hidden Paths

Next, I ran Gobuster against the site looking for pages not linked anywhere on the homepage, targeting PHP, HTML, and TXT files:

```bash
gobuster dir -u http://<MACHINE_IP> -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
```

![gobuster results](https://github.com/user-attachments/assets/a1cc8d10-777a-4ce5-9fc8-153a3523cba2)

This surfaced `login.php`, `robots.txt`, and `portal.php`. Trying to hit `portal.php` directly bounced me (302) back to `login.php`, meaning that page sits behind authentication.

### 2.3 A Look at robots.txt

Rather than the usual crawler rules, `robots.txt` held a lone string sitting by itself. Given the username already in hand, this looked like a likely password.

![robots.txt contents](https://github.com/user-attachments/assets/a5886990-18d0-48c6-9e54-c9bffcced07b)

**Likely password:** `Wubbalubbadubdub`

---

## 3. Logging In

Using the username and password gathered above on `login.php` worked immediately, dropping me into the command portal at `portal.php`.

**Working credentials:** `R1ckRul3s` / `Wubbalubbadubdub`

![Authenticated portal](https://github.com/user-attachments/assets/3cc22bf3-fc21-46e4-8f1b-94be988315f7)

---

## 4. Getting Command Execution — First Ingredient

The portal turned out to be a simple front-end for running shell commands. First step, look around the current directory:

```bash
ls
```

![ls output](https://github.com/user-attachments/assets/a32abf46-4690-4744-a04c-a484c4bcceee)

Sitting right there was `Sup3rS3cretPickl3Ingred.txt`. Reading it with `less` gave up the first ingredient:

![First ingredient file](https://github.com/user-attachments/assets/58f304fd-cc78-4b4d-9b16-28fb61d7a59c)

**Ingredient #1:** `mr. meeseek hair`

---

## 5. Tracking Down the Second Ingredient

With the first one in the bag, I moved on to exploring the wider filesystem, starting at root and working down into `/home`:

```bash
ls /
```

![root filesystem listing](https://github.com/user-attachments/assets/4dffc37d-2268-44ee-888d-55202ddf1a23)

```bash
ls /home
```

![home directory listing](https://github.com/user-attachments/assets/03421824-ae6e-4600-948c-474e5733fa97)

```bash
ls /home/rick
```

![rick's home directory](https://github.com/user-attachments/assets/1623a7bc-bedd-4d4b-a832-831c55272c84)

One of the files there had a space in its name, so I wrapped it in quotes to keep the shell from splitting it into separate arguments:

```bash
less /home/rick/"second ingredients"
```

![Second ingredient file](https://github.com/user-attachments/assets/abfa5e89-0756-4fac-90db-cabd22af4de3)

**Ingredient #2:** `1 jerry tear`

---

## 6. Checking Privileges & Grabbing the Final Ingredient

Before poking around `/root`, it was worth confirming which user context the web shell was actually running under:

```bash
whoami
```

![whoami output](https://github.com/user-attachments/assets/cca825ce-2f7c-4d73-9d57-5c5af8fb8431)

Result: `www-data` — the standard low-privilege web server account.

To see whether that account had any elevated rights, I tried a simple sudo command against root's home folder:

```bash
sudo ls /root
```

![sudo ls /root output](https://github.com/user-attachments/assets/ffe3083a-89ef-4c50-9eec-e87f5f0ccbcf)

It ran instantly with no password prompt — `www-data` had **passwordless sudo access**. *(I didn't run a full `sudo -l`, so the complete list of allowed commands wasn't confirmed, but this one command was enough to reach root's files.)*

Inside `/root` was `3rd.txt`, which I opened using the same elevated access:

```bash
sudo less /root/3rd.txt
```

![Final ingredient file](https://github.com/user-attachments/assets/b208b6de-b395-413e-9b3b-c45bf04ee254)

**Final ingredient:** `fleeb juice`

---

## 7. Wrap-Up: All Three Ingredients

Here's a summary of where each ingredient was found and what it was:

| # | Found In | Ingredient |
|---|---|---|
| 1st | `Sup3rS3cretPickl3Ingred.txt` (web root) | `mr. meeseek hair` |
| 2nd | `/home/rick/second ingredients` | `1 jerry tear` |
| 3rd (final) | `/root/3rd.txt` | `fleeb juice` |

---

## Lessons Learned

- Never underestimate a quick look at page source — leftover comments are a common credential leak.
- `robots.txt` isn't just for search engines; check it as part of standard recon.
- Directory brute-forcing tools like Gobuster reliably surface pages that aren't linked in the UI.
- Passwordless `sudo` for a service account is a serious misconfiguration — always run `sudo -l` to see the full picture of what's allowed.
