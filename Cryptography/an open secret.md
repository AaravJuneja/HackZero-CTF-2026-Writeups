# An Open Secret

## Description

Two systems met somewhere on the network. They exchanged numbers, performed their rituals, and left, confident that no one watching could ever reconstruct what they agreed upon. Unfortunately for them, we were watching. A packet capture of their conversation has been recovered. It’s messy. There’s noise, chatter, and a lot of confidence in how “secure” everything is buried inside, however, is everything you need. The math looks solid. The values look large. Nothing obviously broken. But security isn’t just about choosing the right algorithm it’s about how you use it. If you think this can be solved with a clever shortcut, think again. You might need to go further than you initially expect. Recover the shared secret and use it to unlock what they tried to hide.

## Writeup

* The PCAP mostly has random junk traffic, but one thing to notice is the UDP noise there was a [plaintext debug packet](../images/wireshark.png) leaking the full [Diffie Hellman (DH)](https://cryptohack.gitbook.io/cryptobook/diffie-hellman) key exchange which is an algorithm that lets two parties agree on a shared secret using public values over an insecure channel, relying on the [Discrete Log Problem (DLP)](https://cryptohack.gitbook.io/cryptobook/abstract-algebra/groups/untitled) being hard.

```python
p = 8736489073805684086305179507451312145336960198850222450321684285045258466628525808781552246900007804506276394992600728538533608361706639189532331964901531 #large prime modulus
g = 2 #generator
A = 2794996066442934640383396708993220584783762169186726070188366297468316333265452428474082701535690342706174889597372484008702943319781202470240884210613693 #alice's secret (g^a mod p)
B = 1210454350578824211787039637709213524440747952483190753841197066710749405257527570548081311766060600166395746681022324544415844366753788799012419699450666 #bob's secret (g^b mod p)
```

* Normally after googling a bit, I found out that recovering a or b from `A = g^a mod p` is very infeasible because of the DLP but, the vulnerability here isn’t the DH it’s the parameters which are chosen which is the 1st vulnerability here, after checking `p - 1` on [FactorDB](https://factordb.com/) I got to know that it has a "smooth" component (built up of small prime factors):

```python
p - 1 = 2 * 3^3 * 5 * 13 * 2927
```

* This is very nice because it enables the use of the [Pohlig–Hellman](https://medium.com/asecuritysite-when-bob-met-alice/the-pohlig-hellman-attack-3e9d37e3f4dc) algorithm, which reduces the DLP into smaller subproblems modulo each factor of p - 1 using an implementation of that from github I got,

```python
a mod 10273770
b mod 10273770
```

* Now this can be then recombined using the CRT to get `a0,b0`, but at this point I didn't have all the values but I knew that for some value of `k` this equality will hold,

```python
a = a0 + k * 10273770
b = b0 + k * 10273770
```

* Now comes the 2nd vulnerability, the private exponents are too small(~32 bits), which allows me to brute force `k` easily and get `a,b`:

```python
a = 225118447
b = 234797908
```

* The challenge is practically solved now because once I have either of the exponent(s), I can compute the shared secret `B^a mod p = A^b mod p` and finally using the sha256 hash of the GPG passphrase decrypts the file and we get the flag!

```python
passphrase = sha256(shared_secret_as_bytes) = 5edba444d2ecc490bfbbaa37ce1ce7daa1844e4e881bc05cd1cce0f66e91c181
```

```bash
$ gpg -d --batch --passphrase '5edba444d2ecc490bfbbaa37ce1ce7daa1844e4e881bc05cd1cce0f66e91c181' flag.txt.gpg
gpg: AES256.CFB encrypted data
gpg: encrypted with 1 passphrase
hackzero{br41nfuck_is_not_r34l!}
```

## Solver

```python
import hashlib
from sympy.ntheory import discrete_log
from sympy.ntheory.modular import crt

p = 8736489073805684086305179507451312145336960198850222450321684285045258466628525808781552246900007804506276394992600728538533608361706639189532331964901531; g = 2; A = 2794996066442934640383396708993220584783762169186726070188366297468316333265452428474082701535690342706174889597372484008702943319781202470240884210613693; B = 1210454350578824211787039637709213524440747952483190753841197066710749405257527570548081311766060600166395746681022324544415844366753788799012419699450666

mods = [2, 27, 5, 13, 2927] #small factors of (p - 1)

M = 1
for q in mods:
    M *= q

a_res = []
b_res = []

#solving DL modulo each small factor
for q in mods:
    gg = pow(g, (p - 1) // q, p)  #reduced generator
    a_res.append(discrete_log(p, pow(A, (p - 1) // q, p), gg, order=q))
    b_res.append(discrete_log(p, pow(B, (p - 1) // q, p), gg, order=q))

a0 = int(crt(mods, a_res)[0])
b0 = int(crt(mods, b_res)[0])

#brute-force: a = a0 + k*M
a = a0
cur = pow(g, a, p) #current g^a
step = pow(g, M, p) #jump by M each time

while a < 2**32:
    if cur == A:
        break
    cur = (cur * step) % p
    a += M

"""
b = b0
cur = pow(g, b, p)

while b < 2**32:
    if cur == B:
        break
    cur = (cur * step) % p
    b += M
"""

shared = pow(B, a, p)

from Crypto.Util.number import long_to_bytes
passphrase = hashlib.sha256(long_to_bytes(shared)).hexdigest()

print("a: ", a)
#print("b, ", b)
print("pwd: ", passphrase)
```

## Flag

`hackzero{br41nfuck_is_not_r34l!}`
