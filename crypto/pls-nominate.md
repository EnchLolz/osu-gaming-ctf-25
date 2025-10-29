# pls-nominate

Category: Crypto

Points: 111

Solves: 224

Difficulty: 2/5

>pls help me get the attention of bns by spamming this message to them pls pls pls


## Solution

We are given this script:

```py
from Crypto.Util.number import *

FLAG = open("flag.txt", "rb").read()
message = bytes_to_long(
    b"hello there can you pls nominate my map https://osu.ppy.sh/beatmapsets/2436259 :steamhappy: i can bribe you with a flag if you do: " + FLAG
)

ns = [getPrime(727) * getPrime(727) for _ in range(5)]
e = 5
print(len(FLAG))
print(ns)
print([pow(message, e, n) for n in ns])
```

Since we are given 5 different encrypted versions of the flag and a very small e value, we can use a very classic copper smith attack, specifically the [Håstad's broadcast attack](https://en.wikipedia.org/wiki/Coppersmith%27s_attack#H%C3%A5stad's_broadcast_attack).

```py
# CRT implementation
from typing import List, Tuple, Optional
def egcd(a: int, b: int) -> Tuple[int,int,int]:
    if b == 0:
        return (a, 1, 0)
    g, x1, y1 = egcd(b, a % b)
    return (g, y1, x1 - (a // b) * y1)

def merge_congruence(r1: int, m1: int, r2: int, m2: int) -> Optional[Tuple[int,int]]:
    r1 %= m1
    r2 %= m2
    g, s, t = egcd(m1, m2)
    if (r2 - r1) % g != 0:
        return None  
    m2_g = m2 // g
    k = ((r2 - r1) // g) * s
    k %= m2_g
    r = (r1 + m1 * k) % (m1 * m2_g)
    m = m1 * m2_g
    return (r, m)

def crt(remainders: List[int], moduli: List[int]) -> Optional[Tuple[int,int]]:
    if len(remainders) != len(moduli):
        raise ValueError("remainders and moduli must be same length")
    if not remainders:
        return (0, 1)  
    r, m = remainders[0] % moduli[0], moduli[0]
    for ri, mi in zip(remainders[1:], moduli[1:]):
        merged = merge_congruence(r, m, ri, mi)
        if merged is None:
            return None
        r, m = merged
    return (r, m)

# Given n and c values in output.txt
ns = [2505953962820 ...]
cs = [1234517249314 ...]

rem,mod = crt(cs, ns)
res = rem

import sympy
from Crypto.Util.number import long_to_bytes

# Loop until we reach a perfect power of 5
while not sympy.integer_nthroot(res, 5)[1]:
    res += mod

print(long_to_bytes(sympy.integer_nthroot(res, 5)[0]))
```

Thus we get:
```py
b'hello there can you pls nominate my map https://osu.ppy.sh/beatmapsets/2436259 :steamhappy: i can bribe you with a flag if you do: osu{pr3tty_pl3453_w1th_4_ch3rry_0n_t0p!?:pleading:}'
```