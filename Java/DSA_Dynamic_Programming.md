# Dynamic Programming — Interview Preparation Guide

> **New to DP? Read this first.** DP is just **brute-force recursion + remembering answers you already computed**. If you can write a plain recursive solution, you're most of the way there. Focus on these five classics in order: **Fibonacci**, **Climbing Stairs**, **House Robber**, **Coin Change**, and **Longest Common Subsequence (LCS)**.

---

## Table of Contents

1. [What is Dynamic Programming?](#1-what-is-dynamic-programming)
2. [Two Approaches to DP](#2-two-approaches-to-dp)
3. [How to Identify DP Problems](#3-how-to-identify-dp-problems)
4. [General DP Problem-Solving Template](#4-general-dp-problem-solving-template)
5. [1D DP Problems](#5-1d-dp-problems)
6. [2D DP Problems](#6-2d-dp-problems)
7. [String DP](#7-string-dp)
8. [DP Patterns Summary Table](#8-dp-patterns-summary-table)
9. [Interview Questions & Answers](#9-interview-questions--answers)
10. [Complexity Reference Sheet](#10-complexity-reference-sheet)
11. [Quick Revision Cheat Sheet](#11-quick-revision-cheat-sheet)

---

## 1. What is Dynamic Programming?

Dynamic Programming (DP) solves problems by breaking them into **overlapping subproblems** and storing the results to avoid redundant computation.

**Two properties required:**

**1. Optimal Substructure** — the optimal solution is built from optimal solutions to subproblems.
Example: shortest path A→B→C is optimal only if A→B is optimal for that sub-path.

**2. Overlapping Subproblems** — the same subproblems recur multiple times.
Example: Fibonacci(5) needs Fibonacci(3) twice. Compute once, cache, reuse.

### DP vs Divide and Conquer

| Aspect | Divide and Conquer | Dynamic Programming |
|--------|-------------------|---------------------|
| Subproblems | Non-overlapping | Overlapping |
| Example | Merge Sort | Fibonacci, Knapsack |
| Caching | Not needed | Essential |

### DP vs Greedy

| Aspect | Greedy | Dynamic Programming |
|--------|--------|---------------------|
| Strategy | Locally optimal at each step | Considers all choices |
| Finds optimal? | Not always | Yes (if applicable) |
| Speed | Faster | Slower |
| Example | Activity Selection | 0/1 Knapsack, Edit Distance |

**Key insight:** Coin change with coins [1, 3, 4] and target 6 — Greedy gives 4+1+1 = 3 coins; DP finds 3+3 = 2 coins.

---

## 2. Two Approaches to DP

### Top-Down (Memoization)

Write the natural recursive solution; add a cache to store results. Check before computing.

```java
public long solve(int n) {
    long[] memo = new long[n + 1];
    Arrays.fill(memo, -1);
    return dp(n, memo);
}

private long dp(int n, long[] memo) {
    if (n <= 1) return n;
    if (memo[n] != -1) return memo[n];
    memo[n] = dp(n - 1, memo) + dp(n - 2, memo);
    return memo[n];
}
```

**Prefer top-down when:** only a fraction of subproblems are needed; recursion is intuitive.

### Bottom-Up (Tabulation)

Build iteratively from smallest subproblems upward. Fill a table starting from base cases.

```java
public long solve(int n) {
    if (n <= 1) return n;
    long[] dp = new long[n + 1];
    dp[0] = 0; dp[1] = 1;
    for (int i = 2; i <= n; i++) dp[i] = dp[i-1] + dp[i-2];
    return dp[n];
}
```

**Prefer bottom-up when:** all subproblems are needed; n is large (stack overflow risk); want easier space optimization.

### Space Optimization: 2D → 1D

When `dp[i][j]` only depends on the previous row, use a single 1D array. For 0/1 knapsack, traverse `j` in **reverse** to avoid using updated values (which would allow reuse of items):

```java
// 0/1 knapsack: reverse traversal
for (int i = 0; i < n; i++)
    for (int j = W; j >= w[i]; j--)
        dp[j] = Math.max(dp[j], dp[j - w[i]] + v[i]);
```

---

## 3. How to Identify DP Problems

### Keywords That Signal DP

| Keyword | Example |
|---------|---------|
| "minimum / maximum" | Minimum coins, Maximum profit |
| "count the number of ways" | Climbing stairs, Coin change ways |
| "is it possible / can you reach" | Jump game, Subset sum |
| "longest / shortest" | LCS, Edit distance |
| "partition / split" | Palindrome partitioning |

### Checklist

1. Can the problem be broken into smaller versions of itself?
2. Do the smaller versions overlap?
3. Does the optimal solution depend on optimal sub-solutions?
4. Does it ask for count, min, max, or yes/no over a combinatorial space?

YES to all → almost certainly DP.

---

## 4. General DP Problem-Solving Template

```
Step 1: Define the state — dp[i] means [what this value represents]
Step 2: Write the recurrence — dp[i] = f(dp[i-1], dp[i-2], ...)
Step 3: Identify base cases — dp[0] = ?
Step 4: Determine iteration order — usually left to right; reverse for 0/1 knapsack
Step 5: Extract the answer — return dp[n], dp[target], dp[m][n], etc.
```

---

## 5. 1D DP Problems

### Problem 1: Fibonacci Sequence

**Definition:** F(n) = F(n-1) + F(n-2), F(0)=0, F(1)=1

| Approach | Time | Space |
|----------|------|-------|
| Naive recursion | O(2^n) | O(n) stack |
| Memoization | O(n) | O(n) |
| Tabulation | O(n) | O(n) |
| Space optimized | O(n) | O(1) |

```java
// Space optimized — O(n) time, O(1) space
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
```

---

### Problem 2: Climbing Stairs

**Problem:** n steps, can climb 1 or 2 at a time. How many distinct ways to the top?

**State:** dp[i] = number of ways to reach step i
**Recurrence:** dp[i] = dp[i-1] + dp[i-2]
**Base cases:** dp[0] = 1, dp[1] = 1

```java
public int climbStairs(int n) {
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

**Note:** Structurally identical to Fibonacci (different base cases). Very common interview entry question.

---

### Problem 3: House Robber

**Problem:** Max money from houses; cannot rob two adjacent houses.

**State:** dp[i] = max money from first i houses
**Recurrence:** dp[i] = max(dp[i-1], dp[i-2] + nums[i])

```java
public int rob(int[] nums) {
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

**Variant: House Robber II (circular)** — run the line solution twice (houses `0..n-2` and `1..n-1`), take the max.

---

### Problem 4: Coin Change — Minimum Coins

**Problem:** Given coins and a target amount, find the minimum number of coins. Return -1 if impossible.

**State:** dp[i] = minimum coins to make amount i
**Recurrence:** dp[i] = min over each coin c: dp[i-c] + 1
**Base case:** dp[0] = 0; initialize all others to `amount + 1` (safe "infinity")

```java
public int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1);
    dp[0] = 0;
    for (int i = 1; i <= amount; i++) {
        for (int coin : coins) {
            if (coin <= i && dp[i - coin] != amount + 1) {
                dp[i] = Math.min(dp[i], dp[i - coin] + 1);
            }
        }
    }
    return dp[amount] > amount ? -1 : dp[amount];
}
// Time: O(amount * coins.length), Space: O(amount)
```

**Why `amount+1` and not `Integer.MAX_VALUE`?** `Integer.MAX_VALUE + 1` overflows. `amount+1` is always larger than any valid answer.

**Trace:** coins = [1,3,4], amount = 6 → dp = [0,1,2,1,1,2,2]. Answer: 2 (3+3). Greedy gives 3 coins.

---

### Problem 5: Coin Change — Number of Ways

**State:** dp[i] = number of ways to make amount i; **Base case:** dp[0] = 1

```java
// Order matters (Combination Sum IV): amount outer, coin inner
public int countWaysOrdered(int[] coins, int target) {
    int[] dp = new int[target + 1];
    dp[0] = 1;
    for (int i = 1; i <= target; i++)
        for (int coin : coins)
            if (coin <= i) dp[i] += dp[i - coin];
    return dp[target];
}

// Order does NOT matter: coin outer, amount inner
public int countWaysUnordered(int[] coins, int target) {
    int[] dp = new int[target + 1];
    dp[0] = 1;
    for (int coin : coins)
        for (int i = coin; i <= target; i++)
            dp[i] += dp[i - coin];
    return dp[target];
}
```

**Key difference:** Order matters → amount outer, coin inner. Order doesn't matter → coin outer, amount inner.

---

### Problem 6: Word Break

**Problem:** Can string `s` be segmented into dictionary words?

**State:** dp[i] = true if s[0..i-1] can be segmented
**Recurrence:** dp[i] = true if dp[j] && s[j..i-1] is in dictionary, for some j < i
**Base case:** dp[0] = true

```java
public boolean wordBreak(String s, List<String> wordDict) {
    Set<String> wordSet = new HashSet<>(wordDict);
    int n = s.length();
    boolean[] dp = new boolean[n + 1];
    dp[0] = true;
    for (int i = 1; i <= n; i++) {
        for (int j = 0; j < i; j++) {
            if (dp[j] && wordSet.contains(s.substring(j, i))) {
                dp[i] = true;
                break;
            }
        }
    }
    return dp[n];
}
// Time: O(n^2), Space: O(n + dict size)
```

---

### Problem 7: Jump Game & Jump Game II

**Jump Game** — can you reach the last index? Greedy is optimal:

```java
public boolean canJump(int[] nums) {
    int maxReach = 0;
    for (int i = 0; i < nums.length; i++) {
        if (i > maxReach) return false;
        maxReach = Math.max(maxReach, i + nums[i]);
    }
    return true;
}
// Time: O(n), Space: O(1)
```

**Jump Game II** — minimum jumps to reach the end:

```java
public int jump(int[] nums) {
    int jumps = 0, currentEnd = 0, farthest = 0;
    for (int i = 0; i < nums.length - 1; i++) {
        farthest = Math.max(farthest, i + nums[i]);
        if (i == currentEnd) { jumps++; currentEnd = farthest; }
    }
    return jumps;
}
// Time: O(n), Space: O(1)
```

---

### Problem 8: Decode Ways

**Problem:** Count ways to decode a digit string (A=1...Z=26).

**State:** dp[i] = ways to decode s[0..i-1]
**Base cases:** dp[0] = 1, dp[1] = s[0]!='0' ? 1 : 0

```java
public int numDecodings(String s) {
    int n = s.length();
    if (n == 0 || s.charAt(0) == '0') return 0;
    int[] dp = new int[n + 1];
    dp[0] = 1;
    dp[1] = 1;
    for (int i = 2; i <= n; i++) {
        int oneDigit = s.charAt(i - 1) - '0';
        if (oneDigit != 0) dp[i] += dp[i - 1];
        int twoDigit = Integer.parseInt(s.substring(i - 2, i));
        if (twoDigit >= 10 && twoDigit <= 26) dp[i] += dp[i - 2];
    }
    return dp[n];
}
// Time: O(n), Space: O(n)
```

**Trace:** "226" → dp = [1,1,2,3]. Answer: 3 ({2,2,6}, {22,6}, {2,26}).

---

## 6. 2D DP Problems

### Problem 9: Unique Paths

**Problem:** Robot at top-left of m×n grid, can only move right or down. Count paths to bottom-right.

**State:** dp[i][j] = unique paths to (i,j)
**Recurrence:** dp[i][j] = dp[i-1][j] + dp[i][j-1]
**Base cases:** first row and first column all = 1

```java
public int uniquePaths(int m, int n) {
    int[][] dp = new int[m][n];
    for (int i = 0; i < m; i++) dp[i][0] = 1;
    for (int j = 0; j < n; j++) dp[0][j] = 1;
    for (int i = 1; i < m; i++)
        for (int j = 1; j < n; j++)
            dp[i][j] = dp[i-1][j] + dp[i][j-1];
    return dp[m-1][n-1];
}
// Time: O(m*n), Space: O(m*n) — reducible to O(n) with 1D row
```

**Variant: Unique Paths II (with obstacles)** — set dp[i][j] = 0 on obstacle cells.

---

### Problem 10: Minimum Path Sum

**State:** dp[i][j] = minimum sum to reach (i,j)
**Recurrence:** dp[i][j] = grid[i][j] + min(dp[i-1][j], dp[i][j-1])

```java
public int minPathSum(int[][] grid) {
    int m = grid.length, n = grid[0].length;
    int[][] dp = new int[m][n];
    dp[0][0] = grid[0][0];
    for (int i = 1; i < m; i++) dp[i][0] = dp[i-1][0] + grid[i][0];
    for (int j = 1; j < n; j++) dp[0][j] = dp[0][j-1] + grid[0][j];
    for (int i = 1; i < m; i++)
        for (int j = 1; j < n; j++)
            dp[i][j] = grid[i][j] + Math.min(dp[i-1][j], dp[i][j-1]);
    return dp[m-1][n-1];
}
// Time: O(m*n), Space: O(m*n)
```

---

### Problem 11: Longest Common Subsequence (LCS)

**Subsequence** = characters in order but not necessarily contiguous. LCS finds the longest such sequence common to both strings.

**State:** dp[i][j] = LCS length of s1[0..i-1] and s2[0..j-1]
**Recurrence:**
- s1[i-1] == s2[j-1]: dp[i][j] = dp[i-1][j-1] + 1
- else: dp[i][j] = max(dp[i-1][j], dp[i][j-1])
**Base cases:** dp[0][j] = 0, dp[i][0] = 0

```java
public int lcs(String s1, String s2) {
    int m = s1.length(), n = s2.length();
    int[][] dp = new int[m + 1][n + 1];
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (s1.charAt(i-1) == s2.charAt(j-1))
                dp[i][j] = dp[i-1][j-1] + 1;
            else
                dp[i][j] = Math.max(dp[i-1][j], dp[i][j-1]);
        }
    }
    return dp[m][n];
}
// Time: O(m*n), Space: O(m*n)
```

---

### Problem 12: Edit Distance (Levenshtein Distance)

**Problem:** Minimum insert/delete/replace operations to transform word1 into word2.

**State:** dp[i][j] = min operations to transform word1[0..i-1] into word2[0..j-1]
**Recurrence:**
- Characters match: dp[i][j] = dp[i-1][j-1]
- Mismatch: dp[i][j] = 1 + min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1]) (delete, insert, replace)
**Base cases:** dp[i][0] = i, dp[0][j] = j

```java
public int minDistance(String word1, String word2) {
    int m = word1.length(), n = word2.length();
    int[][] dp = new int[m + 1][n + 1];
    for (int i = 0; i <= m; i++) dp[i][0] = i;
    for (int j = 0; j <= n; j++) dp[0][j] = j;
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (word1.charAt(i-1) == word2.charAt(j-1))
                dp[i][j] = dp[i-1][j-1];
            else
                dp[i][j] = 1 + Math.min(dp[i-1][j],
                                Math.min(dp[i][j-1], dp[i-1][j-1]));
        }
    }
    return dp[m][n];
}
// Time: O(m*n), Space: O(m*n)
```

---

### Problem 13: Longest Common Substring

**Difference from LCS:** Must be contiguous. Mismatch → reset to 0 (don't take max).

**State:** dp[i][j] = length of longest common substring ending at s1[i-1] and s2[j-1]

```java
public int longestCommonSubstring(String s1, String s2) {
    int m = s1.length(), n = s2.length();
    int[][] dp = new int[m + 1][n + 1];
    int maxLen = 0;
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (s1.charAt(i-1) == s2.charAt(j-1)) {
                dp[i][j] = dp[i-1][j-1] + 1;
                maxLen = Math.max(maxLen, dp[i][j]);
            } else {
                dp[i][j] = 0; // reset — substring is broken
            }
        }
    }
    return maxLen;
}
// Time: O(m*n), Space: O(m*n)
```

| Aspect | LCS | Longest Common Substring |
|--------|-----|--------------------------|
| Contiguous? | No | Yes |
| Mismatch case | max(dp[i-1][j], dp[i][j-1]) | 0 (reset) |
| Answer | dp[m][n] | max value across all cells |

---

### Problem 14: 0/1 Knapsack

**Problem:** n items with weight w[i] and value v[i]; capacity W. Maximize value. Each item used at most once.

**State:** dp[i][w] = max value using first i items with capacity w
**Recurrence:** dp[i][w] = max(dp[i-1][w], dp[i-1][w-w[i]] + v[i]) if w[i] <= w
**Base cases:** dp[0][w] = 0, dp[i][0] = 0

```java
// Space-optimized 1D (reverse traversal for 0/1)
public int knapsack(int[] weights, int[] values, int W) {
    int n = weights.length;
    int[] dp = new int[W + 1];
    for (int i = 0; i < n; i++)
        for (int w = W; w >= weights[i]; w--)
            dp[w] = Math.max(dp[w], dp[w - weights[i]] + values[i]);
    return dp[W];
}
// Time: O(n*W), Space: O(W)
```

**Why reverse?** Left-to-right would let item i be selected multiple times (unbounded knapsack). Reverse ensures each item is used at most once.

---

### Problem 15: Subset Sum & Partition Equal Subset Sum

**Subset Sum:** Can any subset sum to target?

```java
public boolean subsetSum(int[] nums, int target) {
    boolean[] dp = new boolean[target + 1];
    dp[0] = true;
    for (int num : nums)
        for (int s = target; s >= num; s--)
            dp[s] = dp[s] || dp[s - num];
    return dp[target];
}
```

**Partition Equal Subset Sum:** Can array be split into two equal-sum halves?

```java
public boolean canPartition(int[] nums) {
    int total = 0;
    for (int num : nums) total += num;
    if (total % 2 != 0) return false;
    int target = total / 2;
    boolean[] dp = new boolean[target + 1];
    dp[0] = true;
    for (int num : nums)
        for (int s = target; s >= num; s--)
            dp[s] = dp[s] || dp[s - num];
    return dp[target];
}
// Time: O(n * total/2), Space: O(total/2)
```

---

### Problem 16: Target Sum

**Problem:** Assign + or - to each element; count ways to reach target.

**Math reduction:** Let P = sum of positive elements. P = (total + target) / 2. Count subsets summing to P.

```java
public int findTargetSumWays(int[] nums, int target) {
    int total = 0;
    for (int num : nums) total += num;
    if ((total + target) % 2 != 0 || Math.abs(target) > total) return 0;
    int subsetSum = (total + target) / 2;
    int[] dp = new int[subsetSum + 1];
    dp[0] = 1;
    for (int num : nums)
        for (int s = subsetSum; s >= num; s--)
            dp[s] += dp[s - num];
    return dp[subsetSum];
}
// Time: O(n * subsetSum), Space: O(subsetSum)
```

---

## 7. String DP

### Longest Palindromic Subsequence (awareness)

LPS(s) = LCS(s, reverse(s)). Reuse the LCS solution directly. O(n²).

### Longest Palindromic Substring

Optimal approach: **expand around center** — O(n²) time, O(1) space.

```java
public String longestPalindrome(String s) {
    int start = 0, maxLen = 1;
    for (int center = 0; center < s.length(); center++) {
        int len1 = expand(s, center, center);     // odd length
        int len2 = expand(s, center, center + 1); // even length
        int len = Math.max(len1, len2);
        if (len > maxLen) { maxLen = len; start = center - (len - 1) / 2; }
    }
    return s.substring(start, start + maxLen);
}

private int expand(String s, int left, int right) {
    while (left >= 0 && right < s.length() && s.charAt(left) == s.charAt(right)) {
        left--; right++;
    }
    return right - left - 1;
}
```

### Distinct Subsequences (awareness)

Count how many distinct subsequences of s equal t. `dp[i][j]` = ways to form t[0..j-1] from s[0..i-1]: always add `dp[i-1][j]` (skip s[i-1]), plus `dp[i-1][j-1]` when characters match. Base: `dp[i][0] = 1`. O(m·n).

---

## 8. DP Patterns Summary Table

| Pattern | Representative Problems | Key State | Key Recurrence |
|---------|------------------------|-----------|----------------|
| Linear 1D | Fibonacci, Climbing Stairs, House Robber | dp[i] = answer for first i elements | dp[i] = f(dp[i-1], dp[i-2]) |
| Linear with decisions | Coin Change (min), Jump Game | dp[i] = min/max for value i | dp[i] = opt over transitions |
| Counting | Coin Change (ways), Decode Ways | dp[i] = number of ways | dp[i] += dp[i - choice] |
| 2D Grid | Unique Paths, Min Path Sum | dp[i][j] = answer at cell | dp[i][j] = f(above, left) |
| Two strings | LCS, Edit Distance | dp[i][j] = answer for prefixes | Match/mismatch cases |
| 0/1 Knapsack | Knapsack, Subset Sum, Partition | dp[i][w] = answer with i items, capacity w | Include or exclude item |
| Unbounded Knapsack | Coin Change, Rod Cutting | dp[w] = answer for capacity w | dp[w] = max(dp[w], dp[w-wi]+vi) |
| Interval DP | Matrix Chain, Burst Balloons | dp[i][j] = answer for interval | Try all split points k |
| String palindrome | LPS, Palindromic Substring | dp[i][j] = answer for s[i..j] | Match ends or expand |
| Tree DP | House Robber III, Max Path Sum | pair[node] = {rob, skip} | Combine children results |

---

## 9. Interview Questions & Answers

**Q1: What makes a problem a DP problem?**
Two properties: **optimal substructure** (optimal solution is built from optimal sub-solutions) and **overlapping subproblems** (same subproblem solved multiple times). Problems typically ask for min/max, count, or feasibility.

---

**Q2: What is the difference between memoization and tabulation?**

| Aspect | Memoization (Top-Down) | Tabulation (Bottom-Up) |
|--------|----------------------|----------------------|
| Direction | Start from problem, recurse | Start from base, build up |
| Implementation | Recursive + cache | Iterative + table |
| Subproblems | Only needed ones | All subproblems |
| Stack overflow risk | Yes | No |
| Space optimization | Harder | Easier |

---

**Q3: How do you identify the state in a DP problem?**
Look at what parameters change between recursive calls — those become your state variables. The state must capture everything needed to make a decision, and no more. Knapsack needs (item index, remaining capacity); Edit Distance needs (position in each string).

---

**Q4: What is the difference between LCS (subsequence) and Longest Common Substring?**

| Aspect | LCS | Longest Common Substring |
|--------|-----|--------------------------|
| Contiguous? | No | Yes |
| Mismatch recurrence | max(dp[i-1][j], dp[i][j-1]) | dp[i][j] = 0 (reset) |
| Answer location | dp[m][n] | max across all cells |

---

**Q5: Walk me through 0/1 Knapsack.**
State: `dp[i][w]` = max value using first i items with capacity w. For each item: either exclude it (take `dp[i-1][w]`) or include it (take `dp[i-1][w-weight[i]] + value[i]`). Time: O(n×W) — pseudo-polynomial. Space-optimized to 1D with reverse traversal to prevent item reuse.

---

**Q6: Why does 0/1 Knapsack 1D use reverse traversal?**
Left-to-right means `dp[w-weight[i]]` may have already been updated in the current iteration, allowing item i to be picked multiple times (unbounded knapsack). Reverse traversal ensures `dp[w-weight[i]]` still reflects the previous item's state.

---

**Q7: What is the time complexity of Edit Distance?**
O(m×n) time to fill the (m+1)×(n+1) table; O(m×n) space reducible to O(min(m,n)) with a 1D rolling array (save the diagonal value before overwriting).

---

**Q8: How do you handle the "impossible" case in Coin Change?**
Initialize `dp[i] = amount + 1` (safe infinity — never overflows unlike `Integer.MAX_VALUE + 1`). After filling, if `dp[amount] > amount`, return -1.

---

**Q9: How do you reconstruct the actual solution from a DP table?**
Keep the full 2D table, then backtrack from the final cell. At each cell, check which decision led to the current value and move to that predecessor. Example for knapsack: if `dp[i][w] != dp[i-1][w]`, item i was included — record it and subtract its weight.

---

**Q10: What is the difference between Coin Change (min coins) and Coin Change (ways)?**

| Aspect | Min Coins | Count Ways |
|--------|-----------|------------|
| Initialization | dp[0]=0, others=amount+1 | dp[0]=1, others=0 |
| Update | dp[i] = min(dp[i], dp[i-coin]+1) | dp[i] += dp[i-coin] |
| Loop order | Either works | Order matters → amount outer; no order → coin outer |

---

**Q11: Why is naive recursive Fibonacci O(2^n)?**
Each call spawns two more (fib(n-1) and fib(n-2)), creating a binary tree of depth n. Many calls are redundant — fib(3) is recomputed many times. Memoization caches each result, cutting it to O(n).

---

**Q12: Can greedy solve 0/1 Knapsack?**
No. Items cannot be split, so taking the highest value/weight ratio first can block a globally better combination. Example: items A(w=1,v=6), B(w=2,v=10), C(w=3,v=12), W=5 — greedy picks A+B=16, DP finds B+C=22.

---

**Q13: 0/1 Knapsack vs Unbounded Knapsack?**

| Aspect | 0/1 Knapsack | Unbounded Knapsack |
|--------|-------------|-------------------|
| Item usage | At most once | Unlimited |
| 1D traversal | Right to left | Left to right |
| Example | Classic knapsack | Coin change |

The only code difference in 1D is the inner loop direction.

---

**Q14: How does Word Break DP work?**
`dp[i]` = true if s[0..i-1] can be segmented. For each i, check all j < i: if `dp[j]` is true AND `s[j..i-1]` is in the dictionary, set `dp[i] = true`. Base case: `dp[0] = true`. Time O(n²).

---

**Q15: When prefer top-down over bottom-up?**
Top-down when only a fraction of subproblems are needed (sparse computation) or the recursion is more intuitive. Bottom-up when all subproblems are visited, n is large (stack overflow risk), or space optimization is needed.

---

**Q16: Why does Coin Change (count ways, order doesn't matter) use coin as outer loop?**
The outer-coin loop ensures each combination is counted once. Since we fix available coins before iterating amounts, {1,2} and {2,1} are never counted separately. Swapping loops (amount outer) counts each ordering separately — permutations instead of combinations.

---

**Q17: What are the limitations of DP?**
(1) **Pseudo-polynomial** — Knapsack is O(n×W); W could be exponentially large in input bits. (2) **Space** — O(n²) or O(n×W) can be prohibitive. (3) **No optimal substructure** — longest simple path in a graph is NP-hard. (4) **Wrong state** — misidentifying state leads to wrong answers.

---

**Q18: How do you debug a wrong DP answer?**
1. Verify base cases manually. 2. Trace a small (3-4 element) example and print the full table. 3. Check iteration order — all dependencies computed before use. 4. Check off-by-one errors. 5. Re-derive the recurrence on paper.

---

**Q19: What DP problems are commonly asked at interviews?**

**Tier 1 (must know):** LCS, 0/1 Knapsack variants (Subset Sum, Partition), Coin Change (both), Edit Distance, House Robber, Fibonacci/Climbing Stairs.

**Tier 2 (should know):** Word Break, Decode Ways, Longest Palindromic Substring, Unique Paths.

**Tier 3 (awareness):** DP on trees, Interval DP, Bitmask DP.

---

**Q20: How do you approach an unseen DP problem in an interview?**
1. Identify it's DP (min/max/count/feasibility + combinatorial space). 2. Define the state explicitly. 3. Write the recurrence on paper with all cases. 4. Identify base cases. 5. Code top-down (memoization) first — easier to verify. 6. Trace a small example. 7. Optimize if asked. State reasoning out loud throughout.

---

## 10. Complexity Reference Sheet

| Problem | Time | Space | Optimized Space |
|---------|------|-------|-----------------|
| Fibonacci | O(n) | O(n) | O(1) |
| Climbing Stairs | O(n) | O(n) | O(1) |
| House Robber | O(n) | O(n) | O(1) |
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
| Longest Palindromic Subsequence | O(n²) | O(n²) | O(n) |
| Longest Palindromic Substring | O(n²) | O(1) | O(1) |
| Distinct Subsequences | O(m·n) | O(m·n) | O(n) |
| House Robber III | O(n) | O(h) | O(h) |

*W = target/capacity, h = tree height*

---

## 11. Quick Revision Cheat Sheet

```
DP Identification:
  ✓ Ask for min/max/count/possible?
  ✓ Optimal substructure?
  ✓ Overlapping subproblems?
  → YES to all = DP

State Design:
  → Identify all changing parameters
  → dp[i] means: "best answer for [first i elements / amount i / prefix i]"

Recurrence by Problem Type:
  Linear (1D):       dp[i] = f(dp[i-1], dp[i-2])
  String/Grid (2D):  dp[i][j] = f(dp[i-1][j], dp[i][j-1], dp[i-1][j-1])
  Knapsack:          dp[i][w] = max(dp[i-1][w], dp[i-1][w-wi]+vi)
  Interval:          dp[i][j] = min/max over k of f(dp[i][k], dp[k+1][j])
  Counting:          dp[i] += dp[i-choice] for all valid choices

Common Iteration Orders:
  Most 1D:                left to right
  0/1 Knapsack 1D:        right to left (reverse)
  Unbounded Knapsack 1D:  left to right
  Interval DP:            by increasing interval length

Space Optimization:
  dp[i] = f(dp[i-1], dp[i-2])  → two variables (prev1, prev2)
  dp[i][j] = f(dp[i-1][...])   → 1D array + careful direction
  Need reconstruction?          → Keep full table
```
