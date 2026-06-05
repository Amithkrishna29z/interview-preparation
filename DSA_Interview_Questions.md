# Data Structures & Algorithms (DSA) Interview Questions

> 🎯 **Goal**: Crack coding rounds in Java developer interviews with simple, clear explanations.

---

## Table of Contents
1. [Arrays & Strings](#arrays--strings)
2. [Linked List](#linked-list)
3. [Stack & Queue](#stack--queue)
4. [Binary Search](#binary-search)
5. [Recursion & Backtracking](#recursion--backtracking)
6. [Sorting Algorithms](#sorting-algorithms)
7. [HashMap Problems](#hashmap-problems)
8. [Two Pointers](#two-pointers)
9. [Trees](#trees)
10. [Common Coding Patterns](#common-coding-patterns)

---

## Arrays & Strings

### Q1: Find the largest and second largest element in an array

**Easy Explanation:** Loop through the array keeping track of two variables — the biggest and the second biggest.

```java
public class LargestSecondLargest {
    public static void find(int[] arr) {
        int largest = Integer.MIN_VALUE;
        int secondLargest = Integer.MIN_VALUE;

        for (int num : arr) {
            if (num > largest) {
                secondLargest = largest; // old largest becomes second
                largest = num;
            } else if (num > secondLargest && num != largest) {
                secondLargest = num;
            }
        }

        System.out.println("Largest: " + largest);
        System.out.println("Second Largest: " + secondLargest);
    }

    public static void main(String[] args) {
        int[] arr = {12, 35, 1, 10, 34, 1};
        find(arr); // Largest: 35, Second Largest: 34
    }
}
```

**Time Complexity:** O(n) | **Space Complexity:** O(1)

---

### Q2: Check if array has duplicates

**Easy Explanation:** Use a HashSet — if adding an element returns false, it's a duplicate.

```java
import java.util.HashSet;

public class CheckDuplicates {
    public static boolean hasDuplicate(int[] arr) {
        HashSet<Integer> seen = new HashSet<>();
        for (int num : arr) {
            if (!seen.add(num)) {  // add() returns false if already present
                return true;
            }
        }
        return false;
    }

    public static void main(String[] args) {
        System.out.println(hasDuplicate(new int[]{1, 2, 3, 4}));     // false
        System.out.println(hasDuplicate(new int[]{1, 2, 3, 1}));     // true
    }
}
```

**Time Complexity:** O(n) | **Space Complexity:** O(n)

---

### Q3: Reverse an array / string

```java
public class ReverseArray {
    // Reverse array in-place
    public static void reverseArray(int[] arr) {
        int left = 0, right = arr.length - 1;
        while (left < right) {
            int temp = arr[left];
            arr[left] = arr[right];
            arr[right] = temp;
            left++;
            right--;
        }
    }

    // Reverse a string
    public static String reverseString(String s) {
        return new StringBuilder(s).reverse().toString();
        // OR manually:
        // char[] chars = s.toCharArray();
        // for (int i = 0, j = s.length()-1; i < j; i++, j--) {
        //     char temp = chars[i]; chars[i] = chars[j]; chars[j] = temp;
        // }
        // return new String(chars);
    }

    public static void main(String[] args) {
        int[] arr = {1, 2, 3, 4, 5};
        reverseArray(arr); // [5, 4, 3, 2, 1]

        System.out.println(reverseString("hello")); // "olleh"
    }
}
```

---

### Q4: Find missing number in array (1 to N)

**Easy Explanation:** Sum of 1 to N is `N*(N+1)/2`. Subtract actual sum to find missing.

```java
public class MissingNumber {
    public static int findMissing(int[] arr, int n) {
        int expectedSum = n * (n + 1) / 2;
        int actualSum = 0;
        for (int num : arr) actualSum += num;
        return expectedSum - actualSum;
    }

    public static void main(String[] args) {
        int[] arr = {1, 2, 4, 5, 6};  // Missing 3
        System.out.println(findMissing(arr, 6));  // 3
    }
}
```

---

### Q5: Check if string is Palindrome

**Easy Explanation:** Compare characters from both ends moving inward.

```java
public class Palindrome {
    public static boolean isPalindrome(String s) {
        s = s.toLowerCase().replaceAll("[^a-z0-9]", ""); // clean up
        int left = 0, right = s.length() - 1;
        while (left < right) {
            if (s.charAt(left) != s.charAt(right)) return false;
            left++;
            right--;
        }
        return true;
    }

    public static void main(String[] args) {
        System.out.println(isPalindrome("racecar"));     // true
        System.out.println(isPalindrome("A man a plan a canal Panama")); // true
        System.out.println(isPalindrome("hello"));       // false
    }
}
```

---

### Q6: Two Sum — find two numbers that add up to target

**Easy Explanation:** Use a HashMap to store what we've seen. For each number, check if (target - number) exists.

```java
import java.util.HashMap;

public class TwoSum {
    public static int[] twoSum(int[] nums, int target) {
        HashMap<Integer, Integer> map = new HashMap<>(); // value -> index

        for (int i = 0; i < nums.length; i++) {
            int complement = target - nums[i];
            if (map.containsKey(complement)) {
                return new int[]{map.get(complement), i};
            }
            map.put(nums[i], i);
        }
        return new int[]{};
    }

    public static void main(String[] args) {
        int[] result = twoSum(new int[]{2, 7, 11, 15}, 9);
        System.out.println(result[0] + ", " + result[1]); // 0, 1 (2+7=9)
    }
}
```

**Time Complexity:** O(n) | **Space Complexity:** O(n)

---

### Q7: Maximum Subarray Sum (Kadane's Algorithm)

**Easy Explanation:** Keep a running sum. If it goes negative, reset to 0 and start fresh.

```java
public class MaxSubarraySum {
    public static int maxSum(int[] arr) {
        int maxSoFar = arr[0];
        int currentSum = arr[0];

        for (int i = 1; i < arr.length; i++) {
            currentSum = Math.max(arr[i], currentSum + arr[i]);
            maxSoFar = Math.max(maxSoFar, currentSum);
        }
        return maxSoFar;
    }

    public static void main(String[] args) {
        int[] arr = {-2, 1, -3, 4, -1, 2, 1, -5, 4};
        System.out.println(maxSum(arr)); // 6 (subarray [4,-1,2,1])
    }
}
```

---

## Linked List

### Q8: Reverse a Linked List

**Easy Explanation:** Reverse the arrows. Keep track of previous, current, and next nodes.

```java
public class ReverseLinkedList {
    static class Node {
        int data;
        Node next;
        Node(int data) { this.data = data; }
    }

    public static Node reverse(Node head) {
        Node prev = null;
        Node current = head;

        while (current != null) {
            Node nextNode = current.next; // save next
            current.next = prev;          // reverse arrow
            prev = current;              // move prev forward
            current = nextNode;          // move current forward
        }
        return prev; // prev is new head
    }

    public static void print(Node head) {
        while (head != null) {
            System.out.print(head.data + " -> ");
            head = head.next;
        }
        System.out.println("null");
    }

    public static void main(String[] args) {
        Node head = new Node(1);
        head.next = new Node(2);
        head.next.next = new Node(3);
        head.next.next.next = new Node(4);

        print(head);              // 1 -> 2 -> 3 -> 4 -> null
        head = reverse(head);
        print(head);              // 4 -> 3 -> 2 -> 1 -> null
    }
}
```

---

### Q9: Detect a cycle in Linked List (Floyd's Algorithm)

**Easy Explanation:** Use two pointers — slow (1 step) and fast (2 steps). If they ever meet, there's a cycle.

```java
public class DetectCycle {
    static class Node {
        int data;
        Node next;
        Node(int data) { this.data = data; }
    }

    public static boolean hasCycle(Node head) {
        Node slow = head;
        Node fast = head;

        while (fast != null && fast.next != null) {
            slow = slow.next;        // 1 step
            fast = fast.next.next;   // 2 steps
            if (slow == fast) return true; // They met = cycle!
        }
        return false; // fast reached end = no cycle
    }
}
```

---

### Q10: Find middle of Linked List

**Easy Explanation:** Slow moves 1 step, fast moves 2 steps. When fast reaches end, slow is at middle.

```java
public static Node findMiddle(Node head) {
    Node slow = head;
    Node fast = head;

    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
    }
    return slow; // slow is at middle
}
```

---

## Stack & Queue

### Q11: Implement a Stack using an Array

**Easy Explanation:** Stack = LIFO (Last In, First Out). Like a stack of plates — add and remove from top.

```java
public class Stack {
    private int[] data;
    private int top = -1;

    public Stack(int size) {
        data = new int[size];
    }

    public void push(int val) {
        if (top == data.length - 1) throw new RuntimeException("Stack Full");
        data[++top] = val;
    }

    public int pop() {
        if (top == -1) throw new RuntimeException("Stack Empty");
        return data[top--];
    }

    public int peek() {
        return data[top];
    }

    public boolean isEmpty() {
        return top == -1;
    }
}
```

---

### Q12: Valid Parentheses (Classic Stack Problem)

**Easy Explanation:** Push opening brackets, pop when you see closing bracket. At end, stack should be empty.

```java
import java.util.Stack;

public class ValidParentheses {
    public static boolean isValid(String s) {
        Stack<Character> stack = new Stack<>();

        for (char c : s.toCharArray()) {
            if (c == '(' || c == '{' || c == '[') {
                stack.push(c);
            } else {
                if (stack.isEmpty()) return false;
                char top = stack.pop();
                if (c == ')' && top != '(') return false;
                if (c == '}' && top != '{') return false;
                if (c == ']' && top != '[') return false;
            }
        }
        return stack.isEmpty();
    }

    public static void main(String[] args) {
        System.out.println(isValid("()[]{}"));  // true
        System.out.println(isValid("([)]"));    // false
        System.out.println(isValid("{[]}"));    // true
    }
}
```

---

## Binary Search

### Q13: Binary Search (Iterative)

**Easy Explanation:** In a sorted array, always check the middle. If target is smaller, go left; if larger, go right.

```java
public class BinarySearch {
    public static int search(int[] arr, int target) {
        int left = 0, right = arr.length - 1;

        while (left <= right) {
            int mid = left + (right - left) / 2; // avoids overflow

            if (arr[mid] == target) return mid;
            else if (arr[mid] < target) left = mid + 1; // go right
            else right = mid - 1; // go left
        }
        return -1; // not found
    }

    public static void main(String[] args) {
        int[] arr = {1, 3, 5, 7, 9, 11};
        System.out.println(search(arr, 7));   // 3 (index)
        System.out.println(search(arr, 4));   // -1 (not found)
    }
}
```

**Time Complexity:** O(log n)

---

## Sorting Algorithms

### Q14: Bubble Sort

**Easy Explanation:** Compare adjacent elements and swap if they're in wrong order. Biggest element "bubbles" to the end each pass.

```java
public class BubbleSort {
    public static void sort(int[] arr) {
        int n = arr.length;
        for (int i = 0; i < n - 1; i++) {
            boolean swapped = false;
            for (int j = 0; j < n - i - 1; j++) {
                if (arr[j] > arr[j + 1]) {
                    int temp = arr[j];
                    arr[j] = arr[j + 1];
                    arr[j + 1] = temp;
                    swapped = true;
                }
            }
            if (!swapped) break; // Already sorted!
        }
    }
}
```

**Time:** O(n²) worst, O(n) best | **Space:** O(1)

---

### Q15: Quick Sort (Most Important)

**Easy Explanation:** Pick a pivot, put smaller elements on left, larger on right. Recursively sort both sides.

```java
public class QuickSort {
    public static void sort(int[] arr, int low, int high) {
        if (low < high) {
            int pivotIndex = partition(arr, low, high);
            sort(arr, low, pivotIndex - 1);
            sort(arr, pivotIndex + 1, high);
        }
    }

    private static int partition(int[] arr, int low, int high) {
        int pivot = arr[high]; // choose last as pivot
        int i = low - 1;       // i = index of smaller element

        for (int j = low; j < high; j++) {
            if (arr[j] <= pivot) {
                i++;
                int temp = arr[i]; arr[i] = arr[j]; arr[j] = temp;
            }
        }
        // Place pivot in correct position
        int temp = arr[i + 1]; arr[i + 1] = arr[high]; arr[high] = temp;
        return i + 1;
    }

    public static void main(String[] args) {
        int[] arr = {64, 25, 12, 22, 11};
        sort(arr, 0, arr.length - 1); // [11, 12, 22, 25, 64]
    }
}
```

**Time:** O(n log n) average, O(n²) worst | **Space:** O(log n)

---

## HashMap Problems

### Q16: Count frequency of characters in a string

```java
import java.util.HashMap;

public class CharFrequency {
    public static HashMap<Character, Integer> count(String s) {
        HashMap<Character, Integer> freq = new HashMap<>();
        for (char c : s.toCharArray()) {
            freq.put(c, freq.getOrDefault(c, 0) + 1);
        }
        return freq;
    }

    public static void main(String[] args) {
        System.out.println(count("hello")); // {h=1, e=1, l=2, o=1}
    }
}
```

---

### Q17: Check if two strings are Anagrams

**Easy Explanation:** Two strings are anagrams if they have same characters with same frequency.

```java
import java.util.Arrays;

public class Anagram {
    public static boolean isAnagram(String s, String t) {
        if (s.length() != t.length()) return false;

        // Method 1: Sort both and compare
        char[] a = s.toCharArray();
        char[] b = t.toCharArray();
        Arrays.sort(a);
        Arrays.sort(b);
        return Arrays.equals(a, b);
    }

    // Method 2: Count frequency (more efficient)
    public static boolean isAnagramFast(String s, String t) {
        if (s.length() != t.length()) return false;
        int[] count = new int[26];
        for (char c : s.toCharArray()) count[c - 'a']++;
        for (char c : t.toCharArray()) count[c - 'a']--;
        for (int n : count) if (n != 0) return false;
        return true;
    }

    public static void main(String[] args) {
        System.out.println(isAnagram("anagram", "nagaram")); // true
        System.out.println(isAnagram("rat", "car")); // false
    }
}
```

---

## Two Pointers

### Q18: Remove duplicates from sorted array

**Easy Explanation:** Use two pointers — slow pointer tracks unique elements position, fast pointer scans ahead.

```java
public class RemoveDuplicates {
    public static int removeDuplicates(int[] nums) {
        if (nums.length == 0) return 0;

        int slow = 0; // position to place next unique element

        for (int fast = 1; fast < nums.length; fast++) {
            if (nums[fast] != nums[slow]) {
                slow++;
                nums[slow] = nums[fast];
            }
        }
        return slow + 1; // length of unique array
    }

    public static void main(String[] args) {
        int[] arr = {1, 1, 2, 3, 3, 4};
        int len = removeDuplicates(arr);
        System.out.println("Unique count: " + len); // 4
        // arr is now [1, 2, 3, 4, ...]
    }
}
```

---

## Trees

### Q19: Binary Tree traversals (Inorder, Preorder, Postorder)

**Easy Explanation:**
- **Inorder** (Left → Root → Right) → gives sorted output for BST
- **Preorder** (Root → Left → Right) → used to copy/serialize tree
- **Postorder** (Left → Right → Root) → used to delete tree

```java
public class BinaryTree {
    static class Node {
        int data;
        Node left, right;
        Node(int data) { this.data = data; }
    }

    // Left -> Root -> Right
    public static void inorder(Node root) {
        if (root == null) return;
        inorder(root.left);
        System.out.print(root.data + " ");
        inorder(root.right);
    }

    // Root -> Left -> Right
    public static void preorder(Node root) {
        if (root == null) return;
        System.out.print(root.data + " ");
        preorder(root.left);
        preorder(root.right);
    }

    // Left -> Right -> Root
    public static void postorder(Node root) {
        if (root == null) return;
        postorder(root.left);
        postorder(root.right);
        System.out.print(root.data + " ");
    }

    public static void main(String[] args) {
        Node root = new Node(1);
        root.left = new Node(2);
        root.right = new Node(3);
        root.left.left = new Node(4);
        root.left.right = new Node(5);

        System.out.print("Inorder: ");   inorder(root);   // 4 2 5 1 3
        System.out.print("\nPreorder: "); preorder(root);  // 1 2 4 5 3
        System.out.print("\nPostorder: ");postorder(root); // 4 5 2 3 1
    }
}
```

---

### Q20: Find height of Binary Tree

```java
public static int height(Node root) {
    if (root == null) return 0;
    int leftHeight = height(root.left);
    int rightHeight = height(root.right);
    return Math.max(leftHeight, rightHeight) + 1;
}
```

---

## Common Coding Patterns

### Q21: FizzBuzz (Classic Interview Warm-up)

```java
public class FizzBuzz {
    public static void fizzBuzz(int n) {
        for (int i = 1; i <= n; i++) {
            if (i % 15 == 0) System.out.println("FizzBuzz");
            else if (i % 3 == 0) System.out.println("Fizz");
            else if (i % 5 == 0) System.out.println("Buzz");
            else System.out.println(i);
        }
    }
}
```

---

### Q22: Fibonacci Series

```java
public class Fibonacci {
    // Iterative (O(n) time, O(1) space)
    public static long fibonacci(int n) {
        if (n <= 1) return n;
        long a = 0, b = 1;
        for (int i = 2; i <= n; i++) {
            long temp = a + b;
            a = b;
            b = temp;
        }
        return b;
    }

    // Recursive (O(2^n) time - not recommended)
    public static long fibRecursive(int n) {
        if (n <= 1) return n;
        return fibRecursive(n - 1) + fibRecursive(n - 2);
    }
}
```

---

### Q23: Check if number is Prime

```java
public class PrimeCheck {
    public static boolean isPrime(int n) {
        if (n < 2) return false;
        if (n == 2) return true;
        if (n % 2 == 0) return false;

        for (int i = 3; i <= Math.sqrt(n); i += 2) {
            if (n % i == 0) return false;
        }
        return true;
    }

    public static void main(String[] args) {
        System.out.println(isPrime(17)); // true
        System.out.println(isPrime(18)); // false
    }
}
```

---

## 📊 Time Complexity Cheat Sheet

| Algorithm | Best | Average | Worst | Space |
|-----------|------|---------|-------|-------|
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) |
| Selection Sort | O(n²) | O(n²) | O(n²) | O(1) |
| Insertion Sort | O(n) | O(n²) | O(n²) | O(1) |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) |
| Binary Search | O(1) | O(log n) | O(log n) | O(1) |
| Linear Search | O(1) | O(n) | O(n) | O(1) |

| Data Structure | Access | Search | Insert | Delete |
|----------------|--------|--------|--------|--------|
| Array | O(1) | O(n) | O(n) | O(n) |
| LinkedList | O(n) | O(n) | O(1) | O(1) |
| Stack | O(n) | O(n) | O(1) | O(1) |
| Queue | O(n) | O(n) | O(1) | O(1) |
| HashMap | - | O(1) avg | O(1) avg | O(1) avg |
| BST | O(log n) | O(log n) | O(log n) | O(log n) |

---

## 🔑 Interview Tips for DSA

1. **Always clarify the problem** — ask about edge cases (empty array, negatives, duplicates)
2. **Think out loud** — explain your approach before coding
3. **Start with brute force**, then optimize
4. **Mention time/space complexity** even if not asked
5. **Test with examples** — trace through your code manually
6. **Handle edge cases** — null inputs, empty arrays, single element

---

## Quick Revision: Most Asked Problems

| Problem | Technique | Time |
|---------|-----------|------|
| Two Sum | HashMap | O(n) |
| Max Subarray | Kadane's | O(n) |
| Valid Parentheses | Stack | O(n) |
| Reverse Linked List | Two pointers | O(n) |
| Binary Search | Divide & Conquer | O(log n) |
| Detect Cycle | Floyd's | O(n) |
| Palindrome Check | Two pointers | O(n) |
| Find Missing Number | Math formula | O(n) |

---

## Graph Algorithms

### Graph Fundamentals
- **Directed vs Undirected graphs**
- **Weighted vs Unweighted graphs**
- **Adjacency Matrix** (O(V²) space, O(1) lookup) vs **Adjacency List** (O(V+E) space, O(degree) lookup)
- **Sparse vs Dense graphs**: adjacency list preferred for sparse, matrix for dense
- **In-degree, Out-degree** in directed graphs

```java
// Graph representation using adjacency list
class Graph {
    private int V;  // number of vertices
    private List<List<Integer>> adj;
    
    public Graph(int V) {
        this.V = V;
        adj = new ArrayList<>();
        for (int i = 0; i < V; i++) adj.add(new ArrayList<>());
    }
    
    public void addEdge(int u, int v) {
        adj.get(u).add(v);
        // For undirected: adj.get(v).add(u);
    }
}

// Weighted graph using pair
class WeightedGraph {
    private List<List<int[]>> adj;  // adj[u] = [[v, weight], ...]
    
    public void addEdge(int u, int v, int w) {
        adj.get(u).add(new int[]{v, w});
    }
}
```

### BFS (Breadth-First Search)
- Explores level by level using a Queue
- Finds shortest path in unweighted graphs
- Time: O(V+E), Space: O(V)

```java
// BFS for shortest path
public int bfsShortestPath(int[][] graph, int src, int dest) {
    int n = graph.length;
    boolean[] visited = new boolean[n];
    Queue<Integer> queue = new LinkedList<>();
    
    visited[src] = true;
    queue.offer(src);
    int steps = 0;
    
    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            int node = queue.poll();
            if (node == dest) return steps;
            for (int neighbor : graph[node]) {
                if (!visited[neighbor]) {
                    visited[neighbor] = true;
                    queue.offer(neighbor);
                }
            }
        }
        steps++;
    }
    return -1;  // not reachable
}

// BFS on 2D grid (find shortest path)
public int bfsGrid(int[][] grid, int[] start, int[] end) {
    int m = grid.length, n = grid[0].length;
    int[][] dirs = {{0,1},{0,-1},{1,0},{-1,0}};
    boolean[][] visited = new boolean[m][n];
    Queue<int[]> queue = new LinkedList<>();
    
    queue.offer(start);
    visited[start[0]][start[1]] = true;
    int steps = 0;
    
    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            int[] curr = queue.poll();
            if (curr[0] == end[0] && curr[1] == end[1]) return steps;
            for (int[] dir : dirs) {
                int nr = curr[0] + dir[0], nc = curr[1] + dir[1];
                if (nr >= 0 && nr < m && nc >= 0 && nc < n
                    && !visited[nr][nc] && grid[nr][nc] != 1) {
                    visited[nr][nc] = true;
                    queue.offer(new int[]{nr, nc});
                }
            }
        }
        steps++;
    }
    return -1;
}
```

### DFS (Depth-First Search)
- Explores as far as possible before backtracking using Stack/recursion
- Used for: cycle detection, topological sort, connected components, path finding
- Time: O(V+E), Space: O(V) for recursion stack

```java
// DFS iterative
public void dfs(List<List<Integer>> adj, int src) {
    int n = adj.size();
    boolean[] visited = new boolean[n];
    Stack<Integer> stack = new Stack<>();
    
    stack.push(src);
    while (!stack.isEmpty()) {
        int node = stack.pop();
        if (visited[node]) continue;
        visited[node] = true;
        System.out.print(node + " ");
        for (int neighbor : adj.get(node)) {
            if (!visited[neighbor]) stack.push(neighbor);
        }
    }
}

// DFS recursive
public void dfsRecursive(List<List<Integer>> adj, int node, boolean[] visited) {
    visited[node] = true;
    System.out.print(node + " ");
    for (int neighbor : adj.get(node)) {
        if (!visited[neighbor]) dfsRecursive(adj, neighbor, visited);
    }
}
```

### Connected Components
```java
// Count connected components in undirected graph
public int countComponents(int n, int[][] edges) {
    List<List<Integer>> adj = new ArrayList<>();
    for (int i = 0; i < n; i++) adj.add(new ArrayList<>());
    for (int[] e : edges) {
        adj.get(e[0]).add(e[1]);
        adj.get(e[1]).add(e[0]);
    }
    
    boolean[] visited = new boolean[n];
    int components = 0;
    for (int i = 0; i < n; i++) {
        if (!visited[i]) {
            dfsRecursive(adj, i, visited);
            components++;
        }
    }
    return components;
}
```

### Cycle Detection

#### Undirected Graph (DFS with parent tracking)
```java
public boolean hasCycleUndirected(List<List<Integer>> adj, int n) {
    boolean[] visited = new boolean[n];
    for (int i = 0; i < n; i++) {
        if (!visited[i] && dfsCycle(adj, i, -1, visited)) return true;
    }
    return false;
}

private boolean dfsCycle(List<List<Integer>> adj, int node, int parent, boolean[] visited) {
    visited[node] = true;
    for (int neighbor : adj.get(node)) {
        if (!visited[neighbor]) {
            if (dfsCycle(adj, neighbor, node, visited)) return true;
        } else if (neighbor != parent) {
            return true;  // back edge to non-parent = cycle
        }
    }
    return false;
}
```

#### Directed Graph (DFS with recursion stack)
```java
public boolean hasCycleDirected(List<List<Integer>> adj, int n) {
    boolean[] visited = new boolean[n];
    boolean[] inStack = new boolean[n];  // nodes in current DFS path
    for (int i = 0; i < n; i++) {
        if (!visited[i] && dfsCycleDirected(adj, i, visited, inStack)) return true;
    }
    return false;
}

private boolean dfsCycleDirected(List<List<Integer>> adj, int node,
                                   boolean[] visited, boolean[] inStack) {
    visited[node] = true;
    inStack[node] = true;
    for (int neighbor : adj.get(node)) {
        if (!visited[neighbor] && dfsCycleDirected(adj, neighbor, visited, inStack)) return true;
        else if (inStack[neighbor]) return true;  // back edge = cycle
    }
    inStack[node] = false;
    return false;
}
```

### Topological Sort

#### DFS-based (Kahn's algorithm is better for cycle detection)
```java
// Stack-based DFS topological sort
public List<Integer> topoSortDFS(List<List<Integer>> adj, int n) {
    boolean[] visited = new boolean[n];
    Stack<Integer> stack = new Stack<>();
    for (int i = 0; i < n; i++) {
        if (!visited[i]) dfsTopoSort(adj, i, visited, stack);
    }
    List<Integer> result = new ArrayList<>();
    while (!stack.isEmpty()) result.add(stack.pop());
    return result;
}

private void dfsTopoSort(List<List<Integer>> adj, int node,
                          boolean[] visited, Stack<Integer> stack) {
    visited[node] = true;
    for (int neighbor : adj.get(node)) {
        if (!visited[neighbor]) dfsTopoSort(adj, neighbor, visited, stack);
    }
    stack.push(node);  // push AFTER all descendants are processed
}
```

#### Kahn's Algorithm (BFS-based, detects cycles)
```java
public List<Integer> topoSortKahn(List<List<Integer>> adj, int n) {
    int[] indegree = new int[n];
    for (int u = 0; u < n; u++)
        for (int v : adj.get(u)) indegree[v]++;
    
    Queue<Integer> queue = new LinkedList<>();
    for (int i = 0; i < n; i++)
        if (indegree[i] == 0) queue.offer(i);
    
    List<Integer> result = new ArrayList<>();
    while (!queue.isEmpty()) {
        int node = queue.poll();
        result.add(node);
        for (int neighbor : adj.get(node)) {
            if (--indegree[neighbor] == 0) queue.offer(neighbor);
        }
    }
    
    // If result.size() != n, graph has a cycle
    return result.size() == n ? result : new ArrayList<>();
}
```

**Use case**: Course prerequisite ordering, build system dependency resolution

### Dijkstra's Algorithm (Shortest Path — Weighted)
```java
public int[] dijkstra(int[][] graph, int src) {
    // graph[u][v] = weight, 0 if no edge
    int n = graph.length;
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[src] = 0;
    
    // PriorityQueue: [distance, node]
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
    pq.offer(new int[]{0, src});
    
    while (!pq.isEmpty()) {
        int[] curr = pq.poll();
        int d = curr[0], u = curr[1];
        
        if (d > dist[u]) continue;  // stale entry
        
        for (int v = 0; v < n; v++) {
            if (graph[u][v] > 0) {
                int newDist = dist[u] + graph[u][v];
                if (newDist < dist[v]) {
                    dist[v] = newDist;
                    pq.offer(new int[]{newDist, v});
                }
            }
        }
    }
    return dist;
}
```

**Time**: O((V+E) log V), **Space**: O(V)
**Limitation**: Does NOT work with negative weights (use Bellman-Ford)

### Union-Find (Disjoint Set Union)
```java
class UnionFind {
    private int[] parent, rank;
    private int components;
    
    public UnionFind(int n) {
        parent = new int[n];
        rank = new int[n];
        components = n;
        for (int i = 0; i < n; i++) parent[i] = i;
    }
    
    public int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]);  // path compression
        return parent[x];
    }
    
    public boolean union(int x, int y) {
        int px = find(x), py = find(y);
        if (px == py) return false;  // already connected
        if (rank[px] < rank[py]) { int t = px; px = py; py = t; }
        parent[py] = px;
        if (rank[px] == rank[py]) rank[px]++;
        components--;
        return true;
    }
    
    public boolean connected(int x, int y) { return find(x) == find(y); }
    public int getComponents() { return components; }
}

// Use cases: detect cycle in undirected graph, count islands, Kruskal's MST
```

---

## Sliding Window Pattern

### Fixed-Size Sliding Window
```java
// Maximum sum of subarray of size k
public int maxSumSubarray(int[] nums, int k) {
    int windowSum = 0, maxSum = 0;
    for (int i = 0; i < k; i++) windowSum += nums[i];
    maxSum = windowSum;
    for (int i = k; i < nums.length; i++) {
        windowSum += nums[i] - nums[i - k];  // add right, remove left
        maxSum = Math.max(maxSum, windowSum);
    }
    return maxSum;
}
```

### Variable-Size Sliding Window
```java
// Longest substring without repeating characters
public int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> lastSeen = new HashMap<>();
    int left = 0, maxLen = 0;
    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        if (lastSeen.containsKey(c) && lastSeen.get(c) >= left) {
            left = lastSeen.get(c) + 1;  // shrink window
        }
        lastSeen.put(c, right);
        maxLen = Math.max(maxLen, right - left + 1);
    }
    return maxLen;
}

// Minimum window substring containing all chars of t
public String minWindow(String s, String t) {
    Map<Character, Integer> need = new HashMap<>(), window = new HashMap<>();
    for (char c : t.toCharArray()) need.merge(c, 1, Integer::sum);
    
    int left = 0, satisfied = 0, minLen = Integer.MAX_VALUE, start = 0;
    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        window.merge(c, 1, Integer::sum);
        if (need.containsKey(c) && window.get(c).equals(need.get(c))) satisfied++;
        
        while (satisfied == need.size()) {
            if (right - left + 1 < minLen) { minLen = right - left + 1; start = left; }
            char l = s.charAt(left++);
            window.merge(l, -1, Integer::sum);
            if (need.containsKey(l) && window.get(l) < need.get(l)) satisfied--;
        }
    }
    return minLen == Integer.MAX_VALUE ? "" : s.substring(start, start + minLen);
}

// Longest subarray with sum <= k
public int longestSubarrayWithSumK(int[] nums, int k) {
    int left = 0, sum = 0, maxLen = 0;
    for (int right = 0; right < nums.length; right++) {
        sum += nums[right];
        while (sum > k) sum -= nums[left++];
        maxLen = Math.max(maxLen, right - left + 1);
    }
    return maxLen;
}
```

### Sliding Window Pattern Template
```
1. Initialize left=0, result, and a window data structure (Map, Set, counter)
2. Expand window by moving right pointer
3. Update window state
4. Shrink window from left when constraint is violated
5. Update result after each valid window
```

---

## Heap / Priority Queue Patterns

### Top K Elements Pattern
```java
// Kth Largest Element in an array
public int findKthLargest(int[] nums, int k) {
    // Min-heap of size k: keeps k largest elements
    PriorityQueue<Integer> minHeap = new PriorityQueue<>();
    for (int num : nums) {
        minHeap.offer(num);
        if (minHeap.size() > k) minHeap.poll();  // remove smallest
    }
    return minHeap.peek();  // smallest of top-k = kth largest
}
// Time: O(n log k), Space: O(k)

// Top K Frequent Elements
public int[] topKFrequent(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int n : nums) freq.merge(n, 1, Integer::sum);
    
    PriorityQueue<Map.Entry<Integer, Integer>> minHeap =
        new PriorityQueue<>(Comparator.comparingInt(Map.Entry::getValue));
    
    for (var entry : freq.entrySet()) {
        minHeap.offer(entry);
        if (minHeap.size() > k) minHeap.poll();
    }
    
    return minHeap.stream().mapToInt(Map.Entry::getKey).toArray();
}
```

### Merge K Sorted Arrays/Lists
```java
public ListNode mergeKLists(ListNode[] lists) {
    PriorityQueue<ListNode> minHeap =
        new PriorityQueue<>(Comparator.comparingInt(n -> n.val));
    
    for (ListNode node : lists) {
        if (node != null) minHeap.offer(node);
    }
    
    ListNode dummy = new ListNode(0), curr = dummy;
    while (!minHeap.isEmpty()) {
        ListNode node = minHeap.poll();
        curr.next = node;
        curr = curr.next;
        if (node.next != null) minHeap.offer(node.next);
    }
    return dummy.next;
}
// Time: O(n log k) where n = total nodes, k = number of lists
```

### Median of Data Stream
```java
class MedianFinder {
    private PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder()); // lower half
    private PriorityQueue<Integer> minHeap = new PriorityQueue<>(); // upper half
    
    public void addNum(int num) {
        maxHeap.offer(num);
        minHeap.offer(maxHeap.poll());  // balance: maxHeap top to minHeap
        if (maxHeap.size() < minHeap.size()) maxHeap.offer(minHeap.poll()); // keep sizes balanced
    }
    
    public double findMedian() {
        if (maxHeap.size() > minHeap.size()) return maxHeap.peek();
        return (maxHeap.peek() + minHeap.peek()) / 2.0;
    }
}
```

### Task Scheduler
```java
// Most frequent task first — greedy + heap
public int leastInterval(char[] tasks, int n) {
    int[] freq = new int[26];
    for (char t : tasks) freq[t - 'A']++;
    PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());
    for (int f : freq) if (f > 0) maxHeap.offer(f);
    
    int time = 0;
    while (!maxHeap.isEmpty()) {
        List<Integer> temp = new ArrayList<>();
        int cycles = n + 1;
        while (cycles > 0 && !maxHeap.isEmpty()) {
            temp.add(maxHeap.poll() - 1);
            cycles--;
        }
        for (int f : temp) if (f > 0) maxHeap.offer(f);
        time += maxHeap.isEmpty() ? n + 1 - cycles : n + 1;
    }
    return time;
}
```

---

## Trie (Prefix Tree)

### Implementation
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
    
    // Autocomplete: find all words with given prefix
    public List<String> autocomplete(String prefix) {
        List<String> results = new ArrayList<>();
        TrieNode curr = root;
        for (char c : prefix.toCharArray()) {
            if (!curr.children.containsKey(c)) return results;
            curr = curr.children.get(c);
        }
        dfsCollect(curr, new StringBuilder(prefix), results);
        return results;
    }
    
    private void dfsCollect(TrieNode node, StringBuilder current, List<String> results) {
        if (node.isEndOfWord) results.add(current.toString());
        for (var entry : node.children.entrySet()) {
            current.append(entry.getKey());
            dfsCollect(entry.getValue(), current, results);
            current.deleteCharAt(current.length() - 1);  // backtrack
        }
    }
}
```
**Time:** O(m) per insert/search/prefix where m = word length
**Space:** O(total characters across all words)
**Use cases:** Autocomplete/type-ahead, IP routing (longest prefix match), spell checkers

---

## Graph Interview Q&A

**Q: What is the difference between BFS and DFS?**
BFS explores layer by layer using a queue — finds shortest path in unweighted graphs. DFS explores as far as possible using a stack/recursion — useful for cycle detection, topological sort, and path finding. BFS guarantees shortest path; DFS does not.

**Q: How do you detect a cycle in a directed graph?**
Use DFS with a "recursion stack" (inStack array). During DFS, mark each node as in-stack when entered and remove it when backtracking. If you encounter a node already in the stack, a cycle exists (back edge).

**Q: What is topological sort? When is it applicable?**
Topological sort orders vertices such that for every directed edge (u→v), u comes before v. Only possible for DAGs (Directed Acyclic Graphs). Kahn's algorithm (BFS-based, uses in-degrees) also detects cycles. Use cases: course prerequisites, build dependency ordering, event scheduling.

**Q: What is Union-Find used for?**
Union-Find (Disjoint Set Union) efficiently answers: "Are these two elements in the same connected component?" It supports union (merge two sets) and find (find root/representative) operations. Used for: detecting cycles in undirected graphs, counting connected components, Kruskal's MST algorithm.

**Q: When would you use Dijkstra vs BFS for shortest path?**
BFS for unweighted graphs (all edges = weight 1) — O(V+E). Dijkstra for weighted graphs with non-negative weights — O((V+E) log V). For negative weights: Bellman-Ford O(VE). BFS is simpler; prefer it for unweighted problems.

**Q: What is the sliding window pattern? When do you use it?**
The sliding window pattern maintains a contiguous subarray/substring between two pointers (left, right). Right expands to include new elements; left shrinks to remove invalid elements. Use when: finding max/min subarray sum, longest substring with constraint, minimum window substring. Key insight: avoids O(n²) nested loops.

**Q: When do you use a min-heap vs max-heap for Top-K problems?**
For "K largest elements": use a min-heap of size K. As you scan, remove the minimum if heap exceeds K — the remaining K elements are the largest, with peek() giving the Kth largest. For "K smallest elements": use a max-heap of size K similarly.
