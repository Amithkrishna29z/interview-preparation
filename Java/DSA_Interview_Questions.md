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
10. [Graph Algorithms](#graph-algorithms)
11. [Sliding Window Pattern](#sliding-window-pattern)
12. [Heap / Priority Queue Patterns](#heap--priority-queue-patterns)
13. [Trie (Prefix Tree)](#trie-prefix-tree)

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

## Graph Algorithms

### Graph Representations

```java
// Adjacency List (preferred for sparse graphs)
class Graph {
    private List<List<Integer>> adj;
    public Graph(int V) {
        adj = new ArrayList<>();
        for (int i = 0; i < V; i++) adj.add(new ArrayList<>());
    }
    public void addEdge(int u, int v) { adj.get(u).add(v); }
}
```

| Representation | Space | Edge Lookup | Best For |
|---|---|---|---|
| Adjacency List | O(V+E) | O(degree) | Sparse graphs |
| Adjacency Matrix | O(V²) | O(1) | Dense graphs |

---

### BFS (Breadth-First Search)

Explores level by level using a Queue. Finds shortest path in unweighted graphs. **Time:** O(V+E)

```java
public int bfsShortestPath(List<List<Integer>> adj, int src, int dest) {
    boolean[] visited = new boolean[adj.size()];
    Queue<Integer> queue = new LinkedList<>();
    visited[src] = true;
    queue.offer(src);
    int steps = 0;
    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            int node = queue.poll();
            if (node == dest) return steps;
            for (int neighbor : adj.get(node))
                if (!visited[neighbor]) { visited[neighbor] = true; queue.offer(neighbor); }
        }
        steps++;
    }
    return -1;
}
```

---

### DFS (Depth-First Search)

Explores as far as possible before backtracking. Used for cycle detection, topological sort, connected components. **Time:** O(V+E)

```java
public void dfsRecursive(List<List<Integer>> adj, int node, boolean[] visited) {
    visited[node] = true;
    System.out.print(node + " ");
    for (int neighbor : adj.get(node))
        if (!visited[neighbor]) dfsRecursive(adj, neighbor, visited);
}
```

---

### Cycle Detection

**Undirected:** DFS with parent tracking — a back edge to a non-parent node means a cycle.

```java
private boolean dfsCycle(List<List<Integer>> adj, int node, int parent, boolean[] visited) {
    visited[node] = true;
    for (int neighbor : adj.get(node)) {
        if (!visited[neighbor]) { if (dfsCycle(adj, neighbor, node, visited)) return true; }
        else if (neighbor != parent) return true;
    }
    return false;
}
```

**Directed:** DFS with an `inStack` array — if you hit a node already in the current path, it's a cycle.

```java
private boolean dfsCycleDirected(List<List<Integer>> adj, int node,
                                  boolean[] visited, boolean[] inStack) {
    visited[node] = inStack[node] = true;
    for (int neighbor : adj.get(node)) {
        if (!visited[neighbor] && dfsCycleDirected(adj, neighbor, visited, inStack)) return true;
        else if (inStack[neighbor]) return true;
    }
    inStack[node] = false;
    return false;
}
```

---

### Topological Sort — Kahn's Algorithm (BFS)

Only valid for DAGs. If result size != n, the graph has a cycle.

```java
public List<Integer> topoSortKahn(List<List<Integer>> adj, int n) {
    int[] indegree = new int[n];
    for (int u = 0; u < n; u++) for (int v : adj.get(u)) indegree[v]++;
    Queue<Integer> queue = new LinkedList<>();
    for (int i = 0; i < n; i++) if (indegree[i] == 0) queue.offer(i);
    List<Integer> result = new ArrayList<>();
    while (!queue.isEmpty()) {
        int node = queue.poll();
        result.add(node);
        for (int neighbor : adj.get(node)) if (--indegree[neighbor] == 0) queue.offer(neighbor);
    }
    return result.size() == n ? result : new ArrayList<>();
}
```
**Use case:** Course prerequisite ordering, build system dependencies.

---

### Dijkstra's Algorithm (Weighted Shortest Path)

```java
public int[] dijkstra(int[][] graph, int src) {
    int n = graph.length;
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[src] = 0;
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
    pq.offer(new int[]{0, src});
    while (!pq.isEmpty()) {
        int[] curr = pq.poll();
        int d = curr[0], u = curr[1];
        if (d > dist[u]) continue;
        for (int v = 0; v < n; v++) {
            if (graph[u][v] > 0 && dist[u] + graph[u][v] < dist[v]) {
                dist[v] = dist[u] + graph[u][v];
                pq.offer(new int[]{dist[v], v});
            }
        }
    }
    return dist;
}
```
**Time:** O((V+E) log V) | Does NOT work with negative weights (use Bellman-Ford).

---

### Union-Find (Disjoint Set Union)

Efficiently answers "are these two nodes connected?" — used for cycle detection and Kruskal's MST.

```java
class UnionFind {
    private int[] parent, rank;
    public UnionFind(int n) {
        parent = new int[n]; rank = new int[n];
        for (int i = 0; i < n; i++) parent[i] = i;
    }
    public int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]); // path compression
        return parent[x];
    }
    public boolean union(int x, int y) {
        int px = find(x), py = find(y);
        if (px == py) return false;
        if (rank[px] < rank[py]) { int t = px; px = py; py = t; }
        parent[py] = px;
        if (rank[px] == rank[py]) rank[px]++;
        return true;
    }
    public boolean connected(int x, int y) { return find(x) == find(y); }
}
```

---

### Graph Interview Q&A

**Q: What is the difference between BFS and DFS?**
BFS explores layer by layer using a queue — guarantees shortest path in unweighted graphs. DFS goes deep first using a stack/recursion — better for cycle detection, topological sort, and path finding.

**Q: How do you detect a cycle in a directed graph?**
Use DFS with an `inStack` boolean array. Mark nodes as in-stack on entry and remove on backtrack. If you encounter a node already in-stack, a back edge (cycle) exists.

**Q: What is topological sort and when do you use it?**
It orders vertices so every directed edge (u→v) has u before v. Only works for DAGs. Use Kahn's algorithm (BFS-based) — it also detects cycles. Common use cases: course prerequisites, build dependencies.

**Q: When to use Dijkstra vs BFS?**
BFS for unweighted graphs — O(V+E). Dijkstra for weighted graphs with non-negative weights — O((V+E) log V). For negative weights use Bellman-Ford.

**Q: What is Union-Find used for?**
Efficiently tracks connected components with near-O(1) union and find operations. Used for detecting cycles in undirected graphs, counting components, and Kruskal's MST.

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

## Heap / Priority Queue Patterns

### Top K Elements — Kth Largest

Use a **min-heap of size K**: scan all elements; when heap exceeds K, remove the minimum. The heap top is the Kth largest.

```java
public int findKthLargest(int[] nums, int k) {
    PriorityQueue<Integer> minHeap = new PriorityQueue<>();
    for (int num : nums) {
        minHeap.offer(num);
        if (minHeap.size() > k) minHeap.poll();
    }
    return minHeap.peek();
}
// Time: O(n log k), Space: O(k)
```

### Merge K Sorted Lists

```java
public ListNode mergeKLists(ListNode[] lists) {
    PriorityQueue<ListNode> minHeap = new PriorityQueue<>(Comparator.comparingInt(n -> n.val));
    for (ListNode node : lists) if (node != null) minHeap.offer(node);
    ListNode dummy = new ListNode(0), curr = dummy;
    while (!minHeap.isEmpty()) {
        ListNode node = minHeap.poll();
        curr.next = node; curr = curr.next;
        if (node.next != null) minHeap.offer(node.next);
    }
    return dummy.next;
}
// Time: O(n log k) where n = total nodes, k = number of lists
```

### Median of Data Stream

Use two heaps: a max-heap for the lower half and a min-heap for the upper half, keeping sizes balanced.

```java
class MedianFinder {
    private PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());
    private PriorityQueue<Integer> minHeap = new PriorityQueue<>();

    public void addNum(int num) {
        maxHeap.offer(num);
        minHeap.offer(maxHeap.poll());
        if (maxHeap.size() < minHeap.size()) maxHeap.offer(minHeap.poll());
    }

    public double findMedian() {
        if (maxHeap.size() > minHeap.size()) return maxHeap.peek();
        return (maxHeap.peek() + minHeap.peek()) / 2.0;
    }
}
```

**Q: When do you use a min-heap vs max-heap for Top-K problems?**
For K largest: use a min-heap of size K (discard minimums, keep K largest). For K smallest: use a max-heap of size K (discard maximums, keep K smallest).

---

## Trie (Prefix Tree)

A tree where each node represents one character. Efficient for prefix-based lookups.

```java
class TrieNode {
    Map<Character, TrieNode> children = new HashMap<>();
    boolean isEndOfWord = false;
}

class Trie {
    private TrieNode root = new TrieNode();

    public void insert(String word) {
        TrieNode curr = root;
        for (char c : word.toCharArray()) {
            curr.children.putIfAbsent(c, new TrieNode());
            curr = curr.children.get(c);
        }
        curr.isEndOfWord = true;
    }

    public boolean search(String word) {
        TrieNode curr = root;
        for (char c : word.toCharArray()) {
            if (!curr.children.containsKey(c)) return false;
            curr = curr.children.get(c);
        }
        return curr.isEndOfWord;
    }

    public boolean startsWith(String prefix) {
        TrieNode curr = root;
        for (char c : prefix.toCharArray()) {
            if (!curr.children.containsKey(c)) return false;
            curr = curr.children.get(c);
        }
        return true;
    }
}
```
**Time:** O(m) per insert/search where m = word length
**Use cases:** Autocomplete, spell checkers, IP routing (longest prefix match)

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
| Kth Largest | Min-heap | O(n log k) |
| Topological Sort | Kahn's BFS | O(V+E) |

---

*Last Updated: 2026-06-18*
