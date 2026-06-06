# Dynamic Programming — Complete Interview Preparation Guide

## Table of Contents

1. [What is Dynamic Programming?](#1-what-is-dynamic-programming)
2. [Two Approaches to DP](#2-two-approaches-to-dp)
3. [How to Identify DP Problems](#3-how-to-identify-dp-problems)
4. [General DP Problem-Solving Template](#4-general-dp-problem-solving-template)
5. [1D DP Problems](#5-1d-dp-problems)
6. [2D DP Problems](#6-2d-dp-problems)
7. [Interval DP](#7-interval-dp)
8. [String DP](#8-string-dp)
9. [DP on Trees](#9-dp-on-trees)
10. [DP Patterns Summary Table](#10-dp-patterns-summary-table)
11. [Space Optimization Techniques](#11-space-optimization-techniques)
12. [Interview Questions & Answers (30+)](#12-interview-questions--answers-30)
13. [Complexity Reference Sheet](#13-complexity-reference-sheet)
14. [Quick Revision Cheat Sheet](#14-quick-revision-cheat-sheet)

> **New to DP? Read this first.** Dynamic Programming sounds scary, but it is really just **brute-force recursion + remembering answers you already computed so you never solve the same subproblem twice.** That's the whole idea. If you can write a plain recursive solution, you are most of the way there — DP simply adds a notebook to store results. Don't try to swallow all 20+ problems at once. Focus first on these five classics, in this order: **Fibonacci**, **Climbing Stairs**, **House Robber**, **Coin Change**, and **Longest Common Subsequence (LCS)**. Once those patterns click, most other DP problems will start to look familiar.

---

## 1. What is Dynamic Programming?

Dynamic Programming (DP) is an algorithmic technique for solving problems by breaking them into **overlapping subproblems** and storing the results of already-solved subproblems to avoid redundant computation.

**Real-world analogy:**
- Imagine calculating your commute time. Instead of re-measuring every road segment each time, you write down segment times once and reuse them.
- DP is essentially **"remember what you've computed so you never compute it twice."**

> **Think of it like:** doing your math homework and writing down the answers to problems you've already solved. The next time the same little calculation shows up, you just copy your earlier answer instead of redoing all the work.

### Two Properties Required for DP

**1. Optimal Substructure**
The optimal solution to the problem can be constructed from optimal solutions to its subproblems.

Example: Shortest path from A to C through B — if A→B→C is optimal, then A→B must be optimal for the sub-path A to B.

> **Think of it like:** planning the cheapest road trip. The best route from your home to a far city is built out of the best routes between the cities along the way — good big answers are made of good smaller answers.

**2. Overlapping Subproblems**
The same subproblems recur multiple times during the computation. Without this, plain recursion or divide-and-conquer suffices — no memoization needed.

Example: Fibonacci(5) requires Fibonacci(3) twice. Computing it once and caching the result is DP.

> **Think of it like:** a question that keeps popping up over and over — "what's 7 times 8?" If your work asks it ten times, you solve it once and remember the answer; you don't recompute it each time.

### DP vs Divide and Conquer

| Aspect | Divide and Conquer | Dynamic Programming |
|--------|-------------------|---------------------|
| Subproblems | Non-overlapping (independent) | Overlapping (repeated) |
| Example | Merge Sort, Quick Sort | Fibonacci, Knapsack |
| Caching | Not needed | Essential |
| Direction | Top-down, no memoization needed | Top-down with memo OR bottom-up |

In Merge Sort, the left half and right half never overlap. In Fibonacci, the subproblems (fib(3), fib(2)) are called repeatedly — that is the key difference.

### DP vs Greedy

| Aspect | Greedy | Dynamic Programming |
|--------|--------|---------------------|
| Strategy | Take locally optimal choice at each step | Consider ALL choices, pick globally optimal |
| Completeness | May not find optimal solution | Always finds optimal solution (if applicable) |
| Speed | Usually faster (O(n log n) or O(n)) | Usually slower (O(n²) or O(n·W)) |
| Example | Activity Selection, Huffman Coding | 0/1 Knapsack, Edit Distance |

**Key insight:** Greedy works when the locally optimal choice leads to a globally optimal solution. DP works when it does not — you need to explore multiple branches.

Example: Coin change with coins [1, 3, 4] and target 6.
- Greedy: 4+1+1 = 3 coins
- DP: 3+3 = 2 coins (optimal)

---

## 2. Two Approaches to DP

### Top-Down (Memoization)

Write the natural recursive solution. Add a cache (HashMap or array) to store computed results. Before computing, check if the result is already in the cache.

> **Think of it like:** solving problems on demand and jotting each answer in a notebook as you go. You start from the big question, dive into whatever smaller questions you need, and write down each answer the first time so you never repeat it.

**When to prefer top-down:**
- Not all subproblems need to be solved (sparse computation)
- The recursion tree is naturally intuitive
- Subproblem space is large but only a fraction is visited
- Easier to implement from the recursive definition

**Java Template — Top-Down:**

```java
import java.util.HashMap;
import java.util.Map;

public class TopDownTemplate {
    private Map<Integer, Long> memo = new HashMap<>();

    public long solve(int n) {
        // Base cases
        if (n <= 1) return n;

        // Check cache
        if (memo.containsKey(n)) return memo.get(n);

        // Compute and store
        long result = solve(n - 1) + solve(n - 2); // example recurrence
        memo.put(n, result);
        return result;
    }
}
```

**Top-Down with array memo (faster than HashMap):**

```java
public class TopDownArrayTemplate {
    private long[] memo;

    public long solve(int n) {
        memo = new long[n + 1];
        Arrays.fill(memo, -1); // -1 = not computed
        return dp(n);
    }

    private long dp(int n) {
        if (n <= 1) return n;
        if (memo[n] != -1) return memo[n];
        memo[n] = dp(n - 1) + dp(n - 2);
        return memo[n];
    }
}
```

### Bottom-Up (Tabulation)

Build the solution iteratively from the smallest subproblems upward. Fill a table (array or 2D array) starting from base cases.

> **Think of it like:** building a pyramid from the ground up. You lay the smallest base cases first, then stack each new answer on top of the ones below it, until you reach the final answer at the peak.

**When to prefer bottom-up:**
- All subproblems need to be solved
- Want to avoid recursion stack overflow (large n)
- Need to optimize space (easier to see which rows/columns to discard)
- Slightly faster in practice (no function call overhead)

**Java Template — Bottom-Up:**

```java
public class BottomUpTemplate {
    public long solve(int n) {
        if (n <= 1) return n;

        long[] dp = new long[n + 1];

        // Base cases
        dp[0] = 0;
        dp[1] = 1;

        // Fill table in order of increasing subproblem size
        for (int i = 2; i <= n; i++) {
            dp[i] = dp[i - 1] + dp[i - 2]; // example recurrence
        }

        return dp[n];
    }
}
```

### Converting Top-Down to Bottom-Up — Step-by-Step

Given a top-down solution, follow these steps:

1. **Identify the state variables** — what parameters change in recursive calls? Those become the dp array indices.
2. **Determine the table dimensions** — one dimension per state variable.
3. **Translate base cases** — recursive base cases become initial dp table values.
4. **Reverse the recurrence** — in top-down, you compute dp[n] from dp[n-1]. In bottom-up, start from dp[0] and go up to dp[n].
5. **Determine iteration order** — ensure that when computing dp[i][j], all values it depends on are already computed.
6. **Return the same value** — top-down returns memo[n], bottom-up returns dp[n].

**Example conversion — Fibonacci:**

```java
// Top-Down
long fibMemo(int n, long[] memo) {
    if (n <= 1) return n;
    if (memo[n] != -1) return memo[n];
    memo[n] = fibMemo(n-1, memo) + fibMemo(n-2, memo);
    return memo[n];
}

// Bottom-Up (direct conversion)
long fibTab(int n) {
    if (n <= 1) return n;
    long[] dp = new long[n + 1];
    dp[0] = 0; dp[1] = 1;          // base cases from recursive base cases
    for (int i = 2; i <= n; i++)   // iterate in increasing order
        dp[i] = dp[i-1] + dp[i-2]; // same recurrence, no memoization needed
    return dp[n];
}
```

### Space Optimization: Reducing 2D DP to 1D DP

When dp[i][j] only depends on dp[i-1][...] (the previous row), you can use a single 1D array and overwrite it row by row.

**Rule:** If the current row only needs the previous row, keep only two arrays (or one with careful traversal direction).

**2D version:**
```java
for (int i = 1; i <= n; i++)
    for (int j = 0; j <= W; j++)
        dp[i][j] = Math.max(dp[i-1][j], dp[i-1][j-w[i]] + v[i]);
```

**1D version (traverse j in reverse to avoid using updated values):**
```java
for (int i = 0; i < n; i++)
    for (int j = W; j >= w[i]; j--)  // REVERSE to use old dp[j-w[i]]
        dp[j] = Math.max(dp[j], dp[j - w[i]] + v[i]);
```

---

## 3. How to Identify DP Problems

### Keywords That Signal DP

| Keyword | Example Problem |
|---------|----------------|
| "minimum / maximum" | Minimum coins, Maximum profit |
| "count the number of ways" | Climbing stairs, Coin change ways |
| "is it possible / can you reach" | Jump game, Subset sum |
| "longest / shortest" | LCS, Edit distance |
| "partition / split" | Palindrome partitioning |
| "all combinations / arrangements" | Combination sum |

### DP Problem Identification Checklist

Ask yourself these questions:
1. Can the problem be broken into smaller versions of itself? (recursive structure)
2. Do the smaller versions overlap? (same subproblem called multiple times)
3. Does an optimal solution to the whole problem depend on optimal solutions to subproblems?
4. Does the problem ask for a count, min, max, or yes/no over a combinatorial space?

If you answer YES to all 4, it is almost certainly a DP problem.

### What is NOT DP

- Sorting problems (no overlapping subproblems)
- Graph traversal (BFS/DFS — unless computing shortest path with DP like Bellman-Ford)
- Pure greedy problems (activity selection)
- Binary search problems

---

## 4. General DP Problem-Solving Template

Follow these 5 steps for every DP problem:

```
Step 1: Define the state
  dp[i] means: [describe what this value represents]
  Example: dp[i] = minimum coins needed to make amount i

Step 2: Write the recurrence relation
  dp[i] = f(dp[i-1], dp[i-2], ...)
  Example: dp[i] = min over all coins c: dp[i-c] + 1

Step 3: Identify base cases
  dp[0] = 0 (base: 0 coins for amount 0)

Step 4: Determine iteration order
  For most 1D DP: left to right (i = 1 to n)
  For 0/1 knapsack: right to left when space-optimized

Step 5: Extract the answer
  Return dp[n], dp[target], dp[m][n], etc.
```

---

## 5. 1D DP Problems

---

### Problem 1: Fibonacci Sequence

**Definition:** F(n) = F(n-1) + F(n-2), F(0)=0, F(1)=1

> **Think of it like:** each number is just the sum of the two numbers before it — like a rabbit family where this month's babies equal last month's plus the month before's. It's the "hello world" of DP because the same small Fibonacci values get asked for again and again.

#### Approach 1 — Naive Recursion: O(2^n) time, O(n) space

```java
public long fibNaive(int n) {
    if (n <= 1) return n;
    return fibNaive(n - 1) + fibNaive(n - 2);
}
// Each call spawns 2 more calls. Exponential time.
// fib(5) computes fib(3) twice, fib(2) three times, etc.
```

#### Approach 2 — Memoization: O(n) time, O(n) space

```java
public long fibMemo(int n) {
    long[] memo = new long[n + 1];   // notebook: memo[i] will hold F(i) once we compute it
    Arrays.fill(memo, -1);           // -1 means "not computed yet"
    return fibHelper(n, memo);
}

private long fibHelper(int n, long[] memo) {
    if (n <= 1) return n;            // base cases: F(0)=0, F(1)=1
    if (memo[n] != -1) return memo[n]; // already solved this? return the saved answer
    // not solved yet: compute it from the two smaller Fibonacci numbers...
    memo[n] = fibHelper(n - 1, memo) + fibHelper(n - 2, memo);
    return memo[n];                 // ...and save it in the notebook before returning
}
// Each subproblem computed exactly once.
```

#### Approach 3 — Tabulation: O(n) time, O(n) space

```java
public long fibTab(int n) {
    if (n <= 1) return n;
    long[] dp = new long[n + 1];     // dp[i] = the i-th Fibonacci number
    dp[0] = 0;                       // base case
    dp[1] = 1;                       // base case
    // build upward: every value uses the two we already filled in below it
    for (int i = 2; i <= n; i++) {
        dp[i] = dp[i - 1] + dp[i - 2];
    }
    return dp[n];                    // the answer sits at the top of the table
}
```

#### Approach 4 — Space Optimized: O(n) time, O(1) space

```java
public long fibOptimal(int n) {
    if (n <= 1) return n;
    long prev2 = 0, prev1 = 1;
    for (int i = 2; i <= n; i++) {
        long curr = prev1 + prev2;
        prev2 = prev1;
        prev1 = curr;
    }
    return prev1;
}
// Only keep the last two values — no array needed.
```

**Complexity Summary:**

| Approach | Time | Space |
|----------|------|-------|
| Naive recursion | O(2^n) | O(n) stack |
| Memoization | O(n) | O(n) |
| Tabulation | O(n) | O(n) |
| Space optimized | O(n) | O(1) |

---

### Problem 2: Climbing Stairs

**Problem:** You are climbing a staircase with n steps. You can climb 1 or 2 steps at a time. How many distinct ways can you reach the top?

> **Think of it like:** standing on any stair and looking back. You could only have arrived here from one step below (a 1-step hop) or two steps below (a 2-step hop), so the ways to reach this stair are just the ways to reach those two stairs added together.

**State definition:** dp[i] = number of distinct ways to reach step i

**Recurrence:** dp[i] = dp[i-1] + dp[i-2]
- To reach step i: come from step i-1 (take 1 step) OR from step i-2 (take 2 steps)

**Base cases:** dp[0] = 1 (one way to stay at ground), dp[1] = 1

```java
// Bottom-up tabulation
public int climbStairs(int n) {
    if (n <= 2) return n;
    int[] dp = new int[n + 1];       // dp[i] = number of ways to reach step i
    dp[0] = 1;                       // 1 way to be at the ground (do nothing)
    dp[1] = 1;                       // 1 way to reach the first step
    // ways to reach step i = ways arriving from i-1 (1-step) + from i-2 (2-step)
    for (int i = 2; i <= n; i++) {
        dp[i] = dp[i - 1] + dp[i - 2];
    }
    return dp[n];                    // ways to reach the top step
}
// Time: O(n), Space: O(n)

// Space optimized
public int climbStairsOptimal(int n) {
    if (n <= 2) return n;
    int prev2 = 1, prev1 = 1;
    for (int i = 2; i <= n; i++) {
        int curr = prev1 + prev2;
        prev2 = prev1;
        prev1 = curr;
    }
    return prev1;
}
// Time: O(n), Space: O(1)
```

**Variant: Climbing stairs with k steps**

You can climb 1, 2, ..., k steps at a time.

```java
public int climbStairsK(int n, int k) {
    int[] dp = new int[n + 1];
    dp[0] = 1;
    for (int i = 1; i <= n; i++) {
        for (int j = 1; j <= k && j <= i; j++) {
            dp[i] += dp[i - j];
        }
    }
    return dp[n];
}
// Time: O(n*k), Space: O(n)
```

**Note:** Climbing stairs is structurally identical to Fibonacci (with different base cases). This is a very common interview entry question.

---

### Problem 3: House Robber

**Problem:** Given an array of non-negative integers representing money in houses, find the maximum amount you can rob. You cannot rob two adjacent houses (alarm triggers).

> **Think of it like:** a burglar walking down a street of houses where robbing two next-door houses sets off the alarm. At each house you make one decision: skip it (keep what you had) or rob it (take its cash plus whatever you'd grabbed up to two houses back).

**State definition:** dp[i] = maximum money robbed considering first i houses

**Recurrence:** dp[i] = max(dp[i-1], dp[i-2] + nums[i])
- Either skip house i (take dp[i-1]) or rob house i (take dp[i-2] + nums[i])

**Base cases:** dp[0] = nums[0], dp[1] = max(nums[0], nums[1])

```java
public int rob(int[] nums) {
    int n = nums.length;
    if (n == 1) return nums[0];      // only one house: just rob it

    int[] dp = new int[n];           // dp[i] = most money robbable from houses 0..i
    dp[0] = nums[0];                 // one house: rob it
    dp[1] = Math.max(nums[0], nums[1]); // two houses: rob the richer one (can't take both)

    for (int i = 2; i < n; i++) {
        // skip house i (keep dp[i-1]) OR rob it (its cash + best up to i-2)
        dp[i] = Math.max(dp[i - 1], dp[i - 2] + nums[i]);
    }
    return dp[n - 1];                // best total after considering every house
}
// Time: O(n), Space: O(n)

// Space optimized
public int robOptimal(int[] nums) {
    int n = nums.length;
    if (n == 1) return nums[0];
    int prev2 = nums[0];
    int prev1 = Math.max(nums[0], nums[1]);
    for (int i = 2; i < n; i++) {
        int curr = Math.max(prev1, prev2 + nums[i]);
        prev2 = prev1;
        prev1 = curr;
    }
    return prev1;
}
// Time: O(n), Space: O(1)
```

**Variant: House Robber II (Circular arrangement)**

Houses are in a circle — first and last are adjacent. Key insight: either the first house is robbed (exclude last) or it is not (exclude first). Run house robber on both subarrays and take the max.

```java
public int robII(int[] nums) {
    int n = nums.length;
    if (n == 1) return nums[0];
    if (n == 2) return Math.max(nums[0], nums[1]);
    // Either rob houses 0..n-2 OR houses 1..n-1
    return Math.max(
        robRange(nums, 0, n - 2),
        robRange(nums, 1, n - 1)
    );
}

private int robRange(int[] nums, int start, int end) {
    int prev2 = 0, prev1 = 0;
    for (int i = start; i <= end; i++) {
        int curr = Math.max(prev1, prev2 + nums[i]);
        prev2 = prev1;
        prev1 = curr;
    }
    return prev1;
}
// Time: O(n), Space: O(1)
```

---

### Problem 4: Coin Change — Minimum Coins

**Problem:** Given coin denominations and a target amount, find the minimum number of coins needed to make up that amount. Return -1 if impossible.

> **Think of it like:** a cashier making change with as few coins as possible. To make 6 cents, you try each coin you own, see how few coins it took to make the leftover amount, and pick whichever choice uses the fewest coins overall.

**State definition:** dp[i] = minimum number of coins to make amount i

**Recurrence:** dp[i] = min over all coins c where c <= i: (dp[i - c] + 1)
- For each coin, try using it and see if we get a better answer

**Base cases:** dp[0] = 0 (zero coins for amount zero)

**Initialization:** dp[i] = Integer.MAX_VALUE (infinity — not yet achievable)

**Full Java Solution with Trace:**

```java
public int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];  // dp[i] = fewest coins to make amount i
    Arrays.fill(dp, amount + 1); // fill with "infinity" (amount+1 is impossible)
    dp[0] = 0;                       // 0 coins needed to make amount 0

    for (int i = 1; i <= amount; i++) {       // solve every amount from 1 up to the target
        for (int coin : coins) {              // try paying with each coin we have
            // coin must fit, and the leftover amount must actually be makeable
            if (coin <= i && dp[i - coin] != amount + 1) {
                // using this coin = 1 coin + best way to make the remainder; keep the smallest
                dp[i] = Math.min(dp[i], dp[i - coin] + 1);
            }
        }
    }
    // still "infinity"? then the amount can't be made → return -1
    return dp[amount] > amount ? -1 : dp[amount];
}
// Time: O(amount * coins.length), Space: O(amount)
```

**Trace Example:** coins = [1, 3, 4], amount = 6

```
dp[0] = 0
dp[1]: coin=1: dp[0]+1=1 → dp[1]=1
dp[2]: coin=1: dp[1]+1=2 → dp[2]=2
dp[3]: coin=1: dp[2]+1=3; coin=3: dp[0]+1=1 → dp[3]=1
dp[4]: coin=1: dp[3]+1=2; coin=3: dp[1]+1=2; coin=4: dp[0]+1=1 → dp[4]=1
dp[5]: coin=1: dp[4]+1=2; coin=3: dp[2]+1=3; coin=4: dp[1]+1=2 → dp[5]=2
dp[6]: coin=1: dp[5]+1=3; coin=3: dp[3]+1=2; coin=4: dp[2]+1=3 → dp[6]=2

Answer: 2 (using 3+3)
```

**Why initialize with amount+1 and not Integer.MAX_VALUE?**
Integer.MAX_VALUE + 1 overflows. Using amount+1 is safe — it is always larger than any valid answer (max valid answer is `amount` using all 1-coins).

---

### Problem 5: Coin Change — Number of Ways (Combination Sum IV)

**Problem:** Count the number of distinct combinations (order matters) of coins that sum to the target.

> **Think of it like:** counting how many different ways a cashier could hand you the same change — not the fewest coins this time, but every possible coin combination that adds up to the amount.

**State definition:** dp[i] = number of ways to make amount i

**Recurrence:** dp[i] += dp[i - coin] for each coin where coin <= i

**Base case:** dp[0] = 1 (one way to make amount 0: use no coins)

```java
// Order matters (like Combination Sum IV / Climbing Stairs)
public int countWaysOrdered(int[] coins, int target) {
    int[] dp = new int[target + 1];
    dp[0] = 1;
    for (int i = 1; i <= target; i++) {
        for (int coin : coins) {
            if (coin <= i) {
                dp[i] += dp[i - coin];
            }
        }
    }
    return dp[target];
}
// Time: O(target * coins.length), Space: O(target)
// [1,2] for target=3: {1,1,1}, {1,2}, {2,1} = 3 ways (order matters)

// Order does NOT matter (classic coin change ways / unbounded knapsack)
public int countWaysUnordered(int[] coins, int target) {
    int[] dp = new int[target + 1];
    dp[0] = 1;
    for (int coin : coins) {          // outer loop: coins
        for (int i = coin; i <= target; i++) { // inner loop: amounts
            dp[i] += dp[i - coin];
        }
    }
    return dp[target];
}
// Time: O(coins.length * target), Space: O(target)
// [1,2] for target=3: {1,1,1}, {1,2} = 2 ways (order doesn't matter)
```

**Key difference between the two:**
- Order matters: outer loop over amounts, inner loop over coins
- Order does NOT matter: outer loop over coins, inner loop over amounts

---

### Problem 6: Word Break

**Problem:** Given a string s and a dictionary of words, determine if s can be segmented into a space-separated sequence of dictionary words.

> **Think of it like:** reading a sign with no spaces, such as "applepie", and checking if you can chop it into real words from your dictionary ("apple" + "pie"). You go left to right asking, "does a valid word end right here, with valid words before it?"

**State definition:** dp[i] = true if s[0..i-1] can be segmented using dictionary words

**Recurrence:** dp[i] = true if there exists j < i such that dp[j] == true AND s[j..i-1] is in the dictionary

**Base case:** dp[0] = true (empty string is always segmentable)

```java
public boolean wordBreak(String s, List<String> wordDict) {
    Set<String> wordSet = new HashSet<>(wordDict);
    int n = s.length();
    boolean[] dp = new boolean[n + 1];
    dp[0] = true; // base case: empty prefix

    for (int i = 1; i <= n; i++) {
        for (int j = 0; j < i; j++) {
            if (dp[j] && wordSet.contains(s.substring(j, i))) {
                dp[i] = true;
                break; // no need to check further for this i
            }
        }
    }
    return dp[n];
}
// Time: O(n^2) — n^2 substrings, each check O(1) with HashSet
// Space: O(n + dict size)
```

**Trace:** s = "leetcode", dict = ["leet", "code"]
```
dp[0] = true
dp[1]: j=0, "l" not in dict → false
dp[2]: j=0, "le" not in dict → false
dp[3]: j=0, "lee" not in dict → false
dp[4]: j=0, dp[0]=true AND "leet" in dict → dp[4] = true
dp[5..7]: no valid split → false
dp[8]: j=4, dp[4]=true AND "code" in dict → dp[8] = true
Answer: true
```

**Variant: Return all possible segmentations**

```java
public List<String> wordBreakAll(String s, List<String> wordDict) {
    Set<String> wordSet = new HashSet<>(wordDict);
    Map<Integer, List<String>> memo = new HashMap<>();
    return helper(s, wordSet, 0, memo);
}

private List<String> helper(String s, Set<String> wordSet, int start,
                             Map<Integer, List<String>> memo) {
    if (memo.containsKey(start)) return memo.get(start);
    List<String> result = new ArrayList<>();
    if (start == s.length()) {
        result.add("");
        return result;
    }
    for (int end = start + 1; end <= s.length(); end++) {
        String word = s.substring(start, end);
        if (wordSet.contains(word)) {
            List<String> rest = helper(s, wordSet, end, memo);
            for (String r : rest) {
                result.add(word + (r.isEmpty() ? "" : " " + r));
            }
        }
    }
    memo.put(start, result);
    return result;
}
// Time: O(n^2 * 2^n) worst case, Space: O(n * 2^n) for all sentences
```

---

### Problem 7: Jump Game

**Problem:** Given an array nums where nums[i] is the max jump length from index i, determine if you can reach the last index.

**Greedy approach (optimal):**

```java
public boolean canJump(int[] nums) {
    int maxReach = 0;
    for (int i = 0; i < nums.length; i++) {
        if (i > maxReach) return false; // can't reach i
        maxReach = Math.max(maxReach, i + nums[i]);
    }
    return true;
}
// Time: O(n), Space: O(1)
```

**DP approach:**

```java
public boolean canJumpDP(int[] nums) {
    int n = nums.length;
    boolean[] dp = new boolean[n];
    dp[0] = true;
    for (int i = 1; i < n; i++) {
        for (int j = 0; j < i; j++) {
            if (dp[j] && j + nums[j] >= i) {
                dp[i] = true;
                break;
            }
        }
    }
    return dp[n - 1];
}
// Time: O(n^2), Space: O(n)
// Greedy is preferred — DP shown for completeness
```

---

### Problem 8: Jump Game II — Minimum Jumps

**Problem:** Find the minimum number of jumps to reach the last index. Assume you can always reach.

**Greedy (optimal):**

```java
public int jump(int[] nums) {
    int jumps = 0, currentEnd = 0, farthest = 0;
    for (int i = 0; i < nums.length - 1; i++) {
        farthest = Math.max(farthest, i + nums[i]);
        if (i == currentEnd) {
            jumps++;
            currentEnd = farthest;
        }
    }
    return jumps;
}
// Time: O(n), Space: O(1)
```

**DP approach:**

```java
public int jumpDP(int[] nums) {
    int n = nums.length;
    int[] dp = new int[n];
    Arrays.fill(dp, Integer.MAX_VALUE);
    dp[0] = 0;
    for (int i = 1; i < n; i++) {
        for (int j = 0; j < i; j++) {
            if (dp[j] != Integer.MAX_VALUE && j + nums[j] >= i) {
                dp[i] = Math.min(dp[i], dp[j] + 1);
            }
        }
    }
    return dp[n - 1];
}
// Time: O(n^2), Space: O(n)
```

---

### Problem 9: Decode Ways

**Problem:** A message containing letters A-Z is encoded as 1-26. Given a digit string, count the number of ways to decode it.

**State definition:** dp[i] = number of ways to decode s[0..i-1]

**Recurrence:**
- If s[i-1] != '0': dp[i] += dp[i-1] (single digit decode)
- If s[i-2..i-1] forms a valid two-digit number (10-26): dp[i] += dp[i-2]

**Base cases:** dp[0] = 1, dp[1] = (s[0] != '0') ? 1 : 0

```java
public int numDecodings(String s) {
    int n = s.length();
    if (n == 0 || s.charAt(0) == '0') return 0;

    int[] dp = new int[n + 1];
    dp[0] = 1; // empty string: one way
    dp[1] = s.charAt(0) != '0' ? 1 : 0;

    for (int i = 2; i <= n; i++) {
        // Single digit decode
        int oneDigit = s.charAt(i - 1) - '0';
        if (oneDigit != 0) {
            dp[i] += dp[i - 1];
        }

        // Two digit decode
        int twoDigit = Integer.parseInt(s.substring(i - 2, i));
        if (twoDigit >= 10 && twoDigit <= 26) {
            dp[i] += dp[i - 2];
        }
    }
    return dp[n];
}
// Time: O(n), Space: O(n)
```

**Trace:** s = "226"
```
dp[0]=1, dp[1]=1 (s[0]='2' != '0')
i=2: oneDigit=2 (!=0) → dp[2]+=dp[1]=1; twoDigit=22 (10-26) → dp[2]+=dp[0]=1 → dp[2]=2
i=3: oneDigit=6 (!=0) → dp[3]+=dp[2]=2; twoDigit=26 (10-26) → dp[3]+=dp[1]=1 → dp[3]=3
Answer: 3 → "226"→{2,2,6},{22,6},{2,26}
```

---

## 6. 2D DP Problems

---

### Problem 10: Unique Paths

**Problem:** A robot starts at top-left of an m×n grid and wants to reach bottom-right. It can only move right or down. Count all unique paths.

> **Think of it like:** walking through a city grid where you can only go right or down. To count the ways to reach any corner, add up the ways to reach the corner just above it and the corner just to its left — those are the only two spots you could have stepped from.

**State definition:** dp[i][j] = number of unique paths to reach cell (i, j)

**Recurrence:** dp[i][j] = dp[i-1][j] + dp[i][j-1]
- Can only arrive from above or from the left

**Base cases:** dp[0][j] = 1 for all j (top row), dp[i][0] = 1 for all i (left column)

```java
public int uniquePaths(int m, int n) {
    int[][] dp = new int[m][n];

    // Base cases: first row and first column
    for (int i = 0; i < m; i++) dp[i][0] = 1;
    for (int j = 0; j < n; j++) dp[0][j] = 1;

    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            dp[i][j] = dp[i - 1][j] + dp[i][j - 1];
        }
    }
    return dp[m - 1][n - 1];
}
// Time: O(m*n), Space: O(m*n)

// Space optimized to O(n)
public int uniquePathsOptimal(int m, int n) {
    int[] dp = new int[n];
    Arrays.fill(dp, 1); // first row = all 1s
    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            dp[j] += dp[j - 1]; // dp[j] was prev row, dp[j-1] is same row
        }
    }
    return dp[n - 1];
}
// Time: O(m*n), Space: O(n)
```

**Variant: Unique Paths II (with obstacles)**

```java
public int uniquePathsWithObstacles(int[][] obstacleGrid) {
    int m = obstacleGrid.length, n = obstacleGrid[0].length;
    if (obstacleGrid[0][0] == 1 || obstacleGrid[m-1][n-1] == 1) return 0;

    int[][] dp = new int[m][n];
    // Fill first column
    for (int i = 0; i < m && obstacleGrid[i][0] == 0; i++) dp[i][0] = 1;
    // Fill first row
    for (int j = 0; j < n && obstacleGrid[0][j] == 0; j++) dp[0][j] = 1;

    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            if (obstacleGrid[i][j] == 0) {
                dp[i][j] = dp[i - 1][j] + dp[i][j - 1];
            }
            // if obstacle, dp[i][j] stays 0
        }
    }
    return dp[m - 1][n - 1];
}
// Time: O(m*n), Space: O(m*n)
```

---

### Problem 11: Minimum Path Sum

**Problem:** Given an m×n grid of non-negative integers, find the path from top-left to bottom-right that minimizes the sum. You can only move right or down.

**State definition:** dp[i][j] = minimum path sum to reach (i, j)

**Recurrence:** dp[i][j] = grid[i][j] + min(dp[i-1][j], dp[i][j-1])

```java
public int minPathSum(int[][] grid) {
    int m = grid.length, n = grid[0].length;
    int[][] dp = new int[m][n];

    dp[0][0] = grid[0][0];
    // Fill first column
    for (int i = 1; i < m; i++) dp[i][0] = dp[i-1][0] + grid[i][0];
    // Fill first row
    for (int j = 1; j < n; j++) dp[0][j] = dp[0][j-1] + grid[0][j];

    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            dp[i][j] = grid[i][j] + Math.min(dp[i-1][j], dp[i][j-1]);
        }
    }
    return dp[m-1][n-1];
}
// Time: O(m*n), Space: O(m*n)

// Space optimized — modify grid in place (or use 1D array)
public int minPathSumOptimal(int[][] grid) {
    int m = grid.length, n = grid[0].length;
    int[] dp = new int[n];

    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (i == 0 && j == 0) dp[j] = grid[0][0];
            else if (i == 0) dp[j] = dp[j-1] + grid[i][j];
            else if (j == 0) dp[j] = dp[j] + grid[i][j];  // dp[j] is from prev row
            else dp[j] = grid[i][j] + Math.min(dp[j], dp[j-1]);
        }
    }
    return dp[n-1];
}
// Time: O(m*n), Space: O(n)
```

---

### Problem 12: Longest Common Subsequence (LCS)

**Definition:**
- **Subsequence:** Characters in order but not necessarily contiguous. "ACE" is a subsequence of "ABCDE".
- **Substring:** Characters in order AND contiguous. "BCD" is a substring of "ABCDE".
- LCS finds the longest subsequence common to both strings.

> **Think of it like:** two friends listing the movies they each watched this year, in the order they saw them. The LCS is the longest list of movies both saw in the same relative order — they don't have to be back-to-back, just in order.

**State definition:** dp[i][j] = length of LCS of s1[0..i-1] and s2[0..j-1]

**Recurrence:**
- If s1[i-1] == s2[j-1]: dp[i][j] = dp[i-1][j-1] + 1 (characters match, extend)
- Else: dp[i][j] = max(dp[i-1][j], dp[i][j-1]) (skip one character from either string)

**Base cases:** dp[0][j] = 0, dp[i][0] = 0 (LCS with empty string is 0)

```java
public int lcs(String s1, String s2) {
    int m = s1.length(), n = s2.length();
    // dp[i][j] = LCS length using first i chars of s1 and first j chars of s2.
    // Row/col 0 stay 0 (an empty string shares nothing) — these are the base cases.
    int[][] dp = new int[m + 1][n + 1];

    for (int i = 1; i <= m; i++) {           // walk through every char of s1
        for (int j = 1; j <= n; j++) {       // against every char of s2
            if (s1.charAt(i - 1) == s2.charAt(j - 1)) {
                // chars match: extend the LCS found before both of these chars
                dp[i][j] = dp[i - 1][j - 1] + 1;
            } else {
                // no match: drop one char from s1 or from s2, keep the better result
                dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
            }
        }
    }
    return dp[m][n];                         // bottom-right cell = LCS of the full strings
}
// Time: O(m*n), Space: O(m*n)
```

**Full Table Trace:** s1 = "ABCBDAB", s2 = "BDCAB"

```
    ""  B  D  C  A  B
""   0  0  0  0  0  0
A    0  0  0  0  1  1
B    0  1  1  1  1  2
C    0  1  1  2  2  2
B    0  1  1  2  2  3
D    0  1  2  2  2  3
A    0  1  2  2  3  3
B    0  1  2  2  3  4

LCS length = 4 ("BCAB" or "BDAB")
```

**Reconstruct the actual LCS:**

```java
public String reconstructLCS(String s1, String s2) {
    int m = s1.length(), n = s2.length();
    int[][] dp = new int[m + 1][n + 1];

    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= n; j++)
            if (s1.charAt(i-1) == s2.charAt(j-1))
                dp[i][j] = dp[i-1][j-1] + 1;
            else
                dp[i][j] = Math.max(dp[i-1][j], dp[i][j-1]);

    // Backtrack from dp[m][n]
    StringBuilder sb = new StringBuilder();
    int i = m, j = n;
    while (i > 0 && j > 0) {
        if (s1.charAt(i-1) == s2.charAt(j-1)) {
            sb.append(s1.charAt(i-1));
            i--; j--;
        } else if (dp[i-1][j] > dp[i][j-1]) {
            i--;
        } else {
            j--;
        }
    }
    return sb.reverse().toString();
}
```

**Variant: Shortest Common Supersequence (SCS)**

The shortest string that has both s1 and s2 as subsequences.
Length of SCS = m + n - LCS(s1, s2)

```java
public String shortestCommonSupersequence(String s1, String s2) {
    int m = s1.length(), n = s2.length();
    int[][] dp = new int[m+1][n+1];
    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= n; j++)
            dp[i][j] = s1.charAt(i-1) == s2.charAt(j-1)
                ? dp[i-1][j-1] + 1
                : Math.max(dp[i-1][j], dp[i][j-1]);

    // Reconstruct SCS by backtracking
    StringBuilder sb = new StringBuilder();
    int i = m, j = n;
    while (i > 0 && j > 0) {
        if (s1.charAt(i-1) == s2.charAt(j-1)) {
            sb.append(s1.charAt(i-1));
            i--; j--;
        } else if (dp[i-1][j] > dp[i][j-1]) {
            sb.append(s1.charAt(i-1));
            i--;
        } else {
            sb.append(s2.charAt(j-1));
            j--;
        }
    }
    while (i > 0) sb.append(s1.charAt(i-- - 1));
    while (j > 0) sb.append(s2.charAt(j-- - 1));
    return sb.reverse().toString();
}
// Time: O(m*n), Space: O(m*n)
```

---

### Problem 13: Edit Distance (Levenshtein Distance)

**Problem:** Given two strings word1 and word2, find the minimum number of operations (insert, delete, replace) to transform word1 into word2.

> **Think of it like:** the autocorrect on your phone deciding how close two words are. It counts the fewest single-letter edits — add a letter, remove a letter, or swap a letter — needed to turn one word into the other.

**State definition:** dp[i][j] = minimum operations to transform word1[0..i-1] into word2[0..j-1]

**Recurrence:**
- If word1[i-1] == word2[j-1]: dp[i][j] = dp[i-1][j-1] (no operation needed)
- Else: dp[i][j] = 1 + min(
    dp[i-1][j],    // delete from word1
    dp[i][j-1],    // insert into word1 (= delete from word2)
    dp[i-1][j-1]   // replace
  )

**Base cases:**
- dp[i][0] = i (delete all i characters from word1)
- dp[0][j] = j (insert all j characters)

```java
public int minDistance(String word1, String word2) {
    int m = word1.length(), n = word2.length();
    int[][] dp = new int[m + 1][n + 1];

    // Base cases
    for (int i = 0; i <= m; i++) dp[i][0] = i;
    for (int j = 0; j <= n; j++) dp[0][j] = j;

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (word1.charAt(i - 1) == word2.charAt(j - 1)) {
                dp[i][j] = dp[i - 1][j - 1]; // no operation
            } else {
                dp[i][j] = 1 + Math.min(
                    dp[i - 1][j],      // delete
                    Math.min(
                        dp[i][j - 1],  // insert
                        dp[i - 1][j - 1] // replace
                    )
                );
            }
        }
    }
    return dp[m][n];
}
// Time: O(m*n), Space: O(m*n)
```

**Full Table Trace:** word1 = "horse", word2 = "ros"

```
    ""  r  o  s
""   0  1  2  3
h    1  1  2  3
o    2  2  1  2
r    3  2  2  2
s    4  3  3  2
e    5  4  4  3

Answer: 3 operations
  horse → rorse (replace 'h' with 'r')
  rorse → rose  (delete 'r')
  rose  → ros   (delete 'e')
```

**Applications:**
- Spell checkers (find closest dictionary word)
- DNA sequence alignment
- Git diff algorithms
- Plagiarism detection

**Space Optimized:**

```java
public int minDistanceOptimal(String word1, String word2) {
    int m = word1.length(), n = word2.length();
    int[] dp = new int[n + 1];
    for (int j = 0; j <= n; j++) dp[j] = j;

    for (int i = 1; i <= m; i++) {
        int prev = dp[0];
        dp[0] = i;
        for (int j = 1; j <= n; j++) {
            int temp = dp[j];
            if (word1.charAt(i-1) == word2.charAt(j-1)) {
                dp[j] = prev;
            } else {
                dp[j] = 1 + Math.min(prev, Math.min(dp[j], dp[j-1]));
            }
            prev = temp;
        }
    }
    return dp[n];
}
// Time: O(m*n), Space: O(n)
```

---

### Problem 14: Longest Common Substring

**Problem:** Find the length of the longest contiguous substring common to both strings.

**Difference from LCS:** Substring must be contiguous. When characters don't match, reset to 0 (unlike LCS which takes max of skipping).

**State definition:** dp[i][j] = length of longest common substring ending at s1[i-1] and s2[j-1]

**Recurrence:**
- If s1[i-1] == s2[j-1]: dp[i][j] = dp[i-1][j-1] + 1
- Else: dp[i][j] = 0

```java
public int longestCommonSubstring(String s1, String s2) {
    int m = s1.length(), n = s2.length();
    int[][] dp = new int[m + 1][n + 1];
    int maxLen = 0;

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (s1.charAt(i - 1) == s2.charAt(j - 1)) {
                dp[i][j] = dp[i - 1][j - 1] + 1;
                maxLen = Math.max(maxLen, dp[i][j]);
            } else {
                dp[i][j] = 0; // KEY difference from LCS: reset, don't take max
            }
        }
    }
    return maxLen;
}
// Time: O(m*n), Space: O(m*n)
```

**LCS vs Longest Common Substring Summary:**

| Aspect | LCS | Longest Common Substring |
|--------|-----|--------------------------|
| Contiguous? | No | Yes |
| Mismatch case | dp[i][j] = max(dp[i-1][j], dp[i][j-1]) | dp[i][j] = 0 |
| Track | dp[m][n] | maxLen across all cells |

---

### Problem 15: 0/1 Knapsack

**Problem:** Given n items, each with weight w[i] and value v[i], and a knapsack of capacity W, maximize the total value. Each item can be taken at most once (0 or 1 times).

> **Think of it like:** packing a backpack with a weight limit before a trip. Each item weighs something and is worth something, and you want the most valuable load that still fits. For every item you face one choice: leave it out or put it in (if there's room).

**State definition:** dp[i][w] = maximum value using first i items with knapsack capacity w

**Recurrence:**
- Exclude item i: dp[i][w] = dp[i-1][w]
- Include item i (if w[i] <= w): dp[i][w] = dp[i-1][w - w[i]] + v[i]
- dp[i][w] = max of the two

**Base cases:** dp[0][w] = 0 (no items → no value), dp[i][0] = 0 (no capacity → no value)

```java
public int knapsack(int[] weights, int[] values, int W) {
    int n = weights.length;
    int[][] dp = new int[n + 1][W + 1];

    for (int i = 1; i <= n; i++) {
        for (int w = 0; w <= W; w++) {
            // Option 1: exclude item i
            dp[i][w] = dp[i - 1][w];
            // Option 2: include item i (if it fits)
            if (weights[i - 1] <= w) {
                dp[i][w] = Math.max(dp[i][w],
                    dp[i - 1][w - weights[i - 1]] + values[i - 1]);
            }
        }
    }
    return dp[n][W];
}
// Time: O(n*W), Space: O(n*W)
```

**Full Table Trace:** items = [{w=2,v=6},{w=2,v=10},{w=3,v=12}], W=5

```
Items: (w=2,v=6), (w=2,v=10), (w=3,v=12)

    w=0  w=1  w=2  w=3  w=4  w=5
i=0:  0    0    0    0    0    0
i=1:  0    0    6    6    6    6   ← item1(2,6): fits at w>=2
i=2:  0    0   10   10   16   16  ← item2(2,10): at w=4: item1+item2=16
i=3:  0    0   10   12   16   22  ← item3(3,12): at w=5: item2+item3=22

Answer: 22 (items 2 and 3: 10+12)
```

**Space Optimization to 1D (MUST iterate w from W down to w[i]):**

```java
public int knapsack1D(int[] weights, int[] values, int W) {
    int n = weights.length;
    int[] dp = new int[W + 1];

    for (int i = 0; i < n; i++) {
        // Traverse from right to left to avoid using item i twice
        for (int w = W; w >= weights[i]; w--) {
            dp[w] = Math.max(dp[w], dp[w - weights[i]] + values[i]);
        }
    }
    return dp[W];
}
// Time: O(n*W), Space: O(W)
```

**Why reverse traversal?** If we go left to right, dp[w - weights[i]] might have already been updated in this iteration (using item i again), turning it into an unbounded knapsack. Reverse traversal ensures we only use each item once.

**Reconstruct Selected Items:**

```java
public List<Integer> knapsackItems(int[] weights, int[] values, int W) {
    int n = weights.length;
    int[][] dp = new int[n + 1][W + 1];
    for (int i = 1; i <= n; i++)
        for (int w = 0; w <= W; w++) {
            dp[i][w] = dp[i-1][w];
            if (weights[i-1] <= w)
                dp[i][w] = Math.max(dp[i][w], dp[i-1][w-weights[i-1]] + values[i-1]);
        }

    // Backtrack
    List<Integer> items = new ArrayList<>();
    int w = W;
    for (int i = n; i >= 1; i--) {
        if (dp[i][w] != dp[i-1][w]) {
            items.add(i - 1); // item index
            w -= weights[i - 1];
        }
    }
    return items;
}
```

---

### Problem 16: Subset Sum

**Problem:** Given an array of positive integers and a target sum, determine if any subset sums to the target.

> **Think of it like:** checking whether you can pick some bills from your wallet to pay an exact price with no change. You don't need every bill — just some combination that adds up exactly to the target.

**Relation to Knapsack:** Treat each number as an item with weight = value. Can we fill capacity exactly?

**State definition:** dp[i][s] = true if a subset of first i elements sums to s

**Recurrence:**
- Exclude element: dp[i][s] = dp[i-1][s]
- Include element: dp[i][s] |= dp[i-1][s - nums[i-1]] (if nums[i-1] <= s)

```java
public boolean subsetSum(int[] nums, int target) {
    int n = nums.length;
    boolean[] dp = new boolean[target + 1];
    dp[0] = true; // empty subset sums to 0

    for (int num : nums) {
        // Traverse right to left (0/1 — each element used at most once)
        for (int s = target; s >= num; s--) {
            dp[s] = dp[s] || dp[s - num];
        }
    }
    return dp[target];
}
// Time: O(n * target), Space: O(target)
```

---

### Problem 17: Partition Equal Subset Sum

**Problem:** Given an array, determine if it can be partitioned into two subsets with equal sum.

**Key insight:** If total sum is odd, impossible. Otherwise, find a subset summing to total/2.

```java
public boolean canPartition(int[] nums) {
    int total = 0;
    for (int num : nums) total += num;
    if (total % 2 != 0) return false; // odd total → impossible

    int target = total / 2;
    boolean[] dp = new boolean[target + 1];
    dp[0] = true;

    for (int num : nums) {
        for (int s = target; s >= num; s--) {
            dp[s] = dp[s] || dp[s - num];
        }
    }
    return dp[target];
}
// Time: O(n * total/2), Space: O(total/2)
```

---

### Problem 18: Target Sum (Assign +/- to Array Elements)

**Problem:** Given an integer array nums and an integer target, assign + or - to each element and count the number of ways to reach the target.

**Mathematical reduction to Subset Sum:**
Let P = sum of elements with +, N = sum of elements with -.
P + N = total, P - N = target
→ P = (total + target) / 2

So count subsets with sum = (total + target) / 2.

```java
public int findTargetSumWays(int[] nums, int target) {
    int total = 0;
    for (int num : nums) total += num;

    // If (total + target) is odd or target > total, no solution
    if ((total + target) % 2 != 0 || Math.abs(target) > total) return 0;

    int subsetSum = (total + target) / 2;
    int[] dp = new int[subsetSum + 1];
    dp[0] = 1;

    for (int num : nums) {
        for (int s = subsetSum; s >= num; s--) {
            dp[s] += dp[s - num];
        }
    }
    return dp[subsetSum];
}
// Time: O(n * subsetSum), Space: O(subsetSum)
```

**Alternative: Direct DP without reduction**

```java
public int findTargetSumWaysDirect(int[] nums, int target) {
    Map<Integer, Integer> memo = new HashMap<>();
    return dfs(nums, target, 0, 0, memo);
}

private int dfs(int[] nums, int target, int idx, int curr,
                Map<Integer, Integer> memo) {
    String key = idx + "," + curr;
    if (memo.containsKey(key.hashCode())) return memo.get(key.hashCode());
    if (idx == nums.length) return curr == target ? 1 : 0;

    int add = dfs(nums, target, idx + 1, curr + nums[idx], memo);
    int sub = dfs(nums, target, idx + 1, curr - nums[idx], memo);
    memo.put(key.hashCode(), add + sub);
    return add + sub;
}
```

---

## 7. Interval DP

Interval DP solves problems on contiguous subarrays or subsequences. The state is typically dp[i][j] = answer for interval [i, j], and we try all "split points" k.

---

### Problem 19: Matrix Chain Multiplication

**Problem:** Given dimensions of matrices A1, A2, ..., An where Ai has dimensions p[i-1] x p[i], find the minimum number of scalar multiplications to compute the product.

**State definition:** dp[i][j] = minimum multiplications to compute product of matrices i through j

**Recurrence:** dp[i][j] = min over all k from i to j-1:
  dp[i][k] + dp[k+1][j] + p[i-1] * p[k] * p[j]

**Base case:** dp[i][i] = 0 (single matrix, no multiplication needed)

```java
public int matrixChainMultiplication(int[] p) {
    int n = p.length - 1; // number of matrices
    int[][] dp = new int[n][n];
    // dp[i][j] = min cost to multiply matrices i..j (0-indexed)

    // Length of chain (l = 2 means two matrices, etc.)
    for (int len = 2; len <= n; len++) {
        for (int i = 0; i <= n - len; i++) {
            int j = i + len - 1;
            dp[i][j] = Integer.MAX_VALUE;
            for (int k = i; k < j; k++) {
                int cost = dp[i][k] + dp[k + 1][j] + p[i] * p[k + 1] * p[j + 1];
                dp[i][j] = Math.min(dp[i][j], cost);
            }
        }
    }
    return dp[0][n - 1];
}
// Time: O(n^3), Space: O(n^2)
```

**Trace:** p = [10, 30, 5, 60] → matrices: A(10x30), B(30x5), C(5x60)

```
dp[0][0]=0, dp[1][1]=0, dp[2][2]=0

len=2:
  dp[0][1]: k=0: dp[0][0]+dp[1][1]+10*30*5 = 0+0+1500 = 1500
  dp[1][2]: k=1: dp[1][1]+dp[2][2]+30*5*60 = 0+0+9000 = 9000

len=3:
  dp[0][2]: 
    k=0: dp[0][0]+dp[1][2]+10*30*60 = 0+9000+18000 = 27000
    k=1: dp[0][1]+dp[2][2]+10*5*60 = 1500+0+3000 = 4500
  dp[0][2] = 4500
```

---

### Problem 20: Burst Balloons

**Problem:** Given n balloons with values nums[i], bursting balloon i gives nums[i-1] * nums[i] * nums[i+1] coins. Maximize total coins.

**Key insight:** Think about which balloon is burst LAST in interval [i, j], not first. This avoids dependency issues.

**State definition:** dp[i][j] = max coins from bursting all balloons in (i, j) exclusive

**Recurrence:** dp[i][j] = max over k from i+1 to j-1:
  dp[i][k] + dp[k][j] + nums[i] * nums[k] * nums[j]

```java
public int maxCoins(int[] nums) {
    int n = nums.length;
    // Add boundary balloons with value 1
    int[] balloons = new int[n + 2];
    balloons[0] = balloons[n + 1] = 1;
    for (int i = 0; i < n; i++) balloons[i + 1] = nums[i];

    int size = n + 2;
    int[][] dp = new int[size][size];

    // len = length of the open interval (i, j)
    for (int len = 2; len < size; len++) {
        for (int i = 0; i < size - len; i++) {
            int j = i + len;
            for (int k = i + 1; k < j; k++) {
                dp[i][j] = Math.max(dp[i][j],
                    dp[i][k] + dp[k][j] + balloons[i] * balloons[k] * balloons[j]);
            }
        }
    }
    return dp[0][size - 1];
}
// Time: O(n^3), Space: O(n^2)
```

---

### Problem 21: Palindrome Partitioning II (Minimum Cuts)

**Problem:** Given a string s, find the minimum cuts needed so that every substring is a palindrome.

**State definition:** dp[i] = minimum cuts for s[0..i]

**Precompute:** isPalin[i][j] = whether s[i..j] is a palindrome

```java
public int minCut(String s) {
    int n = s.length();
    boolean[][] isPalin = new boolean[n][n];

    // Precompute palindromes using expand-around-center / DP
    for (int i = n - 1; i >= 0; i--) {
        for (int j = i; j < n; j++) {
            isPalin[i][j] = s.charAt(i) == s.charAt(j)
                && (j - i <= 2 || isPalin[i + 1][j - 1]);
        }
    }

    int[] dp = new int[n];
    Arrays.fill(dp, Integer.MAX_VALUE);

    for (int i = 0; i < n; i++) {
        if (isPalin[0][i]) {
            dp[i] = 0; // whole prefix is palindrome, no cut needed
        } else {
            for (int j = 1; j <= i; j++) {
                if (isPalin[j][i] && dp[j - 1] != Integer.MAX_VALUE) {
                    dp[i] = Math.min(dp[i], dp[j - 1] + 1);
                }
            }
        }
    }
    return dp[n - 1];
}
// Time: O(n^2), Space: O(n^2)
```

---

## 8. String DP

---

### Problem 22: Longest Palindromic Subsequence

**Problem:** Find the length of the longest subsequence of a string that is a palindrome.

**Key insight:** LPS of s = LCS(s, reverse(s))

**State definition:** dp[i][j] = length of LPS in s[i..j]

**Recurrence:**
- If s[i] == s[j]: dp[i][j] = dp[i+1][j-1] + 2
- Else: dp[i][j] = max(dp[i+1][j], dp[i][j-1])

**Base cases:** dp[i][i] = 1 (single char is palindrome)

```java
public int longestPalindromeSubseq(String s) {
    int n = s.length();
    int[][] dp = new int[n][n];

    // Base case: single characters
    for (int i = 0; i < n; i++) dp[i][i] = 1;

    // Fill for increasing lengths
    for (int len = 2; len <= n; len++) {
        for (int i = 0; i <= n - len; i++) {
            int j = i + len - 1;
            if (s.charAt(i) == s.charAt(j)) {
                dp[i][j] = (len == 2) ? 2 : dp[i + 1][j - 1] + 2;
            } else {
                dp[i][j] = Math.max(dp[i + 1][j], dp[i][j - 1]);
            }
        }
    }
    return dp[0][n - 1];
}
// Time: O(n^2), Space: O(n^2)
```

---

### Problem 23: Longest Palindromic Substring

**Problem:** Find the longest contiguous substring of s that is a palindrome.

**DP Approach:**

```java
// DP: O(n^2) time, O(n^2) space
public String longestPalindromeDP(String s) {
    int n = s.length();
    boolean[][] dp = new boolean[n][n];
    int start = 0, maxLen = 1;

    // Single characters are palindromes
    for (int i = 0; i < n; i++) dp[i][i] = true;

    // Check length 2
    for (int i = 0; i < n - 1; i++) {
        if (s.charAt(i) == s.charAt(i + 1)) {
            dp[i][i + 1] = true;
            start = i; maxLen = 2;
        }
    }

    // Check lengths 3 and above
    for (int len = 3; len <= n; len++) {
        for (int i = 0; i <= n - len; i++) {
            int j = i + len - 1;
            if (s.charAt(i) == s.charAt(j) && dp[i + 1][j - 1]) {
                dp[i][j] = true;
                if (len > maxLen) { start = i; maxLen = len; }
            }
        }
    }
    return s.substring(start, start + maxLen);
}
// Time: O(n^2), Space: O(n^2)
```

**Expand-Around-Center (optimal — O(n) space):**

```java
// O(n^2) time, O(1) space
public String longestPalindromeExpand(String s) {
    int n = s.length();
    int start = 0, maxLen = 1;

    for (int center = 0; center < n; center++) {
        // Odd length palindromes
        int len1 = expandAroundCenter(s, center, center);
        // Even length palindromes
        int len2 = expandAroundCenter(s, center, center + 1);

        int len = Math.max(len1, len2);
        if (len > maxLen) {
            maxLen = len;
            start = center - (len - 1) / 2;
        }
    }
    return s.substring(start, start + maxLen);
}

private int expandAroundCenter(String s, int left, int right) {
    while (left >= 0 && right < s.length()
           && s.charAt(left) == s.charAt(right)) {
        left--;
        right++;
    }
    return right - left - 1; // length of palindrome
}
// Time: O(n^2), Space: O(1)
```

---

### Problem 24: Distinct Subsequences

**Problem:** Given strings s and t, count the number of distinct subsequences of s that equal t.

**State definition:** dp[i][j] = number of ways to form t[0..j-1] using s[0..i-1]

**Recurrence:**
- Always: dp[i][j] = dp[i-1][j] (don't use s[i-1])
- If s[i-1] == t[j-1]: dp[i][j] += dp[i-1][j-1] (use s[i-1] to match t[j-1])

**Base cases:** dp[i][0] = 1 (empty t matched by any prefix of s in one way)

```java
public int numDistinct(String s, String t) {
    int m = s.length(), n = t.length();
    long[][] dp = new long[m + 1][n + 1];

    for (int i = 0; i <= m; i++) dp[i][0] = 1;

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            dp[i][j] = dp[i - 1][j]; // skip s[i-1]
            if (s.charAt(i - 1) == t.charAt(j - 1)) {
                dp[i][j] += dp[i - 1][j - 1]; // use s[i-1]
            }
        }
    }
    return (int) dp[m][n];
}
// Time: O(m*n), Space: O(m*n)
```

---

## 9. DP on Trees

---

### Problem 25: House Robber III (Binary Tree)

**Problem:** Houses are arranged in a binary tree. Adjacent (parent-child) houses cannot be robbed together. Maximize total value.

**Key insight:** For each node, we have two choices: rob this node or skip it. Return a pair [robThis, skipThis] for each subtree.

```java
public int rob(TreeNode root) {
    int[] result = robHelper(root);
    return Math.max(result[0], result[1]);
}

// Returns [maxIfRobbed, maxIfSkipped]
private int[] robHelper(TreeNode node) {
    if (node == null) return new int[]{0, 0};

    int[] left = robHelper(node.left);
    int[] right = robHelper(node.right);

    // Rob this node: cannot rob children
    int robThis = node.val + left[1] + right[1];

    // Skip this node: children can be robbed or skipped
    int skipThis = Math.max(left[0], left[1]) + Math.max(right[0], right[1]);

    return new int[]{robThis, skipThis};
}
// Time: O(n), Space: O(h) — h = tree height
```

---

### Problem 26: Maximum Path Sum in Binary Tree

**Problem:** Find the maximum path sum in a binary tree. The path may start and end at any node.

**Key insight:** For each node, compute the max "gain" if we include this node as an extension (can only go in one direction — left or right, not both for passing through). But the answer might use this node as the "top" of a path going both ways.

```java
private int maxSum = Integer.MIN_VALUE;

public int maxPathSum(TreeNode root) {
    maxSum = Integer.MIN_VALUE;
    maxGain(root);
    return maxSum;
}

private int maxGain(TreeNode node) {
    if (node == null) return 0;

    // Max gain from left/right subtrees (ignore negative contributions)
    int leftGain = Math.max(maxGain(node.left), 0);
    int rightGain = Math.max(maxGain(node.right), 0);

    // Path through this node (using both left and right)
    int pathThroughNode = node.val + leftGain + rightGain;
    maxSum = Math.max(maxSum, pathThroughNode);

    // Return max gain if this node is used as extension (only one direction)
    return node.val + Math.max(leftGain, rightGain);
}
// Time: O(n), Space: O(h)
```

---

## 10. DP Patterns Summary Table

| Pattern | Representative Problems | Key State Definition | Key Recurrence |
|---------|------------------------|---------------------|----------------|
| Linear 1D | Fibonacci, Climbing Stairs, House Robber | dp[i] = answer for first i elements | dp[i] = f(dp[i-1], dp[i-2]) |
| Linear with decisions | Coin Change (min), Jump Game | dp[i] = min/max for value i | dp[i] = opt over transitions to i |
| Counting | Coin Change (ways), Decode Ways | dp[i] = number of ways for i | dp[i] += dp[i - choice] |
| 2D Grid | Unique Paths, Min Path Sum | dp[i][j] = answer at cell (i,j) | dp[i][j] = f(dp[i-1][j], dp[i][j-1]) |
| Two strings | LCS, Edit Distance, Distinct Subseq | dp[i][j] = answer for s1[0..i-1], s2[0..j-1] | Match/mismatch cases |
| 0/1 Knapsack | Knapsack, Subset Sum, Partition | dp[i][w] = answer with i items, capacity w | Include or exclude item |
| Unbounded Knapsack | Coin Change, Rod Cutting | dp[w] = answer for capacity w | dp[w] = max(dp[w], dp[w-wi]+vi) |
| Interval DP | Matrix Chain, Burst Balloons, Palindrome Cuts | dp[i][j] = answer for interval [i,j] | Try all split points k |
| String palindrome | LPS, Palindromic Substring, Min Cuts | dp[i][j] = answer for s[i..j] | Match ends or expand |
| Tree DP | House Robber III, Max Path Sum | pair[node] = {rob, skip} | Combine children results |

---

## 11. Space Optimization Techniques

### Technique 1: When Current Row Only Depends on Previous Row

**Rule:** If dp[i][j] only depends on dp[i-1][...], you can use a 1D array.

**Example: LCS (0/1 version — space reduction not as clean for LCS due to diagonal dependency)**

For problems like Knapsack and Subset Sum where dp[i][j] = f(dp[i-1][j], dp[i-1][j-w]):

```java
// 2D version
for (int i = 1; i <= n; i++)
    for (int j = 0; j <= W; j++)
        dp[i][j] = Math.max(dp[i-1][j], w[i]<=j ? dp[i-1][j-w[i]]+v[i] : 0);

// 1D version — iterate j in reverse so dp[j-w[i]] is still from previous row
for (int i = 0; i < n; i++)
    for (int j = W; j >= w[i]; j--)
        dp[j] = Math.max(dp[j], dp[j - w[i]] + v[i]);
```

### Technique 2: Current + Previous Variables (Rolling Variables)

**Use when:** dp[i] only depends on dp[i-1] and dp[i-2].

```java
// Fibonacci / House Robber / Climbing Stairs
int prev2 = base0, prev1 = base1;
for (int i = 2; i <= n; i++) {
    int curr = prev1 + prev2; // or: max(prev1, prev2 + nums[i])
    prev2 = prev1;
    prev1 = curr;
}
return prev1;
```

### Technique 3: Rolling Array for 2D DP

**Use when:** dp[i][j] only depends on dp[i-1][j], dp[i][j-1], dp[i-1][j-1].

```java
// Two-row rolling array
int[][] dp = new int[2][n + 1];
for (int i = 1; i <= m; i++) {
    int curr = i % 2;
    int prev = 1 - curr;
    for (int j = 1; j <= n; j++) {
        // use dp[prev][j] instead of dp[i-1][j]
        // use dp[curr][j-1] instead of dp[i][j-1]
    }
}
```

### When NOT to Optimize Space

- When you need to reconstruct the actual solution (need the full table for backtracking)
- When the space savings are negligible compared to code complexity
- When the problem is 1D already

---

## 12. Interview Questions & Answers (30+)

---

**Q1: What makes a problem a DP problem?**

A problem is suitable for DP if it has two key properties:
1. **Optimal substructure:** The optimal solution contains optimal solutions to subproblems. For example, the shortest path from A to C through B requires the shortest path from A to B.
2. **Overlapping subproblems:** The same subproblem is encountered multiple times. For example, computing Fibonacci(5) requires Fibonacci(3) in multiple branches.

Additionally, the problem typically asks for: minimum/maximum value, count of ways, or feasibility (can it be done?).

---

**Q2: What is the difference between memoization and tabulation?**

| Aspect | Memoization (Top-Down) | Tabulation (Bottom-Up) |
|--------|----------------------|----------------------|
| Direction | Start from original problem, recurse down | Start from base cases, build up |
| Implementation | Recursive + cache | Iterative + table |
| Subproblems computed | Only needed ones | All subproblems |
| Overhead | Function call stack | None (iterative) |
| Stack overflow risk | Yes (deep recursion) | No |
| Space optimization | Harder | Easier to reduce dimensions |
| Code clarity | Often mirrors the recurrence | Sometimes requires thinking in reverse |

For large inputs where recursion depth could cause stack overflow, prefer tabulation. For problems where only a fraction of subproblems are needed, memoization avoids unnecessary computation.

---

**Q3: How do you identify the state in a DP problem?**

The state captures everything you need to know to solve a subproblem — no more, no less.

Steps to identify the state:
1. Look at what changes between recursive calls. Those are your state variables.
2. Ask: "What information do I need to make a decision at this step?" Those become the parameters.
3. The state should be minimal — adding unnecessary state variables increases time/space complexity.

Examples:
- Fibonacci: state = n (only depends on position)
- 0/1 Knapsack: state = (item_index, remaining_capacity) → dp[i][w]
- Edit Distance: state = (i, j) = positions in both strings → dp[i][j]
- Burst Balloons: state = (i, j) = the open interval → dp[i][j]

---

**Q4: What is optimal substructure? Give an example where it does NOT hold.**

Optimal substructure means the optimal solution to the whole problem is composed of optimal solutions to its subproblems.

**Holds:** Shortest path (Dijkstra/Bellman-Ford). If the shortest path from A to C goes through B, then A→B and B→C are also shortest paths.

**Does NOT hold:** Longest simple path in a graph with cycles. The longest path from A to C might go through B, but the longest path from A to B might be a different route that makes A→C longer overall. Greedy/DP fails; this problem is NP-hard.

---

**Q5: What is the difference between LCS (subsequence) and Longest Common Substring?**

| Aspect | Longest Common Subsequence | Longest Common Substring |
|--------|--------------------------|-------------------------|
| Contiguous? | No — characters in order but may skip | Yes — contiguous characters |
| Example | LCS("ABCDE", "ACE") = "ACE" (length 3) | LCS("ABCDE", "BCEF") = "BCE" (length 3) |
| Mismatch recurrence | dp[i][j] = max(dp[i-1][j], dp[i][j-1]) | dp[i][j] = 0 (reset) |
| Answer location | dp[m][n] | max value across all dp[i][j] |
| Typical use | DNA matching, diff tools | Pattern matching, plagiarism |

The key code difference: on a character mismatch, LCS takes the max of skipping either character, while Longest Common Substring resets to 0 because the substring is broken.

---

**Q6: Walk me through the 0/1 Knapsack problem.**

"Given n items with weights and values, and a knapsack of capacity W, maximize the total value while keeping total weight ≤ W. Each item can be used at most once."

**State:** dp[i][w] = maximum value using first i items with capacity w.

**Decision at each step:** For item i, either:
- Exclude it: dp[i][w] = dp[i-1][w]
- Include it (if it fits): dp[i][w] = dp[i-1][w - weight[i]] + value[i]

**Take the max of both options.**

**Base cases:** dp[0][w] = 0 (no items) and dp[i][0] = 0 (no capacity).

**Time complexity:** O(n × W). This is pseudo-polynomial — polynomial in the numeric value of W, not in the number of bits needed to represent W.

**Space optimization:** We can reduce to 1D dp[W+1] by traversing w from W down to weight[i] for each item. Reverse traversal ensures we use each item at most once (otherwise we'd be allowing reuse — which is the unbounded knapsack variant).

---

**Q7: How do you reduce knapsack from 2D to 1D DP?**

The 2D recurrence is: dp[i][w] = max(dp[i-1][w], dp[i-1][w-wt[i]] + val[i])

Notice that row i only depends on row i-1. So we can use a single 1D array, but we must be careful about overwriting values we still need.

If we iterate w from left to right (increasing), when we compute dp[w - wt[i]], that value may have already been updated in this same i-iteration (meaning item i was already added). That would count item i multiple times — turning it into an unbounded knapsack.

**Solution:** Iterate w from right to left (W down to wt[i]). This ensures that when we access dp[w - wt[i]], it still reflects the state from the previous item (i-1), not the current item.

```java
for (int i = 0; i < n; i++)
    for (int w = W; w >= weights[i]; w--)  // RIGHT to LEFT
        dp[w] = Math.max(dp[w], dp[w - weights[i]] + values[i]);
```

For unbounded knapsack (item can be used multiple times), iterate LEFT to RIGHT.

---

**Q8: What is the time and space complexity of Edit Distance?**

- **Time:** O(m × n) where m = length of word1, n = length of word2. We fill an (m+1) × (n+1) table, each cell in O(1).
- **Space:** O(m × n) for the full table. Can be reduced to O(min(m, n)) using a 1D rolling array.

**Space optimization explanation:** dp[i][j] depends on dp[i-1][j-1] (diagonal), dp[i-1][j] (above), dp[i][j-1] (left). When processing row i, we only need row i-1. We use a 1D array but save the diagonal value (dp[i-1][j-1]) in a `prev` variable before overwriting.

---

**Q9: How do you reconstruct the actual solution (not just the optimal value) from a DP table?**

Store the full DP table (do not space-optimize), then backtrack from the final answer cell.

**General approach:**
1. At cell dp[i][j], check which decision led to the current value.
2. Move to the predecessor cell that was chosen.
3. Repeat until you reach a base case.

**Example — 0/1 Knapsack reconstruction:**
```java
int w = W;
for (int i = n; i >= 1; i--) {
    if (dp[i][w] != dp[i-1][w]) {
        // Item i was included
        selectedItems.add(i);
        w -= weights[i-1];
    }
    // else: item i was excluded, dp[i][w] == dp[i-1][w]
}
```

**Example — LCS reconstruction:** At dp[i][j], if characters match, go diagonal (both characters are in LCS). If not, go in the direction of the larger value (skip that character).

**Key rule:** The reconstruction simply reverses the decision made when filling the table.

---

**Q10: What is the difference between Coin Change (min coins) and Coin Change (ways)?**

| Aspect | Min Coins | Count Ways |
|--------|-----------|------------|
| Goal | Minimum number of coins | Number of distinct combinations |
| Initialization | dp[0]=0, others=∞ | dp[0]=1, others=0 |
| Recurrence | dp[i] = min(dp[i], dp[i-coin]+1) | dp[i] += dp[i-coin] |
| Return impossible as | -1 if dp[amount] stays ∞ | 0 naturally (dp[amount] stays 0) |
| Order matters? | No (same structure) | Depends on loop order |
| Loop structure | Either outer-loop order works | Order matters → amount outer; No order → coin outer |

**Critical distinction for "count ways":** If order matters ({1,2} and {2,1} are different), the outer loop should be over the amount. If order does NOT matter ({1,2} and {2,1} are the same combination), the outer loop should be over coins. This is the difference between permutations and combinations.

---

**Q11: What is the time complexity of the naive recursive Fibonacci? Why?**

O(2^n). Each call to fib(n) makes two recursive calls — fib(n-1) and fib(n-2). This creates a binary tree of calls. The tree has depth n, so the number of nodes is approximately 2^n. Many of these are redundant (fib(3) is computed O(2^(n-3)) times), which is exactly why memoization brings it down to O(n).

---

**Q12: Can greedy solve the 0/1 Knapsack problem? Why or why not?**

No. Greedy approaches (like taking highest value/weight ratio first) do not guarantee an optimal solution for 0/1 Knapsack because items cannot be split (unlike the Fractional Knapsack, which greedy solves optimally).

Counterexample:
- Items: A(w=1, v=6), B(w=2, v=10), C(w=3, v=12), W=5
- Greedy by v/w ratio: A(6), B(5), C(4) → Take A+B (w=3, v=16) or A+C (w=4, v=18) or B+C (w=5, v=22)
- Greedy would pick A first (ratio=6), then B (ratio=5), giving v=16
- DP finds B+C = v=22

The problem is that choosing the locally best item can prevent a globally better combination.

---

**Q13: What is the difference between 0/1 Knapsack and Unbounded Knapsack?**

| Aspect | 0/1 Knapsack | Unbounded Knapsack |
|--------|-------------|-------------------|
| Item usage | Each item at most once | Each item unlimited times |
| 1D traversal | Right to left (prevent reuse) | Left to right (allow reuse) |
| Example | Classic knapsack | Coin change (min coins) |
| Recurrence | dp[w] = max(dp[w], dp[w-wt]+val) with reverse traversal | Same but forward traversal |

The only code difference in the 1D version is the direction of the inner loop.

---

**Q14: Explain the "all intervals" iteration pattern in interval DP.**

In interval DP (like Matrix Chain Multiplication), we cannot iterate i from 1 to n in the outer loop because dp[i][j] might depend on dp[i][k] and dp[k][j] which aren't computed yet.

Instead, iterate by **interval length**:
```java
for (int len = 2; len <= n; len++) {       // increasing interval length
    for (int i = 0; i <= n - len; i++) {   // starting index
        int j = i + len - 1;               // ending index
        for (int k = i; k < j; k++) {      // all split points
            dp[i][j] = min/max(dp[i][j], dp[i][k] + dp[k+1][j] + cost);
        }
    }
}
```

When processing an interval of length L, all intervals of length < L have already been computed, so dependencies are satisfied.

---

**Q15: How does the Word Break DP work, and what is the state?**

dp[i] = true if s[0..i-1] can be segmented into dictionary words.

For each position i, we check all j < i: "Is dp[j] true AND is s[j..i-1] in the dictionary?" If both are true, dp[i] = true.

Base case: dp[0] = true (empty string requires no words).

The answer is dp[n] where n = length of s.

Time: O(n²) with HashSet for O(1) dictionary lookup. Space: O(n + dict_size).

---

**Q16: When would you use top-down over bottom-up in production code?**

**Use top-down (memoization) when:**
- Only a fraction of subproblems will be needed (sparse computation)
- The problem has complex state that is hard to enumerate in order
- You want to write code quickly and the recursive formulation is intuitive
- The recursion depth is manageable (n < ~10,000)

**Use bottom-up (tabulation) when:**
- All or most subproblems will be visited
- n is large and you risk stack overflow with recursion
- You need to optimize space (easier to drop rows you no longer need)
- Performance is critical (no function call overhead)

In interviews, either is usually acceptable. Mention both, then implement the one that comes most naturally.

---

**Q17: What is the Bellman-Ford algorithm and how is it DP?**

Bellman-Ford finds shortest paths from a source in a graph with potentially negative weights. It is a DP algorithm:

- **State:** dp[v][k] = shortest path to vertex v using at most k edges
- **Recurrence:** dp[v][k] = min over all edges (u,v): dp[u][k-1] + weight(u,v)
- **Iterations:** Run n-1 rounds (any shortest path has at most n-1 edges)
- **Time:** O(V × E)

The DP insight is that the shortest path from source to v using at most k hops is built from the shortest path using at most k-1 hops.

---

**Q18: How do you handle the "impossible" case in Coin Change (minimum coins)?**

Initialize dp[i] = amount + 1 for all i > 0 (a value larger than any valid answer — since the maximum valid answer using 1-coins is `amount`).

If dp[amount] remains > amount after filling the table, it was never reachable, so return -1.

Do not use Integer.MAX_VALUE directly because Integer.MAX_VALUE + 1 overflows to Integer.MIN_VALUE, causing incorrect comparisons.

```java
Arrays.fill(dp, amount + 1);  // safe "infinity"
dp[0] = 0;
// ... fill table ...
return dp[amount] > amount ? -1 : dp[amount];
```

---

**Q19: What is memoization's relationship to the call graph?**

Memoization ensures each unique input to a recursive function is computed exactly once. The call graph for an unmemoized function is a tree (each call spawns new children). With memoization, repeated subtrees are collapsed — the call graph becomes a DAG (Directed Acyclic Graph). The time complexity drops from exponential (tree size) to linear or polynomial (DAG node count).

---

**Q20: Describe the Longest Palindromic Subsequence and its relationship to LCS.**

LPS of a string s = LCS(s, reverse(s)).

**Why?** A palindrome reads the same forward and backward. So the longest palindromic subsequence of s is the longest subsequence that matches the reverse of s.

Example: s = "BBBAB", reverse = "BABBB"
LCS("BBBAB", "BABBB") = "BBBB" → length 4. This is the LPS.

This reduction allows us to reuse the LCS solution directly.

---

**Q21: What are the limitations of Dynamic Programming?**

1. **Pseudo-polynomial time:** Knapsack is O(n × W), which looks polynomial but W could be exponentially large in terms of input bits. True polynomial is O(n × log W).
2. **Space:** Many DP problems require O(n²) or O(n × W) space, which can be prohibitive for large inputs.
3. **Not applicable without optimal substructure:** Problems like longest simple path in a graph (NP-hard) cannot be solved with DP.
4. **State explosion:** If the state requires many dimensions, the table size explodes (curse of dimensionality).
5. **Problem framing:** Identifying the correct state definition is non-trivial. Wrong state → wrong recurrence → wrong answer.

---

**Q22: How is Climbing Stairs different from Fibonacci?**

They are structurally identical but with different base cases:
- Fibonacci: F(0)=0, F(1)=1 → 0,1,1,2,3,5,8...
- Climbing Stairs: cs(1)=1, cs(2)=2 → 1,2,3,5,8,13...

Climbing Stairs is Fibonacci shifted by one: cs(n) = Fib(n+1).

The underlying recurrence is the same: f(n) = f(n-1) + f(n-2).

---

**Q23: Why does Coin Change (count ways, order doesn't matter) use coin as outer loop?**

The outer-coin/inner-amount pattern ensures each combination is counted exactly once. When we process coin c, dp[amount] accumulates counts using coins seen so far. Since we fix the set of coins available before iterating amounts, we never count {1,2} and {2,1} as different — the coin is always added in the order we encounter it in the outer loop.

If we swapped (amount outer, coin inner), every ordering of the same coins would be counted separately, giving permutations instead of combinations.

---

**Q24: What is the time complexity of LCS and can it be improved?**

Standard DP LCS is O(m × n) time and space.

Space can be improved to O(min(m, n)) using a rolling array (only two rows needed).

Time improvement: The Hunt-Szymanski algorithm is O((r + n) log n) where r is the number of matching pairs. For sparse matches, this is faster. For dense matches (many common characters), it degenerates.

In practice, for most interview purposes, O(m × n) is the expected answer.

---

**Q25: Explain why Edit Distance has 3 operations in the recurrence.**

When characters s1[i-1] and s2[j-1] don't match:
- **Delete:** Remove s1[i-1]. Now align s1[0..i-2] with s2[0..j-1] → dp[i-1][j]
- **Insert:** Insert s2[j-1] into s1 at position i. Now align s1[0..i-1] with s2[0..j-2] → dp[i][j-1]
- **Replace:** Replace s1[i-1] with s2[j-1]. Now align s1[0..i-2] with s2[0..j-2] → dp[i-1][j-1]

Each operation costs 1, and we take the minimum. When characters match, no operation is needed → dp[i][j] = dp[i-1][j-1] (free diagonal move).

---

**Q26: What is the difference between "change the minimum coins" and "can we make exact change"?**

- Min coins: dp[i] = min coins to make exactly amount i. Initialize dp[0]=0, others=∞. Final: dp[amount] or -1 if ∞.
- Exact change (subset sum style): dp[i] = boolean — can we make exactly amount i. Initialize dp[0]=true, others=false. Final: dp[target] is true/false.

Both use the same loop structure. The difference is in the DP array type (int vs boolean), initialization, and update operation (min vs OR).

---

**Q27: How would you solve House Robber III without using a pair return?**

Use two separate recursive functions: one for "rob this node" and one for "skip this node". But this leads to O(n²) due to recomputation. The pair-return approach computes both in a single O(n) pass.

Alternatively, use a Map<TreeNode, int[]> as memoization:

```java
Map<TreeNode, int[]> memo = new HashMap<>();

int[] dp(TreeNode node) {
    if (node == null) return new int[]{0, 0};
    if (memo.containsKey(node)) return memo.get(node);
    int[] left = dp(node.left), right = dp(node.right);
    int rob = node.val + left[1] + right[1];
    int skip = Math.max(left[0], left[1]) + Math.max(right[0], right[1]);
    int[] result = {rob, skip};
    memo.put(node, result);
    return result;
}
```

---

**Q28: In the 0/1 Knapsack 1D optimization, what happens if you iterate left to right?**

Iterating left to right means when computing dp[w], dp[w - weight[i]] has already been updated for item i in the same iteration. This means item i can be selected multiple times (once when processing dp[w - weight[i]] and again when processing dp[w]). This accidentally implements the unbounded knapsack (each item can be used any number of times).

Right-to-left traversal ensures dp[w - weight[i]] still reflects the state before item i was considered, so each item is used at most once.

---

**Q29: Describe a scenario where memoization might use more space than tabulation.**

In problems with large but sparse state spaces, memoization uses a HashMap and only stores visited states. But HashMap has overhead (~40 bytes per entry in Java due to Entry object, next pointer, hash). A tabulation array of the same size uses just 4 bytes per integer.

Also, for deep recursion, memoization adds O(depth) stack frames on top of the cache. Tabulation uses no stack space.

However, if only 5% of a 1,000,000-cell table is visited, HashMap memoization uses ~50,000 entries × 40 bytes = 2MB vs tabulation's 4MB array. So memoization can be more space-efficient in sparse cases despite per-entry overhead.

---

**Q30: What is the difference between overlapping subproblems and repeated work in general recursion?**

All repeated work in recursion is not necessarily overlapping subproblems. In Merge Sort, the same array positions are processed in every call, but each call processes a different subarray — there is no re-computation of the same subproblem.

Overlapping subproblems specifically means: the same function call with the same arguments is made multiple times. In naive Fibonacci, fib(3) is called with argument 3 from multiple parent calls — that is true overlap. In Merge Sort, each subarray range [l, r] is processed exactly once — no overlap.

DP only helps when there is genuine overlap — when caching saves repeated computation.

---

**Q31: How do you debug a DP solution that gives a wrong answer?**

1. **Verify base cases:** Print dp[0], dp[1], dp[0][0], etc. and check they are correct manually.
2. **Trace on a small example:** Run on a 3-4 element input and print the full DP table. Verify each cell manually.
3. **Check iteration order:** Ensure that when computing dp[i][j], all dependencies (dp[i-1][j], dp[i][j-1], etc.) are already computed.
4. **Check off-by-one errors:** Is dp[n] or dp[n-1] the answer? Are indices shifted by 1 (dp[i] represents s[0..i-1] vs s[0..i])?
5. **Verify recurrence:** Re-derive the recurrence from scratch on paper. Misidentifying the decision choices is the most common error.
6. **Test edge cases:** Empty input, single element, target=0, impossible cases.

---

**Q32: What DP problems are commonly asked at FAANG-level interviews?**

Tier 1 (must know perfectly):
- Longest Common Subsequence
- 0/1 Knapsack and variants (Subset Sum, Partition Equal Subset Sum)
- Coin Change (both variants)
- Edit Distance
- Longest Palindromic Subsequence/Substring
- House Robber and variants

Tier 2 (should understand well):
- Word Break
- Decode Ways
- Burst Balloons
- Matrix Chain Multiplication
- Distinct Subsequences
- Maximum Profit in Stock (with cooldown, transaction limits)

Tier 3 (good to know):
- DP on trees (House Robber III, Max Path Sum)
- DP on intervals (palindrome partitioning)
- Bitmask DP (Travelling Salesman Problem)

---

**Q33: Explain how "Decode Ways" handles the leading zero edge case.**

A '0' cannot be decoded as a single digit (there is no letter 0). It can only be decoded as part of a two-digit number (10 or 20).

In the recurrence:
- Single digit decode: only if s[i-1] != '0' → dp[i] += dp[i-1]
- Two digit decode: only if s[i-2..i-1] is between 10 and 26 → dp[i] += dp[i-2]

If s starts with '0', dp[1] = 0 (no valid decoding for the first character), and this cascades to give dp[n] = 0.

If a '0' appears in the middle and the preceding digit is neither 1 nor 2, there is no valid decoding → dp[i] = 0, propagating failure forward.

---

**Q34: What is bitmask DP and when is it used?**

Bitmask DP uses a bitmask (integer) to represent a subset of elements as the DP state. It is used when:
- The number of elements is small (typically ≤ 20-25)
- You need to track which subset of elements has been processed

**Classic example: Travelling Salesman Problem (TSP)**

State: dp[mask][v] = minimum cost to visit all cities in `mask` ending at city `v`

```java
// TSP using bitmask DP
int n = dist.length;
int[][] dp = new int[1 << n][n];
// Fill with infinity, set dp[1][0] = 0 (start at city 0)
for (int mask = 1; mask < (1 << n); mask++) {
    for (int v = 0; v < n; v++) {
        if ((mask & (1 << v)) == 0) continue; // v not in mask
        int prevMask = mask ^ (1 << v);
        for (int u = 0; u < n; u++) {
            if ((prevMask & (1 << u)) == 0) continue;
            dp[mask][v] = Math.min(dp[mask][v], dp[prevMask][u] + dist[u][v]);
        }
    }
}
// Time: O(2^n * n^2), Space: O(2^n * n)
```

---

**Q35: How do you approach a DP problem you've never seen before in an interview?**

Follow this systematic approach:

1. **Identify it's DP:** Does it ask for min/max/count/feasibility? Is there a combinatorial space? Can subproblems overlap?

2. **Define the state:** What are the changing parameters? What does dp[...] represent? State this explicitly: "dp[i] means the maximum profit considering the first i transactions."

3. **Write the recurrence on paper:** Consider what decisions you can make at each state. Write all cases (include/exclude, match/mismatch, etc.).

4. **Identify base cases:** What is the trivially known answer? (dp[0], dp[i][0], etc.)

5. **Check iteration order:** Can you compute dp[i] from dp[i-1]? Or do you need a specific order like interval DP?

6. **Code it up top-down first:** It is easier to code recursion + memoization and verify correctness before optimizing.

7. **Verify on a small example:** Trace through a 2-3 element example by hand.

8. **Optimize if asked:** Reduce space, convert to bottom-up.

State your reasoning out loud throughout — interviewers value the thought process as much as the final solution.

---

## 13. Complexity Reference Sheet

| Problem | Time | Space | Optimized Space |
|---------|------|-------|-----------------|
| Fibonacci | O(n) | O(n) | O(1) |
| Climbing Stairs | O(n) | O(n) | O(1) |
| House Robber | O(n) | O(n) | O(1) |
| House Robber II | O(n) | O(1) | O(1) |
| Coin Change (min) | O(n·W) | O(W) | O(W) |
| Coin Change (ways) | O(n·W) | O(W) | O(W) |
| Word Break | O(n²) | O(n) | O(n) |
| Jump Game | O(n) | O(1) | O(1) |
| Jump Game II | O(n) | O(1) | O(1) |
| Decode Ways | O(n) | O(n) | O(1) |
| Unique Paths | O(m·n) | O(m·n) | O(n) |
| Min Path Sum | O(m·n) | O(m·n) | O(n) |
| LCS | O(m·n) | O(m·n) | O(min(m,n)) |
| Edit Distance | O(m·n) | O(m·n) | O(min(m,n)) |
| Longest Common Substring | O(m·n) | O(m·n) | O(min(m,n)) |
| 0/1 Knapsack | O(n·W) | O(n·W) | O(W) |
| Subset Sum | O(n·W) | O(n·W) | O(W) |
| Partition Equal Subset | O(n·sum) | O(sum) | O(sum) |
| Target Sum | O(n·sum) | O(sum) | O(sum) |
| Matrix Chain Multiplication | O(n³) | O(n²) | O(n²) |
| Burst Balloons | O(n³) | O(n²) | O(n²) |
| Palindrome Partitioning II | O(n²) | O(n²) | O(n²) |
| Longest Palindromic Subsequence | O(n²) | O(n²) | O(n) |
| Longest Palindromic Substring | O(n²) | O(1) expand-center | O(1) |
| Distinct Subsequences | O(m·n) | O(m·n) | O(n) |
| House Robber III | O(n) | O(h) | O(h) |
| Maximum Path Sum | O(n) | O(h) | O(h) |

*W = target/capacity, h = tree height*

---

## 14. Quick Revision Cheat Sheet

```
DP Identification:
  ✓ Ask for min/max/count/possible?
  ✓ Optimal substructure?
  ✓ Overlapping subproblems?
  → YES to all = DP

State Design:
  → Identify all changing parameters
  → dp[i] means: "best answer for [first i elements / amount i / prefix of length i]"

Recurrence Strategy by Problem Type:
  Linear (1D):       dp[i] = f(dp[i-1], dp[i-2])
  String/Grid (2D):  dp[i][j] = f(dp[i-1][j], dp[i][j-1], dp[i-1][j-1])
  Knapsack:          dp[i][w] = max(dp[i-1][w], dp[i-1][w-wi]+vi)
  Interval:          dp[i][j] = min/max over k in [i,j] of f(dp[i][k], dp[k+1][j])
  Counting:          dp[i] += dp[i-choice] for all valid choices

Common Iteration Orders:
  Most 1D: left to right
  0/1 Knapsack 1D: right to left (reverse)
  Unbounded Knapsack 1D: left to right
  Interval DP: by increasing interval length

Space Optimization:
  dp[i] = f(dp[i-1], dp[i-2]) → two variables (prev1, prev2)
  dp[i][j] = f(dp[i-1][...]) → 1D array, careful about direction
  Need reconstruction? → Keep full table (don't optimize space)
```
