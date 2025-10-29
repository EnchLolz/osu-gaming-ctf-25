# linear-feedback

Category: Crypto

Points: 119

Solves: 142

Difficulty: 2/5

>this owc map is so fire btw :steamhappy: https://osu.ppy.sh/beatmapsets/2451798#osu/5355997


## Solution

We are given this script:

```py
from secrets import randbits
from math import floor
from hashlib import sha256

class LFSR:
    def __init__(self, key, taps, format):
        self.key = key
        self.taps = taps
        self.state = list(map(int, list(format.format(key))))
    
    def _clock(self):
        ob = self.state[0]
        self.state = self.state[1:] + [sum([self.state[t] for t in self.taps]) % 2]
        return ob

def xnor_gate(a, b):
    if a == 0 and b == 0:
        return 1
    elif a == 0 and b == 1:
        return 0
    elif a == 1 and b == 0:
        return 0
    else:
        return 1

key1 = randbits(21)
key2 = randbits(29)
L1 = LFSR(key1, [2, 4, 5, 1, 7, 9, 8], "{:021b}")
L2 = LFSR(key2, [5, 3, 5, 5, 9, 9, 7], "{:029b}")

bits = [xnor_gate(L1._clock(), L2._clock()) for _ in range(floor(72.7))]
print(bits)

FLAG = open("flag.txt", "rb").read()
keystream = sha256((str(key1) + str(key2)).encode()).digest() * 2
print(bytes([b1 ^ b2 for b1, b2 in zip(FLAG, keystream)]).hex())
```

Analysing this code, we can first see how the `LSFR` clock is generated. It takes in the initial randbits `key`, and an array of indices `taps`. The clock's state is initially set to the key. The clock then keep generating new bits by appending the xor of the bits at indices in `taps` in the current state, the drops and returns the first bit in the state. Importantly, since we know the taps, if we can recover the original key, we recover the entire clock sequence for however long we need.

Next, we do another XNOR cipher between `key1` clock and `key2` clock and get the output:
```py
[0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 0, 0, 0, 1, 1, 0, 0, 0, 0, 1, 1, 1, 0, 1, 1, 1, 0, 1, 1, 0, 1, 1, 0, 0, 1, 0, 1, 0, 1, 0, 1, 1, 1, 1, 0, 1, 0, 1, 0, 0, 1, 1, 1, 0, 1, 1, 0, 0, 0, 0, 0, 1, 1, 0, 0, 0, 0, 1, 0, 1, 0, 0]
```

Since XNOR is reversible just like XOR, if we have one key we can recover the other. Then once we recover `key1` and `key2` we can compute the sha256 keystream, and decode the flag.

To do this, we can realize that `key1` is relatively small, with only 21 bits, we can just bruteforce over all 2 million `key1` possibilities. Since the rest of the bits are determined determinently, we can recover a possible `key2` from reversing the XNOR with the output bits. We then double check that the guessed `key1` and `key2` actually satisfy the XNOR.

```py
class LFSR:
    def __init__(self, key, taps, format):
        self.key = key
        self.taps = taps
        self.state = list(map(int, list(format.format(key))))
    
    def _clock(self):
        ob = self.state[0]
        self.state = self.state[1:] + [sum([self.state[t] for t in self.taps]) % 2]
        return ob

def xnor_gate(a, b):
    if a == 0 and b == 0:
        return 1
    elif a == 0 and b == 1:
        return 0
    elif a == 1 and b == 0:
        return 0
    else:
        return 1


outbits = [0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 0, 0, 0, 1, 1, 0, 0, 0, 0, 1, 1, 1, 0, 1, 1, 1, 0, 1, 1, 0, 1, 1, 0, 0, 1, 0, 1, 0, 1, 0, 1, 1, 1, 1, 0, 1, 0, 1, 0, 0, 1, 1, 1, 0, 1, 1, 0, 0, 0, 0, 0, 1, 1, 0, 0, 0, 0, 1, 0, 1, 0, 0]

from tqdm import tqdm
# loop through all possible key1's
for guess in tqdm(range(2**21)):
    L1 = LFSR(guess, [2, 4, 5, 1, 7, 9, 8], "{:021b}")
    recover_key2 = []
    # reverse xnor of key1 and bits to get key2
    for i in range(29):
        l1_bit = L1._clock()
        recover_key2.append(xnor_gate(l1_bit, outbits[i]))

    recover_key2_int = int("".join(map(str, recover_key2)), 2)

    # check that key1 and key2 actually produce the outbits
    L1 = LFSR(guess, [2, 4, 5, 1, 7, 9, 8], "{:021b}")
    L2 = LFSR(recover_key2_int, [5, 3, 5, 5, 9, 9, 7], "{:029b}")
    test_outbits = [xnor_gate(L1._clock(), L2._clock()) for _ in range(72)]

    if test_outbits == outbits:
        print("Found keys!")
        print("Key1:", guess)
        print("Key2:", recover_key2_int)
```

After running this script we get the candidates:

```py
Found keys!
Key1: 272504
Key2: 464471258

Found keys!
Key1: 776071
Key2: 340835109

Found keys!
Key1: 1321080
Key2: 196035802

Found keys!
Key1: 1824647
Key2: 72399653
```

After testing through these keys, we get that the 2nd one is the solution.

```py
>>> key1 = 776071
>>> key2 = 340835109                                                                                             >>> flagenc = bytes.fromhex("9f7f799ec2fb64e743d8ed06ca6be98e24724c9ca48e21013c8baefe83b5a304af3f7ad6c4cc64fa4380e854e8")
>>> keystream = sha256((str(key1) + str(key2)).encode()).digest() * 2
>>> print(bytes([b1 ^ b2 for b1, b2 in zip(flagenc, keystream)]))
b'osu{th1s_hr1_i5_th3_m0st_fun_m4p_3v3r_1n_0wc}'
```
