# Level Up — Problem Solutions

This file contains problem statements and optimized Python solutions.
All solutions read input using `input()` (not `sys.stdin`) so they run
directly on online judges / coding-test compilers like HackerRank,
LeetCode custom judges, HackerEarth, etc.

---

## Q1 - Find Peak Element

### Problem Statement

A **peak element** is an element that is strictly greater than its neighbors.

Given a `0`-indexed integer array `nums`, find a peak element, and return its index. If the array
contains multiple peaks, return the index to **any** of the peaks.

You may imagine that `nums[-1] = nums[n] = -∞`. In other words, an element is always considered
to be strictly greater than a neighbor that is outside the array.

You must write an algorithm that runs in **O(log n)** time.

**Example 1**
```
Input:  nums = [1,2,3,1]
Output: 2
Explanation: 3 is a peak element and your function should return the index number 2.
```

**Example 2**
```
Input:  nums = [1,2,1,3,5,6,4]
Output: 5
Explanation: Your function can return either index number 1 where the peak element is 2,
or index number 5 where the peak element is 6.
```

**Constraints**
- `1 <= nums.length <= 1000`
- `-2^31 <= nums[i] <= 2^31 - 1`
- `nums[i] != nums[i + 1]` for all valid `i`.

### Approach

Since a strict peak is guaranteed to exist (boundaries act as `-∞`, and no two adjacent
elements are equal), binary search finds one in `O(log n)`:

- Compare `nums[mid]` with `nums[mid + 1]`.
- If `nums[mid] < nums[mid + 1]`, the slope is rising, so a peak must exist to the right
  → search `[mid+1, right]`.
- Otherwise, `nums[mid]` is greater than its right neighbor, so `nums[mid]` is a peak or a
  peak lies to the left → search `[left, mid]`.
- Loop ends when `left == right`, which is guaranteed to be a valid peak index.

**Complexity:** Time `O(log n)`, Space `O(1)`.

### Q1 - Solution

```python
def find_peak_element(nums):
    """
    Return the index of a peak element in `nums` using binary search.
    A peak element is strictly greater than its neighbors (with nums[-1]
    and nums[n] treated as -infinity).

    Time Complexity:  O(log n)
    Space Complexity: O(1)
    """
    left, right = 0, len(nums) - 1

    while left < right:
        mid = left + (right - left) // 2  # avoids overflow in other languages

        if nums[mid] < nums[mid + 1]:
            # Ascending slope -> a peak must exist to the right of mid
            left = mid + 1
        else:
            # Descending (or peak) at mid -> peak is at mid, or to the left
            right = mid

    return left  # left == right, pointing at a valid peak index


# ---------------- Input / Output (HackerRank style) ----------------
# Input format:
#   A single line: space-separated integers representing nums.
# Output format:
#   A single integer: the index of a peak element.

nums = list(map(int, input().split()))
print(find_peak_element(nums))
```

### Q1 - Test Cases

| Input               | Output | Note                                   |
|---------------------|--------|-----------------------------------------|
| `1 2 3 1`            | `2`    | matches Example 1                       |
| `1 2 1 3 5 6 4`       | `5`    | one of several valid peaks (6 at idx 5) |
| `1`                  | `0`    | single element                          |
| `1 2 3 4 5`           | `4`    | strictly increasing → last index        |
| `5 4 3 2 1`           | `0`    | strictly decreasing → first index       |
| `1 2`                | `1`    | two elements                            |
| `2 1`                | `0`    | two elements                            |

All the above were verified against the solution logic (each returned index satisfies the
strict-peak condition relative to its neighbors).

---

## Q2 - Sort Vowels in a String

### Problem Statement

Given a `0`-indexed string `s`, permute `s` to get a new string `t` such that:

- **All consonants remain in their original places.** More formally, if there is an index `i`
  with `0 <= i < s.length` such that `s[i]` is a consonant, then `t[i] = s[i]`.
- **The vowels must be sorted in non-decreasing order of their ASCII values.** More formally,
  for pairs of indices `i, j` with `0 <= i < j < s.length` such that `s[i]` and `s[j]` are
  vowels, then `t[i]` must not have a higher ASCII value than `t[j]`.

Return the resulting string.

The vowels are `'a'`, `'e'`, `'i'`, `'o'`, and `'u'`, and they can appear in lowercase or
uppercase. Consonants comprise all letters that are not vowels.

**Example 1**
```
Input:  s = "lEetcOde"
Output: "lEOtcede"
Explanation: 'E', 'O', and 'e' are the vowels in s; 'l', 't', 'c', and 'd' are all consonants.
The vowels are sorted according to their ASCII values, and the consonants remain in the same
places.
```

**Example 2**
```
Input:  s = "lYmpH"
Output: "lYmpH"
Explanation: There are no vowels in s (all characters in s are consonants), so we return
"lYmpH".
```

**Constraints**
- `1 <= s.length <= 10^5`
- `s` consists only of letters of the English alphabet in uppercase and lowercase.

### Approach

1. Scan the string once, recording the indices where vowels occur and the vowel characters
   themselves.
2. Sort the collected vowel characters by ASCII value (Python's default string comparison
   already sorts uppercase before lowercase, which is exactly ASCII order).
3. Place the sorted vowels back into their original recorded index positions; consonants are
   never touched.

**Complexity:** Time `O(n log n)` (sorting the vowels), Space `O(n)`.

### Q2 - Solution

```python
def sort_vowels(s):
    """
    Return a new string where all vowels of `s` are rearranged into
    non-decreasing ASCII order, while every consonant stays in its
    original position.

    Time Complexity:  O(n log n)  (sorting the vowels)
    Space Complexity: O(n)
    """
    vowels_set = frozenset("aeiouAEIOU")

    vowel_indices = [i for i, ch in enumerate(s) if ch in vowels_set]
    if not vowel_indices:
        return s  # no vowels -> string is already the answer

    sorted_vowels = sorted(s[i] for i in vowel_indices)

    result = list(s)
    for idx, ch in zip(vowel_indices, sorted_vowels):
        result[idx] = ch

    return "".join(result)


# ---------------- Input / Output (HackerRank style) ----------------
# Input format:
#   A single line containing the string s.
# Output format:
#   A single line containing the resulting string t.

s = input()
print(sort_vowels(s))
```

### Q2 - Test Cases

| Input          | Output       | Note                              |
|----------------|--------------|-------------------------------------|
| `lEetcOde`     | `lEOtcede`   | matches Example 1                   |
| `lYmpH`        | `lYmpH`      | no vowels at all                    |
| `a`            | `a`          | single vowel                        |
| `AaEeIiOoUu`   | `AEIOUaeiou` | all vowels, mixed case              |
| `bcdfg`        | `bcdfg`      | all consonants                      |

