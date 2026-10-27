# Rock You Like a Hash Cat

**Category:** Crypto / Password Cracking
**Challenge:** *An attacker dumped four password hashes from a database. Recover all four passwords and pass them (in id order) to `unlock()` to decrypt the flag. The flag key is derived from the passwords, so they genuinely have to be cracked.*

## Recon

Four hashes, three algorithms, one salt:

| id | algo | salt | note |
|---|---|---|---|
| 1 | md5 | — | unsalted |
| 2 | sha1 | — | unsalted |
| 3 | sha256 | — | unsalted |
| 4 | sha256 | `ntnu_salt_2025$` | salt **prepended**: `sha256(salt + password)` |

The flag itself is XOR-encrypted with `sha256("|".join(passwords))`, so there's no shortcut — every password is needed, in the right order.

The provided `task2_hashes.py` already gives `hash_password(algo, password, salt)` and `unlock()`. The only missing piece is the cracking loop at the `# TODO`.

## Why this works at all

Hashes are **one-way** — there is no `unhash()`. "Cracking" is guess-and-check: take a candidate password, run it *forward* through the same hash the database used, and compare. If the digests match, the guess is the password, because the same input always yields the same digest and collisions are infeasible.

The consequence: you can only ever recover a password that's already in your candidate list. The hints spell out how real users pick passwords, so the candidate list is `wordlist.txt` **plus the usual mutations**:

- **Capitalise** — `dragon` → `Dragon`, `DRAGON`
- **Leetspeak** — `a→@`, `o→0`, `e→3`, `i→1`, `s→$`, `t→7`
- **Appended digits / years** — `+123`, `+2025`, `+!`

## The mutation engine

The one non-obvious part is leetspeak. Replacing *every* letter at once (`dragon` → `dr@g0n` only) misses passwords that swap just some letters. The fix is to treat each character as a choice between *itself and its leet form*, then take the cartesian product:

```python
import itertools
LEET = {'a':'@','o':'0','e':'3','i':'1','s':'$','t':'7'}

def leets(word):
    # each char becomes a set: {itself} plus its leet form if one exists
    opts = [{c} | ({LEET[c.lower()]} if c.lower() in LEET else set()) for c in word]
    for combo in itertools.product(*opts):
        yield "".join(combo)

SUFFIXES = ["", "1", "123", "12", "2025", "2024", "2023", "!"]

def mutations(word):
    for base in {word, word.capitalize(), word.upper(), word.lower()}:
        for lv in leets(base):
            for suffix in SUFFIXES:
                yield lv + suffix
```

`itertools.product` over the per-character option sets yields every on/off combination: `dragon`, `dr@gon`, `drag0n`, `dr@g0n`, and so on.

## The crack loop

Dropped in where the TODO was. Salt needs no special handling — `hash_password` already prepends `t["salt"]`, so target 4 just works:

```python
if __name__ == "__main__":
    found = {}
    words = [line.strip() for line in open("wordlist.txt")]
    for word in words:
        for candidate in mutations(word):
            for t in TARGETS:
                if hash_password(t["algo"], candidate, t["salt"]) == t["hash"]:
                    found[t["id"]] = candidate
    unlock([found.get(t["id"], "") for t in TARGETS])
```

![task2_hashes.py prints the decrypted flag](images/01-flag.png)

### Cracked passwords

Each target deliberately demonstrates a different mutation rule:

| id | algo | password | rule shown |
|---|---|---|---|
| 1 | md5 | `letmein` | raw wordlist hit |
| 2 | sha1 | `Monkey123` | capitalise + digits |
| 3 | sha256 | `Dr@g0n` | leetspeak |
| 4 | sha256 (salted) | `sunshine2025` | year suffix |

## Flag

```
NTNU{WEAK_PASSWORDS_FALL_TO_A_WORDLIST}
```

## Takeaway

None of these passwords is "strong" — every one is a dictionary word with a predictable mutation, which is exactly what rule-based cracking (hashcat's `-r` rules, John's mangling) is built to chew through. The wordlist shipped with the challenge, but `rockyou.txt` (14M real leaked passwords) would catch the same four instantly.

Two defensive lessons sit inside the challenge:

- **Fast hashes are the wrong tool for passwords.** MD5/SHA1/SHA256 are designed to be *fast*, which helps the attacker try billions of candidates per second. Password storage should use a slow, memory-hard KDF — bcrypt, scrypt, or Argon2.
- **Salt defeats precomputation, not weak passwords.** The salt on target 4 means an attacker can't use a precomputed rainbow table and can't crack all users at once with a shared table — each hash must be attacked individually. But `sunshine2025` still fell in milliseconds, because salting does nothing for a guessable password. Salt + slow KDF + actual password strength are three separate controls, and you need all three.
