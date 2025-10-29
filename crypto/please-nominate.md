# please-nominate

Category: Crypto

Points: 142

Solves: 72

Difficulty: 3/5

>ok this time i'm going to be a bit more nice and personal when sending my message


## Solution

We are given this script:

```py
from Crypto.Util.number import *

FLAG = open("flag.txt").read()

BNS = ["Plus4j", "Mattay", "Dailycore"]
print(len(FLAG))

for BN in BNS:
    print("message for", BN)
    message = bytes_to_long(
        (f"hi there {BN}, " + FLAG).encode()
    )
    n = getPrime(727) * getPrime(727)
    e = 3
    print(n)
    print(pow(message, e, n))
```

Unlike `pls-nominate` it looks like we are only given 3 secret messages, and our exponent `e` has been reduced to `3`. However, unlike in `pls-nominate` the message string isn't the same across the 3 encryptions as we replace the `"hi there {name}, "` with a different name each time (Plus4j, Mattay, Dailycore).

So in this case, we can no longer use the Håstad's broadcast attack, instead we will use the [Franklin–Reiter related-message attack](https://en.wikipedia.org/wiki/Coppersmith%27s_attack#Franklin%E2%80%93Reiter_related-message_attack). Essentially this attack is possible, since the messages are basically the same except for the initial part. However since we know how the messages are related by just changing the names, we know the difference between the messages.

Essentialy, instead of treating each messages as completely independent as $M_1, M_2, M_3$ we can treat them as:

$$M_1 = A_1+M$$
$$M_2 = A_2+M$$
$$M_3 = A_3+M$$

In this case $M$ is the `FLAG`, while $A_1,A_2,A_3$ are the different prefixes that we add in front of `FLAG`.

More specifically 

$$A_1 = \text{``hi there Plus5j, " + ``$\backslash$x00"} \times \text{len(FLAG)}$$
$$A_1 = \text{``hi there Mattay, " + ``$\backslash$x00"} \times \text{len(FLAG)}$$
$$A_1 = \text{``hi there Dailycore, " + ``$\backslash$x00"} \times \text{len(FLAG)}$$

Then we have the 3 equations:

$$ C_1 \equiv M_1^e \equiv (A_1+M)^e \equiv(A_1+M)^3 \pmod{N_1} $$
$$ C_2 \equiv M_2^e \equiv (A_2+M)^e \equiv(A_2+M)^3 \pmod{N_2} $$
$$ C_3 \equiv M_3^e \equiv (A_3+M)^e \equiv(A_3+M)^3 \pmod{N_3} $$

Expanding out the polynomials we get that:

$$M^3+3 \cdot A_i \cdot M^2 + 3 \cdot A_i^2 \cdot M + A_i - C_i \equiv 0 \pmod{N_i}$$


Since we have 3 of these equations, and we are solving for $M$, we can CRT on the coefficients for the 3 equations to find a polynomial mod $N_1N_2N_3$ with the root being the flag. Since we are now working with some large modulus compared to $M$, we can get sage to solve it for us.

We then can create the following sage solution:


```py
# run sage -python
from sage.all import Integer, ZZ, PolynomialRing, crt, floor, IntegerModRing

# Replace these with your values 
n1 = Integer(...)
n2 = Integer(...)
n3 = Integer(...)
c1 = Integer(...)
c2 = Integer(...)
c3 = Integer(...)

flag_len = 147 

# Prefix values
pref1 = b"hi there Plus4j, "+ b"\x00" * flag_len
pref2 = b"hi there Mattay, "+ b"\x00" * flag_len
pref3 = b"hi there Dailycore, "+ b"\x00" * flag_len

# compute A_i as integers
A1 = Integer(int.from_bytes(pref1, 'big'))
A2 = Integer(int.from_bytes(pref2, 'big'))
A3 = Integer(int.from_bytes(pref3, 'big'))


# Build the coefficient lists for each f_i(x) = x^3 + 3 A_i x^2 + 3 A_i^2 x + (A_i^3 - c_i)
# We will CRT combine coefficients so they are correct modulo N
n_list = [n1, n2, n3]
N = n1 * n2 * n3
print("N bits:", N.nbits())

# collect coefficient residues modulo each n_i
# coefficients ordered as [coef_x3, coef_x2, coef_x1, coef_x0]
coeffs_mods = []
for (A, c, n) in [(A1, c1, n1), (A2, c2, n2), (A3, c3, n3)]:
    # f = x^3 + 3A x^2 + 3A^2 x + (A^3 - c)
    coef3 = Integer(1) % n
    coef2 = (3 * A) % n
    coef1 = (3 * (A**2)) % n
    coef0 = (A**3 - c) % n
    coeffs_mods.append([coef3, coef2, coef1, coef0])

# Perform CRT on each coefficient across moduli to get coefficients modulo N
C = []
for j in range(4):
    residues = [coeffs_mods[i][j] for i in range(3)]
    modulii = n_list
    Cj = crt(residues, modulii) 
    C.append(Integer(Cj))


# Build polynomial F(x) with coefficients C (degree 3)
R = PolynomialRing(IntegerModRing(N), 'x')
x = R.gen()
F = C[0]*x**3 + C[1]*x**2 + C[2]*x + C[3]


# Sanity: F(x) mod n_i should equal f_i(x) mod n_i
for (A, c, n) in [(A1,c1,n1),(A2,c2,n2),(A3,c3,n3)]:
    fi = (x**3 + 3*A*x**2 + (3*A**2)*x + (A**3 - c))
    assert all((Integer(F.coefficient(k) - fi.coefficient(k)) % n) == 0 for k in range(4))

print("Constructed F(x) modulo N. Now running Coppersmith small_roots...")


X = Integer(2) ** (8 * flag_len)  # exact upper bound

# attempt small_roots. try beta ~ 1/3, and tweak epsilon value
roots = F.small_roots(X=X, beta=1/3, epsilon=0.01)  
print("roots:", roots)

# used long_to_bytes from Crypto.Util.number in python sandbox
```


output:
```py
N bits: 4360
Constructed F(x) modulo N. Now running Coppersmith small_roots...
roots: [414765597335564325713128441496359414877305966964858030459688398455857076654116366633149181171065139093295118031977095186092505818217885344608787735025146445474520500176696374689993381199009150949007352617402851237269028496993633201304414809673335792565405554244415871966289328621927248123612373940163781093006891249362074566104870295069035952672662954365]
```

```py
>>> long_to_bytes(414765597335564325713128441496359414877305966964858030459688398455857076654116366633149181171065139093295118031977095186092505818217885344608787735025146445474520500176696374689993381199009150949007352617402851237269028496993633201304414809673335792565405554244415871966289328621927248123612373940163781093006891249362074566104870295069035952672662954365)
b'guys please play resplendence https://osu.ppy.sh/beatmapsets/2436259#osu/5311353 6 digit represent osu{0mg_my_m4p_f1n4lly_g0t_r4nk3d_1m_s0_h4ppy!!}'
```


Note, `epsilon` in the solver was very important here. Setting it higher even at `0.02` makes it so that sage cannot find a root.