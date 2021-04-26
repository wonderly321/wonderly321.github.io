---
layout: post
title: "Log-Code Jam to IO for Women 2021"
categories: code
author: "Wonder"
meta: "Code Jam to IO for Women 2021"
---

一转眼过去一年半了。

终于又再次做了一直想搞清楚而终于花时间搞清楚了的事情。

#### 1. Impartial Offerings

按照宠物的体格大小为宠物公平分配食物，求最小食物份数

思路：

按照动物体格分级统计数量

code:

```python
def Process(len, list):
    new_list = sorted(set(list))
    dict = {v: w for w, v in enumerate(new_list, 1)}
    ans = 0
    for v in list:
        ans += dict[v]
    return ans

if __name__ == '__main__':
    t = input()
    for cas in xrange(1, t+1):
        len = input()
        list_int = [int(s) for s in raw_input().split(" ")]
        ans = Process(len, list_int)
        print("Case #{}: {}".format(cas, ans))
```

<h4>2. Inconstant Ordering</h4>
字符串成块排列按照第奇或偶块呈升序或降序，要求块连接处也符合这一标准。求字母序最小的字符串

思路：

按照题意怼， 仅需要注意连接处的选择。为使字母序最小，则每一块都有应保证最小字母为A，且在连接处，如有需要应更改前一个block的最末一位保证符合降序顺序要求。

code:

```python
def Process(len, list):
    ans = 'A'
    al = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ'
    for i, l in enumerate(list, 1):
        idx = al.index(ans[-1])
        if i % 2 == 1:
            ans += al[idx+1:idx+l+1]
        else:
            if ans[-1] <= al[l-1]:
                ans = ans[:-1]
                ans += al[l]
            ans += al[l-1::-1]
    return ans

if __name__ == '__main__':
    t = input()
    for cas in xrange(1, t+1):
        len = input()
        list_int = [int(s) for s in raw_input().split(" ")]
        ans = Process(len, list_int)
        print("Case #{}: {}".format(cas, ans))
```

#### 3. Introductions Organization

组织团建，一个组织中有经理和普通员工两种人。有些互相认识，有些不认识。需要经过经理介绍才能互相认识。

思路：

建图，人为节点，认识则有边，意识到两个互相不认识的人的最优认识路径的中间节点必定是经理节点。

code:

```python
def Process(m, n, mtx, q):
    h = [[int(i != j) if mtx[i][j] == 'Y' else 110 for i in range(m)] for j in range(m)]
    for k in range(m):
        for i in range(m):
            for j in range(m):
                if h[i][j] > h[i][k] + h[k][j]:
                    h[i][j] = h[i][k] + h[k][j]
    ans = []
    for [x, y] in q:
        if mtx[x-1][y-1] == 'Y':
            ans.append('0')
            continue
        ret = 110
        for i in range(m):
            for j in range(m):
                if mtx[x-1][i] == 'Y' and mtx[j][y-1] == 'Y':
                    if h[i][j] == 110:
                        continue
                    now = 3 + h[i][j]
                    c = 0
                    while now > 2:
                        now -= now/3
                        c += 1
                    ret = min(c, ret)
        ans.append(str(ret) if ret != 110 else '-1')

    return " ".join(ans)

if __name__ == '__main__':
    t = input()
    for cas in xrange(1, t + 1):
        m, n, p = [int(x) for x in raw_input().split(" ")]
        mtx = []
        for i in range(m + n):
            mtx.append(raw_input())
        q = []
        for i in range(p):
            x, y = [int(x) for x in raw_input().split(" ")]
            q.append([x, y])
        ans = Process(m, n, mtx, q)
        print("Case #{}: {}".format(cas, ans))
```

#### 4. Irrefutable Outcome

回合制游戏，一人只能在字符串的左右两端取走'I', 一人只能取'O',不能取则输了。且假定每个人都一定做对自己最有力的决策。

思路：

递归，树，记忆化

code:

```python
l = 'IO'

def Process(s, c, dp):
    if s in dp[c]: return dp[c][s]
    if not s:
        return l[1-c], 1
    if s[0] != l[c] and s[-1] != l[c]:
        dp[c][s] = (l[1 - c], len(s)+1)
        return dp[c][s]
    elif s[0] == l[c] and s[-1] != l[c]:

        dp[c][s] = Process(s[1:], 1 - c, dp)
        return dp[c][s]
    elif s[0] != l[c] and s[-1] == l[c]:
        dp[c][s] = Process(s[:-1], 1 - c, dp)
        return dp[c][s]
    else:
        retl = Process(s[1:], 1-c, dp)
        retr = Process(s[:-1], 1-c, dp)
        if retl[0] == l[c] and retr[0] == l[c]:
            dp[c][s] = (l[c], max(retl[1], retr[1]))
            return dp[c][s]
        elif retl[0] != l[c] and retr[0] != l[c]:
            dp[c][s] = (l[1-c], min(retl[1], retr[1]))
            return dp[c][s]
        elif retl[0] == l[c]:
            dp[c][s] = retl
            return retl
        else:
            dp[c][s] = retr
            return retr

if __name__ == '__main__':
    t = input()
    for cas in xrange(1, t + 1):
        s = raw_input()
        dp = [{}, {}]
        ans = Process(s, 0, dp)
        print("Case #{}: {} {}".format(cas, ans[0], ans[1]))
```