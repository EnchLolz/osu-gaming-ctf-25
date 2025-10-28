# xnor-xnor-xnor

Category: Crypto

Points: 107

Solves: 348

Difficulty: 1/5

>https://osu.ppy.sh/beatmapsets/1236927#osu/2573164


## Solution

We are given this script

```py
import os
flag = open("flag.txt", "rb").read()

def xnor_gate(a, b):
    if a == 0 and b == 0:
        return 1
    elif a == 0 and b == 1:
        return 0
    elif a == 1 and b == 0:
        return 0
    else:
        return 1

def str_to_bits(s):
    bits = []
    for x in s:
        bits += [(x >> i) & 1 for i in range(8)][::-1]
    return bits

def bits_to_str(bits):
    return bytes([sum(x * 2 ** j for j, x in enumerate(bits[i:i+8][::-1])) for i in range(0, len(bits), 8)])

def xnor(pt_bits, key_bits):
    print(pt_bits, key_bits)
    return [xnor_gate(pt_bit, key_bit) for pt_bit, key_bit in zip(pt_bits, key_bits)]

key = os.urandom(4) * (1 + len(flag) // 4)
key_bits = str_to_bits(key)
flag_bits = str_to_bits(flag)
enc_flag = xnor(xnor(xnor(flag_bits, key_bits), key_bits), key_bits)

print(bits_to_str(enc_flag).hex())
# 7e5fa0f2731fb9b9671fb1d62254b6e5645fe4ff2273b8f04e4ee6e5215ae6ed6c
```

Essentially, the script generates a key the length of the flag, then encrypts it the same was as an xor cipher, just in this case it uses xnor and does it 3 times. Conceptually, it's the same as xor, as xnor is reversible in the same way as xor.

Further lets see how the key is constructed:

```py
key = os.urandom(4) * (1 + len(flag) // 4)
```

Note that the key is actually just 4 random bytes then duplicated till the end of the flag. Further we know that the first 4 bytes of the flag are `osu{`. We can thus then reverse the key.


```py
>>> def xnor(byte1,byte2):
...     return ~(byte1^byte2) & 0xFF
... 
>>> flagenc = bytes.fromhex("7e5fa0f2731fb9b9671fb1d62254b6e5645fe4ff2273b8f04e4ee6e5215ae6ed6c")
>>> key = bytearray(xnor(flagenc[i],b"osu{"[i]) for i in range(4))
>>> ''.join(chr(xnor(flagenc[i],key[i%4])) for i in range(len(flagenc)))
'osu{b3l0v3d_3xclus1v3_my_b3l0v3d}'
```