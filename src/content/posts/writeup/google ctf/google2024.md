---
title: 2024 Google CTF writeup
published: 2024-06-24
description: 2 solve
image: './googlectf.png'
tags: [block-cipher, DES, elliptic]
category: Crypto
draft: false
---

# 2024 GOOGLE CTF
기말고사 기간이라 ~~열심히 공부하다~~ 잠깐 했다<br>
나머지 문제는 기말 끝나고 풀어야겠다
+ 7/9  추가 - 기말 개망함

## DISFUNCTIONAL
### source code
```python
import signal
import os
import random
import sys
from Crypto.Cipher import DES3

class Desfunctional:
    def __init__(self):
        self.key = os.urandom(24)
        self.iv = os.urandom(8)
        self.flipped_bits = set(range(0, 192, 8))
        self.challenge = os.urandom(64)
        self.counter = 128

    def get_flag(self, plain):
        if plain == self.challenge:
            with open("flag.txt", "rb") as f:
                FLAG = f.read()
            return FLAG
        raise Exception("Not quite right")

    def get_challenge(self):
        cipher = DES3.new(self.key, mode=DES3.MODE_CBC, iv=self.iv)
        return cipher.encrypt(self.challenge)

    def corruption(self):
        if len(self.flipped_bits) == 192:
            self.flipped_bits = set(range(0, 192, 8))
        remaining = list(set(range(192)) - self.flipped_bits)
        num_flips = random.randint(1, len(remaining))
        self.flipped_bits = self.flipped_bits.union(
            random.choices(remaining, k=num_flips))
        mask = int.to_bytes(sum(2**i for i in self.flipped_bits), 24)
        return bytes(i ^ j for i, j in zip(self.key, mask))

    def decrypt(self, text: bytes):
        self.counter -= 1
        if self.counter < 0:
            raise Exception("Out of balance")
        key = self.corruption()
        if len(text) % 8 != 0:
            return b''
        cipher = DES3.new(key, mode=DES3.MODE_CBC, iv=self.iv)
        return cipher.decrypt(text)


if __name__ == "__main__":
    chall = Desfunctional()
    PROMPT = ("Choose an API option\n"
              "1. Get challenge\n"
              "2. Decrypt\n"
              "3. Get the flag\n")
    signal.alarm(128)
    while True:
        try:
            option = int(input(PROMPT))
            if option == 1:
                print(chall.get_challenge().hex())
            elif option == 2:
                ct = bytes.fromhex(input("(hex) ct: "))
                print(chall.decrypt(ct).hex())
            elif option == 3:
                pt = bytes.fromhex(input("(hex) pt: "))
                print(chall.get_flag(pt))
                sys.exit(0)
        except Exception as e:
            print(e)
            sys.exit(1)
```
`key`로 encrypt 된 `challenge`를 얻을 수 있고, `key`의 일부 비트가 반전된 `key'`로 input을 decrypt할 수 있다.<br>
`corruption`함수는 평균 9번 정도 실행할 경우 `mask = b'\xff'*24`가 된다.<br>
`mask = b'\xff' * 24`일 경우 `key`의 모든 비트가 반전되므로 굉장히 수상하다. 

`key' = key ^ mask`로 여러 값들을 decrypt하고, `mask`와 xor 한 값들을 관찰한 결과

```python
import os
from pwn import *
from Crypto.Cipher import DES3

key = os.urandom(24)
iv = os.urandom(8)
challenge = os.urandom(64)

mask = lambda n : b'\xff' * n
encrypt = lambda key, iv, pt : DES3.new(key, mode=DES3.MODE_CBC, iv=iv).encrypt(pt)
decrypt = lambda key, iv, ct : DES3.new(key, mode=DES3.MODE_CBC, iv=iv).decrypt(ct)

enc = encrypt(key, iv, challenge)
enc_x_mask = xor(enc, mask(64))

key_x_mask = xor(key, mask(24))
pt = decrypt(key_x_mask, iv, enc_x_mask)
pt_x_mask = xor(pt, mask(64))

print(challenge.hex())
print(enc.hex())
print(enc_x_mask.hex())
print(pt.hex())
print(pt_x_mask.hex())

"""
challenge            = 3e6076df89cf1d5b df30a22177f891c99099d47aaf1e28cd0291f876e5c9da38ffef568b7808cd8fcff914e9ccac8fc94017b3bcd54743ae14b72209785bef0a
enc                  = 851cba77dd6d0415 ba2cd9600ae3adbef9d163a7cf1ac398e3c68a820c51379512dcc8667b9384be0b0d7db26e70da106af87a793a2097f09263fd5616166c56
enc ^ mask           = 7ae345882292fbea 45d3269ff51c5241062e9c5830e53c671c39757df3aec86aed233799846c7b41f4f2824d918f25ef95078586c5df680f6d9c02a9e9e993a9
D(enc ^ mask)        = c19f89207630e2a4 df30a22177f891c99099d47aaf1e28cd0291f876e5c9da38ffef568b7808cd8fcff914e9ccac8fc94017b3bcd54743ae14b72209785bef0a
D(enc ^ mask) ^ mask = 3e6076df89cf1d5b 20cf5dde88076e366f662b8550e1d732fd6e07891a3625c70010a97487f732703006eb1633537036bfe84c432ab8bc51eb48ddf687a410f5
"""
```