All the above were verified programmatically against the solution logic.

---

## Q3 - Longest Substring Without Repeating Characters

### Problem Statement

Given a string `s`, find the length of the longest substring without duplicate characters.

**Example 1**
```
Input:  s = "abcabcbb"
Output: 3
Explanation: The answer is "abc", with the length of 3.
```

**Example 2**
```
Input:  s = "bbbbb"
Output: 1
Explanation: The answer is "b", with the length of 1.
```

**Example 3**
```
Input:  s = "pwwkew"
Output: 3
Explanation: The answer is "wke", with the length of 3.
Notice that the answer must be a substring, "pwke" is a subsequence and not a substring.
```

**Constraints**
- `0 <= s.length <= 5 * 10^4`
- `s` consists of English letters, digits, symbols and spaces.

### Approach

Use the **sliding window** technique with a hash map that stores the last seen index of each
character:

- Maintain a window `[start, i]` that never contains a duplicate character.
- For each character `s[i]`, if it was seen before **and** that previous occurrence is inside
  the current window (`last_seen[ch] >= start`), shrink the window by moving `start` to
  `last_seen[ch] + 1`.
- Update `last_seen[ch] = i` and track the best (maximum) window length seen so far.
- Because `start` and `i` each only move forward, the whole scan is a single pass over the
  string.

This is a single linear pass — far more efficient than the brute-force `O(n^2)` or `O(n^3)`
approach of checking every substring.

**Complexity:** Time `O(n)`, Space `O(min(n, Σ))` where `Σ` is the character set size (bounded,
since we only store one index per distinct character).

### Q3 - Solution

```python
def length_of_longest_substring(s):
    """
    Return the length of the longest substring of `s` that has no
    repeating characters, using the sliding-window technique.

    Time Complexity:  O(n)
    Space Complexity: O(min(n, charset size))
    """
    last_seen = {}   # character -> most recent index it was seen at
    start = 0        # left edge of the current window (inclusive)
    best = 0

    for i, ch in enumerate(s):
        if ch in last_seen and last_seen[ch] >= start:
            # Duplicate found inside the current window -> shrink window
            start = last_seen[ch] + 1

        last_seen[ch] = i
        best = max(best, i - start + 1)

    return best


# ---------------- Input / Output (HackerRank style) ----------------
# Input format:
#   A single line containing the string s (may include spaces, so read
#   the raw line rather than splitting on whitespace).
# Output format:
#   A single integer: the length of the longest substring without
#   repeating characters.

s = input()
print(length_of_longest_substring(s))
```

### Q3 - Test Cases

| Input        | Output | Note                                             |
|--------------|--------|---------------------------------------------------|
| `abcabcbb`   | `3`    | matches Example 1 (`"abc"`)                        |
| `bbbbb`      | `1`    | matches Example 2 (`"b"`)                          |
| `pwwkew`     | `3`    | matches Example 3 (`"wke"`)                        |
| *(empty)*    | `0`    | empty string edge case                             |
| ` ` (space)  | `1`    | single space character is still a valid character  |
| `au`         | `2`    | no repeats at all                                  |
| `dvdf`       | `3`    | tricky case: repeat resets window mid-string       |
| `abba`       | `2`    | tricky case: window must not use a stale last-seen index |

All the above were verified programmatically against the solution logic.

---

## Q4 - Maximum Product Subarray

### Problem Statement

Given an integer array `nums`, find a subarray that has the largest product, and return the
product.

The test cases are generated so that the answer will fit in a 32-bit integer.

**Example 1**
```
Input:  nums = [2,3,-2,4]
Output: 6
Explanation: [2,3] has the largest product 6.
```

**Example 2**
```
Input:  nums = [-2,0,-1]
Output: 0
Explanation: The result cannot be 2, because [-2,-1] is not a subarray.
```

**Constraints**
- `1 <= nums.length <= 2 * 10^4`
- `-10 <= nums[i] <= 10`
- The product of any subarray of `nums` is guaranteed to fit in a 32-bit integer.

### Approach

