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
