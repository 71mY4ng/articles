# Dynamic Programming MIT notes

__Dp is just like "careful brute force"__

__DP is like subproblems + "reuse"__

__DP is like recursion + memorization__

***DP*** : 
- memoize ( remember )
- re-use solutions
- important to define subproblems that
- help solve the real problem

Examples

- memorization & subproblems
- fibonacci
- shortest paths
- guessing & DAG view

## Example : Fibonacci numbers

```
f_1 = f_2 = 1
f_n = f_n-1 + f_n-2
```

goal : compute f_n

### Naive Recursive algorithm

```
fib(n):
    if n <= 2: f = 1
    else: f = fib(n-1) + fib(n-2)
    return f
```

### Memorized dp algorithm

```
memo = {}
fib(n):
    if n in memo: return memo[n]
    if n <= 2: f = 1
    else: f = fib(n-1) + fib(n-2)
    memo[n] = f
    return f
```

相较于递归穷举，memo 记录了递归树中的重复子问题，这样在遇到已计算的重复子问题，就不需要再计算。

- Memorized calls cost constant time (θ(1))
- 


### bottom-up dp algorithm

```
fib = {}
for k in range(1,n + 1):
    if k <= 2: f = 1
    else: f = fib[k - 1] + fib[k - 2]
    fib[k] = f
return fib[n]
```


## Example: Shortest paths