:::note[WOW!]
`challenge = (D(enc ^ mask) ^ mask)[:8] + D(enc ^ mask)[8:]`
:::

### exploit
```python
from pwn import *
from tqdm import tqdm

context.log_level = 'error'

while True:
    p = remote("desfunctional.2024.ctfcompetition.com", 1337)

    p.sendlineafter(b"flag\n", b"1")
    enc = bytes.fromhex(p.recvline(keepends=False).decode())
    enc = xor(enc, b'\xff'*64)

    for _ in tqdm(range(9)):
        p.sendlineafter(b"flag\n", b"2")
        p.sendlineafter(b"ct: ", enc.hex().encode())
        dec = bytes.fromhex(p.recvline(keepends=False).decode())

    p.sendlineafter(b"flag\n", b"3")
    chall = xor(dec[:8], b'\xff'*8) + dec[8:]
    p.sendlineafter(b"pt: ", chall.hex().encode())

    flag = p.recvline()
    if flag.find(b"CTF") != -1: 
        import re
        flag = re.search('CTF{.*}', flag.decode()).group()
        raise ZeroDivisionError(flag)    
```
> `CTF{y0u_m4y_NOT_g3t_th3_k3y_but_y0u_m4y_NOT_g3t_th3_c1ph3rt3xt_as_w3ll}`

## BLINDERS
### source code
```python
from ecdsa.curves import NIST256p
from ecdsa.numbertheory import jacobi, square_root_mod_prime
from ecdsa.ellipticcurve import Point
from Crypto.Random import random
import hashlib

curve = NIST256p.curve

def H(id):
    a, b, p = curve.a(), curve.b(), curve.p()

    hash = hashlib.sha256(f'id={id}'.encode()).digest()
    x = int.from_bytes(hash, 'big')

    while True:
        y2 = (x**3 + a*x + b) % p
        if jacobi(y2, p) == 1: break
        x += 1

    y = square_root_mod_prime(y2, p)
    return Point(curve, x, y)

# Implements Blinders, a private set membership protocol.
class BlindersServer:
    def __init__(self, S):
        self.S = S
    
    def handle(self, client_eid):
        # 2.1. Generate a random secret key k
        k = random.randrange(0, NIST256p.order)
        # Compute eid1 = H(id1)^K, ..., eidn = H(idn)^K
        eids = [H(id) * k for id in self.S]
        # Compute doubly-encrypted identifier deid = eid^K
        deid = client_eid * k
        # Return (eid1, ..., eidn) and deid to P1
        return eids, deid

def challenge():
    # S = {0, 1, ..., 255} \ {x} for some 0 <= x < 256
    S = list(range(256))
    S.remove(random.getrandbits(8))
    server = BlindersServer(S)

    for _ in range(3):
        operation, *params = input().split()
        if operation == 'handle':
            client_eid = Point(curve, int(params[0]), int(params[1]))
            eids, deid = server.handle(client_eid)
            print([(eid.x(), eid.y()) for eid in eids])
            print((deid.x(), deid.y()))
        elif operation == 'submit':
            client_S_hash = bytes.fromhex(params[0])
            S_hash = hashlib.sha256(','.join(map(str, server.S)).encode()).digest()
            return client_S_hash == S_hash
        else:
            return False

if __name__ == '__main__':
    with open('/flag.txt', 'r') as f:
        FLAG = f.read().strip()

    # Convince me 16 times and I will give you the flag :)
    for _ in range(16):
        if challenge():
            print('OK!')
        else:
            print('Nope.')
            break
    else:
        print(FLAG)
```
`S = [0, 1, 2, ...  255] - [x] for some 0 <= x < 256`<br>
`eids = [H(id) * k for id in S]`<br>
`deid = k * Point(NIST256p, *user input())`<br>
`eids, deid`를 바탕으로 난수 하나가 빠진 배열 `S`를 알아내야 한다.<br>

point를 생성하는 함수인 `H(id)`는 Hash를 이용하기 때문에 `H(0), H(1), ... H(255)`의 값을 알 수 있다.

