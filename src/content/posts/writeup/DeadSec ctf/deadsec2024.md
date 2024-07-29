---
title: 2024 DeadSec CTF writeup
published: 2024-07-28
description: '방학 전 마지막 CTF'
image: './deadsec.png'
tags: ['RSA', 'LLL', 'secret-sharing', 'subset-sum', 'etc']
category: 'Crypto'
draft: false
---
# 2024 DeadSec CTF

잔류인데 공부하기 싫었다. 마침 [액슨](https://blog.exon.kr/)의 꼬드김에 넘어가서 둘이서 참여하게 되었다.

**Table of Contents**

1. [Writeup (English)](#writeup-en) My English skills are poor 😅
    1. [Flag Killer](#flag-killer)
    2. [SSP](#ssp)
    3. [Raul Rosas](#raul-rosas)
    4. [Password Guesser](#password-guesser)
    5. [Not an Active Field for a Reason](#not-an-active-field-for-a-reason)
2. [Writeup (Korean)](#writeup-ko)
    1. [Flag Killer](#flag-killer-1)
    2. [SSP](#ssp-1)
    3. [Raul Rosas](#raul-rosas-1)
    4. [Password Guesser](#password-guesser-1)
    5. [Not an Active Field for a Reason](#not-an-active-field-for-a-reason-1)

<br>
<br>
<br>
<br>
<br>
<br>

# Writeup (En)

## Flag Killer

### source code

```python
#!/usr/bin/python3

from binascii import hexlify

flag = hexlify(b'DEAD{test}').decode()

index = 0
output = ''

def FLAG_KILLER(value):
    index = 0
    temp = []
    output = 0
    while value > 0:
        temp.append(2 - (value % 4) if value % 2 != 0 else 0)
        value = (value - temp[index])/2
        index += 1
    temp = temp[::-1]
    for index in range(len(temp)):
        output += temp[index] * 3 ** (len(temp) - index - 1)
    return output

while index < len(flag):
    output += '%05x' % int(FLAG_KILLER(int(flag[index:index+3],16)))
    index += 3

print(output)
```

### exploit

```python
k = set([FLAG_KILLER(i) for i in range(0x1000000)])
assert len(k) == 0x1000000 # True
```

The `FLAG_KILLER()` function is a one-to-one function. Since `FLAG_KILLER(i)` is unique for each `i`, you can find the value of `i` in reverse from `enc` after calculating the values for all `i` in [0, 0xffffff].

```python
from binascii import unhexlify
from itertools import product

p = "0123456789abcdef"

enc = "0e98b103240e99c71e320dd330dd430de2629ce326a4a2b6b90cd201030926a090cfc5269f904f740cd1001c290cd10002900cd100ee59269a8269a026a4a2d05a269a82aa850d03a2b6b900883"
enc = [int(enc[i:i+5], 16) for i in range(0, len(enc), 5)]

def FLAG_KILLER(value):
    index = 0
    temp = []
    output = 0
    while value > 0:
        temp.append(2 - (value % 4) if value % 2 != 0 else 0)
        value = (value - temp[index])/2
        index += 1
    temp = temp[::-1]
    for index in range(len(temp)):
        output += temp[index] * 3 ** (len(temp) - index - 1)
    return output

table = {}
for i in product(p, repeat=3):
    i = ''.join(i)
    v = FLAG_KILLER(int(i,16))
    table[v] = i

f = ''.join([table[e] for e in enc])

f1 = f
f2 = f[:-3]+f[-2:]
f3 = f[:-3]+f[-1:]

for flag in (f1, f2, f3):
    if len(flag)&1 == 0:
        raise ZeroDivisionError(bytes.fromhex(flag).decode())

>> ZeroDivisionError: DEAD{263f871e880e9dc7d2401000304fc60e98c7c588}
```

## SSP

### source code

```python
MAX_N = 100
RAND_MAX = 10 ** 200 # You can't even solve with knapsack method

from random import randrange

for n in range(1, MAX_N + 1):
    print(f"Stage {n}")

    arr = [ randrange(0, RAND_MAX) for _ in range(n) ]
    counts = randrange(0, n + 1)
    
    used = set()
    
    while True:
        idx = randrange(0, n)
        used.add(idx)

        if len(used) >= counts:
            break
    
    s = 0
    for idx in used:
        s += arr[idx]
    
    for a in arr:
        print(a, end=' ')
    print(s)

    answer = list(map(int, input().split()))

    user_sum = 0
    for i in answer:
        user_sum += arr[i]
    
    if user_sum != s:
        print("You are wrong!")
        exit(0)

    print(f"Stage {n} Clear")

print("Long time waiting... Here's your flag.")

with open('./flag', 'r') as f:
    print(f.read())
```

### exploit

$$
sum = \sum\limits_{i=0}^{n-1} x_iarr_i;\ x_i=0,1
$$
$$
M =
\begin{bmatrix}
I_{n\times n} & -arr_{n\times  1} \\
0_{1\times n} & sum
\end{bmatrix}
$$
Applying _LLL_ to $M$, we can get a vector that only contains 0,1.

```python
from tqdm import tqdm
from pwn import *
import re

# context.log_level = True
# p = process(["python3", "chall.py"])
p = remote("34.121.62.108", 31401)

MAX_N = 100
RAND_MAX = 10 ** 200

for n in tqdm(range(1, MAX_N+1)):
    p.recvuntil(f"Stage {n}\n".encode()) # stage

    *arr, s = p.recvline().decode().split()
    arr = [*map(int, arr)]
    s = int(s)

    L = matrix(QQ, n + 1, n + 1)
    for i in range(n):
        L[i, i] = 1
        L[i, n] = -arr[i]

    L[n, n] = s

    for v in L.LLL():
        v = list(v)
        if all((i in [0,1]) for i in v):
            sol = []
            for idx in range(len(v)):
                if v[idx] == 1:
                    sol.append(idx)
            sol = map(str, sol)
            break

    p.sendline(" ".join(sol).encode())

raise ZeroDivisionError(re.search('DEAD{.*}', p.recvall().decode()).group())

>> ZeroDivisionError: DEAD{T00_B1g_Number_Causes_Pr0blem...}
```

## Raul Rosas

### source code

```python
from  Crypto.Util.number  import  *
from  sympy  import  nextprime

p1  =  bin(getPrime(1024))[2:]
p2  =  p1[:605]
p2  =  p2  + ('0'*(len(p1)-len(p2))) # 0 * 419

p1  =  int(p1,2)
p2  =  nextprime(int(p2,2))

q1  =  getPrime(300)
q2  =  getPrime(300)

n1  =  p1*p1*q1
n2  =  p2*p2*q2

e  =  65537
flag  =  bytes_to_long(b'REDACTED')
c1  =  pow(flag,e,n1)
c2  =  pow(flag,e,n2)

print(f'{n1=}')
print(f'{n2=}')
print(f'{c1=}')
print(f'{c2=}')


"""
n1=33914684861748025775039281034732118800210172226202865626649257734640860626122496857824722482435571212266837521062975265470108636677204118801674455876175256919094583111702086440374440069720564836535455468886946320281180036997133848753476194808776154286740338853149382219104098930424628379244203425638143586895732678175237573473771798480275214400819978317207532566320561087373402673942574292313462136068626729114505686759701305592972367260477978324301469299251420212283758756993372112866755859599750559165005003201133841030574381795101573167606659158769490361449603797836102692182242091338045317594471059984757228202609971840405638858696334676026230362235521239830379389872765912383844262135900613776738814453
n2=45676791074605066998943099103364315794006332282441283064976666268034083630735700946472676852534025506807314001461603559827433723291528233236210007601454376876234611894686433890588598497194981540553814858726066215204034517808726230108550384400665772370055344973309767254730566845236167460471232855535131280959838577294392570538301153645042892860893604629926657287846345355440026453883519493151299226289819375073507978835796436834205595029397133882344120359631326071197504087811348353107585352525436957117561997040934067881585416375733220284897170841715716721313708208669285280362958902914780961119036511592607473063247721427765849962400322051875888323638189434117452309193654141881914639294164650898861297303
c1=5901547799381070840359392038174495588170513247847714273595411167296183629412915012222227027356430642556122066895371444948863326101566394976530551223412292667644441453331065752759544619792554573114517925105448879969399346787436142706971884168511458472259984991259195488997495087540800463362289424481986635322685691583804462882482621269852340750338483349943910768394808039522826196641550659069967791745064008046300108627004744686494254057929843770761235779923141642086541365488201157760211440185514437408144860842733403640608261720306139244013974182714767738134497204545868435961883422098094282377180143072849852529146164709312766146939608395412424617384059645917698095750364523710239164016515753752257367489
c2=3390569979784056878736266202871557824004856366694719533085092616630555208111973443587439052592998102055488632207160968490605754861061546019836966349190018267098889823086718042220586285728994179393183870155266933282043334755304139243271973119125463775794806745935480171168951943663617953860813929121178431737477240925668994665543833309966378218572247768170043609879504955562993281112055931542971553613629203301798161781786253559679002805820092716314906043601765180455118897800232982799905604384587625502913096329061269176369601390578862509347479694697409545495592160695530037113884443071693090949908858172105089597051790694863761129626857737468493438459158669342430468741236573321658187309329276080990875017
"""
```

### exploit

```python
bin(p2) = 0b???????????????????...0000000000000000000000 + k(which is small value)
bin(n2) = 0b10110001101000...00000000000000000000...11100110111010000010101011000000011010100111100000010000101011001001011100001111111101100011100100011011111110000001110110010010111111001100111100011010101000010001100100000010000111010011010001101111011101101011111011111000000111100001001100001100111011110011011010001000110111110000101010110010111000111010010111
= p2*q2 = (???????0000000+k)*q2 
= ???????? * q2 + 000000000..000 * q2 + k*q2
```

Factorize the LSB of $n_2$ to obtain $q_2$.

```python
from Crypto.Util.number import long_to_bytes

n2=45676791074605066998943099103364315794006332282441283064976666268034083630735700946472676852534025506807314001461603559827433723291528233236210007601454376876234611894686433890588598497194981540553814858726066215204034517808726230108550384400665772370055344973309767254730566845236167460471232855535131280959838577294392570538301153645042892860893604629926657287846345355440026453883519493151299226289819375073507978835796436834205595029397133882344120359631326071197504087811348353107585352525436957117561997040934067881585416375733220284897170841715716721313708208669285280362958902914780961119036511592607473063247721427765849962400322051875888323638189434117452309193654141881914639294164650898861297303
c2=3390569979784056878736266202871557824004856366694719533085092616630555208111973443587439052592998102055488632207160968490605754861061546019836966349190018267098889823086718042220586285728994179393183870155266933282043334755304139243271973119125463775794806745935480171168951943663617953860813929121178431737477240925668994665543833309966378218572247768170043609879504955562993281112055931542971553613629203301798161781786253559679002805820092716314906043601765180455118897800232982799905604384587625502913096329061269176369601390578862509347479694697409545495592160695530037113884443071693090949908858172105089597051790694863761129626857737468493438459158669342430468741236573321658187309329276080990875017

small_n2 = int("11100110111010000010101011000000011010100111100000010000101011001001011100001111111101100011100100011011111110000001110110010010111111001100111100011010101000010001100100000010000111010011010001101111011101101011111011111000000111100001001100001100111011110011011010001000110111110000101010110010111000111010010111", 2)

q2 = max(list(factor(small_n2)))[0]
assert q2.nbits() == 300 and q2.is_prime()

p2 = ZZ(n2//q2).nth_root(2)
assert p2.nbits() == 1024 and p2.is_prime()
assert p2^2*q2 == n2

phi = p2*(p2-1)*(q2-1)
d2 = pow(65537, -1, phi)

print(long_to_bytes(int(pow(c2, d2, n2))))

>> ZeroDivisionError: DEAD{Rual_R0s4s_Chiweweiner!!}
```

## Password Guesser

### source code

```python
from  collections  import  Counter
from  Crypto.Util.number  import  *
from  Crypto.Cipher  import  AES
import  hashlib
from  Crypto.Util.Padding  import  pad
import  math
  
flag  =  b'<REDACTED>'
P  =  13**37
password  =  b'<REDACTED>'
pl  =  list(password)
pl  =  sorted(pl)
assert  math.prod(pl) %  P  ==  sum(pl) %  P
password2  =  bytes(pl)
  
print(f"counts = {[cnt  for  _, cnt  in  Counter(password2).items()]}")
cipher  =  AES.new(hashlib.sha256(password2).digest(), AES.MODE_CBC)
print(f"c = {cipher.encrypt(pad(flag, 16))}")
print(f"iv = {cipher.iv}")
  
'''
counts = [5, 4, 7, 5, 5, 8, 9, 4, 5, 7, 4, 4, 7, 5, 7, 8, 4, 2, 5, 5, 4, 3, 10, 4, 5, 7, 4, 4, 4, 6, 5, 12, 5, 5, 5, 8, 7, 9, 2, 3, 2, 5, 8, 6, 4, 4, 7, 2, 4, 5, 7, 9, 4, 9, 7, 4, 7, 8, 4, 2, 4, 4, 4, 4, 3, 3, 7, 4, 6, 9, 4, 4, 4, 6, 7, 4, 4, 4, 1, 3, 5, 8, 4, 9, 11, 7, 4, 2, 4]
c = b'q[\n\x05\xad\x99\x94\xfb\xc1W9\xcb`\x96\xb9|CA\xb8\xb5\xe0v\x93\xff\x85\xaa\xa7\x86\xeas#c'
iv = b'+\xd5}\xd8\xa7K\x88j\xb5\xf7\x8b\x95)n53'
'''
```

### exploit

+ charset of password is `string.printable`.
+ If `pl` contains a multiple of 13, then `prod(pl)` is highly likely to have `P` as a factor, making `prod(pl) mod P` equal to 0.

Therefore, if we first remove the multiples of 13 from `printable`, we are left with 92 candidates. By examining all possible combinations of $_{92}C_{89}$, we can find the correct `key`.

```python
from string import printable
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad
import hashlib

P = 13^37
counts = [5, 4, 7, 5, 5, 8, 9, 4, 5, 7, 4, 4, 7, 5, 7, 8, 4, 2, 5, 5, 4, 3, 10, 4, 5, 7, 4, 4, 4, 6, 5, 12, 5, 5, 5, 8, 7, 9, 2, 3, 2, 5, 8, 6, 4, 4, 7, 2, 4, 5, 7, 9, 4, 9, 7, 4, 7, 8, 4, 2, 4, 4, 4, 4, 3, 3, 7, 4, 6, 9, 4, 4, 4, 6, 7, 4, 4, 4, 1, 3, 5, 8, 4, 9, 11, 7, 4, 2, 4]

for perm in Combinations([ord(i) for i in sorted(printable) if ord(i)%13 != 0], 89):
    key, pd, sm = b"", 1, 0
    for i in range(89):
        pd *= perm[i] ^ counts[i]
        sm += perm[i] * counts[i]
        key += bytes([perm[i]]) * counts[i]
    if pd % P == sm % P:
        ct = b'q[\n\x05\xad\x99\x94\xfb\xc1W9\xcb`\x96\xb9|CA\xb8\xb5\xe0v\x93\xff\x85\xaa\xa7\x86\xeas#c'
        iv = b'+\xd5}\xd8\xa7K\x88j\xb5\xf7\x8b\x95)n53'
        cipher = AES.new(hashlib.sha256(key).digest(), AES.MODE_CBC, iv)

        raise ZeroDivisionError(unpad(cipher.decrypt(ct), 16).decode())

>> ZeroDivisionError: DEAD{y0u_Gu3ssEd_mY_p4s5w0rD}
```

It would have been complicated if the `password` contained multiples of 13, but fortunately, the flag was obtained in one go.

## Not an active field for a reason

### source code

The `machine.py` is simply code that implements the [TPM](https://en.wikipedia.org/wiki/Neural_cryptography).

```python
from Crypto.Cipher import AES
import hashlib
from machine import TreeParityMachine
from secret import flag
import numpy as np
from Crypto.Util.Padding import pad

def encrypt(key, plaintext):
    cipher = AES.new(key, AES.MODE_ECB)
    return cipher.encrypt(pad(plaintext, 16)).hex()

k, l, n = 7, 10, 10
Alice = TreeParityMachine(k, n, l, "hebian")
Bob = TreeParityMachine(k, n, l, "hebian")

inputs = []
alice_taus = []
bob_taus = []

for _ in range(1000):
    x = np.random.randint(-25, 26, Alice.n * Alice.k)
    t1 = Alice.forward(x)
    t2 = Bob.forward(x)
    inputs.append(list(x))
    alice_taus.append(t1)
    bob_taus.append(t2)
    if t1 == t2:
        Alice.backward(Bob.tau)
        Bob.backward(Alice.tau)
        
assert np.array_equal(Bob.W, Alice.W)
assert Bob.W.shape == (k, n)

sha256 = hashlib.sha256()
sha256.update(Alice.W.tobytes())
key = sha256.digest()
ct = encrypt(key, flag)

with open("output.txt", "w") as f:
    f.write(f"ct = {ct}\n")
    f.write(f"inputs = {inputs}\n")
    f.write(f"alice_taus = {alice_taus}\n")
    f.write(f"bob_taus = {bob_taus}\n")
```

### exploit

Since this was an unfamiliar key exchange protocol and there seemed to be no vulnerabilities in `machine.py` itself, I explored some research papers and discovered the [geometry attack](https://arxiv.org/pdf/0711.2411.pdf#page=33).<br>
In summary, `Eve` generates a TPM (tree parity machine) first, and when `Alice.tau == Bob.tau`:<br>

+ If `Alice.tau == Eve.tau` - `Eve` can apply the learning rule (the process of making their W's identical).<br>
+ If `Alice.tau != Eve.tau` - `Eve` can use the `geometry attack` method to make her W identical to theirs.

```python
# This code worked on Windows, not Ubuntu.
import hashlib
import numpy as np
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad
from machine import TreeParityMachine
from output import ct, inputs, alice_taus, bob_taus

# https://arxiv.org/pdf/0711.2411.pdf#page=33
def geometry(TPM : TreeParityMachine, tau):
    wx = np.sum(TPM.x * TPM.W, axis=1)
    h_i = wx / np.sqrt(TPM.n)
    min_idx = np.argmin(np.abs(h_i))
    nonzero = np.where(TPM.roe == 0, -1, TPM.roe)
    TPM.roe[min_idx] = -nonzero[min_idx]
    TPM.tau = np.sign(np.prod(TPM.roe))
    
    if TPM.tau == tau:
        TPM.backward(tau)

Eve = TreeParityMachine(7, 10, 10, "hebian")
for i in range(1000):
    if alice_taus[i] == bob_taus[i]:
        if alice_taus[i] == Eve.forward(np.array(inputs[i])): 
            Eve.backward(alice_taus[i])
        else: geometry(Eve, alice_taus[i])

raise ZeroDivisionError(unpad(AES.new(hashlib.sha256(Eve.W.tobytes()).digest(), AES.MODE_ECB).decrypt(ct), 16).decode())

>> ZeroDivisionError:DEAD{Hamoor_added_AI_so_crypto_people_think_its_hard}
```

Interestingly, through the `geometry attack`, `Eve.W` becomes equal to `Alice.W` exactly after 1000 iterations.

---
---
---
---
---

# Writeup (Ko)

## Flag Killer

### source code

```python
#!/usr/bin/python3

from binascii import hexlify

flag = hexlify(b'DEAD{test}').decode()

index = 0
output = ''

def FLAG_KILLER(value):
    index = 0
    temp = []
    output = 0
    while value > 0:
        temp.append(2 - (value % 4) if value % 2 != 0 else 0)
        value = (value - temp[index])/2
        index += 1
    temp = temp[::-1]
    for index in range(len(temp)):
        output += temp[index] * 3 ** (len(temp) - index - 1)
    return output

while index < len(flag):
    output += '%05x' % int(FLAG_KILLER(int(flag[index:index+3],16)))
    index += 3

print(output)
```

### exploit

```python
k = set([FLAG_KILLER(i) for i in range(0x1000000)])
assert len(k) == 0x1000000 # True
```

 `FLAG_KILLER()`함수는 정의역과 공역이 모두 [0, 0xffffff]인 일대일 함수로 볼 수 있다. 각 `i`에 대해 `FLAG_KILLER(i)`가 유일하므로 모든 `i`에 대해 값을 구한 후 `enc`에서 역으로 `i`를 구할 수 있다.

```python
from binascii import unhexlify
from itertools import product

p = "0123456789abcdef"

enc = "0e98b103240e99c71e320dd330dd430de2629ce326a4a2b6b90cd201030926a090cfc5269f904f740cd1001c290cd10002900cd100ee59269a8269a026a4a2d05a269a82aa850d03a2b6b900883"
enc = [int(enc[i:i+5], 16) for i in range(0, len(enc), 5)]

def FLAG_KILLER(value):
    index = 0
    temp = []
    output = 0
    while value > 0:
        temp.append(2 - (value % 4) if value % 2 != 0 else 0)
        value = (value - temp[index])/2
        index += 1
    temp = temp[::-1]
    for index in range(len(temp)):
        output += temp[index] * 3 ** (len(temp) - index - 1)
    return output

table = {}
for i in product(p, repeat=3):
    i = ''.join(i)
    v = FLAG_KILLER(int(i,16))
    table[v] = i

f = ''.join([table[e] for e in enc])

f1 = f
f2 = f[:-3]+f[-2:]
f3 = f[:-3]+f[-1:]

for flag in (f1, f2, f3):
    if len(flag)&1 == 0:
        raise ZeroDivisionError(bytes.fromhex(flag).decode())

>> ZeroDivisionError: DEAD{263f871e880e9dc7d2401000304fc60e98c7c588}
```

## SSP

지금 내 옆에 앉아있는 친구가 낸 문제이다.

### source code

```python
MAX_N = 100
RAND_MAX = 10 ** 200 # You can't even solve with knapsack method

from random import randrange

for n in range(1, MAX_N + 1):
    print(f"Stage {n}")

    arr = [ randrange(0, RAND_MAX) for _ in range(n) ]
    counts = randrange(0, n + 1)
    
    used = set()
    
    while True:
        idx = randrange(0, n)
        used.add(idx)

        if len(used) >= counts:
            break
    
    s = 0
    for idx in used:
        s += arr[idx]
    
    for a in arr:
        print(a, end=' ')
    print(s)

    answer = list(map(int, input().split()))

    user_sum = 0
    for i in answer:
        user_sum += arr[i]
    
    if user_sum != s:
        print("You are wrong!")
        exit(0)

    print(f"Stage {n} Clear")

print("Long time waiting... Here's your flag.")

with open('./flag', 'r') as f:
    print(f.read())
```

### exploit

$$
sum = \sum\limits_{i=0}^{n-1} x_iarr_i;\ x_i=0,1
$$
`user_sum`은 위와 같은 식으로 나타낼 수 있다. 누가봐도 LLL을 쓰면 풀릴거 같다.
$$
M =
\begin{bmatrix}
I_{n\times n} & -arr_{n\times  1} \\
0_{1\times n} & sum
\end{bmatrix}
$$
$M$에 LLL을 적용하면 0,1만 존재하는 `answer`가 담긴 행이 존재한다.

```python
from tqdm import tqdm
from pwn import *
import re

# context.log_level = True
# p = process(["python3", "chall.py"])
p = remote("34.121.62.108", 31401)

MAX_N = 100
RAND_MAX = 10 ** 200

for n in tqdm(range(1, MAX_N+1)):
    p.recvuntil(f"Stage {n}\n".encode()) # stage

    *arr, s = p.recvline().decode().split()
    arr = [*map(int, arr)]
    s = int(s)

    L = matrix(QQ, n + 1, n + 1)
    for i in range(n):
        L[i, i] = 1
        L[i, n] = -arr[i]

    L[n, n] = s

    for v in L.LLL():
        v = list(v)
        if all((i in [0,1]) for i in v):
            sol = []
            for idx in range(len(v)):
                if v[idx] == 1:
                    sol.append(idx)
            sol = map(str, sol)
            break

    p.sendline(" ".join(sol).encode())

raise ZeroDivisionError(re.search('DEAD{.*}', p.recvall().decode()).group())

>> ZeroDivisionError: DEAD{T00_B1g_Number_Causes_Pr0blem...}
```

## Raul Rosas

### source code

```python
from  Crypto.Util.number  import  *
from  sympy  import  nextprime

p1  =  bin(getPrime(1024))[2:]
p2  =  p1[:605]
p2  =  p2  + ('0'*(len(p1)-len(p2))) # 0 * 419

p1  =  int(p1,2)
p2  =  nextprime(int(p2,2))

q1  =  getPrime(300)
q2  =  getPrime(300)

n1  =  p1*p1*q1
n2  =  p2*p2*q2

e  =  65537
flag  =  bytes_to_long(b'REDACTED')
c1  =  pow(flag,e,n1)
c2  =  pow(flag,e,n2)

print(f'{n1=}')
print(f'{n2=}')
print(f'{c1=}')
print(f'{c2=}')


"""
n1=33914684861748025775039281034732118800210172226202865626649257734640860626122496857824722482435571212266837521062975265470108636677204118801674455876175256919094583111702086440374440069720564836535455468886946320281180036997133848753476194808776154286740338853149382219104098930424628379244203425638143586895732678175237573473771798480275214400819978317207532566320561087373402673942574292313462136068626729114505686759701305592972367260477978324301469299251420212283758756993372112866755859599750559165005003201133841030574381795101573167606659158769490361449603797836102692182242091338045317594471059984757228202609971840405638858696334676026230362235521239830379389872765912383844262135900613776738814453
n2=45676791074605066998943099103364315794006332282441283064976666268034083630735700946472676852534025506807314001461603559827433723291528233236210007601454376876234611894686433890588598497194981540553814858726066215204034517808726230108550384400665772370055344973309767254730566845236167460471232855535131280959838577294392570538301153645042892860893604629926657287846345355440026453883519493151299226289819375073507978835796436834205595029397133882344120359631326071197504087811348353107585352525436957117561997040934067881585416375733220284897170841715716721313708208669285280362958902914780961119036511592607473063247721427765849962400322051875888323638189434117452309193654141881914639294164650898861297303
c1=5901547799381070840359392038174495588170513247847714273595411167296183629412915012222227027356430642556122066895371444948863326101566394976530551223412292667644441453331065752759544619792554573114517925105448879969399346787436142706971884168511458472259984991259195488997495087540800463362289424481986635322685691583804462882482621269852340750338483349943910768394808039522826196641550659069967791745064008046300108627004744686494254057929843770761235779923141642086541365488201157760211440185514437408144860842733403640608261720306139244013974182714767738134497204545868435961883422098094282377180143072849852529146164709312766146939608395412424617384059645917698095750364523710239164016515753752257367489
c2=3390569979784056878736266202871557824004856366694719533085092616630555208111973443587439052592998102055488632207160968490605754861061546019836966349190018267098889823086718042220586285728994179393183870155266933282043334755304139243271973119125463775794806745935480171168951943663617953860813929121178431737477240925668994665543833309966378218572247768170043609879504955562993281112055931542971553613629203301798161781786253559679002805820092716314906043601765180455118897800232982799905604384587625502913096329061269176369601390578862509347479694697409545495592160695530037113884443071693090949908858172105089597051790694863761129626857737468493438459158669342430468741236573321658187309329276080990875017
"""
```

### exploit

```python
bin(p2) = 0b???????????????????...000000000000000000...??????
bin(n2) = 0b10110001101000...000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000011100110111010000010101011000000011010100111100000010000101011001001011100001111111101100011100100011011111110000001110110010010111111001100111100011010101000010001100100000010000111010011010001101111011101101011111011111000000111100001001100001100111011110011011010001000110111110000101010110010111000111010010111
```

$n_2$의 하위 비트를 보면 $0$이 반복되는 구간이 있고, 이것이 $p_2$의 $0$이 반복되는 구간과 $q_2$를 곱한 부분이라고 추측할 수 있다. 따라서 $0$이 반복되기 전까지의 $n_2$값을 소인수분해하면 $k*q_2$를 얻을 수 있고, $q_2$를 얻었다면 $p_2$를 얻을 수 있다.

```python
from Crypto.Util.number import long_to_bytes

n2=45676791074605066998943099103364315794006332282441283064976666268034083630735700946472676852534025506807314001461603559827433723291528233236210007601454376876234611894686433890588598497194981540553814858726066215204034517808726230108550384400665772370055344973309767254730566845236167460471232855535131280959838577294392570538301153645042892860893604629926657287846345355440026453883519493151299226289819375073507978835796436834205595029397133882344120359631326071197504087811348353107585352525436957117561997040934067881585416375733220284897170841715716721313708208669285280362958902914780961119036511592607473063247721427765849962400322051875888323638189434117452309193654141881914639294164650898861297303
c2=3390569979784056878736266202871557824004856366694719533085092616630555208111973443587439052592998102055488632207160968490605754861061546019836966349190018267098889823086718042220586285728994179393183870155266933282043334755304139243271973119125463775794806745935480171168951943663617953860813929121178431737477240925668994665543833309966378218572247768170043609879504955562993281112055931542971553613629203301798161781786253559679002805820092716314906043601765180455118897800232982799905604384587625502913096329061269176369601390578862509347479694697409545495592160695530037113884443071693090949908858172105089597051790694863761129626857737468493438459158669342430468741236573321658187309329276080990875017

small_n2 = int("11100110111010000010101011000000011010100111100000010000101011001001011100001111111101100011100100011011111110000001110110010010111111001100111100011010101000010001100100000010000111010011010001101111011101101011111011111000000111100001001100001100111011110011011010001000110111110000101010110010111000111010010111", 2)

q2 = max(list(factor(small_n2)))[0]
assert q2.nbits() == 300 and q2.is_prime()

p2 = ZZ(n2//q2).nth_root(2)
assert p2.nbits() == 1024 and p2.is_prime()
assert p2^2*q2 == n2

phi = p2*(p2-1)*(q2-1)
d2 = pow(65537, -1, phi)

print(long_to_bytes(int(pow(c2, d2, n2))))

>> ZeroDivisionError: DEAD{Rual_R0s4s_Chiweweiner!!}
```

## Password Guesser

### source code

```python
from  collections  import  Counter
from  Crypto.Util.number  import  *
from  Crypto.Cipher  import  AES
import  hashlib
from  Crypto.Util.Padding  import  pad
import  math
  
flag  =  b'<REDACTED>'
P  =  13**37
password  =  b'<REDACTED>'
pl  =  list(password)
pl  =  sorted(pl)
assert  math.prod(pl) %  P  ==  sum(pl) %  P
password2  =  bytes(pl)
  
print(f"counts = {[cnt  for  _, cnt  in  Counter(password2).items()]}")
cipher  =  AES.new(hashlib.sha256(password2).digest(), AES.MODE_CBC)
print(f"c = {cipher.encrypt(pad(flag, 16))}")
print(f"iv = {cipher.iv}")
  
'''
counts = [5, 4, 7, 5, 5, 8, 9, 4, 5, 7, 4, 4, 7, 5, 7, 8, 4, 2, 5, 5, 4, 3, 10, 4, 5, 7, 4, 4, 4, 6, 5, 12, 5, 5, 5, 8, 7, 9, 2, 3, 2, 5, 8, 6, 4, 4, 7, 2, 4, 5, 7, 9, 4, 9, 7, 4, 7, 8, 4, 2, 4, 4, 4, 4, 3, 3, 7, 4, 6, 9, 4, 4, 4, 6, 7, 4, 4, 4, 1, 3, 5, 8, 4, 9, 11, 7, 4, 2, 4]
c = b'q[\n\x05\xad\x99\x94\xfb\xc1W9\xcb`\x96\xb9|CA\xb8\xb5\xe0v\x93\xff\x85\xaa\xa7\x86\xeas#c'
iv = b'+\xd5}\xd8\xa7K\x88j\xb5\xf7\x8b\x95)n53'
'''
```

### exploit

+ password의 charset은 `string.printable`이다.
+ `prod(pl) mod P`에서 `pl`에 13에 배수가 있다면 mod에 걸려 0이 될 가능성이 있다.

따라서 먼저 `printable`에서 13의 배수를 제거한다면 92개의 후보가 존재하고, $_{92}C_{89}$의 가능한 쌍을 모두 검사해 올바른 `key`를 구할 수 있다.

```python
from string import printable
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad
import hashlib

P = 13^37
counts = [5, 4, 7, 5, 5, 8, 9, 4, 5, 7, 4, 4, 7, 5, 7, 8, 4, 2, 5, 5, 4, 3, 10, 4, 5, 7, 4, 4, 4, 6, 5, 12, 5, 5, 5, 8, 7, 9, 2, 3, 2, 5, 8, 6, 4, 4, 7, 2, 4, 5, 7, 9, 4, 9, 7, 4, 7, 8, 4, 2, 4, 4, 4, 4, 3, 3, 7, 4, 6, 9, 4, 4, 4, 6, 7, 4, 4, 4, 1, 3, 5, 8, 4, 9, 11, 7, 4, 2, 4]

for perm in Combinations([ord(i) for i in sorted(printable) if ord(i)%13 != 0], 89):
    key, pd, sm = b"", 1, 0
    for i in range(89):
        pd *= perm[i] ^ counts[i]
        sm += perm[i] * counts[i]
        key += bytes([perm[i]]) * counts[i]
    if pd % P == sm % P:
        ct = b'q[\n\x05\xad\x99\x94\xfb\xc1W9\xcb`\x96\xb9|CA\xb8\xb5\xe0v\x93\xff\x85\xaa\xa7\x86\xeas#c'
        iv = b'+\xd5}\xd8\xa7K\x88j\xb5\xf7\x8b\x95)n53'
        cipher = AES.new(hashlib.sha256(key).digest(), AES.MODE_CBC, iv)

        raise ZeroDivisionError(unpad(cipher.decrypt(ct), 16).decode())

>> ZeroDivisionError: DEAD{y0u_Gu3ssEd_mY_p4s5w0rD}
```

`password`에 13의 배수가 있었다면 복잡했겠지만 다행히 한 번에 flag가 나왔다.

## Not an Active Field for a Reason

### source code

`machine.py`는 단순히 [TPM](https://en.wikipedia.org/wiki/Neural_cryptography)을 구현한 코드이다.

```python
from Crypto.Cipher import AES
import hashlib
from machine import TreeParityMachine
from secret import flag
import numpy as np
from Crypto.Util.Padding import pad

def encrypt(key, plaintext):
    cipher = AES.new(key, AES.MODE_ECB)
    return cipher.encrypt(pad(plaintext, 16)).hex()

k, l, n = 7, 10, 10
Alice = TreeParityMachine(k, n, l, "hebian")
Bob = TreeParityMachine(k, n, l, "hebian")

inputs = []
alice_taus = []
bob_taus = []

for _ in range(1000):
    x = np.random.randint(-25, 26, Alice.n * Alice.k)
    t1 = Alice.forward(x)
    t2 = Bob.forward(x)
    inputs.append(list(x))
    alice_taus.append(t1)
    bob_taus.append(t2)
    if t1 == t2:
        Alice.backward(Bob.tau)
        Bob.backward(Alice.tau)
        
assert np.array_equal(Bob.W, Alice.W)
assert Bob.W.shape == (k, n)

sha256 = hashlib.sha256()
sha256.update(Alice.W.tobytes())
key = sha256.digest()
ct = encrypt(key, flag)

with open("output.txt", "w") as f:
    f.write(f"ct = {ct}\n")
    f.write(f"inputs = {inputs}\n")
    f.write(f"alice_taus = {alice_taus}\n")
    f.write(f"bob_taus = {bob_taus}\n")
```

### exploit

처음 보는 키 교환 프로토콜이고, `machine.py`자체에는 취약점이 없어보였기 때문에 논문을 탐색한 결과, [geometry attack](https://arxiv.org/pdf/0711.2411.pdf#page=33)을 발견할 수 있었다.
요약하자면 `Eve`도 TPM을 생성하고, `Alice.tau == Bob.tau`일때,

+ `Alice.tau == Eve.tau`라면 `Eve`도 learning rule(서로의 W를 똑같이 만드는 과정)을 적용하고,
+ `Alice.tau != Eve.tau`라면 `geometry attack`을 적용하는 방법으로 `Eve`의 W를 똑같이 만들 수 있다.

```python
import hashlib
import numpy as np
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad
from machine import TreeParityMachine
from output import ct, inputs, alice_taus, bob_taus

# https://arxiv.org/pdf/0711.2411.pdf#page=33
def geometry(TPM : TreeParityMachine, tau):
    wx = np.sum(TPM.x * TPM.W, axis=1)
    h_i = wx / np.sqrt(TPM.n)
    min_idx = np.argmin(np.abs(h_i))
    nonzero = np.where(TPM.roe == 0, -1, TPM.roe)
    TPM.roe[min_idx] = -nonzero[min_idx]
    TPM.tau = np.sign(np.prod(TPM.roe))
    
    if TPM.tau == tau:
        TPM.backward(tau)

Eve = TreeParityMachine(7, 10, 10, "hebian")
for i in range(1000):
    if alice_taus[i] == bob_taus[i]:
        if alice_taus[i] == Eve.forward(np.array(inputs[i])): 
            Eve.backward(alice_taus[i])
        else: geometry(Eve, alice_taus[i])

raise ZeroDivisionError(unpad(AES.new(hashlib.sha256(Eve.W.tobytes()).digest(), AES.MODE_ECB).decrypt(ct), 16).decode())

>> ZeroDivisionError:DEAD{Hamoor_added_AI_so_crypto_people_think_its_hard}
```

신기하게도 `geometry attack`을 통해 딱 1000번 반복시 `Alice.W == Eve.W`가 된다.

### tmi

`ex.py`를 ubuntu에서 코딩했는데, `geometry attack`구현에 수많은 시행착오를 겪었다. 그렇게 완성된 코드를 ubuntu에서 실행하면 flag를 찾지 못하지만, window에서 실행하면 flag가 나온다(? ? ? ? ? ? ? ? ? ? ? ? ? ? ?). 왜 이런 차이가 발생하는지 모르겠지만 망할 os차이만 없어도 퍼블이었다.

# 후기

한동안 CTF안하고 있었는데, 이제 슬슬 CTF 대회시즌이 다가오니 재활을 열심히 해야겠다. 방학때 완벽한 계획을 세워 갓생을 살며 공부 + 운동 + IT 모두 하는 완벽한 계획을 세울 계획을 계획해야겠다.

### 읽어주셔서 감사합니다

![Amelia Watson Winking](https://cdn3.emoji.gg/emojis/7050_Amelia_Watson_Winking.gif)
