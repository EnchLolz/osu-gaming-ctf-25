# ssss

Category: Crypto

Points: 122

Solves: 128

Difficulty: 2/5

>can you ss this secret sharing scheme?


## Solution

We are given this script:

```py
#!/usr/local/bin/python3
from Crypto.Util.number import *
import random

p = 2**255 - 19
k = 15
SECRET = random.randrange(0, p)

def lcg(x, a, b, p):
    return (a * x + b) % p

a = random.randrange(0, p)
b = random.randrange(0, p)
poly = [SECRET]
while len(poly) != k: poly.append(lcg(poly[-1], a, b, p))

def evaluate_poly(f, x):
    return sum(c * pow(x, i, p) for i, c in enumerate(f)) % p

print("welcome to ssss", flush=True)
for _ in range(k - 1):
    x = int(input())
    assert 0 < x < p, "no cheating!"
    print(evaluate_poly(poly, x), flush=True)

if int(input("secret? ")) == SECRET:
    FLAG = open("flag.txt").read()
    print(FLAG, flush=True)
```

It looks the code generates a degree `14` polynomials where each coefficient are generated from an LCG, and the `0th` degree term is the basecase/secret that we are trying to guess. To guess the secret, we are given 14 queries of the polynomial.