$$
P_1=\sum\limits^{127}_{i=0}H(2i)=H(0)+H(2)+...+H(254) \\
P_2=\sum\limits^{127}_{i=0}H(2i+1)=H(1)+H(3)+...+H(255)
$$
을 보내면

$$
\text{deid}_1=k_1\sum\limits^{127}_{i=0}H(2i) \\
\text{deid}_2=k_2\sum\limits^{127}_{i=0}H(2i+1)
$$
이제 brute-force를 통해 `S`를 복구할 수 있다.


편의상 사용자 지정 (로컬) 변수는 앞에 `_`를 붙이고, 서버측 변수는 그냥 쓰겠다.<br>
`_x == x`인 경우 `_S = S = [0, 1, 2, ... x-1, x+1, ... 255]`가 된다.<br>
그러면 `eids = [H(0)*k, H(1)*k, H(2)*k, ...  H(x-1)*k, H(x+1)*k, ... H(255)*k]`<br>

+ `x`가 홀수인 경우<br>
`_S, eids`에서 `H(_S[i]) * k = eids[i]`이다 ( _x != x일 경우 인덱스가 밀려 성립x ).<br>
S는 홀수 하나가 없기 때문에 `eids`에 H(2n)*k 꼴의 point는 모두 존재한다.<br>
따라서 `_S`의 짝수 인덱스를 모두 구하고, `eids`의 해당 인덱스만 더하면<br>
$$
k\sdot(H(0)+H(2)+...+H(x-1)+H(x+1)+...+H(254))=k\sum\limits^{127}_{i=0}H(2i)=\text{deid}_1
$$
+ `x`가 짝수인 경우도 마찬가지다.

:::note[WOW!]
$0\le \text{x} <256$ 이므로 가능한 모든 `x`에 대해 시도하면 유일한 해 `_x`가 존재한다.
:::


### exploit
```python
import hashlib
from tqdm import tqdm
from ecdsa.numbertheory import square_root_mod_prime
from pwn import *

io = remote("blinders.2024.ctfcompetition.com", int(1337))
context.log_level = 'error'

io.recvline() # recv  '== proof-of-work: disabled =='

p = 0xffffffff00000001000000000000000000000000ffffffffffffffffffffffff
K = GF(p)
a = K(0xffffffff00000001000000000000000000000000fffffffffffffffffffffffc)
b = K(0x5ac635d8aa3a93e7b3ebbd55769886bc651d06b0cc53b0f63bce3c3e27d2604b)
E = EllipticCurve(K, (a, b))

def H(id):
    hash = hashlib.sha256(f'id={id}'.encode()).digest()
    x = int.from_bytes(hash, 'big')

    while True:
        y2 = (x**3 + a*x + b) % p
        if jacobi_symbol(y2, p) == 1: break
        x += 1

    y = square_root_mod_prime(*map(int, [y2, p]))
    return E(x, y)

eid_1 = sum([H(i) for i in range(0, 256, 2)])
eid_2 = sum([H(i) for i in range(1, 256, 2)])

def sol():
    io.sendline(f"handle {eid_1.xy()[0]} {eid_1.xy()[1]}".encode())

    eids1 = eval(io.recvline())
    deid1 = eval(io.recvline())

    eids1 = [E(*eids1[i]) for i in range(255)]
    deid1 = E(*deid1)

    io.sendline(f"handle {eid_2.xy()[0]} {eid_2.xy()[1]}".encode())

    eids2 = eval(io.recvline())
    deid2 = eval(io.recvline())

    eids2 = [E(*eids2[i]) for i in range(255)]
    deid2 = E(*deid2)

    for i in range(256):
        S = [*range(256)]
        S.remove(i)

        if i%2:
            idx = [S.index(j) for j in range(0, 256, 2)]
            if sum([eids1[j] for j in idx]) == deid1:
                return S

        else:
            idx = [S.index(j) for j in range(1, 256, 2)]
            if sum([eids2[j] for j in idx]) == deid2:
                return S

for _ in tqdm(range(16)):
    S = sol()
    io.sendline(f"submit {hashlib.sha256(','.join(map(str,S)).encode()).hexdigest()}".encode())
    io.recvuntil(b"OK!\n")

raise ZeroDivisionError(io.recvline(keepends=False).decode())
```
> `CTF{pr1v4t3_s3t_m3mb3rsh1p_qu3r135_m3d4_m0r3_p0w3rfu1}`<br>


### 뉴비의 글 읽어주셔서 감사합니다!<br>
![Amelia Watson Winking](https://cdn3.emoji.gg/emojis/7050_Amelia_Watson_Winking.gif)