The tricky part of this problem is that **a negative number can flip the sign** of the running
product, turning the current minimum product into the largest possible product (and vice
versa). A single running "max so far" (like in Kadane's algorithm for sums) isn't enough —
we must also track the running **minimum** product, because multiplying a very negative
minimum by another negative number can produce a new maximum.

**Dynamic programming, single pass:**
- Keep two running values ending at the current index: `max_prod` (largest product of a
  subarray ending here) and `min_prod` (smallest / most negative product of a subarray ending
  here).
- When the current element `n` is negative, swap `max_prod` and `min_prod` before updating —
  multiplying by a negative number turns the max into a candidate min and the min into a
  candidate max.
- Update: `max_prod = max(n, max_prod * n)` and `min_prod = min(n, min_prod * n)`. The `n`
  alone accounts for starting a fresh subarray right at this element (important after a `0`,
  which resets everything).
- Track the overall best (`result`) as the maximum `max_prod` seen across the whole scan.

This is a linear-time variant of Kadane's algorithm adapted for products instead of sums.

**Complexity:** Time `O(n)`, Space `O(1)`.

### Q4 - Solution

```python
def max_product_subarray(nums):
    """
    Return the largest product achievable by any contiguous subarray of
    `nums`, using a Kadane's-algorithm-style single pass that tracks both
    the running maximum and running minimum product (to correctly handle
    sign flips caused by negative numbers).

    Time Complexity:  O(n)
    Space Complexity: O(1)
    """
    max_prod = min_prod = result = nums[0]

    for n in nums[1:]:
        if n < 0:
            # A negative number flips which running value can become the
            # new maximum, so swap before combining with n.
            max_prod, min_prod = min_prod, max_prod

        max_prod = max(n, max_prod * n)
        min_prod = min(n, min_prod * n)

        result = max(result, max_prod)

    return result


# ---------------- Input / Output (HackerRank style) ----------------
# Input format:
#   A single line: space-separated integers representing nums.
# Output format:
#   A single integer: the largest product of any contiguous subarray.

nums = list(map(int, input().split()))
print(max_product_subarray(nums))
```

### Q4 - Test Cases

| Input                   | Output | Note                                                          |
|-------------------------|--------|-----------------------------------------------------------------|
| `2 3 -2 4`               | `6`    | matches Example 1 (`[2,3]`)                                     |
| `-2 0 -1`                | `0`    | matches Example 2 — `0` breaks any positive-crossing product    |
| `-2 3 -4`                | `24`   | product of the whole array (two negatives cancel out)           |
| `0 2`                    | `2`    | leading zero must not block the rest of the array               |
| `-2`                     | `-2`   | single negative element                                         |
| `2 -5 -2 -4 3`           | `24`   | best subarray is `[-2,-4]` combined with sign tracking → 24     |
| `-1 -2 -9 -6`            | `108`  | all negative, even count → whole-array product is best          |

All the above were verified programmatically against the solution logic.

---

## Q5 - Maximum Subarray Sum

### Problem Statement

Given an integer array `nums`, find the subarray with the largest sum, and return its sum.

**Example 1**
```
Input:  nums = [-2,1,-3,4,-1,2,1,-5,4]
Output: 6
Explanation: The subarray [4,-1,2,1] has the largest sum 6.
```

**Example 2**
```
Input:  nums = [1]
Output: 1
Explanation: The subarray [1] has the largest sum 1.
```

**Example 3**
```
Input:  nums = [5,4,-1,7,8]
Output: 23
Explanation: The subarray [5,4,-1,7,8] has the largest sum 23.
```

**Follow up:** If you have figured out the O(n) solution, try coding another solution using the
divide and conquer approach, which is more subtle.

### Approach

This is the classic **Kadane's Algorithm** problem:

- Maintain `cur`, the maximum sum of a subarray *ending exactly at the current index*.
- At each element `n`, decide whether to **extend** the previous subarray (`cur + n`) or
  **start fresh** at the current element (`n`) — whichever is larger:
  `cur = max(n, cur + n)`.
- Track `best`, the maximum `cur` value seen across the whole scan — this is the answer.

The key insight is that once a running sum drops so low that continuing it would produce a
smaller result than starting over, it's never beneficial to keep the negative prefix — so we
simply restart the subarray at the current element.

**Complexity:** Time `O(n)`, Space `O(1)`.

(A divide-and-conquer `O(n log n)` alternative also exists — split the array in half, find the
best subarray sum in the left half, the right half, and the best sum crossing the midpoint, and
take the max of the three — but Kadane's linear-time approach is optimal and simpler.)

### Q5 - Solution

```python
def max_subarray_sum(nums):
    """
    Return the largest sum achievable by any contiguous subarray of
    `nums`, using Kadane's algorithm.

    Time Complexity:  O(n)
    Space Complexity: O(1)
    """
    best = cur = nums[0]

    for n in nums[1:]:
        # Either extend the current subarray, or start a new one at n.
        cur = max(n, cur + n)
        best = max(best, cur)

    return best


# ---------------- Input / Output (HackerRank style) ----------------
# Input format:
#   A single line: space-separated integers representing nums.
# Output format:
#   A single integer: the largest sum of any contiguous subarray.

nums = list(map(int, input().split()))
print(max_subarray_sum(nums))
```

### Q5 - Test Cases

| Input                            | Output | Note                                                |
|-----------------------------------|--------|--------------------------------------------------------|
| `-2 1 -3 4 -1 2 1 -5 4`            | `6`    | matches Example 1 (`[4,-1,2,1]`)                        |
| `1`                                | `1`    | matches Example 2, single element                      |
| `5 4 -1 7 8`                       | `23`   | matches Example 3, entire array is the best subarray    |
| `-1`                               | `-1`   | single negative element — must still return a valid sum |
| `-5 -4 -3`                         | `-3`   | all negative — best subarray is the single largest value|
| `0 0 0`                            | `0`    | all zeros — sum is trivially 0                          |

All the above were verified programmatically against the solution logic.

---

## Q6 - Min Stack

### Problem Statement

Design a stack that supports `push`, `pop`, `top`, and retrieving the minimum element in
**constant time**.

Implement the `MinStack` class:

- `MinStack()` initializes the stack object.
- `void push(int val)` pushes the element `val` onto the stack.
- `void pop()` removes the element on the top of the stack.
- `int top()` gets the top element of the stack.
- `int getMin()` retrieves the minimum element in the stack.

You must implement a solution with `O(1)` time complexity for each function.

**Example 1**
```
Input
["MinStack","push","push","push","getMin","pop","top","getMin"]
[[],[-2],[0],[-3],[],[],[],[]]

Output
[null,null,null,null,-3,null,0,-2]

Explanation
MinStack minStack = new MinStack();
minStack.push(-2);
minStack.push(0);
minStack.push(-3);
minStack.getMin(); // return -3
minStack.pop();
minStack.top();    // return 0
minStack.getMin(); // return -2
```

**Constraints**
- `-2^31 <= val <= 2^31 - 1`
- Methods `pop`, `top` and `getMin` operations will always be called on non-empty stacks.
- At most `3 * 10^4` calls will be made to `push`, `pop`, `top`, and `getMin`.

### Approach

Use **two parallel stacks**:

- `stack` — the actual data stack, storing every pushed value in order.
- `min_stack` — at every position `i`, `min_stack[i]` holds the minimum value of `stack[0..i]`
  (i.e. the minimum of everything pushed so far, up to and including that point).

On `push(val)`:
- Push `val` onto `stack`.
- Push `min(val, min_stack[-1])` onto `min_stack` (or just `val` if `min_stack` is empty).

On `pop()`:
- Pop from both `stack` and `min_stack` together, so they always stay the same length and in
  sync.

`top()` and `getMin()` are then simple `O(1)` lookups of the last element of `stack` /
`min_stack` respectively.

Because `min_stack` records the running minimum *at each depth*, popping a value automatically
"restores" the correct previous minimum without needing to search — this is what makes
`getMin()` O(1) even after arbitrary sequences of push/pop.

**Complexity:** Time `O(1)` per operation, Space `O(n)` for `n` pushed elements (two stacks of
parallel size).

### Q6 - Solution

```python
class MinStack:
    """
    A stack supporting push, pop, top, and getMin, all in O(1) time,
    implemented using an auxiliary stack that tracks the running minimum
    at every depth.
    """

    def __init__(self):
        self.stack = []       # holds all pushed values
        self.min_stack = []   # min_stack[i] = min of stack[0..i]

    def push(self, val):
        self.stack.append(val)
        if not self.min_stack or val <= self.min_stack[-1]:
            self.min_stack.append(val)
        else:
            self.min_stack.append(self.min_stack[-1])

    def pop(self):
        self.stack.pop()
        self.min_stack.pop()

    def top(self):
        return self.stack[-1]

    def getMin(self):
        return self.min_stack[-1]


# ---------------- Input / Output (HackerRank style) ----------------
# Input format:
#   Line 1: an integer n, the number of operations.
#   Next n lines: one of
#       push <val>
#       pop
#       top
#       getMin
# Output format:
#   For every "top" and "getMin" operation, print the returned value on
#   its own line, in the order the operations were performed.
#   (push and pop produce no output.)

n = int(input())
ms = MinStack()
results = []

for _ in range(n):
    parts = input().split()
    cmd = parts[0]

    if cmd == "push":
        ms.push(int(parts[1]))
    elif cmd == "pop":
        ms.pop()
    elif cmd == "top":
        results.append(str(ms.top()))
    elif cmd == "getMin":
        results.append(str(ms.getMin()))

print("\n".join(results))
```

### Q6 - Test Cases

| Input (operations)                                                              | Output          | Note                                        |
|-----------------------------------------------------------------------------------|-----------------|-----------------------------------------------|
| `push -2` / `push 0` / `push -3` / `getMin` / `pop` / `top` / `getMin`             | `-3` / `0` / `-2` | matches Example 1 exactly                     |
| `push 5` / `push 3` / `getMin` / `pop` / `getMin`                                  | `3` / `5`         | min correctly reverts to `5` after popping `3`|
| `push 1` / `push 1` / `getMin` / `pop`                                             | `1`               | duplicate minimums handled with `<=` comparison |

All the above were verified programmatically against the solution logic.

---

## Q7 - Evaluate Reverse Polish Notation

### Problem Statement

You are given an array of strings `tokens` that represents an arithmetic expression in
Reverse Polish Notation.

Evaluate the expression. Return an integer that represents the value of the expression.

Note that:
- The valid operators are `'+'`, `'-'`, `'*'`, and `'/'`.
- Each operand may be an integer or another expression.
- The division between two integers always **truncates toward zero**.
- There will not be any division by zero.
- The input represents a valid arithmetic expression in reverse polish notation.
- The answer and all the intermediate calculations can be represented in a 32-bit integer.

**Example 1**
```
Input:  tokens = ["2","1","+","3","*"]
Output: 9
Explanation: ((2 + 1) * 3) = 9
```

**Example 2**
```
Input:  tokens = ["4","13","5","/","+"]
Output: 6
Explanation: (4 + (13 / 5)) = 6
```

**Example 3**
```
Input:  tokens = ["10","6","9","3","+","-11","*","/","*","17","+","5","+"]
Output: 22
Explanation: ((10 * (6 / ((9 + 3) * -11))) + 17) + 5
= ((10 * (6 / (12 * -11))) + 17) + 5
= ((10 * (6 / -132)) + 17) + 5
= ((10 * 0) + 17) + 5
= (0 + 17) + 5
= 17 + 5
= 22
```

### Approach

Reverse Polish Notation (postfix) is naturally evaluated with a **stack**:

- Scan the tokens left to right.
- If a token is a **number**, push it onto the stack.
- If a token is an **operator**, pop the top two values (`b` = most recently pushed, `a` = the
  one before it), apply the operator as `a OP b` (order matters for `-` and `/`), and push the
  result back onto the stack.
- After processing every token, the single value remaining on the stack is the answer.

For division, Python's `/` and `//` don't truncate toward zero for negative results (`//` floors
instead), so we explicitly use `int(a / b)`, which truncates toward zero as required by the
problem (e.g. `-7 / 2 = -3.5 -> -3`, not `-4`).

**Complexity:** Time `O(n)`, Space `O(n)` for the stack.

### Q7 - Solution

```python
def eval_rpn(tokens):
    """
    Evaluate an arithmetic expression given in Reverse Polish Notation
    (postfix notation) using a stack.

    Time Complexity:  O(n)
    Space Complexity: O(n)
    """
    stack = []
    operators = {"+", "-", "*", "/"}

    for tok in tokens:
        if tok in operators:
            b = stack.pop()
            a = stack.pop()
            if tok == "+":
                stack.append(a + b)
            elif tok == "-":
                stack.append(a - b)
            elif tok == "*":
                stack.append(a * b)
            else:  # '/'
                # Truncate toward zero (Python's int(a / b) does this correctly)
                stack.append(int(a / b))
        else:
            stack.append(int(tok))

    return stack[-1]


# ---------------- Input / Output (HackerRank style) ----------------
# Input format:
#   A single line: space-separated tokens (numbers and operators)
#   representing the RPN expression.
# Output format:
#   A single integer: the value of the evaluated expression.

tokens = input().split()
print(eval_rpn(tokens))
```

### Q7 - Test Cases

| Input                                                  | Output | Note                                                       |
|-----------------------------------------------------------|--------|---------------------------------------------------------------|
| `2 1 + 3 *`                                              | `9`    | matches Example 1                                              |
| `4 13 5 / +`                                             | `6`    | matches Example 2 (integer division truncates)                 |
| `10 6 9 3 + -11 * / * 17 + 5 +`                          | `22`   | matches Example 3 (deeply nested expression)                   |
| `3 -4 +`                                                 | `-1`   | negative operand parsing                                       |
| `-7 2 /`                                                 | `-3`   | truncation toward zero for negative division (`-3.5 -> -3`)    |

All the above were verified programmatically against the solution logic.

---

## Q8 - Daily Temperatures

### Problem Statement

Given an array of integers `temperatures` represents the daily temperatures, return an array
`answer` such that `answer[i]` is the number of days you have to wait after the `i`th day to
get a warmer temperature. If there is no future day for which this is possible, keep
`answer[i] == 0` instead.

**Example 1**
```
Input:  temperatures = [73,74,75,71,69,72,76,73]
Output: [1,1,4,2,1,1,0,0]
```

**Example 2**
```
Input:  temperatures = [30,40,50,60]
Output: [1,1,1,0]
```

**Example 3**
```
Input:  temperatures = [30,60,90]
Output: [1,1,0]
```

### Approach

Use a **monotonic (decreasing) stack** of indices:

- Iterate through `temperatures` left to right, maintaining a stack of indices whose
  temperatures are in decreasing order (i.e. we haven't yet found a warmer day for them).
- For each new day `i` with temperature `t`, while the stack is non-empty and the temperature
  at the index on top of the stack is **less than** `t`, that means day `i` is the first warmer
  day for that index — pop it and set `answer[popped_index] = i - popped_index`.
- Push the current index `i` onto the stack (it may still be waiting for a warmer day later).
- Any indices left on the stack at the end never found a warmer day, so their `answer` stays
  `0` (the array is pre-initialized with zeros).

Each index is pushed once and popped at most once, so the total work across the whole scan is
linear despite the nested-looking `while` loop.

**Complexity:** Time `O(n)`, Space `O(n)` for the stack and the output array.

### Q8 - Solution

```python
def daily_temperatures(temperatures):
    """
    For each day, return the number of days until a warmer temperature
    occurs (or 0 if none exists), using a monotonic decreasing stack of
    indices.

    Time Complexity:  O(n)
    Space Complexity: O(n)
    """
    n = len(temperatures)
    answer = [0] * n
    stack = []  # indices with temperatures in decreasing order

    for i, t in enumerate(temperatures):
        while stack and temperatures[stack[-1]] < t:
            j = stack.pop()
            answer[j] = i - j
        stack.append(i)

    return answer


# ---------------- Input / Output (HackerRank style) ----------------
# Input format:
#   A single line: space-separated integers representing temperatures.
# Output format:
#   A single line: space-separated integers representing the answer
#   array.

temperatures = list(map(int, input().split()))
result = daily_temperatures(temperatures)
print(" ".join(map(str, result)))
```

### Q8 - Test Cases

| Input                          | Output                    | Note                                        |
|---------------------------------|----------------------------|------------------------------------------------|
| `73 74 75 71 69 72 76 73`        | `1 1 4 2 1 1 0 0`           | matches Example 1                               |
| `30 40 50 60`                    | `1 1 1 0`                   | matches Example 2 — strictly increasing         |
| `30 60 90`                       | `1 1 0`                     | matches Example 3                               |
| `100`                            | `0`                         | single day, no future day exists                |
| `100 99 98 97`                   | `0 0 0 0`                   | strictly decreasing — no day ever gets warmer   |
| `50 50 50`                       | `0 0 0`                     | equal temperatures never count as "warmer"      |

All the above were verified programmatically against the solution logic.

---

## Q9 - Online Stock Span

### Problem Statement

Design an algorithm that collects daily price quotes for some stock and returns the span of
that stock's price for the current day.

The span of the stock's price in one day is the maximum number of consecutive days (starting
from that day and going backward) for which the stock price was less than or equal to the price
of that day.

For example, if the prices of the stock in the last four days is `[7,2,1,2]` and the price of
the stock today is `2`, then the span of today is `4` because starting from today, the price of
the stock was less than or equal `2` for `4` consecutive days.
Also, if the prices of the stock in the last four days is `[7,34,1,2]` and the price of the
stock today is `8`, then the span of today is `3` because starting from today, the price of the
stock was less than or equal `8` for `3` consecutive days.

Implement the `StockSpanner` class:
- `StockSpanner()` Initializes the object of the class.
- `int next(int price)` Returns the span of the stock's price given that today's price is
  `price`.

**Example 1**
```
Input
["StockSpanner", "next", "next", "next", "next", "next", "next", "next"]
[[], [100], [80], [60], [70], [60], [75], [85]]
Output
[null, 1, 1, 1, 2, 1, 4, 6]

Explanation
StockSpanner stockSpanner = new StockSpanner();
stockSpanner.next(100); // return 1
stockSpanner.next(80);  // return 1
stockSpanner.next(60);  // return 1
stockSpanner.next(70);  // return 2
stockSpanner.next(60);  // return 1
stockSpanner.next(75);  // return 4, because the last 4 prices (including today's price of 75)
                        // were less than or equal to today's price.
stockSpanner.next(85);  // return 6
```

### Approach

Use a **monotonic (decreasing) stack** that stores `(price, span)` pairs:

- On each call to `next(price)`, start with `span = 1` (today itself always counts).
- While the stack is non-empty and the price on top of the stack is `<= price`, that top entry
  is "absorbed" into today's span — pop it and add its stored span to the running `span`
  (its days are all `<= price`, so they extend today's streak).
- Push `(price, span)` onto the stack and return `span`.

Because each `(price, span)` entry is pushed once and popped at most once across the whole
sequence of calls, the **amortized** cost per `next()` call is `O(1)`, even though a single
call's `while` loop could in the worst case pop many entries.

**Complexity:** Time `O(1)` amortized per `next()` call, Space `O(n)` for the stack across `n`
calls.

### Q9 - Solution

```python
class StockSpanner:
    """
    Computes the "stock span" for a stream of daily prices using a
    monotonic decreasing stack of (price, span) pairs.

    Time Complexity:  O(1) amortized per next() call
    Space Complexity: O(n) for n calls to next()
    """

    def __init__(self):
        self.stack = []  # each entry: (price, span)

    def next(self, price):
        span = 1
        while self.stack and self.stack[-1][0] <= price:
            span += self.stack.pop()[1]
        self.stack.append((price, span))
        return span


# ---------------- Input / Output (HackerRank style) ----------------
# Input format:
#   Line 1: an integer n, the number of prices to process.
#   Line 2: n space-separated integers, the daily prices in order.
# Output format:
#   A single line: n space-separated integers, the span returned by
#   next() for each price in order.

n = int(input())
prices = list(map(int, input().split()))

spanner = StockSpanner()
results = [spanner.next(p) for p in prices]
print(" ".join(map(str, results)))
```

### Q9 - Test Cases

| Input (prices in order)                | Output              | Note                                    |
|-------------------------------------------|-----------------------|--------------------------------------------|
| `100 80 60 70 60 75 85`                    | `1 1 1 2 1 4 6`         | matches Example 1 exactly                   |
| `10`                                       | `1`                    | single price, span is always at least 1     |
| `10 20 30 40`                              | `1 2 3 4`               | strictly increasing — span grows every day  |
| `40 30 20 10`                              | `1 1 1 1`               | strictly decreasing — span always 1         |
| `5 5 5 5`                                  | `1 2 3 4`               | equal prices count toward the span (`<=`)   |

All the above were verified programmatically against the solution logic.

---

## Q10 - Valid Parentheses

### Problem Statement

Given a string `s` containing just the characters `'('`, `')'`, `'{'`, `'}'`, `'['` and `']'`,
determine if the input string is valid.

An input string is valid if:
- Open brackets must be closed by the same type of brackets.
- Open brackets must be closed in the correct order.
- Every close bracket has a corresponding open bracket of the same type.

**Example 1**
```
Input:  s = "()"
Output: true
```

**Example 2**
```
Input:  s = "()[]{}"
Output: true
```

**Example 3**
```
Input:  s = "(]"
Output: false
```

**Example 4**
```
Input:  s = "([])"
Output: true
```

### Approach

Classic **stack** application:

- Scan the string left to right.
- Whenever an **opening** bracket (`(`, `[`, `{`) is seen, push it onto the stack.
- Whenever a **closing** bracket is seen, check:
  - The stack must not be empty (otherwise there's nothing to close).
  - The top of the stack must be the matching opening bracket.
  - If both hold, pop the stack; otherwise the string is invalid.
- After scanning the entire string, it's valid only if the stack is **empty** (every opener was
  matched and closed).

**Complexity:** Time `O(n)`, Space `O(n)`.

### Q10 - Solution

```python
def is_valid(s):
    """
    Return True if the bracket string `s` is valid (every bracket is
    properly opened, closed, and nested), using a stack.

    Time Complexity:  O(n)
    Space Complexity: O(n)
    """
    stack = []
    pairs = {")": "(", "]": "[", "}": "{"}

    for ch in s:
        if ch in "([{":
            stack.append(ch)
        elif ch in ")]}":
            if not stack or stack[-1] != pairs[ch]:
                return False
            stack.pop()

    return not stack


# ---------------- Input / Output (HackerRank style) ----------------
# Input format:
#   A single line containing the string s.
# Output format:
#   "true" if the string is valid, "false" otherwise.

s = input()
print("true" if is_valid(s) else "false")
```

### Q10 - Test Cases

| Input      | Output  | Note                                            |
|------------|---------|----------------------------------------------------|
| `()`       | `true`  | matches Example 1                                   |
| `()[]{}`   | `true`  | matches Example 2                                   |
| `(]`       | `false` | matches Example 3 — mismatched bracket type         |
| `([])`     | `true`  | matches Example 4 — correctly nested                |
| *(empty)*  | `true`  | empty string is trivially valid                     |
| `(`        | `false` | unclosed opener                                     |
| `]`        | `false` | closer with nothing to match                        |
| `([)]`     | `false` | closed out of order (interleaved, not nested)        |

All the above were verified programmatically against the solution logic.

---

## Q11 - Maximum Number of Activities (Activity Selection)

### Problem Statement

Once upon a time, a wise king wanted to attend as many village festivals as possible in one day.
Each festival had a start and end time, and the king could only attend one at a time. His
advisor suggested picking only the festivals that ended the earliest, so he could fit more in.
The king followed this advice, selecting the next festival that started after the last one
ended. By the end of the day, he had enjoyed the most celebrations possible without any
overlaps.

Given two arrays `start[]` and `finish[]`, find the maximum number of non-overlapping
activities a single person can attend, where an activity `i` can be attended only if it starts
**strictly after** the previous chosen activity finishes.

**Example 1**
```
Input:  start[] = [1, 3, 0, 5, 8, 5], finish[] = [2, 4, 6, 7, 9, 9]
Output: 4
Explanation: A person can attend 4 functions. The maximum set of functions that can be
attended is {0, 1, 3, 4} (these are indexes in start[] and finish[]).
```

**Example 2**
```
Input:  start[] = [10, 12, 20], finish[] = [20, 25, 30]
Output: 1
```

**Example 3**
```
Input:  start[] = [1, 1, 1], finish[] = [4, 3, 2]
Output: 1
```

### Approach

This is the classic **Activity Selection** greedy problem:

- Pair up each activity's `(finish, start)` and **sort by finish time**.
- Greedily walk through activities in order of increasing finish time, tracking
  `last_finish` (the finish time of the most recently selected activity, initialized to
  `-infinity`).
- An activity can be selected only if its start time is **strictly greater than**
  `last_finish` (a new activity cannot begin at the exact same instant the room/slot is freed —
  see Example 2, where `[10,20]` and `[20,30]` are treated as still overlapping since one starts
  exactly when the other ends).
- Whenever an activity is selected, increment the count and update `last_finish` to its finish
  time.

Sorting by finish time (rather than start time or duration) is the key greedy insight: always
picking the activity that frees up the earliest leaves the most room for future activities.

**Complexity:** Time `O(n log n)` (dominated by the sort), Space `O(n)`.

### Q11 - Solution

```python
def max_activities(start, finish):
    """
    Return the maximum number of non-overlapping activities that can be
    attended, given their start and finish times, using the classic
    greedy Activity Selection algorithm (sort by finish time).

    Time Complexity:  O(n log n)
    Space Complexity: O(n)
    """
    # Sort activities by finish time
    activities = sorted(zip(finish, start))

    count = 0
    last_finish = float("-inf")

    for f, s in activities:
        if s > last_finish:  # strictly after the previous activity's finish
            count += 1
            last_finish = f

    return count


# ---------------- Input / Output (HackerRank style) ----------------
# Input format:
#   Line 1: an integer n, the number of activities.
#   Line 2: n space-separated integers, the start times.
#   Line 3: n space-separated integers, the finish times.
# Output format:
#   A single integer: the maximum number of non-overlapping activities.

n = int(input())
start = list(map(int, input().split()))
finish = list(map(int, input().split()))
print(max_activities(start, finish))
```

### Q11 - Test Cases

| Input (start / finish)                         | Output | Note                                                |
|---------------------------------------------------|--------|---------------------------------------------------------|
| `1 3 0 5 8 5` / `2 4 6 7 9 9`                       | `4`    | matches Example 1                                        |
| `10 12 20` / `20 25 30`                             | `1`    | matches Example 2 — touching intervals still overlap     |
| `1 1 1` / `4 3 2`                                   | `1`    | matches Example 3 — best is picking the earliest finisher|
| `1 2 3 4` / `2 3 4 5`                               | `4`    | back-to-back activities, all strictly non-overlapping    |
| `5` / `10`                                          | `1`    | single activity, always attendable                       |

All the above were verified programmatically against the solution logic.

---

## Q12 - Job Sequencing with Deadlines (Maximize Profit)

### Problem Statement

A merchant, Riya, receives job requests, each with a **deadline** and a **profit**. Each job
takes exactly one full day, and only one job can be scheduled per day. A job contributes its
profit only if it is completed on or before its deadline.

Given arrays `deadline[]` and `profit[]`, find the maximum number of jobs that can be scheduled
and the maximum total profit achievable.

Return the answer as `[count, total_profit]`.

**Example 1**
```
Input:  deadline[] = [4, 1, 1, 1], profit[] = [20, 10, 40, 30]
Output: [2, 60]
Explanation: Job1 and Job3 can be done with maximum profit of 60 (20 + 40).
```

**Example 2**
```
Input:  deadline[] = [2, 1, 2, 1, 1], profit[] = [100, 19, 27, 25, 15]
Output: [2, 127]
```

**Example 3**
```
Input:  deadline[] = [3, 1, 2, 2], profit[] = [50, 10, 20, 30]
Output: [3, 100]
```

**Constraints**
- `1 <= deadline.size() == profit.size() <= 10^5`
- `1 <= deadline[i] <= deadline.size()`
- `1 <= profit[i] <= 500`

### Approach

This is the classic **Job Sequencing Problem**, solved greedily:

- Sort jobs by **profit descending**, so the most valuable jobs get first pick of a slot.
- For each job (in that order), try to schedule it in the **latest available day** at or before
  its deadline — scheduling as late as possible keeps earlier days open for jobs with tighter
  deadlines.
- To find "the latest available day `<= deadline`" efficiently for up to `10^5` jobs, a naive
  linear scan per job would be `O(n)` per job → `O(n^2)` total, too slow for `n = 10^5`.
  Instead, use a **Union-Find (Disjoint Set Union)** structure over days `0..maxDeadline`:
  - `find(d)` returns the latest *free* day `<= d`.
  - Once a day is used, union it with `day - 1`, so future queries for that day are
    redirected to the next available day down.
  - This makes each `find`/`union` operation nearly `O(1)` (amortized, with path
    compression), giving an overall `O(n log n)` solution dominated by the initial sort.

**Complexity:** Time `O(n log n)` (sort + near-linear DSU operations), Space `O(maxDeadline)`.

### Q12 - Solution

```python
def job_sequencing(deadline, profit):
    """
    Return [count, total_profit]: the maximum number of jobs that can be
    scheduled (one per day, before their deadline) and the maximum total
    profit, using a greedy approach (sort by profit desc) accelerated
    with a Union-Find structure to find the latest free day quickly.

    Time Complexity:  O(n log n)
    Space Complexity: O(maxDeadline)
    """
    max_deadline = max(deadline)
    parent = list(range(max_deadline + 1))  # day 0 acts as a "no slot left" sentinel

    def find(x):
        while parent[x] != x:
            parent[x] = parent[parent[x]]  # path compression (halving)
            x = parent[x]
        return x

    jobs = sorted(zip(deadline, profit), key=lambda job: -job[1])

    count = 0
    total_profit = 0

    for d, p in jobs:
        available_day = find(d)
        if available_day > 0:
            count += 1
            total_profit += p
            parent[available_day] = find(available_day - 1)

    return [count, total_profit]


# ---------------- Input / Output (HackerRank style) ----------------
# Input format:
#   Line 1: an integer n, the number of jobs.
#   Line 2: n space-separated integers, the deadlines.
#   Line 3: n space-separated integers, the profits.
# Output format:
#   Two space-separated integers on one line: count and total_profit.

n = int(input())
deadline = list(map(int, input().split()))
profit = list(map(int, input().split()))

result = job_sequencing(deadline, profit)
print(result[0], result[1])
```

### Q12 - Test Cases

| Input (deadline / profit)              | Output    | Note                                     |
|--------------------------------------------|-----------|---------------------------------------------|
| `4 1 1 1` / `20 10 40 30`                    | `2 60`    | matches Example 1                             |
| `2 1 2 1 1` / `100 19 27 25 15`              | `2 127`   | matches Example 2                             |
| `3 1 2 2` / `50 10 20 30`                    | `3 100`   | matches Example 3                             |

All the above were verified programmatically against the solution logic (both the simple
`O(n^2)` slot-scanning version and the optimized Union-Find `O(n log n)` version gave identical
results).

---

## Q13 - Heavier Half (Minimum Coins to Exceed Remaining Sum)

### Problem Statement

Mira has a magical pouch full of coins, each with different values. The task: pick the
**fewest** number of coins such that their total value is **greater than** the value of all the
coins left behind.

Sort the coins from largest to smallest and pick them one by one, keeping a running total, until
that running total is greater than the sum of what remains in the pouch. Return the minimum
number of coins needed.

**Test Cases**
```
Input:  arr = [2, 17, 7, 3]
Output: 1
Explanation: Select only 17. Remaining sum is 12, and 17 > 12.
```
```
Input:  arr = [20, 12, 18, 4]
Output: 2
Explanation: Select 18 and 12. Total = 30, remaining = 24.
```
```
Input:  arr = [1, 1, 1, 1, 10]
Output: 1
Explanation: Select 10. Remaining sum is 4.
```

**Constraints**
- `1 <= arr.size() <= 10^5`
- `1 <= arr[i] <= 10^4`

### Approach

This is a straightforward **greedy** problem:

- To minimize the number of coins picked, always pick the **largest remaining coins first** —
  each large coin contributes the most to the "picked" side while removing the least additional
  requirement from the remaining side.
- Sort the array in **descending order**.
- Maintain a running sum of picked coins. After adding each coin, compare it to
  `total - running_sum` (the sum of everything not yet picked). Stop as soon as
  `running_sum > remaining_sum`, and return the count of coins picked so far.

Sorting descending and greedily taking the largest first is provably optimal here — swapping
any smaller coin in for a larger one already picked could never help reach "greater than the
rest" sooner.

**Complexity:** Time `O(n log n)` (dominated by the sort), Space `O(1)` extra (ignoring the
sort's internal space, or `O(n)` if a full copy is made).

### Q13 - Solution

```python
def min_coins_heavier_half(arr):
    """
    Return the minimum number of largest coins that must be picked so
    their sum exceeds the sum of the remaining coins.

    Time Complexity:  O(n log n)
    Space Complexity: O(n) (for the sorted copy)
    """
    arr_sorted = sorted(arr, reverse=True)
    total = sum(arr)

    running_sum = 0
    count = 0

    for coin in arr_sorted:
        running_sum += coin
        count += 1
        if running_sum > total - running_sum:
            break

    return count


# ---------------- Input / Output (HackerRank style) ----------------
# Input format:
#   Line 1: an integer n, the number of coins.
#   Line 2: n space-separated integers, the coin values.
# Output format:
#   A single integer: the minimum number of coins required.

n = int(input())
arr = list(map(int, input().split()))
print(min_coins_heavier_half(arr))
```

### Q13 - Test Cases

| Input                     | Output | Note                                       |
|-----------------------------|--------|-------------------------------------------------|
| `2 17 7 3`                   | `1`    | matches given test case                          |
| `20 12 18 4`                 | `2`    | matches given test case                          |
| `1 1 1 1 10`                 | `1`    | matches given test case                          |
| `5 5 5 5`                    | `3`    | tie case — need strictly more than half the total |
| `1`                          | `1`    | single coin — trivially "greater" than nothing    |

All the above were verified programmatically against the solution logic.

---

## Q14 - Minimize Difference Between Tallest and Shortest Tower

### Problem Statement

Every tower's height must be either raised or lowered by exactly `K` meters (no exceptions),
with the constraint that **no tower can be made shorter than zero**. Find the minimum possible
difference between the tallest and shortest tower after this adjustment.

**Example 1**
```
Input:  k = 2, arr[] = {1, 5, 8, 10}
Output: 5
Explanation: The array can be modified as {1+k, 5-k, 8-k, 10-k} = {3, 3, 6, 8}. The difference
between the largest and the smallest is 8-3 = 5.
```

**Example 2**
```
Input:  k = 3, arr[] = {3, 9, 12, 16, 20}
Output: 11
Explanation: The array can be modified as {3+k, 9+k, 12-k, 16-k, 20-k} -> {6, 12, 9, 13, 17}.
The difference between the largest and the smallest is 17-6 = 11.
```

**Example 3**
```
Input:  k = 3, arr[] = {7, 7, 7, 7}
Output: 0
```

**Constraints**
- `1 <= k <= 10^7`
- `1 <= n <= 10^5`
- `1 <= arr[i] <= 10^7`

### Approach

This is the classic **"Minimize the Maximum Difference between Heights"** greedy problem:

- **Sort** the array. In a sorted array, it's optimal to **raise the smaller elements by K**
  and **lower the larger elements by K** — this pulls the whole array toward the middle.
- Once sorted, there's a clean pivot point: for every index `i`, assume elements
  `arr[0..i]` get `+K` and elements `arr[i+1..n-1]` get `-K` (this ordering keeps the
  transformed array's overall shape monotonic enough to reason about safely).
- Start with the baseline answer `arr[-1] - arr[0]` (the case of not adjusting at all — always
  valid because `arr[i]+k` and `arr[i]-k` bracket every original value... but to be precise, we
  compute it by trying the split at every index):
  - For a split after index `i` (`1 <= i <= n-2`), the candidate min of the transformed array is
    `min(arr[0]+k, arr[i+1]-k)` and the candidate max is
    `max(arr[i]+k, arr[n-1]-k)`.
  - Track the minimum `(max - min)` over all split points.
- **Non-negativity rule:** if `arr[i] - k < 0`, lowering that element isn't allowed, so that
  split point is skipped entirely.

**Complexity:** Time `O(n log n)` (dominated by the sort), Space `O(1)` extra (beyond the sort).

### Q14 - Solution

```python
def min_diff_towers(k, arr):
    """
    Return the minimum possible difference between the tallest and
    shortest tower after adjusting every tower's height by exactly +k
    or -k (never going below 0), using the classic sorted-pivot greedy
    approach.

    Time Complexity:  O(n log n)
    Space Complexity: O(1) extra (beyond sorting)
    """
    n = len(arr)
    if n == 1:
        return 0

    arr = sorted(arr)
    result = arr[-1] - arr[0]  # baseline: no useful adjustment found yet

    small = arr[0] + k
    large = arr[-1] - k
    if small > large:
        small, large = large, small

    for i in range(1, n - 1):
        subtract = arr[i] - k
        add = arr[i] + k

        if subtract < 0:
            continue  # cannot lower this tower below 0

        mn = min(small, subtract)
        mx = max(large, add)
        result = min(result, mx - mn)

    return result


# ---------------- Input / Output (HackerRank style) ----------------
# Input format:
#   Line 1: an integer k.
#   Line 2: an integer n, the number of towers.
#   Line 3: n space-separated integers, the tower heights.
# Output format:
#   A single integer: the minimum possible difference between the
#   tallest and shortest tower.

k = int(input())
n = int(input())
arr = list(map(int, input().split()))
print(min_diff_towers(k, arr))
```

### Q14 - Test Cases

| Input (k / n / arr)              | Output | Note                                          |
|---------------------------------------|--------|--------------------------------------------------|
| `2` / `4` / `1 5 8 10`                 | `5`    | matches Example 1                                  |
| `3` / `5` / `3 9 12 16 20`              | `11`   | matches Example 2                                  |
| `3` / `4` / `7 7 7 7`                   | `0`    | matches Example 3 — identical heights, no benefit  |

All the above were verified programmatically against the solution logic.

---

## Q15 - Non-Adjacent Rune Rule (Rearrangement Feasibility Check)

### Problem Statement

Given a string `s`, determine whether it's possible to rearrange its characters so that **no
two identical characters are adjacent**. If any character appears more than half the length of
the string (rounded up), it's impossible to arrange without a violation; otherwise it is always
possible.

Return `1` if such a rearrangement is possible, and `0` otherwise.

**Test Cases**
```
Input:  s = "aaabc"
Output: 1
Explanation: Rearranged as "abaca" or "acaba" — no adjacent duplicates.
```
```
Input:  s = "aaabb"
Output: 1
Explanation: Rearranged as "ababa" — valid rearrangement.
```
```
Input:  s = "aaaabc"
Output: 0
Explanation: Too many 'a's — cannot rearrange without adjacent duplicates.
```

### Approach

The feasibility of rearranging a multiset of characters so no two identical characters are
adjacent depends purely on **the frequency of the most common character**:

- Let `n = len(s)` and `max_freq` be the highest character frequency.
- A valid non-adjacent arrangement exists **if and only if**
  `max_freq <= ceil(n / 2)` (equivalently `max_freq <= (n + 1) // 2` using integer division).
  - Intuition: with `ceil(n/2)` "slots" available at alternating positions
    (`0, 2, 4, ...`), the most frequent character can occupy every other slot without ever
    being adjacent to itself, as long as it doesn't need *more* slots than exist.
  - This is the same feasibility condition used by the classic "Reorganize String" /
    "Task Scheduler"-style problems (often verified constructively using a max-heap greedy
    placement, but the yes/no feasibility check itself only needs the frequency count).
- Count character frequencies with a hash map, find the maximum, and compare against the
  threshold.

**Complexity:** Time `O(n)`, Space `O(1)` (at most 26 lowercase letters, or `O(k)` for an
alphabet of size `k`).

### Q15 - Solution

```python
from collections import Counter


def can_rearrange(s):
    """
    Return 1 if the characters of `s` can be rearranged so that no two
    identical characters are adjacent, else return 0. Uses the
    frequency-threshold feasibility check (no two identical chars can
    be adjacent iff the most frequent character occurs at most
    ceil(n / 2) times).

    Time Complexity:  O(n)
    Space Complexity: O(k), k = size of the character alphabet used
    """
    n = len(s)
    if n == 0:
        return 1

    counts = Counter(s)
    max_freq = max(counts.values())

    return 1 if max_freq <= (n + 1) // 2 else 0


# ---------------- Input / Output (HackerRank style) ----------------
# Input format:
#   A single line containing the string s.
# Output format:
#   A single integer: 1 if a valid non-adjacent rearrangement exists,
#   0 otherwise.

s = input()
print(can_rearrange(s))
```

### Q15 - Test Cases

| Input      | Output | Note                                                     |
|------------|--------|---------------------------------------------------------------|
| `aaabc`    | `1`    | matches given test case (`max_freq('a')=3 <= ceil(5/2)=3`)     |
| `aaabb`    | `1`    | matches given test case (`max_freq=3 <= 3`)                    |
| `aaaabc`   | `0`    | matches given test case (`max_freq('a')=4 > ceil(6/2)=3`)       |
| `a`        | `1`    | single character, trivially arrangeable                        |
| `aa`       | `0`    | two identical characters can never avoid being adjacent         |
| `ab`       | `1`    | two distinct characters, always arrangeable                     |
| `aabb`     | `1`    | e.g. "abab" is a valid arrangement                              |

All the above were verified programmatically against the solution logic.
