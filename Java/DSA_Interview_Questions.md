# Data Structures & Algorithms (DSA) Interview Questions

> **Goal**: Crack coding rounds in Java developer interviews with simple, clear explanations.

---

## Table of Contents
1. [Arrays & Strings](#arrays--strings)
2. [Linked List](#linked-list)
3. [Stack & Queue](#stack--queue)
4. [Binary Search](#binary-search)
5. [Sorting Algorithms](#sorting-algorithms)
6. [HashMap Problems](#hashmap-problems)
7. [Two Pointers](#two-pointers)
8. [Trees](#trees)
9. [Common Coding Patterns](#common-coding-patterns)
10. [Sliding Window Pattern](#sliding-window-pattern)

---

## Arrays & Strings

### Q1: Find the largest and second largest element

```java
public static void find(int[] arr) {
    int largest = Integer.MIN_VALUE, secondLargest = Integer.MIN_VALUE;
    for (int num : arr) {
        if (num > largest) { secondLargest = largest; largest = num; }
        else if (num > secondLargest && num != largest) secondLargest = num;
    }
    System.out.println("Largest: " + largest + ", Second: " + secondLargest);
}
```
**Time:** O(n) | **Space:** O(1)

---

### Q2: Check if array has duplicates

Use a HashSet — if `add()` returns false, it's a duplicate.

```java
public static boolean hasDuplicate(int[] arr) {
    HashSet<Integer> seen = new HashSet<>();
    for (int num : arr) if (!seen.add(num)) return true;
    return false;
}
```
**Time:** O(n) | **Space:** O(n)

---

### Q3: Reverse an array / string

```java
public static void reverseArray(int[] arr) {
    int left = 0, right = arr.length - 1;
    while (left < right) { int t = arr[left]; arr[left++] = arr[right]; arr[right--] = t; }
}

public static String reverseString(String s) {
    return new StringBuilder(s).reverse().toString();
}
```

---

### Q4: Find missing number in array (1 to N)

Sum formula: expected `N*(N+1)/2` minus actual sum = missing number.

```java
public static int findMissing(int[] arr, int n) {
    int actualSum = 0;
    for (int num : arr) actualSum += num;
    return n * (n + 1) / 2 - actualSum;
}
```

---

### Q5: Check if string is Palindrome

```java
public static boolean isPalindrome(String s) {
    s = s.toLowerCase().replaceAll("[^a-z0-9]", "");
    int left = 0, right = s.length() - 1;
    while (left < right) {
        if (s.charAt(left++) != s.charAt(right--)) return false;
    }
    return true;
}
```

---

### Q6: Two Sum

Use a HashMap: for each number, check if `(target - number)` was seen before.

```java
public static int[] twoSum(int[] nums, int target) {
    HashMap<Integer, Integer> map = new HashMap<>();
    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];
        if (map.containsKey(complement)) return new int[]{map.get(complement), i};
        map.put(nums[i], i);
    }
    return new int[]{};
}
```
**Time:** O(n) | **Space:** O(n)

---

### Q7: Maximum Subarray Sum (Kadane's Algorithm)

Keep a running sum; reset to current element if the running sum goes negative.

```java
public static int maxSum(int[] arr) {
    int maxSoFar = arr[0], currentSum = arr[0];
    for (int i = 1; i < arr.length; i++) {
        currentSum = Math.max(arr[i], currentSum + arr[i]);
        maxSoFar = Math.max(maxSoFar, currentSum);
    }
    return maxSoFar;
}
// {-2,1,-3,4,-1,2,1,-5,4} → 6 (subarray [4,-1,2,1])
```

---

## Linked List

### Q8: Reverse a Linked List

Reverse the arrows: track previous, current, and next nodes.

```java
public static Node reverse(Node head) {
    Node prev = null, current = head;
    while (current != null) {
        Node next = current.next;
        current.next = prev;
        prev = current;
        current = next;
    }
    return prev;
}
```

---

### Q9: Detect a cycle (Floyd's Algorithm)

Slow pointer moves 1 step, fast moves 2. If they meet, there's a cycle.

```java
public static boolean hasCycle(Node head) {
    Node slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) return true;
    }
    return false;
}
```

---

### Q10: Find middle of Linked List

Same slow/fast trick — when fast reaches end, slow is at middle.

```java
public static Node findMiddle(Node head) {
    Node slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
    }
    return slow;
}
```

---

## Stack & Queue

### Q11: Implement a Stack using an Array

Stack = LIFO (Last In, First Out).

```java
public class Stack {
    private int[] data;
    private int top = -1;
    public Stack(int size) { data = new int[size]; }
    public void push(int val) { data[++top] = val; }
    public int pop() { return data[top--]; }
    public int peek() { return data[top]; }
    public boolean isEmpty() { return top == -1; }
}
```

---

### Q12: Valid Parentheses

Push opening brackets; pop and match on closing brackets. Stack must be empty at the end.

```java
public static boolean isValid(String s) {
    Stack<Character> stack = new Stack<>();
    for (char c : s.toCharArray()) {
        if (c == '(' || c == '{' || c == '[') stack.push(c);
        else {
            if (stack.isEmpty()) return false;
            char top = stack.pop();
            if (c == ')' && top != '(') return false;
            if (c == '}' && top != '{') return false;
            if (c == ']' && top != '[') return false;
        }
    }
    return stack.isEmpty();
}
```

---

## Binary Search

### Q13: Binary Search (Iterative)

In a sorted array, check the middle; go left if target is smaller, right if larger.

```java
public static int search(int[] arr, int target) {
    int left = 0, right = arr.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2; // avoids overflow
        if (arr[mid] == target) return mid;
        else if (arr[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;
}
```
**Time:** O(log n)

---

## Sorting Algorithms

### Q14: Bubble Sort

Compare adjacent elements and swap if out of order. Largest bubbles to end each pass.

```java
public static void bubbleSort(int[] arr) {
    int n = arr.length;
    for (int i = 0; i < n - 1; i++) {
        boolean swapped = false;
        for (int j = 0; j < n - i - 1; j++) {
            if (arr[j] > arr[j + 1]) {
                int t = arr[j]; arr[j] = arr[j + 1]; arr[j + 1] = t;
                swapped = true;
            }
        }
        if (!swapped) break;
    }
}
```
**Time:** O(n²) worst, O(n) best | **Space:** O(1)

---

### Q15: Quick Sort

Pick a pivot, partition smaller elements left and larger right, then recurse.

```java
public static void sort(int[] arr, int low, int high) {
    if (low < high) {
        int pivotIndex = partition(arr, low, high);
        sort(arr, low, pivotIndex - 1);
        sort(arr, pivotIndex + 1, high);
    }
}

private static int partition(int[] arr, int low, int high) {
    int pivot = arr[high], i = low - 1;
    for (int j = low; j < high; j++) {
        if (arr[j] <= pivot) { i++; int t = arr[i]; arr[i] = arr[j]; arr[j] = t; }
    }
    int t = arr[i + 1]; arr[i + 1] = arr[high]; arr[high] = t;
    return i + 1;
}
```
**Time:** O(n log n) avg, O(n²) worst | **Space:** O(log n)

---

## HashMap Problems

### Q16: Count character frequency

```java
public static HashMap<Character, Integer> count(String s) {
    HashMap<Character, Integer> freq = new HashMap<>();
    for (char c : s.toCharArray()) freq.put(c, freq.getOrDefault(c, 0) + 1);
    return freq;
}
```

---

### Q17: Check if two strings are Anagrams

```java
// O(n) approach using frequency array
public static boolean isAnagram(String s, String t) {
    if (s.length() != t.length()) return false;
    int[] count = new int[26];
    for (char c : s.toCharArray()) count[c - 'a']++;
    for (char c : t.toCharArray()) count[c - 'a']--;
    for (int n : count) if (n != 0) return false;
    return true;
}
```

---

## Two Pointers

### Q18: Remove duplicates from sorted array

Slow pointer tracks the unique position; fast pointer scans ahead.

```java
public static int removeDuplicates(int[] nums) {
    if (nums.length == 0) return 0;
    int slow = 0;
    for (int fast = 1; fast < nums.length; fast++) {
        if (nums[fast] != nums[slow]) nums[++slow] = nums[fast];
    }
    return slow + 1;
}
```

---

## Trees

### Q19: Binary Tree traversals

- **Inorder** (Left → Root → Right) — gives sorted output for BST
- **Preorder** (Root → Left → Right) — used to copy/serialize a tree
- **Postorder** (Left → Right → Root) — used to delete a tree

```java
public static void inorder(Node root) {
    if (root == null) return;
    inorder(root.left);
    System.out.print(root.data + " ");
    inorder(root.right);
}

public static void preorder(Node root) {
    if (root == null) return;
    System.out.print(root.data + " ");
    preorder(root.left);
    preorder(root.right);
}

public static void postorder(Node root) {
    if (root == null) return;
    postorder(root.left);
    postorder(root.right);
    System.out.print(root.data + " ");
}
```

---

### Q20: Find height of Binary Tree

```java
public static int height(Node root) {
    if (root == null) return 0;
    return Math.max(height(root.left), height(root.right)) + 1;
}
```

---

## Common Coding Patterns

### Q21: FizzBuzz

```java
for (int i = 1; i <= n; i++) {
    if (i % 15 == 0) System.out.println("FizzBuzz");
    else if (i % 3 == 0) System.out.println("Fizz");
    else if (i % 5 == 0) System.out.println("Buzz");
    else System.out.println(i);
}
```

---

### Q22: Fibonacci Series

```java
// Iterative — O(n) time, O(1) space (preferred)
public static long fibonacci(int n) {
    if (n <= 1) return n;
    long a = 0, b = 1;
    for (int i = 2; i <= n; i++) { long t = a + b; a = b; b = t; }
    return b;
}
// Recursive is O(2^n) — avoid in interviews unless asked
```

---

### Q23: Check if number is Prime

```java
public static boolean isPrime(int n) {
    if (n < 2) return false;
    if (n == 2) return true;
    if (n % 2 == 0) return false;
    for (int i = 3; i <= Math.sqrt(n); i += 2)
        if (n % i == 0) return false;
    return true;
}
```

---

## Sliding Window Pattern

**Template:**
1. Initialize `left=0`, result, and a window state (Map, counter, etc.)
2. Expand window by moving `right`
3. Shrink from `left` when the constraint is violated
4. Update result after each valid window

### Fixed-Size Window — Max sum of size k

```java
public int maxSumSubarray(int[] nums, int k) {
    int windowSum = 0;
    for (int i = 0; i < k; i++) windowSum += nums[i];
    int maxSum = windowSum;
    for (int i = k; i < nums.length; i++) {
        windowSum += nums[i] - nums[i - k];
        maxSum = Math.max(maxSum, windowSum);
    }
    return maxSum;
}
```

### Variable-Size Window — Longest substring without repeating characters

```java
public int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> lastSeen = new HashMap<>();
    int left = 0, maxLen = 0;
    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        if (lastSeen.containsKey(c) && lastSeen.get(c) >= left)
            left = lastSeen.get(c) + 1;
        lastSeen.put(c, right);
        maxLen = Math.max(maxLen, right - left + 1);
    }
    return maxLen;
}
```

**Q: What is the sliding window pattern and when do you use it?**
It maintains a contiguous subarray between two pointers, expanding right and shrinking left. Avoids O(n²) nested loops for problems like max subarray sum, longest substring with a constraint, or minimum window substring.

---

## Time Complexity Cheat Sheet

| Algorithm | Best | Average | Worst | Space |
|---|---|---|---|---|
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) |
| Binary Search | O(1) | O(log n) | O(log n) | O(1) |
| Linear Search | O(1) | O(n) | O(n) | O(1) |

| Data Structure | Access | Search | Insert | Delete |
|---|---|---|---|---|
| Array | O(1) | O(n) | O(n) | O(n) |
| LinkedList | O(n) | O(n) | O(1) | O(1) |
| Stack / Queue | O(n) | O(n) | O(1) | O(1) |
| HashMap | - | O(1) avg | O(1) avg | O(1) avg |
| BST | O(log n) | O(log n) | O(log n) | O(log n) |

---

## Interview Tips

1. **Clarify the problem** — ask about edge cases (empty input, negatives, duplicates)
2. **Think out loud** — explain your approach before coding
3. **Start with brute force**, then optimize
4. **State time/space complexity** even if not asked
5. **Test with examples** — trace through manually
6. **Handle edge cases** — null, empty array, single element

---

## Quick Reference: Most Asked Problems

| Problem | Technique | Time |
|---|---|---|
| Two Sum | HashMap | O(n) |
| Max Subarray | Kadane's | O(n) |
| Valid Parentheses | Stack | O(n) |
| Reverse Linked List | Two pointers | O(n) |
| Binary Search | Divide & Conquer | O(log n) |
| Detect Cycle (LL) | Floyd's | O(n) |
| Palindrome Check | Two pointers | O(n) |
| Find Missing Number | Math formula | O(n) |

---

*Last Updated: 2026-06-18*
