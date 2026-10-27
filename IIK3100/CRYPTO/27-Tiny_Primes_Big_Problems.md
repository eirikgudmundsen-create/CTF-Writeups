# Tiny Primes, Big Problems

**Category:** Crypto / RSA
**Challenge:** *A developer copied the RSA key-generation slide but used primes that are far too small. The flag was encrypted with textbook RSA (no padding): split into 6-byte chunks (last chunk NUL-padded), each chunk `m` encrypted as `c = m^e mod n`. Recover the flag.*

## Recon

```python
n = 889970796180013      # 50 bits
e = 65537
ciphertext = [283355017161443, 300083924892126, 15528015238632,
              591455819941687, 491941932604380]
```

RSA's entire security rests on one assumption: that `n = p·q` is too large to factor. Here `n` is **50 bits** — `n ≈ 8.9 × 10¹⁴`, so `√n ≈ 3 × 10⁷`. Factoring it is a sub-second problem. Real RSA uses `n` of **2048 bits or more**, where the same approach would outlast the universe.

## The attack

Textbook RSA decryption is `m = c^d mod n`, where `d` is the private exponent. We don't have `d`, but we can derive it the moment we know `p` and `q` — which is exactly what factoring hands us.

### Step 1 — factor n

No library needed. Trial division up to `√n` finds the smaller prime, and the other is just `n // p`:

```python
def factor(n):
    i = 2
    while i * i <= n:
        if n % i == 0:
            return i, n // i
        i += 1
    return None

p, q = factor(n)
# p = 29594171, q = 30072503
assert p * q == n
```

The loop stops the instant `i` divides `n`, so it runs to `p` (~29.6M), not all the way to `√n`.

### Step 2 — rebuild the private key

With both primes known, the rest of key generation is reproducible:

```python
phi = (p - 1) * (q - 1)
d = pow(e, -1, phi)        # modular inverse of e mod phi
```

`d` is defined as `e⁻¹ mod phi`, i.e. `e·d ≡ 1 (mod phi)`. By Euler's theorem that makes `(m^e)^d ≡ m^(e·d) ≡ m (mod n)`, so `d` undoes the encryption. Recovering `phi` requires `p` and `q` — which is why factoring *is* the break.

### Step 3 — decrypt and reassemble

Each block decrypts to an integer; converting it back to 6 big-endian bytes is the inverse of how the encryptor read each chunk:

```python
flag = b""
for c in ciphertext:
    m = pow(c, d, n)
    flag += m.to_bytes(6, "big")
print(flag.rstrip(b"\x00").decode())
```

The raw output ends in padding — `b'NTNU{SMALL_PRIMES_FALL_FAST}\x00\x00'` — from the last chunk being NUL-filled to 6 bytes. `rstrip(b"\x00")` trims those trailing nulls.

![task3_rsa.py prints the recovered flag](images/01-flag.png)

## Flag

```
NTNU{SMALL_PRIMES_FALL_FAST}
```

## Takeaway

The math of RSA was implemented perfectly — the break is purely the **key size**. Factoring is believed hard only for large `n`; at 50 bits, trial division ends it instantly, and even a 512-bit modulus falls to tools like CADO-NFS. Modern RSA needs **≥ 2048-bit** `n`.

This challenge also quietly shows why **textbook RSA** is never used in practice, independent of key size:

- **No padding is deterministic.** The same plaintext always encrypts to the same ciphertext, so identical blocks leak, and short/structured messages are attackable. Real RSA uses OAEP padding, which randomizes each encryption.
- **Block-by-block encryption leaks structure.** Splitting the flag into independent chunks means each `c` is a separate textbook-RSA problem. Proper systems encrypt a random symmetric key with RSA-OAEP once, then encrypt the data with AES (hybrid encryption).

Two independent failures, then: a toy key size, and raw RSA with no padding. Either alone is fatal; here both appear on the same slide.
