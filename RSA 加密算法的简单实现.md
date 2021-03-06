RSA 加密算法是目前广泛使用的一种非对称加密算法。RSA 加密算法的可靠性依赖于极大整数因数分解的困难度。本文主要根据 RSA 加密算法实现一个简单版本的 Python 代码实现，包括私钥、公钥生成与加密、解密方法。

## RSA 加密算法概述

RSA 加密算法为非对称加密算法，意味着加密和解密使用不同的秘钥，称为 **公钥** 和 **私钥** ，公钥可以任意分发并用于加密，私钥需妥善保存用于解密。

### 公钥

公钥由两个数 E、N 组成，使用公钥加密时运算

```
密文 = (明文 ** E) % N
```

### 私钥

私钥由两个数 D、N 组成，使用私钥解密时运算

```
明文 = (密文 ** D) % N
```

所以生成 RSA 加密算法的秘钥就是生成 E、D、N 这三个数。生成过程大致如下：

1. 生成两个大质数 p 和 q，计算 N = p*q
2. 求得 L = (p-1) * (q-1)
3. 选择一个大于 1 小于 L 的正整数 E，使 E 与 L 互质
4. 求得 E 关于 L 的模逆元，即为 D，D 小于 L

## 实现

### 质数生成函数

为了生成质数，我们需要判断一个数是否是质数，我们可以使用数学上的[费马小定理](https://zh.wikipedia.org/wiki/%E8%B4%B9%E9%A9%AC%E5%B0%8F%E5%AE%9A%E7%90%86)：

> 如果 a 是一个整数，n 是一个质数，那么 a ** n ≡ a (mod n)
>
> ———— 费马小定理

符合费马小定理是一个数是质数的必要而非充分条件，所以为了判断一个数 p 是否是质数，通常需要使用 k 个不同的 a 值来进行验证，称为[费马素性检验](https://zh.wikipedia.org/wiki/%E8%B4%B9%E9%A9%AC%E7%B4%A0%E6%80%A7%E6%A3%80%E9%AA%8C)，如果随机出来的 a 值都符合费马小定理，我们认为该数是一个质数。

质数判断函数：

```py
import random

# 使用费马小定理检查一个数 n 是不是质数
def is_prime(n):
    if n < 2:
        return False
    
    # 100次 费马素性检测
    for _ in range(100):
        a = random.randint(2, n-1)
        if pow(a, n, n) != a:
            return False
    return True
```

质数生成函数：

```py
# 生成一个随机的大质数(介于 10^100 与 10^110 之间)
def generate_prime():
    while True:
        expect_number = random.randint(1E100, 1E110)
        if is_prime(expect_number):
            break
    return expect_number
```

### 生成 p、q、N、L：

```py
p = generate_prime()
q = generate_prime()

N = p*q

L = (p-1) * (q-1)
```

### 生成 E：

根据算法，E 是介于 1 与 L 之间与 L 互质的数（最大公约数为 1）：

```py
import math

# E 的随机生成函数
def generate_e():
    while True:
        expect_e = random.randint(2, L)
        if math.gcd(expect_e, L) == 1:
            break
    return expect_e

E = generate_e()
```

### 生成 D：

根据算法，D 是 E 关于 L 的[模逆元](https://zh.wikipedia.org/wiki/%E6%A8%A1%E5%8F%8D%E5%85%83%E7%B4%A0)，也就是：

```
E * D ≡ 1 (mod L)
```

我们当然可以从 1 开始一直试到 L 来寻找 D，但是在 D 很大大的情况下计算量是非常大的。这里我们使用[扩展欧几里得算法](https://zh.wikipedia.org/wiki/%E6%89%A9%E5%B1%95%E6%AC%A7%E5%87%A0%E9%87%8C%E5%BE%97%E7%AE%97%E6%B3%95)（以下代码出自该 wiki 条目）：

```py
# 扩展欧几里得算法求模逆元
def ext_euclid(E, L):
     if L == 0:
         return 1, 0, E
     else:
         x, y, q = ext_euclid(L, E % L) # q = gcd(E, L) = gcd(L, E%L)
         x, y = y, (x - (E // L) * y)
         return x, y, q

D = ext_euclid(E, L)[0]
```

## 加密与解密算法

至此，我们的秘钥已经生成完成：公钥 (N, E)、私钥：(N, D)。

```py
# 加密算法：
# 密文 = 明文 ** E % N
def encrypt(message):
    return pow(message, E, N)

# 解密算法
# 明文 = 密文 ** D % N
def decrypt(secret):
    return pow(secret, D, N)
```

下边为我们根据上述算法生成的一对秘钥：

```
公钥 (N, E) = (
    4035201027052526716476770815301885547930386611424746846836729207720402684586903067390911032635700387807809913761690633108036479608875715316392163632899103649241282491070030290136720168432012867861620750453121897352765277, 
    3008241542036098467083169586257729095679728537524058797442421603938590337844782185494379049491888737248445705876804690371511766177598271941925223485143156067146179224661664800047568667114630699761924649833705169821421939
)

私钥：(N, D) = (
    4035201027052526716476770815301885547930386611424746846836729207720402684586903067390911032635700387807809913761690633108036479608875715316392163632899103649241282491070030290136720168432012867861620750453121897352765277, 
    263104268796040141692115220957372648791994375662679669674392390239402717514924248311933394103742807630673302384124516817979074152829180217297193870626824603082561775343291294186200630148873101176593862045968382744834439
)
```

我们尝试一次加密与解密：

```py
message = 3141592653

secret = encrypt(message)
print(secret)
# 1926350294561951470341018025045828648112140614621362576015267907023473556254498224885881403355559114529772490367097950170850223416554454126315058562501008274952527345854593658423828584546446402279730866190911413291563786

decrypted_message = decrypt(secret)
print(decrypted_message)
# 3141592653

message == decrypted_message
# True
```

解密后的明文和开始的明文相同，加密解密成功。
