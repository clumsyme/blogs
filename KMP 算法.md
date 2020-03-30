<!-- KMP 字符串查找算法用于在一个主字符串 `source` 里查找子字符串 `pattern`。在介绍 KMP 算法之前 -->

如何在一个主字符串 `source` 里查找子字符串 `pattern`？这个问题可以用[字符串查找算法](https://zh.wikipedia.org/wiki/%E5%AD%97%E4%B8%B2%E6%90%9C%E5%B0%8B%E6%BC%94%E7%AE%97%E6%B3%95)解决，最直观的字符串查找算法就是暴力搜索。但是暴力搜索在字符串有很多重复部分的情况下会导致重复对比，最差情况下可能达到 O(MN) 的时间复杂度。使用 KMP 算法，可以避免不必要的重复对比，使时间复杂度降低到 O(M + N)。

## 暴力搜索

```py
def search(source: str, pattern: str) -> int:
    """ 在 source 内搜索 pattern，
        如果匹配到，返回 pattern 所在的 index，
        否则，返回 -1
    """
    source_length = len(source)
    pattern_length = len(pattern)

    for start in range(source_length - pattern_length + 1):
        for i in range(pattern_length):
            if source[start + i] != pattern[i]:
                break
        else:
            return start
    else:
        return -1
```

### 暴力搜索的问题

比如我们在 `abcdef` 中 搜索 `cd`，那么暴力搜索的过程是：

```
a   b   c   d   e   f
-----------------------------------
c                           不匹配
    c                       不匹配
        c                   匹配
        c   d               匹配（完成）
```

可以发现，每次匹配失败，都会将 pattern 回退到 0，将 source 回退到 start+1（起始位置前进一步）。

那么在考虑下边这个情形，在 `aaaaab` 中匹配 `aab`：

```
a   a   a   b
------------------------------
a                       匹配
a   a                   匹配
a   a   b               不匹配
    a                   匹配
    a   a               匹配
    a   a   b           匹配（完成）
```

我们那从 start = 0 开始，在 `aaa` 没有匹配 `aab` 的时候，我们完全回退了 pattern，然后回退 source，从 start = 1 的位置开始对比。但是我们观察后可以发现，在这里，虽然 `aaa` 匹配 `aab` 失败了，但是后两个 `aa` 是匹配了 `aab` 的前两个 `aa` 的，所以回退就导致了重复的对比。

## 减少不必要的回退

KMP 算法的思想就是减少回退，尽量利用已知的匹配信息。事实上，KMP 算法从不回退 source，只回退 pattern。

所谓回退 pattern，就是知道在匹配了 `j` 个字符的状态下，再尝试匹配字符 `c`，会变成匹配到几个字符的状态？

试想在任意时刻，我们的 source 在位置 `i` 已经匹配了 pattern 的 `j` 个字符，source 的下个字符是 `c`，在尝试匹配 `c` 后，有两种情况：

-   匹配成功：那么我们就匹配到了 pattern 中的 `j+1` 个字符，特别是，如果 j+1 的长度和 pattern 相同，我们就完成了匹配
-   匹配失败：那么我们需要**回退 pattern**

比如对于 pattern：`aab`，如果我们已经匹配了 `aa` 2 个字符，那么 `j=2`。这时尝试继续匹配下一个字符，下一个字符可能是 `'a'` 或者 `'b'`：

-   `'b'`:匹配成功，`j = 2+1`，匹配完成
-   `'a'`:匹配失败，但是 `aaa` 可以匹配 pattern 的前 2 个字符 `aa`，所以回退到匹配了 2 个字符的状态。

## 状态转换函数 `δ`

**试想，如果我们有一个函数 `δ`，输入当前匹配的字符数 `j` 和下一个字符 `c`，可以输出更新后的匹配字符数 `j'`，那么我们就可以用这个函数去迭代 source 字符串即可。**

```py
# 初始只匹配了 0 个字符
j = 0
for char in source:
    # 输入当前匹配字符数，和下一个字符
    # 输出更新后匹配的字符数
    j = f(j, char)
    # 如果 j 更新后和 pattern 长度一样，那说明我们已经完全匹配了 patteren
    if j == pattern_length:
        break
```

所以关键就是如何构造这个函数 `δ`。

## 确定有限状态自动机

`f` 函数可以用一个[确定有限状态自动机](https://zh.wikipedia.org/wiki/%E7%A1%AE%E5%AE%9A%E6%9C%89%E9%99%90%E7%8A%B6%E6%80%81%E8%87%AA%E5%8A%A8%E6%9C%BA)实现。

确定有限状态自动机 M 是由：

-   一个非空有限的状态集合 Q
-   一个输入字母表 Σ（非空有限的字符集合）
-   一个转移函数 δ，接收上一个状态和一个输入，更新状态
-   一个开始状态 q0
-   一个中止状态的集合 F

如果 pattern 为 `aab`，并且所有字符均为 `'a'` 或者 `'b'` ，那么这个确定有限状态自动机 M：

-   Q = {0, 1, 2, 3}：一共有四个状态，状态 `j` 表示匹配到了 `j` 个字符
-   Σ = {a, b}：可以输入字符 `'a'` 或者 `'b'`
-   q0 = 0：开始状态，匹配了 0 个字符
-   F = { 3 }：在状态 3 中止
-   δ：状态转移函数如下：

| 当前状态 | 输入 | 输入 |
| -------- | ---- | ---- |
|          | a    | b    |
| 0        | 1    | 0    |
| 1        | 2    | 0    |
| 2        | 2    | 3    |

`δ(0, 'a') = 1`，也就是在状态 0 输入字符 `a` 会变成状态 1;
`δ(2, 'a') = 1`，也就是在状态 2 输入字符 `a` 会保持状态 2;
`δ(2, 'b') = 3`，也就是在状态 2 输入字符 `b` 会保持状态 3;完成匹配。

也可以用下边的状态转换图来表示：

![kmp](https://raw.githubusercontent.com/clumsyme/blogs/master/imgs/kmp/dfa.png)。

数据结构：

如果我们用 `m` 个字典构成一个列表 `dfa` ，列表的第 `j` 项表示状态为 `j` 时的字典，字典的 `key` 为所有可能的字符，`value` 为在当前状态输入字符 `key` 会转变成的状态。也就是 `dfa[j][c]` 表示状态为 `j` 时输入字符 `c` 得到的下一个状态。例如 pattern `aab` 的 dfa 可以构造为：

```py
dfa = [
    {
        'a': 1, 'b': 0,
    },
    {
        'a': 2, 'b': 0,
    },
    {
        'a': 2, 'b': 3
    }
]

δ(j, c) = dfa[j][c]
```

_实际应用中，更常见的是使用字符的 charCode 来构建数组来表示状态转换关系。此处使用字典是避免 charCode 转换使代码更易读。_

## 构建 dfa

对于 pattern：<code>p<sub>1</sub>p<sub>2</sub>...p<sub>m</sub></code>，有 `m` 个状态可以接收输入转变状态，所以对于长度为 `m` 的 pattern，我们需要一个长度为 m 的数组。

数组的每一项初始化为 value 为 0 的初始字典。

```py
pattern_length = len(pattern)
input_set = {'a', 'b'}
init_dict = {char: 0 for char in input_set}
dfa = [{**init_dict} for _ in range(patteren)]
```

在状态为 0 的时候，如果匹配了 pattern 的第一个字符，应该转变为状态 1，否则仍然保持状态 0.

```py
first_char = pattern[0]
dfa[0][first_char] = 1
```

状态 0 的所有转换到此完成了。我们从状态 1 开始，我们记录一个重启状态：`restart_j`，表示在状态 `j` 尝试匹配字符 `c` 失败后，我们应该将状态重设为 `restart_j`，并输入字符 `c` 来尝试匹配。所以，如果在 `j` 状态匹配失败的话，转变成的状态会是重启状态匹配字符 `c` 得到的状态。

**如果在状态 1 匹配失败，应该以状态 0 来进行重启。**

<!-- *（首先肯定不会以大于 1 的进行重启，以状态 1 重启会导致无限重启，所以应该从状态 0 重启）* -->

```py
# 起始重启状态为 0
restart_j = 0

# 从状态 1 到状态 pattern_length-1，计算 dfa
for j in range(1, pattern_length):
    match_char = pattern[j]

    # 对于输入字符集内的每一个字符 char
    # 如果在状态 j 匹配失败，以状态 restart_j 重启，那么状态应该是 dfa[restart_j][char]
    for char in input_set:
        dfa[j][char] = dfa[restart_j][char]

    # 如果匹配成功，应该将状态+1
    dfa[j][match_char] = j + 1

    # 在这次匹配前，重启状态为 restart_j，在 restart_j 状态尝试匹配了 match_char 后，重启状态应该更新为 dfa[restart_j][char]
    restart_j = dfa[restart_j][char]
```

综合起来，构建 dfa 以及状态转换函数的代码如下：

```py
input_set = {'a', 'b'}
def make_trans_func(pattern: str):
    pattern_length = len(pattern)
    init_dict = {char: 0 for char in input_set}
    dfa = [{**init_dict} for _ in range(pattern_length)]

    first_char = pattern[0]
    dfa[0][first_char] = 1

    restart_j = 0

    for j in range(1, pattern_length):
        match_char = pattern[j]
        for char in input_set:
            dfa[j][char] = dfa[restart_j][char]
        dfa[j][match_char] = j + 1
        restart_j = dfa[restart_j][char]

    def trans_func(j, char):
        return dfa[j][char]

    return trans_func
```

## KMP 算法实现

有了状态转换函数，就可以完成我们的 KMP 算法代码：

```py
def kmp_search(source: str, pattern: str) -> int:
    trans_func = make_trans_func(pattern)

    pattern_length = len(pattern)
    j = 0
    for index, char in enumerate(source):
        j = trans_func(j, char)
        if j == pattern_length:
            return index - pattern_length + 1
    else:
        return -1
```

### 测试

我们以 ASCII 编码为 `input_set`，来进行一些测试。

```py
from string import ascii_letters
input_set = {char for char in ascii_letters}

source = 'abcdabcabcabcdabceamansmantomtoaotomjerrybcdabceababc'
patterns = [ 
    'abcdabce',
    'tom',
    'jerry',
    'toao',
]

for pattern in patterns:
    assert kmp_search(source, pattern) == source.find(pattern)

for pattern in patterns:
    print(kmp_search(source, pattern), source.find(pattern))
# 10 10
# 26 26
# 36 36
# 29 29
```

### 性能

```py
> %timeit kmp_search(source, pattern)  
# 56 µs ± 2.99 µs per loop (mean ± std. dev. of 7 runs, 10000 loops each)  

> %timeit source.find(pattern) 
# 166 ns ± 3.34 ns per loop (mean ± std. dev. of 7 runs, 10000000 loops each)
```

看起来还不错。

## 复杂度分析