# Crack the Hash — Write-up

A walkthrough on identifying and cracking different hash types, starting from basic lookups and moving to Hashcat with rockyou.txt.

Didn't know how to even start at first. Looked it up and the first step is always: figure out what kind of hash you're dealing with.

## Tools Used

- nmap: initial recon
- hashid: hash type identification
- **CrackStation**: online hash lookup
- **John the Ripper**: password cracking
- **Hashcat**: GPU-accelerated password cracking

My attack box had a network connection problem so I couldn't install hashid. Used CrackStation instead to identify the hashes.

## Level 1: Identifying the Hashes

| # | Hash Type | Password | Notes |
| --- | --- | --- | --- |
| 1 | MD5 | *see screenshot* | Rated "easy" on CrackStation |
| 2 | SHA-1 | `password123` |  |
| 3 | SHA-256 | `letmein` |  |
| 4 | bcrypt | `bleh` | Not solvable via CrackStation, needed Hashcat (below) |
| 5 | MD4 | `Eternity22` |  |

!Screenshot 2026-10-20 at 21.17.53.png

!Screenshot 2026-10-20 at 21.20.07.png

!Screenshot 2026-10-20 at 21.23.12.png

### Hash Algorithms Reference

| Algorithm | Year | Output | Status |
| --- | --- | --- | --- |
| MD4 | 1990 | 128-bit | Completely broken, collisions in milliseconds |
| MD5 | 1991 | 128-bit | Cryptographically broken, collisions possible |
| SHA-1 | 1995 | 160-bit | Deprecated, practical collision demoed in 2017 (SHAttered) |
| SHA-256 | 2001 | 256-bit | Secure, industry standard |
| bcrypt | 1999 | variable | Secure, intentionally slow, resists brute-force |

### Cracking Hash #4 (bcrypt)

CrackStation couldn't resolve it because the hash starts with $2y$12$, a **bcrypt** hash, not MD5:

- $2y$ → PHP's bcrypt variant
- 12 → cost factor (computational difficulty)
- remainder → salt + hash

Bcrypt is deliberately slow, so it needs a dedicated tool: **Hashcat**.

```
echo '$2y$12$Dwt1BZj6pcyc3Dy1FWZ5ieeUznr71EeNkJkUlypTsgbX1H68wsRom' > hash.txt
hashcat -m 3200 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

Running the full rockyou.txt against bcrypt was estimated at 29 days. Too slow, not a real option.

Saw online that since the password was only 4 characters long, I should filter the wordlist down first:

```
grep -oE '\b[a-z]{4}\b' /usr/share/wordlists/rockyou.txt > rockyou_4.txt
```

After very little time it found it:

```
hashcat -m 3200 hash.txt --show
$2y$12$Dwt1BZj6pcyc3Dy1FWZ5ieeUznr71EeNkJkUlypTsgbX1H68wsRom:bleh
```

Finally got Kali running through WSL to do this part.

## Level 2: Hashcat Challenges

Still need to solve the Hashcat ones.

| Hash | Algorithm | Mode | Result |
| --- | --- | --- | --- |
| `1DFECA0C002AE40B8619ECF94819CC1B` | NTLM |  | `n63umy8lkf4i` |
| `F09EDCB1FCEFC6DFB23DC3505A882655FF77375ED8AA2D1C13F640FCCC2D0C85` | SHA-256 |  | `paule` |
| `$6$aReallyHardSalt$6WKUTqzq.UQQmrm0p/T7MPpMbGNnzXPMAXi4bJMl9be.cfi3/qxIf.hsGpS41BqMhSrHVXgMpdjS6xeKZAs02` | SHA-512 (salted) | `-m 1800` | *command run, result not recorded in notes* |
| `e5d8870e5bdd26602cab8dbe07a942c8669e56d6` | HMAC-SHA1 | `-m 160` | `tryhackme` |

To solve the last two of Level 2, I just needed to check the type of each one.

### SHA-512 (salted)

```
echo '$6$aReallyHardSalt$6WKUTqzq.UQQmrm0p/T7MPpMbGNnzXPMAXi4bJMl9be.cfi3/qxIf.hsGpS41BqMhSrHVXgMpdjS6xeKZAs02' > hash_sha512.txt
hashcat -m 1800 -a 0 hash_sha512.txt /usr/share/wordlists/rockyou.txt
```

SHA-512 is still fast enough on CPU that a dictionary attack is viable, even though it's a more secure algorithm than MD5/SHA-1.

### HMAC-SHA1 (salted)

The last hash had a salt in it, had to look up what that meant first.

A **salt** is random data mixed into the hashing process so identical passwords don't produce identical hashes. It's what defeats precomputed lookup services like CrackStation.

```
echo 'e5d8870e5bdd26602cab8dbe07a942c8669e56d6:tryhackme' > hash_hmac.txt
hashcat -m 160 -a 0 hash_hmac.txt /usr/share/wordlists/rockyou.txt
```

```
hashcat -m 160 hash_hmac.txt --show
e5d8870e5bdd26602cab8dbe07a942c8669e56d6:tryhackme:481616481616
```

## About rockyou.txt

Already had rockyou.txt locally, so could just point Hashcat straight at it.

The default go-to wordlist for password cracking. Leaked from the 2009 RockYou breach, containing 14M+ real passwords (123456, password, pet names, dates...). Commonly pre-installed at /usr/share/wordlists/rockyou.txt on Kali.

## Key Takeaways

- Always identify the hash type first. The tool and approach depend entirely on it.
- Fast hashes (MD5, SHA-1, NTLM) are quick to brute-force; bcrypt and salted hashes are deliberately slow and need targeted wordlists.
- Narrowing a wordlist by length/charset can turn an unfeasible attack (29 days) into a fast one.
- Salts defeat hash-lookup services. Use purpose-built tools like Hashcat instead.