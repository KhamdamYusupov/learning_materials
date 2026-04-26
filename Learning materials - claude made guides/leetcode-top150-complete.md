# LeetCode Top 150 Interview Questions — Complete Guide (Java)

---

# CATEGORY 1: Array / String

---

# #88 — Merge Sorted Array

## 1. Problem Description
Two sorted arrays `nums1` (size m+n, last n slots are 0) and `nums2` (size n). Merge `nums2` into `nums1` in-place, sorted.

## 2. Key Algorithmic Pattern
**Two Pointers (from the end)** — merge from back to avoid overwriting valid data.

## 3. Intuition
Empty slots are at the end of `nums1`. Compare largest unplaced elements from both arrays and fill from the back. Never overwrite data we still need.

## 4. Step-by-Step Approach
1. `p1 = m-1`, `p2 = n-1`, `p = m+n-1`
2. Place the larger of `nums1[p1]` or `nums2[p2]` at `nums1[p]`, advance that pointer and `p`
3. Copy any remaining `nums2` elements

## 5. Time and Space Complexity
- **Time:** O(m+n) — each element visited once
- **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public void merge(int[] nums1, int m, int[] nums2, int n) {
        int p1 = m - 1, p2 = n - 1, p = m + n - 1;
        while (p1 >= 0 && p2 >= 0) {
            if (nums1[p1] > nums2[p2]) nums1[p--] = nums1[p1--];
            else nums1[p--] = nums2[p2--];
        }
        while (p2 >= 0) nums1[p--] = nums2[p2--];
    }
}
```

---

# #27 — Remove Element

## 1. Problem Description
Remove all occurrences of `val` in-place. Return `k` (count of remaining elements).

## 2. Key Algorithmic Pattern
**Two Pointers (write pointer)**

## 3. Intuition
A slow pointer `k` marks where to write the next valid element. A fast pointer scans all elements, copying non-`val` values.

## 4. Step-by-Step Approach
1. `k = 0`
2. For each element: if `!= val`, write to `nums[k]`, increment `k`
3. Return `k`

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public int removeElement(int[] nums, int val) {
        int k = 0;
        for (int i = 0; i < nums.length; i++)
            if (nums[i] != val) nums[k++] = nums[i];
        return k;
    }
}
```

---

# #26 — Remove Duplicates from Sorted Array

## 1. Problem Description
Remove duplicates in-place from sorted array so each unique element appears once. Return `k`.

## 2. Key Algorithmic Pattern
**Two Pointers** — because sorted, duplicates are adjacent.

## 3. Intuition
Only write a new value when it differs from the last written. Skip consecutive duplicates automatically.

## 4. Step-by-Step Approach
1. `k = 1` (first element always unique)
2. For `i` from 1: if `nums[i] != nums[k-1]`, write to `nums[k]`, increment `k`
3. Return `k`

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public int removeDuplicates(int[] nums) {
        int k = 1;
        for (int i = 1; i < nums.length; i++)
            if (nums[i] != nums[k - 1]) nums[k++] = nums[i];
        return k;
    }
}
```

---

# #80 — Remove Duplicates from Sorted Array II

## 1. Problem Description
Same but allow each element **at most twice**. Return `k`.

## 2. Key Algorithmic Pattern
**Two Pointers** — compare with element 2 positions back in write buffer.

## 3. Intuition
If `nums[i] == nums[k-2]`, it would be a third occurrence — skip it. Generalizes: to allow at most `m` duplicates, compare with `nums[k-m]`.

## 4. Step-by-Step Approach
1. `k = 2` (first two always valid)
2. For `i` from 2: if `nums[i] != nums[k-2]`, write and increment `k`

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public int removeDuplicates(int[] nums) {
        if (nums.length <= 2) return nums.length;
        int k = 2;
        for (int i = 2; i < nums.length; i++)
            if (nums[i] != nums[k - 2]) nums[k++] = nums[i];
        return k;
    }
}
```

---

# #169 — Majority Element

## 1. Problem Description
Find the element appearing more than ⌊n/2⌋ times. Guaranteed to exist.

## 2. Key Algorithmic Pattern
**Boyer-Moore Voting Algorithm**

## 3. Intuition
Cancel one majority vote against one minority vote. The majority always survives because it appears more than half the time.

## 4. Step-by-Step Approach
1. `candidate = nums[0]`, `count = 1`
2. For each subsequent element: if `count == 0`, pick new candidate; else increment/decrement count
3. Return candidate

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public int majorityElement(int[] nums) {
        int candidate = nums[0], count = 1;
        for (int i = 1; i < nums.length; i++) {
            if (count == 0) { candidate = nums[i]; count = 1; }
            else if (nums[i] == candidate) count++;
            else count--;
        }
        return candidate;
    }
}
```

---

# #189 — Rotate Array

## 1. Problem Description
Rotate array to the right by `k` steps in-place.

## 2. Key Algorithmic Pattern
**Reversal Algorithm** — three reversals produce a rotation in O(1) space.

## 3. Intuition
Reversing the entire array then reversing each segment separately restores internal order while rearranging groups. `k = k % n` handles over-rotation.

## 4. Step-by-Step Approach
1. `k = k % n`
2. Reverse entire array
3. Reverse `nums[0..k-1]`
4. Reverse `nums[k..n-1]`

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public void rotate(int[] nums, int k) {
        int n = nums.length;
        k = k % n;
        reverse(nums, 0, n - 1);
        reverse(nums, 0, k - 1);
        reverse(nums, k, n - 1);
    }
    private void reverse(int[] nums, int l, int r) {
        while (l < r) { int t = nums[l]; nums[l++] = nums[r]; nums[r--] = t; }
    }
}
```

---

# #121 — Best Time to Buy and Sell Stock

## 1. Problem Description
One buy and one sell. Maximize profit. Return 0 if no profit possible.

## 2. Key Algorithmic Pattern
**Greedy** — track running minimum price.

## 3. Intuition
For each day, the best profit is selling at today's price after buying at the lowest price seen so far. Track `minPrice` and `maxProfit` simultaneously.

## 4. Step-by-Step Approach
1. `minPrice = MAX_VALUE`, `maxProfit = 0`
2. For each price: update `minPrice`, update `maxProfit = max(maxProfit, price - minPrice)`

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public int maxProfit(int[] prices) {
        int minPrice = Integer.MAX_VALUE, maxProfit = 0;
        for (int price : prices) {
            minPrice = Math.min(minPrice, price);
            maxProfit = Math.max(maxProfit, price - minPrice);
        }
        return maxProfit;
    }
}
```

---

# #122 — Best Time to Buy and Sell Stock II

## 1. Problem Description
Unlimited transactions. Maximize total profit.

## 2. Key Algorithmic Pattern
**Greedy** — capture every upward price movement.

## 3. Intuition
Buying at day 0 and selling at day 3 equals the sum of daily gains (day0→1) + (day1→2) + (day2→3). So collect every positive consecutive difference.

## 4. Step-by-Step Approach
1. For each consecutive pair: if `prices[i] > prices[i-1]`, add the difference to profit

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public int maxProfit(int[] prices) {
        int profit = 0;
        for (int i = 1; i < prices.length; i++)
            if (prices[i] > prices[i - 1]) profit += prices[i] - prices[i - 1];
        return profit;
    }
}
```

---

# #55 — Jump Game

## 1. Problem Description
`nums[i]` = max jump from index `i`. Can you reach the last index?

## 2. Key Algorithmic Pattern
**Greedy** — track farthest reachable index.

## 3. Intuition
If at any point `i > maxReach`, we're stuck — can't continue. Otherwise, constantly update how far we can reach.

## 4. Step-by-Step Approach
1. `maxReach = 0`
2. For each `i`: if `i > maxReach` return false; update `maxReach = max(maxReach, i + nums[i])`
3. Return true

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public boolean canJump(int[] nums) {
        int maxReach = 0;
        for (int i = 0; i < nums.length; i++) {
            if (i > maxReach) return false;
            maxReach = Math.max(maxReach, i + nums[i]);
        }
        return true;
    }
}
```

---

# #45 — Jump Game II

## 1. Problem Description
Minimum jumps to reach the last index (guaranteed reachable).

## 2. Key Algorithmic Pattern
**Greedy (BFS-level stepping)**

## 3. Intuition
Treat as BFS: each "level" = all positions reachable in the same number of jumps. From all positions in current level, find the farthest reach — that's the next level. Count levels.

## 4. Step-by-Step Approach
1. `jumps = 0`, `currentEnd = 0`, `farthest = 0`
2. For `i` from 0 to n-2: update `farthest`; when `i == currentEnd`, increment `jumps`, `currentEnd = farthest`

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public int jump(int[] nums) {
        int jumps = 0, currentEnd = 0, farthest = 0;
        for (int i = 0; i < nums.length - 1; i++) {
            farthest = Math.max(farthest, i + nums[i]);
            if (i == currentEnd) { jumps++; currentEnd = farthest; }
        }
        return jumps;
    }
}
```

---

# #274 — H-Index

## 1. Problem Description
Return H-Index: largest `h` such that at least `h` papers have ≥ `h` citations.

## 2. Key Algorithmic Pattern
**Sorting + Linear Scan**

## 3. Intuition
Sort ascending. For each index `i`, `n - i` papers have at least `citations[i]` citations. The first time `citations[i] >= n - i`, that's our answer.

## 4. Step-by-Step Approach
1. Sort ascending
2. For each `i`: if `citations[i] >= n - i`, return `n - i`

## 5. Time and Space Complexity
- **Time:** O(n log n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public int hIndex(int[] citations) {
        Arrays.sort(citations);
        int n = citations.length, h = 0;
        for (int i = 0; i < n; i++) {
            int papers = n - i;
            if (citations[i] >= papers) { h = papers; break; }
        }
        return h;
    }
}
```

---

# #380 — Insert Delete GetRandom O(1)

## 1. Problem Description
Design a set with O(1) average insert, remove, and getRandom.

## 2. Key Algorithmic Pattern
**HashMap + ArrayList** — swap-with-last trick for O(1) delete.

## 3. Intuition
HashMap gives O(1) lookup/delete index. ArrayList gives O(1) random access. To delete: swap target with last element, update map, pop last.

## 4. Step-by-Step Approach
- **Insert:** add to list end, store index in map
- **Remove:** swap with last, update map for swapped, remove last
- **GetRandom:** `list.get(random index)`

## 5. Time and Space Complexity
- **Time:** O(1) average all ops, **Space:** O(n)

## 6. Java Implementation
```java
class RandomizedSet {
    private Map<Integer, Integer> indexMap = new HashMap<>();
    private List<Integer> list = new ArrayList<>();
    private Random random = new Random();

    public boolean insert(int val) {
        if (indexMap.containsKey(val)) return false;
        list.add(val);
        indexMap.put(val, list.size() - 1);
        return true;
    }

    public boolean remove(int val) {
        if (!indexMap.containsKey(val)) return false;
        int idx = indexMap.get(val), lastVal = list.get(list.size() - 1);
        list.set(idx, lastVal);
        indexMap.put(lastVal, idx);
        list.remove(list.size() - 1);
        indexMap.remove(val);
        return true;
    }

    public int getRandom() { return list.get(random.nextInt(list.size())); }
}
```

---

# #238 — Product of Array Except Self

## 1. Problem Description
Return array where `answer[i]` = product of all elements except `nums[i]`. No division, O(n).

## 2. Key Algorithmic Pattern
**Prefix and Suffix Products**

## 3. Intuition
`answer[i]` = (product of all left of i) × (product of all right of i). Forward pass builds left products; backward pass multiplies in right products.

## 4. Step-by-Step Approach
1. Forward: `answer[i] = answer[i-1] * nums[i-1]`
2. Backward: multiply `answer[i] *= rightProduct`, then `rightProduct *= nums[i]`

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(1) extra

## 6. Java Implementation
```java
class Solution {
    public int[] productExceptSelf(int[] nums) {
        int n = nums.length;
        int[] answer = new int[n];
        answer[0] = 1;
        for (int i = 1; i < n; i++) answer[i] = answer[i - 1] * nums[i - 1];
        int right = 1;
        for (int i = n - 1; i >= 0; i--) { answer[i] *= right; right *= nums[i]; }
        return answer;
    }
}
```

---

# #134 — Gas Station

## 1. Problem Description
Find starting gas station index to complete a circular route. Return -1 if impossible.

## 2. Key Algorithmic Pattern
**Greedy** — if tank goes negative, the start must be after the failing station.

## 3. Intuition
If we run out at station `j` starting from `i`, no station between `i` and `j` can be the answer (all have less surplus). Jump start to `j+1`.

## 4. Step-by-Step Approach
1. `totalGas = 0`, `currentTank = 0`, `startStation = 0`
2. For each station: add net gain; if `currentTank < 0`, reset start to `i+1`, tank to 0
3. Return `startStation` if `totalGas >= 0`, else -1

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public int canCompleteCircuit(int[] gas, int[] cost) {
        int totalGas = 0, currentTank = 0, startStation = 0;
        for (int i = 0; i < gas.length; i++) {
            int net = gas[i] - cost[i];
            totalGas += net;
            currentTank += net;
            if (currentTank < 0) { startStation = i + 1; currentTank = 0; }
        }
        return totalGas >= 0 ? startStation : -1;
    }
}
```

---

# #135 — Candy

## 1. Problem Description
Minimum candies: each child ≥ 1, higher-rated child must get more than adjacent lower-rated neighbor.

## 2. Key Algorithmic Pattern
**Greedy (Two-Pass)**

## 3. Intuition
One pass can't capture both directions. Left pass handles left-neighbor constraint; right pass handles right-neighbor constraint. Take max of both at each position.

## 4. Step-by-Step Approach
1. Fill all with 1
2. Left→right: if `ratings[i] > ratings[i-1]`, `candies[i] = candies[i-1] + 1`
3. Right→left: if `ratings[i] > ratings[i+1]`, `candies[i] = max(candies[i], candies[i+1] + 1)`
4. Sum all

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(n)

## 6. Java Implementation
```java
class Solution {
    public int candy(int[] ratings) {
        int n = ratings.length;
        int[] candies = new int[n];
        Arrays.fill(candies, 1);
        for (int i = 1; i < n; i++)
            if (ratings[i] > ratings[i - 1]) candies[i] = candies[i - 1] + 1;
        for (int i = n - 2; i >= 0; i--)
            if (ratings[i] > ratings[i + 1]) candies[i] = Math.max(candies[i], candies[i + 1] + 1);
        int total = 0;
        for (int c : candies) total += c;
        return total;
    }
}
```

---

# #42 — Trapping Rain Water

## 1. Problem Description
Given elevation map, compute total water trapped.

## 2. Key Algorithmic Pattern
**Two Pointers**

## 3. Intuition
Water at `i` = `min(maxLeft, maxRight) - height[i]`. The shorter side is the bottleneck. Move the pointer at the shorter bar — the taller side guarantees at least that height.

## 4. Step-by-Step Approach
1. `left = 0`, `right = n-1`, `leftMax = 0`, `rightMax = 0`, `water = 0`
2. If `height[left] <= height[right]`: update `leftMax`, add `leftMax - height[left]`, advance `left`
3. Else: update `rightMax`, add `rightMax - height[right]`, advance `right`

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public int trap(int[] height) {
        int left = 0, right = height.length - 1;
        int leftMax = 0, rightMax = 0, water = 0;
        while (left < right) {
            if (height[left] <= height[right]) {
                leftMax = Math.max(leftMax, height[left]);
                water += leftMax - height[left++];
            } else {
                rightMax = Math.max(rightMax, height[right]);
                water += rightMax - height[right--];
            }
        }
        return water;
    }
}
```

---

# #13 — Roman to Integer

## 1. Problem Description
Convert Roman numeral string to integer. Subtract when smaller value precedes larger.

## 2. Key Algorithmic Pattern
**Linear Scan with HashMap**

## 3. Intuition
If `value[i] < value[i+1]`, subtract `value[i]` (subtractive notation). Otherwise add.

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public int romanToInt(String s) {
        Map<Character, Integer> map = Map.of('I',1,'V',5,'X',10,'L',50,'C',100,'D',500,'M',1000);
        int result = 0;
        for (int i = 0; i < s.length(); i++) {
            int curr = map.get(s.charAt(i));
            int next = i + 1 < s.length() ? map.get(s.charAt(i + 1)) : 0;
            result += curr < next ? -curr : curr;
        }
        return result;
    }
}
```

---

# #12 — Integer to Roman

## 1. Problem Description
Convert integer (1–3999) to Roman numeral.

## 2. Key Algorithmic Pattern
**Greedy with value-symbol table** (including subtractive pairs like CM, CD).

## 3. Intuition
Greedily subtract the largest fitting value, appending its symbol, until 0.

## 5. Time and Space Complexity
- **Time:** O(1) — bounded by 3999, **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public String intToRoman(int num) {
        int[]    values  = {1000,900,500,400,100,90,50,40,10,9,5,4,1};
        String[] symbols = {"M","CM","D","CD","C","XC","L","XL","X","IX","V","IV","I"};
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < values.length; i++)
            while (num >= values[i]) { sb.append(symbols[i]); num -= values[i]; }
        return sb.toString();
    }
}
```

---

# #58 — Length of Last Word

## 1. Problem Description
Return length of the last word in string `s` (may have trailing spaces).

## 2. Key Algorithmic Pattern
**Linear Scan from End**

## 6. Java Implementation
```java
class Solution {
    public int lengthOfLastWord(String s) {
        int i = s.length() - 1;
        while (i >= 0 && s.charAt(i) == ' ') i--;
        int len = 0;
        while (i >= 0 && s.charAt(i) != ' ') { len++; i--; }
        return len;
    }
}
```
**Time:** O(n), **Space:** O(1)

---

# #14 — Longest Common Prefix

## 1. Problem Description
Return longest common prefix of all strings in array.

## 2. Key Algorithmic Pattern
**Vertical Scanning** — compare column by column across all strings.

## 6. Java Implementation
```java
class Solution {
    public String longestCommonPrefix(String[] strs) {
        for (int i = 0; i < strs[0].length(); i++) {
            char c = strs[0].charAt(i);
            for (int j = 1; j < strs.length; j++)
                if (i >= strs[j].length() || strs[j].charAt(i) != c)
                    return strs[0].substring(0, i);
        }
        return strs[0];
    }
}
```
**Time:** O(S) total characters, **Space:** O(1)

---

# #151 — Reverse Words in a String

## 1. Problem Description
Reverse word order; remove extra spaces.

## 2. Key Algorithmic Pattern
**Split + Reverse + Join**

## 6. Java Implementation
```java
class Solution {
    public String reverseWords(String s) {
        String[] words = s.trim().split("\\s+");
        int l = 0, r = words.length - 1;
        while (l < r) { String t = words[l]; words[l++] = words[r]; words[r--] = t; }
        return String.join(" ", words);
    }
}
```
**Time:** O(n), **Space:** O(n)

---

# #6 — Zigzag Conversion

## 1. Problem Description
Write string in zigzag across `numRows`, read row by row.

## 2. Key Algorithmic Pattern
**Simulation with direction flag**

## 3. Intuition
Simulate zigzag: track `currentRow` and `direction` (+1 down, -1 up). Flip at top and bottom rows.

## 6. Java Implementation
```java
class Solution {
    public String convert(String s, int numRows) {
        if (numRows == 1) return s;
        StringBuilder[] rows = new StringBuilder[numRows];
        for (int i = 0; i < numRows; i++) rows[i] = new StringBuilder();
        int cur = 0, dir = -1;
        for (char c : s.toCharArray()) {
            rows[cur].append(c);
            if (cur == 0 || cur == numRows - 1) dir = -dir;
            cur += dir;
        }
        StringBuilder res = new StringBuilder();
        for (StringBuilder row : rows) res.append(row);
        return res.toString();
    }
}
```
**Time:** O(n), **Space:** O(n)

---

# #28 — Find the Index of the First Occurrence in a String

## 1. Problem Description
Return first occurrence index of `needle` in `haystack`, or -1.

## 2. Key Algorithmic Pattern
**Sliding Window** (or KMP for O(n+m))

## 6. Java Implementation
```java
class Solution {
    public int strStr(String haystack, String needle) {
        int n = haystack.length(), m = needle.length();
        for (int i = 0; i <= n - m; i++)
            if (haystack.substring(i, i + m).equals(needle)) return i;
        return -1;
    }
}
```
**Time:** O(n*m) naive, **Space:** O(1)

---

# #68 — Text Justification

## 1. Problem Description
Format text to exactly `maxWidth` chars per line with full justification (spaces distributed evenly; last line left-justified).

## 2. Key Algorithmic Pattern
**Greedy line packing + String simulation**

## 3. Intuition
Greedily pack words. For each line: distribute `totalSpaces / (gaps)` per gap, with leftmost gaps getting one extra for the remainder.

## 6. Java Implementation
```java
class Solution {
    public List<String> fullJustify(String[] words, int maxWidth) {
        List<String> result = new ArrayList<>();
        int i = 0, n = words.length;
        while (i < n) {
            int lineLen = words[i].length(), j = i + 1;
            while (j < n && lineLen + 1 + words[j].length() <= maxWidth)
                lineLen += 1 + words[j].length(), j++;
            int numWords = j - i;
            int numSpaces = maxWidth - lineLen + (numWords - 1);
            StringBuilder sb = new StringBuilder(words[i]);
            if (j == n || numWords == 1) {
                for (int k = i + 1; k < j; k++) sb.append(' ').append(words[k]);
                while (sb.length() < maxWidth) sb.append(' ');
            } else {
                int spacePerGap = numSpaces / (numWords - 1);
                int extra = numSpaces % (numWords - 1);
                for (int k = i + 1; k < j; k++) {
                    int sp = spacePerGap + (k - i <= extra ? 1 : 0);
                    sb.append(" ".repeat(sp)).append(words[k]);
                }
            }
            result.add(sb.toString());
            i = j;
        }
        return result;
    }
}
```
**Time:** O(n), **Space:** O(n)

---

# CATEGORY 2: Two Pointers

---

# #125 — Valid Palindrome

## 1. Problem Description
Return true if string is a palindrome after keeping only alphanumeric chars (case-insensitive).

## 2. Key Algorithmic Pattern
**Two Pointers**

## 3. Intuition
Skip non-alphanumeric from both ends, compare case-insensitively. Mismatch means not palindrome.

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public boolean isPalindrome(String s) {
        int left = 0, right = s.length() - 1;
        while (left < right) {
            while (left < right && !Character.isLetterOrDigit(s.charAt(left))) left++;
            while (left < right && !Character.isLetterOrDigit(s.charAt(right))) right--;
            if (Character.toLowerCase(s.charAt(left)) != Character.toLowerCase(s.charAt(right))) return false;
            left++; right--;
        }
        return true;
    }
}
```

---

# #392 — Is Subsequence

## 1. Problem Description
Return true if `s` is a subsequence of `t` (all chars of `s` appear in `t` in order).

## 2. Key Algorithmic Pattern
**Two Pointers** — advance `s` pointer only on match.

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public boolean isSubsequence(String s, String t) {
        int i = 0, j = 0;
        while (i < s.length() && j < t.length()) {
            if (s.charAt(i) == t.charAt(j)) i++;
            j++;
        }
        return i == s.length();
    }
}
```

---

# #167 — Two Sum II - Input Array Is Sorted

## 1. Problem Description
Find two numbers in sorted array summing to target. Return 1-based indices.

## 2. Key Algorithmic Pattern
**Two Pointers** — sorted property allows convergence.

## 3. Intuition
Start at both ends. Sum too large → move right pointer left. Sum too small → move left pointer right.

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public int[] twoSum(int[] numbers, int target) {
        int left = 0, right = numbers.length - 1;
        while (left < right) {
            int sum = numbers[left] + numbers[right];
            if (sum == target) return new int[]{left + 1, right + 1};
            else if (sum < target) left++;
            else right--;
        }
        return new int[]{-1, -1};
    }
}
```

---

# #11 — Container With Most Water

## 1. Problem Description
Find two lines maximizing the container area (water).

## 2. Key Algorithmic Pattern
**Two Pointers**

## 3. Intuition
Start widest possible. The shorter bar is the bottleneck. Moving the taller bar inward can never increase water (width decreases, height limited by the shorter). Always move the pointer at the shorter bar.

## 4. Step-by-Step Approach
1. `left = 0`, `right = n-1`, `maxWater = 0`
2. `water = min(h[left], h[right]) * (right - left)`, update max
3. Move pointer at shorter bar

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public int maxArea(int[] height) {
        int left = 0, right = height.length - 1, maxWater = 0;
        while (left < right) {
            maxWater = Math.max(maxWater, Math.min(height[left], height[right]) * (right - left));
            if (height[left] < height[right]) left++;
            else right--;
        }
        return maxWater;
    }
}
```

---

# #15 — 3Sum

## 1. Problem Description
Find all unique triplets in array summing to zero.

## 2. Key Algorithmic Pattern
**Sort + Two Pointers**

## 3. Intuition
Fix one element, then two-pointer scan the rest. Sort enables skipping duplicates and converging efficiently.

## 4. Step-by-Step Approach
1. Sort `nums`
2. For each `i`: skip duplicates, break if `nums[i] > 0`
3. Two-pointer scan `left = i+1`, `right = n-1`; skip duplicates on match

## 5. Time and Space Complexity
- **Time:** O(n²), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public List<List<Integer>> threeSum(int[] nums) {
        Arrays.sort(nums);
        List<List<Integer>> result = new ArrayList<>();
        for (int i = 0; i < nums.length - 2; i++) {
            if (i > 0 && nums[i] == nums[i - 1]) continue;
            if (nums[i] > 0) break;
            int left = i + 1, right = nums.length - 1;
            while (left < right) {
                int sum = nums[i] + nums[left] + nums[right];
                if (sum == 0) {
                    result.add(Arrays.asList(nums[i], nums[left], nums[right]));
                    while (left < right && nums[left] == nums[left + 1]) left++;
                    while (left < right && nums[right] == nums[right - 1]) right--;
                    left++; right--;
                } else if (sum < 0) left++;
                else right--;
            }
        }
        return result;
    }
}
```

---

# CATEGORY 3: Sliding Window

---

# #209 — Minimum Size Subarray Sum

## 1. Problem Description
Minimal length contiguous subarray with sum ≥ target (positive integers). Return 0 if none.

## 2. Key Algorithmic Pattern
**Sliding Window** — all values positive so window sum is monotone.

## 3. Intuition
Expand right until sum ≥ target. Then contract left to minimize while maintaining constraint.

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public int minSubArrayLen(int target, int[] nums) {
        int left = 0, sum = 0, minLen = Integer.MAX_VALUE;
        for (int right = 0; right < nums.length; right++) {
            sum += nums[right];
            while (sum >= target) {
                minLen = Math.min(minLen, right - left + 1);
                sum -= nums[left++];
            }
        }
        return minLen == Integer.MAX_VALUE ? 0 : minLen;
    }
}
```

---

# #3 — Longest Substring Without Repeating Characters

## 1. Problem Description
Length of longest substring with all unique characters.

## 2. Key Algorithmic Pattern
**Sliding Window + HashMap**

## 3. Intuition
Track last seen index of each character. On duplicate, jump `left` past the previous occurrence rather than advancing one-by-one.

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(min(n, 128))

## 6. Java Implementation
```java
class Solution {
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
}
```

---

# #30 — Substring with Concatenation of All Words

## 1. Problem Description
Find all start indices of substrings that are exact concatenations of all words (any order, each once).

## 2. Key Algorithmic Pattern
**Sliding Window + HashMap (per word-length offset)**

## 3. Intuition
All words are the same length `w`. Run `w` separate sliding windows (one per offset 0..w-1). Each window tracks word frequency and a match count.

## 5. Time and Space Complexity
- **Time:** O(n * w) amortized, **Space:** O(k)

## 6. Java Implementation
```java
class Solution {
    public List<Integer> findSubstring(String s, String[] words) {
        List<Integer> result = new ArrayList<>();
        if (s.isEmpty() || words.length == 0) return result;
        int w = words[0].length(), k = words.length, n = s.length();
        Map<String, Integer> wordFreq = new HashMap<>();
        for (String word : words) wordFreq.merge(word, 1, Integer::sum);
        for (int i = 0; i < w; i++) {
            Map<String, Integer> windowFreq = new HashMap<>();
            int left = i, count = 0;
            for (int right = i; right + w <= n; right += w) {
                String word = s.substring(right, right + w);
                if (wordFreq.containsKey(word)) {
                    windowFreq.merge(word, 1, Integer::sum);
                    count++;
                    while (windowFreq.get(word) > wordFreq.get(word)) {
                        String lw = s.substring(left, left + w);
                        windowFreq.merge(lw, -1, Integer::sum);
                        count--; left += w;
                    }
                    if (count == k) result.add(left);
                } else { windowFreq.clear(); count = 0; left = right + w; }
            }
        }
        return result;
    }
}
```

---

# #76 — Minimum Window Substring

## 1. Problem Description
Smallest substring of `s` containing all characters of `t` (including duplicates). Return "" if none.

## 2. Key Algorithmic Pattern
**Sliding Window + Frequency Maps**

## 3. Intuition
Expand `right` until all of `t` is covered. Then contract `left` to minimize. Track `formed` (distinct chars meeting required frequency) to know when the window is valid.

## 5. Time and Space Complexity
- **Time:** O(|s| + |t|), **Space:** O(|s| + |t|)

## 6. Java Implementation
```java
class Solution {
    public String minWindow(String s, String t) {
        if (t.isEmpty()) return "";
        Map<Character, Integer> tFreq = new HashMap<>();
        for (char c : t.toCharArray()) tFreq.merge(c, 1, Integer::sum);
        int required = tFreq.size(), formed = 0;
        Map<Character, Integer> windowFreq = new HashMap<>();
        int left = 0, minLen = Integer.MAX_VALUE, minLeft = 0;
        for (int right = 0; right < s.length(); right++) {
            char c = s.charAt(right);
            windowFreq.merge(c, 1, Integer::sum);
            if (tFreq.containsKey(c) && windowFreq.get(c).equals(tFreq.get(c))) formed++;
            while (formed == required) {
                if (right - left + 1 < minLen) { minLen = right - left + 1; minLeft = left; }
                char lc = s.charAt(left++);
                windowFreq.merge(lc, -1, Integer::sum);
                if (tFreq.containsKey(lc) && windowFreq.get(lc) < tFreq.get(lc)) formed--;
            }
        }
        return minLen == Integer.MAX_VALUE ? "" : s.substring(minLeft, minLeft + minLen);
    }
}
```

---

# CATEGORY 4: Matrix

---

# #36 — Valid Sudoku

## 1. Problem Description
Determine if a partially filled 9×9 Sudoku board is valid (no row/column/box repeats).

## 2. Key Algorithmic Pattern
**HashSet Validation**

## 3. Intuition
For each digit, check row, column, and 3×3 box simultaneously using sets. Box index = `(r/3)*3 + c/3`.

## 5. Time and Space Complexity
- **Time:** O(1) (fixed 81 cells), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public boolean isValidSudoku(char[][] board) {
        Set<Character>[] rows = new HashSet[9], cols = new HashSet[9], boxes = new HashSet[9];
        for (int i = 0; i < 9; i++) { rows[i] = new HashSet<>(); cols[i] = new HashSet<>(); boxes[i] = new HashSet<>(); }
        for (int r = 0; r < 9; r++) {
            for (int c = 0; c < 9; c++) {
                char val = board[r][c];
                if (val == '.') continue;
                int boxIdx = (r / 3) * 3 + c / 3;
                if (!rows[r].add(val) || !cols[c].add(val) || !boxes[boxIdx].add(val)) return false;
            }
        }
        return true;
    }
}
```

---

# #54 — Spiral Matrix

## 1. Problem Description
Return all elements of an m×n matrix in clockwise spiral order.

## 2. Key Algorithmic Pattern
**Layer-by-Layer Boundary Simulation**

## 3. Intuition
Maintain four shrinking boundaries (top, bottom, left, right). Traverse each edge in order, then shrink that boundary.

## 5. Time and Space Complexity
- **Time:** O(m*n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public List<Integer> spiralOrder(int[][] matrix) {
        List<Integer> result = new ArrayList<>();
        int top = 0, bottom = matrix.length - 1, left = 0, right = matrix[0].length - 1;
        while (top <= bottom && left <= right) {
            for (int c = left; c <= right; c++) result.add(matrix[top][c]); top++;
            for (int r = top; r <= bottom; r++) result.add(matrix[r][right]); right--;
            if (top <= bottom) { for (int c = right; c >= left; c--) result.add(matrix[bottom][c]); bottom--; }
            if (left <= right) { for (int r = bottom; r >= top; r--) result.add(matrix[r][left]); left++; }
        }
        return result;
    }
}
```

---

# #48 — Rotate Image

## 1. Problem Description
Rotate an n×n matrix 90° clockwise in-place.

## 2. Key Algorithmic Pattern
**Transpose + Reverse each row**

## 3. Intuition
Transpose (swap `[i][j]` and `[j][i]`) reorders diagonally. Then reversing each row completes the clockwise rotation.

## 5. Time and Space Complexity
- **Time:** O(n²), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public void rotate(int[][] matrix) {
        int n = matrix.length;
        for (int i = 0; i < n; i++)
            for (int j = i + 1; j < n; j++) { int t = matrix[i][j]; matrix[i][j] = matrix[j][i]; matrix[j][i] = t; }
        for (int[] row : matrix) {
            int l = 0, r = n - 1;
            while (l < r) { int t = row[l]; row[l++] = row[r]; row[r--] = t; }
        }
    }
}
```

---

# #73 — Set Matrix Zeroes

## 1. Problem Description
If any cell is 0, set its entire row and column to 0. In-place.

## 2. Key Algorithmic Pattern
**In-Place Marking using first row/column as flags**

## 3. Intuition
Use first row and first column to remember which rows/columns need zeroing. Handle them separately to avoid corrupting the flags.

## 5. Time and Space Complexity
- **Time:** O(m*n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public void setZeroes(int[][] matrix) {
        int m = matrix.length, n = matrix[0].length;
        boolean firstRow = false, firstCol = false;
        for (int j = 0; j < n; j++) if (matrix[0][j] == 0) firstRow = true;
        for (int i = 0; i < m; i++) if (matrix[i][0] == 0) firstCol = true;
        for (int i = 1; i < m; i++) for (int j = 1; j < n; j++) if (matrix[i][j] == 0) { matrix[i][0] = 0; matrix[0][j] = 0; }
        for (int i = 1; i < m; i++) for (int j = 1; j < n; j++) if (matrix[i][0] == 0 || matrix[0][j] == 0) matrix[i][j] = 0;
        if (firstRow) Arrays.fill(matrix[0], 0);
        if (firstCol) for (int i = 0; i < m; i++) matrix[i][0] = 0;
    }
}
```

---

# #289 — Game of Life

## 1. Problem Description
Apply one step of Conway's Game of Life in-place.

## 2. Key Algorithmic Pattern
**In-Place Encoding** (2 = alive→dead, -1 = dead→alive)

## 3. Intuition
Encode transitions with extra values so we can read original state (by magnitude/sign) while writing new state. Second pass decodes.

## 5. Time and Space Complexity
- **Time:** O(m*n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public void gameOfLife(int[][] board) {
        int m = board.length, n = board[0].length;
        int[] dirs = {-1, 0, 1};
        for (int i = 0; i < m; i++) {
            for (int j = 0; j < n; j++) {
                int live = 0;
                for (int di : dirs) for (int dj : dirs) {
                    if (di == 0 && dj == 0) continue;
                    int ni = i + di, nj = j + dj;
                    if (ni >= 0 && ni < m && nj >= 0 && nj < n && Math.abs(board[ni][nj]) == 1) live++;
                }
                if (board[i][j] == 1 && (live < 2 || live > 3)) board[i][j] = 2;
                else if (board[i][j] == 0 && live == 3) board[i][j] = -1;
            }
        }
        for (int i = 0; i < m; i++) for (int j = 0; j < n; j++) board[i][j] = board[i][j] > 0 ? 1 : 0;
    }
}
```

---

# CATEGORY 5: HashMap

---

# #383 — Ransom Note

## 1. Problem Description
Can `ransomNote` be built using letters from `magazine` (each used at most once)?

## 2. Key Algorithmic Pattern
**Frequency Count**

## 6. Java Implementation
```java
class Solution {
    public boolean canConstruct(String ransomNote, String magazine) {
        int[] freq = new int[26];
        for (char c : magazine.toCharArray()) freq[c - 'a']++;
        for (char c : ransomNote.toCharArray()) if (--freq[c - 'a'] < 0) return false;
        return true;
    }
}
```
**Time:** O(n), **Space:** O(1)

---

# #205 — Isomorphic Strings

## 1. Problem Description
Return true if `s` and `t` are isomorphic (bijective char-to-char mapping).

## 2. Key Algorithmic Pattern
**Bidirectional HashMap** — one map isn't enough; need both directions.

## 6. Java Implementation
```java
class Solution {
    public boolean isIsomorphic(String s, String t) {
        Map<Character, Character> sToT = new HashMap<>(), tToS = new HashMap<>();
        for (int i = 0; i < s.length(); i++) {
            char sc = s.charAt(i), tc = t.charAt(i);
            if (sToT.containsKey(sc) && sToT.get(sc) != tc) return false;
            if (tToS.containsKey(tc) && tToS.get(tc) != sc) return false;
            sToT.put(sc, tc); tToS.put(tc, sc);
        }
        return true;
    }
}
```
**Time:** O(n), **Space:** O(1)

---

# #290 — Word Pattern

## 1. Problem Description
Return true if string `s` (space-separated words) follows the same pattern as `pattern` (bijective char↔word mapping).

## 2. Key Algorithmic Pattern
**Bidirectional HashMap** (char→word and word→char)

## 6. Java Implementation
```java
class Solution {
    public boolean wordPattern(String pattern, String s) {
        String[] words = s.split(" ");
        if (pattern.length() != words.length) return false;
        Map<Character, String> cw = new HashMap<>(); Map<String, Character> wc = new HashMap<>();
        for (int i = 0; i < pattern.length(); i++) {
            char c = pattern.charAt(i); String w = words[i];
            if (cw.containsKey(c) && !cw.get(c).equals(w)) return false;
            if (wc.containsKey(w) && wc.get(w) != c) return false;
            cw.put(c, w); wc.put(w, c);
        }
        return true;
    }
}
```
**Time:** O(n), **Space:** O(n)

---

# #242 — Valid Anagram

## 1. Problem Description
Return true if `t` is an anagram of `s` (same characters, same frequencies).

## 2. Key Algorithmic Pattern
**Frequency Count**

## 6. Java Implementation
```java
class Solution {
    public boolean isAnagram(String s, String t) {
        if (s.length() != t.length()) return false;
        int[] freq = new int[26];
        for (char c : s.toCharArray()) freq[c - 'a']++;
        for (char c : t.toCharArray()) if (--freq[c - 'a'] < 0) return false;
        return true;
    }
}
```
**Time:** O(n), **Space:** O(1)

---

# #49 — Group Anagrams

## 1. Problem Description
Group strings that are anagrams of each other.

## 2. Key Algorithmic Pattern
**HashMap with sorted-string key**

## 3. Intuition
Anagrams produce the same string when sorted. Use sorted form as the grouping key.

## 5. Time and Space Complexity
- **Time:** O(n * k log k), **Space:** O(n*k)

## 6. Java Implementation
```java
class Solution {
    public List<List<String>> groupAnagrams(String[] strs) {
        Map<String, List<String>> map = new HashMap<>();
        for (String str : strs) {
            char[] chars = str.toCharArray(); Arrays.sort(chars);
            map.computeIfAbsent(new String(chars), k -> new ArrayList<>()).add(str);
        }
        return new ArrayList<>(map.values());
    }
}
```

---

# #1 — Two Sum

## 1. Problem Description
Find indices of two numbers that add to `target`. Exactly one solution; can't reuse same element.

## 2. Key Algorithmic Pattern
**HashMap (complement lookup)** — one pass.

## 3. Intuition
For each number, check if its complement (`target - num`) was already seen. Store `num → index` as we go.

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(n)

## 6. Java Implementation
```java
class Solution {
    public int[] twoSum(int[] nums, int target) {
        Map<Integer, Integer> seen = new HashMap<>();
        for (int i = 0; i < nums.length; i++) {
            int comp = target - nums[i];
            if (seen.containsKey(comp)) return new int[]{seen.get(comp), i};
            seen.put(nums[i], i);
        }
        return new int[]{};
    }
}
```

---

# #202 — Happy Number

## 1. Problem Description
A happy number eventually reaches 1 when repeatedly replacing `n` with the sum of squares of its digits. Return true if happy.

## 2. Key Algorithmic Pattern
**HashSet for cycle detection**

## 3. Intuition
Either process reaches 1 (happy) or enters a cycle. Track seen values in a set.

## 6. Java Implementation
```java
class Solution {
    public boolean isHappy(int n) {
        Set<Integer> seen = new HashSet<>();
        while (n != 1 && !seen.contains(n)) {
            seen.add(n); int s = 0;
            while (n > 0) { int d = n % 10; s += d * d; n /= 10; }
            n = s;
        }
        return n == 1;
    }
}
```
**Time:** O(log n), **Space:** O(log n)

---

# #219 — Contains Duplicate II

## 1. Problem Description
Return true if any two identical values are at most `k` indices apart.

## 2. Key Algorithmic Pattern
**HashMap (value → last seen index)**

## 6. Java Implementation
```java
class Solution {
    public boolean containsNearbyDuplicate(int[] nums, int k) {
        Map<Integer, Integer> last = new HashMap<>();
        for (int i = 0; i < nums.length; i++) {
            if (last.containsKey(nums[i]) && i - last.get(nums[i]) <= k) return true;
            last.put(nums[i], i);
        }
        return false;
    }
}
```
**Time:** O(n), **Space:** O(n)

---

# #128 — Longest Consecutive Sequence

## 1. Problem Description
Find length of longest consecutive sequence in unsorted array. Must be O(n).

## 2. Key Algorithmic Pattern
**HashSet** — only start counting from sequence beginnings.

## 3. Intuition
Put all numbers in a set. For each `num` where `num-1` is NOT in the set (i.e., it's a sequence start), count forward. This ensures each number is processed at most twice total.

## 4. Step-by-Step Approach
1. Add all to a HashSet
2. For each `num` where `num-1` not in set: count streak forward
3. Track max streak

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(n)

## 6. Java Implementation
```java
class Solution {
    public int longestConsecutive(int[] nums) {
        Set<Integer> set = new HashSet<>();
        for (int n : nums) set.add(n);
        int max = 0;
        for (int n : set) {
            if (!set.contains(n - 1)) {
                int streak = 1;
                while (set.contains(n + streak)) streak++;
                max = Math.max(max, streak);
            }
        }
        return max;
    }
}
```

---

# CATEGORY 6: Intervals

---

# #228 — Summary Ranges

## 1. Problem Description
Given sorted unique integer array, return list of ranges covering every number.

## 2. Key Algorithmic Pattern
**Linear Scan**

## 6. Java Implementation
```java
class Solution {
    public List<String> summaryRanges(int[] nums) {
        List<String> result = new ArrayList<>();
        int i = 0, n = nums.length;
        while (i < n) {
            int start = nums[i];
            while (i + 1 < n && nums[i + 1] == nums[i] + 1) i++;
            result.add(nums[i] == start ? "" + start : start + "->" + nums[i]);
            i++;
        }
        return result;
    }
}
```
**Time:** O(n), **Space:** O(1)

---

# #56 — Merge Intervals

## 1. Problem Description
Given array of intervals, merge all overlapping ones.

## 2. Key Algorithmic Pattern
**Sort by start + linear merge**

## 3. Intuition
After sorting by start, two intervals overlap if `next.start <= current.end`. Extend current end = max of both ends.

## 5. Time and Space Complexity
- **Time:** O(n log n), **Space:** O(n)

## 6. Java Implementation
```java
class Solution {
    public int[][] merge(int[][] intervals) {
        Arrays.sort(intervals, (a, b) -> a[0] - b[0]);
        List<int[]> result = new ArrayList<>();
        int[] cur = intervals[0];
        for (int i = 1; i < intervals.length; i++) {
            if (intervals[i][0] <= cur[1]) cur[1] = Math.max(cur[1], intervals[i][1]);
            else { result.add(cur); cur = intervals[i]; }
        }
        result.add(cur);
        return result.toArray(new int[0][]);
    }
}
```

---

# #57 — Insert Interval

## 1. Problem Description
Insert `newInterval` into sorted non-overlapping intervals list, merging as needed.

## 2. Key Algorithmic Pattern
**Three-Phase Linear Scan**

## 3. Intuition
Phase 1: add intervals ending before `newInterval` starts. Phase 2: merge all overlapping with `newInterval`. Phase 3: add remaining.

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(n)

## 6. Java Implementation
```java
class Solution {
    public int[][] insert(int[][] intervals, int[] newInterval) {
        List<int[]> result = new ArrayList<>();
        int i = 0, n = intervals.length;
        while (i < n && intervals[i][1] < newInterval[0]) result.add(intervals[i++]);
        while (i < n && intervals[i][0] <= newInterval[1]) {
            newInterval[0] = Math.min(newInterval[0], intervals[i][0]);
            newInterval[1] = Math.max(newInterval[1], intervals[i][1]); i++;
        }
        result.add(newInterval);
        while (i < n) result.add(intervals[i++]);
        return result.toArray(new int[0][]);
    }
}
```

---

# #452 — Minimum Number of Arrows to Burst Balloons

## 1. Problem Description
Balloons are intervals on a number line. Minimum arrows to burst all (an arrow at `x` bursts all intervals containing `x`).

## 2. Key Algorithmic Pattern
**Greedy — sort by end coordinate**

## 3. Intuition
Sort by end. Shoot at first balloon's end — bursts all currently overlapping balloons. Skip those, shoot again at next unbursted balloon's end.

## 5. Time and Space Complexity
- **Time:** O(n log n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public int findMinArrowShots(int[][] points) {
        Arrays.sort(points, (a, b) -> Integer.compare(a[1], b[1]));
        int arrows = 1, pos = points[0][1];
        for (int i = 1; i < points.length; i++) {
            if (points[i][0] > pos) { arrows++; pos = points[i][1]; }
        }
        return arrows;
    }
}
```

---

# CATEGORY 7: Stack

---

# #20 — Valid Parentheses

## 1. Problem Description
Return true if bracket string `()[]{}` is valid (properly opened and closed in order).

## 2. Key Algorithmic Pattern
**Stack**

## 3. Intuition
Push open brackets. On close bracket, check if top of stack is the matching open. If not (or stack empty), invalid.

## 6. Java Implementation
```java
class Solution {
    public boolean isValid(String s) {
        Deque<Character> stack = new ArrayDeque<>();
        for (char c : s.toCharArray()) {
            if (c == '(' || c == '[' || c == '{') stack.push(c);
            else {
                if (stack.isEmpty()) return false;
                char top = stack.pop();
                if (c == ')' && top != '(' || c == ']' && top != '[' || c == '}' && top != '{') return false;
            }
        }
        return stack.isEmpty();
    }
}
```
**Time:** O(n), **Space:** O(n)

---

# #71 — Simplify Path

## 1. Problem Description
Simplify a Unix absolute file path (resolve `.`, `..`, multiple slashes).

## 2. Key Algorithmic Pattern
**Stack**

## 3. Intuition
Split on `/`. Push directories; `".."` pops; `.` and empty strings are ignored. Reconstruct from stack.

## 6. Java Implementation
```java
class Solution {
    public String simplifyPath(String path) {
        Deque<String> stack = new ArrayDeque<>();
        for (String part : path.split("/")) {
            if (part.equals("..")) { if (!stack.isEmpty()) stack.pop(); }
            else if (!part.isEmpty() && !part.equals(".")) stack.push(part);
        }
        StringBuilder sb = new StringBuilder();
        for (String dir : stack) sb.insert(0, "/" + dir);
        return sb.length() == 0 ? "/" : sb.toString();
    }
}
```
**Time:** O(n), **Space:** O(n)

---

# #155 — Min Stack

## 1. Problem Description
Stack supporting `push`, `pop`, `top`, and `getMin` — all O(1).

## 2. Key Algorithmic Pattern
**Auxiliary Min Stack**

## 3. Intuition
A second stack tracks the current minimum at every level. Push current min alongside each value; pop both together.

## 6. Java Implementation
```java
class MinStack {
    Deque<Integer> stack = new ArrayDeque<>(), minStack = new ArrayDeque<>();
    public void push(int val) {
        stack.push(val);
        minStack.push(minStack.isEmpty() ? val : Math.min(val, minStack.peek()));
    }
    public void pop() { stack.pop(); minStack.pop(); }
    public int top() { return stack.peek(); }
    public int getMin() { return minStack.peek(); }
}
```
**Time:** O(1) all ops, **Space:** O(n)

---

# #150 — Evaluate Reverse Polish Notation

## 1. Problem Description
Evaluate postfix expression with `+`, `-`, `*`, `/` (truncate toward zero).

## 2. Key Algorithmic Pattern
**Stack**

## 3. Intuition
Push numbers. On operator: pop two, apply, push result. Final stack element is the answer.

## 6. Java Implementation
```java
class Solution {
    public int evalRPN(String[] tokens) {
        Deque<Integer> stack = new ArrayDeque<>();
        for (String t : tokens) {
            if ("+-*/".contains(t)) {
                int b = stack.pop(), a = stack.pop();
                switch (t) {
                    case "+": stack.push(a + b); break; case "-": stack.push(a - b); break;
                    case "*": stack.push(a * b); break; case "/": stack.push(a / b); break;
                }
            } else stack.push(Integer.parseInt(t));
        }
        return stack.pop();
    }
}
```
**Time:** O(n), **Space:** O(n)

---

# #224 — Basic Calculator

## 1. Problem Description
Evaluate expression string with `+`, `-`, `(`, `)`, spaces. No `*` or `/`.

## 2. Key Algorithmic Pattern
**Stack for sign context**

## 3. Intuition
Track running `result` and `sign`. On `(`: push current result and sign, reset. On `)`: apply saved sign and add saved result. This restores context from before the parenthesis.

## 6. Java Implementation
```java
class Solution {
    public int calculate(String s) {
        Deque<Integer> stack = new ArrayDeque<>();
        int result = 0, sign = 1, num = 0;
        for (char c : s.toCharArray()) {
            if (Character.isDigit(c)) num = num * 10 + (c - '0');
            else if (c == '+') { result += sign * num; num = 0; sign = 1; }
            else if (c == '-') { result += sign * num; num = 0; sign = -1; }
            else if (c == '(') { stack.push(result); stack.push(sign); result = 0; sign = 1; }
            else if (c == ')') { result += sign * num; num = 0; result *= stack.pop(); result += stack.pop(); }
        }
        return result + sign * num;
    }
}
```
**Time:** O(n), **Space:** O(n)

---

# CATEGORY 8: Linked List

---

# #141 — Linked List Cycle

## 1. Problem Description
Detect if a linked list has a cycle.

## 2. Key Algorithmic Pattern
**Floyd's Cycle Detection (Fast/Slow Pointers)**

## 3. Intuition
`slow` moves 1 step, `fast` moves 2. In a cycle they will inevitably meet. If `fast` reaches null, no cycle.

## 6. Java Implementation
```java
public class Solution {
    public boolean hasCycle(ListNode head) {
        ListNode slow = head, fast = head;
        while (fast != null && fast.next != null) {
            slow = slow.next; fast = fast.next.next;
            if (slow == fast) return true;
        }
        return false;
    }
}
```
**Time:** O(n), **Space:** O(1)

---

# #2 — Add Two Numbers

## 1. Problem Description
Two linked lists represent integers in reverse order. Return their sum as a linked list.

## 2. Key Algorithmic Pattern
**Simulation with carry**

## 3. Intuition
Process digit by digit, tracking carry. Continue until both lists exhausted and carry is 0.

## 6. Java Implementation
```java
class Solution {
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        ListNode dummy = new ListNode(0), curr = dummy; int carry = 0;
        while (l1 != null || l2 != null || carry != 0) {
            int sum = carry;
            if (l1 != null) { sum += l1.val; l1 = l1.next; }
            if (l2 != null) { sum += l2.val; l2 = l2.next; }
            carry = sum / 10; curr.next = new ListNode(sum % 10); curr = curr.next;
        }
        return dummy.next;
    }
}
```
**Time:** O(max(m,n)), **Space:** O(max(m,n))

---

# #21 — Merge Two Sorted Lists

## 1. Problem Description
Merge two sorted linked lists into one sorted list.

## 2. Key Algorithmic Pattern
**Iterative merge with dummy head**

## 6. Java Implementation
```java
class Solution {
    public ListNode mergeTwoLists(ListNode l1, ListNode l2) {
        ListNode dummy = new ListNode(0), curr = dummy;
        while (l1 != null && l2 != null) {
            if (l1.val <= l2.val) { curr.next = l1; l1 = l1.next; }
            else { curr.next = l2; l2 = l2.next; }
            curr = curr.next;
        }
        curr.next = l1 != null ? l1 : l2;
        return dummy.next;
    }
}
```
**Time:** O(m+n), **Space:** O(1)

---

# #138 — Copy List with Random Pointer

## 1. Problem Description
Deep copy a linked list where each node has `next` and `random` pointers.

## 2. Key Algorithmic Pattern
**HashMap (original → clone)**

## 3. Intuition
Two passes: first create all clones (stored in map). Second pass wire `next` and `random` using the map.

## 6. Java Implementation
```java
class Solution {
    public Node copyRandomList(Node head) {
        if (head == null) return null;
        Map<Node, Node> map = new HashMap<>();
        Node curr = head;
        while (curr != null) { map.put(curr, new Node(curr.val)); curr = curr.next; }
        curr = head;
        while (curr != null) {
            map.get(curr).next = map.get(curr.next);
            map.get(curr).random = map.get(curr.random);
            curr = curr.next;
        }
        return map.get(head);
    }
}
```
**Time:** O(n), **Space:** O(n)

---

# #92 — Reverse Linked List II

## 1. Problem Description
Reverse nodes from position `left` to `right` in one pass.

## 2. Key Algorithmic Pattern
**Head-insertion in sublist**

## 3. Intuition
Navigate to the node just before `left`. Then repeatedly take the node after the sublist front and insert it at the front (head insertion).

## 6. Java Implementation
```java
class Solution {
    public ListNode reverseBetween(ListNode head, int left, int right) {
        ListNode dummy = new ListNode(0); dummy.next = head; ListNode pre = dummy;
        for (int i = 1; i < left; i++) pre = pre.next;
        ListNode curr = pre.next;
        for (int i = 0; i < right - left; i++) {
            ListNode next = curr.next; curr.next = next.next; next.next = pre.next; pre.next = next;
        }
        return dummy.next;
    }
}
```
**Time:** O(n), **Space:** O(1)

---

# #25 — Reverse Nodes in k-Group

## 1. Problem Description
Reverse every `k` consecutive nodes. Leave remaining nodes (< k) as-is.

## 2. Key Algorithmic Pattern
**Recursive group reversal**

## 3. Intuition
Check if k nodes remain. If yes, reverse that group. Recursively process the rest. Connect reversed group's tail to recursive result.

## 6. Java Implementation
```java
class Solution {
    public ListNode reverseKGroup(ListNode head, int k) {
        ListNode curr = head; int count = 0;
        while (curr != null && count < k) { curr = curr.next; count++; }
        if (count < k) return head;
        ListNode prev = null; curr = head;
        for (int i = 0; i < k; i++) { ListNode next = curr.next; curr.next = prev; prev = curr; curr = next; }
        head.next = reverseKGroup(curr, k);
        return prev;
    }
}
```
**Time:** O(n), **Space:** O(n/k) stack frames

---

# #19 — Remove Nth Node From End of List

## 1. Problem Description
Remove the n-th node from the end of a linked list.

## 2. Key Algorithmic Pattern
**Two Pointers with n-step gap**

## 3. Intuition
Place `fast` pointer n+1 steps ahead of `slow`. When `fast` reaches null, `slow` is just before the target.

## 6. Java Implementation
```java
class Solution {
    public ListNode removeNthFromEnd(ListNode head, int n) {
        ListNode dummy = new ListNode(0); dummy.next = head;
        ListNode fast = dummy, slow = dummy;
        for (int i = 0; i <= n; i++) fast = fast.next;
        while (fast != null) { fast = fast.next; slow = slow.next; }
        slow.next = slow.next.next;
        return dummy.next;
    }
}
```
**Time:** O(n), **Space:** O(1)

---

# #82 — Remove Duplicates from Sorted List II

## 1. Problem Description
Delete all nodes with duplicate values (keep only nodes that appeared exactly once).

## 2. Key Algorithmic Pattern
**Dummy node + skip entire duplicate runs**

## 6. Java Implementation
```java
class Solution {
    public ListNode deleteDuplicates(ListNode head) {
        ListNode dummy = new ListNode(0, head), prev = dummy;
        while (prev.next != null) {
            ListNode curr = prev.next;
            if (curr.next != null && curr.val == curr.next.val) {
                while (curr.next != null && curr.val == curr.next.val) curr = curr.next;
                prev.next = curr.next;
            } else prev = prev.next;
        }
        return dummy.next;
    }
}
```
**Time:** O(n), **Space:** O(1)

---

# #61 — Rotate List

## 1. Problem Description
Rotate linked list to the right by `k` places.

## 2. Key Algorithmic Pattern
**Make circular, find new tail, break**

## 3. Intuition
Connect tail to head (circular). New head is at position `n - (k % n)`. Find the node just before it as the new tail, break the circle.

## 6. Java Implementation
```java
class Solution {
    public ListNode rotateRight(ListNode head, int k) {
        if (head == null || k == 0) return head;
        int len = 1; ListNode tail = head;
        while (tail.next != null) { tail = tail.next; len++; }
        k = k % len; if (k == 0) return head;
        tail.next = head;
        ListNode newTail = head;
        for (int i = 0; i < len - k - 1; i++) newTail = newTail.next;
        ListNode newHead = newTail.next; newTail.next = null;
        return newHead;
    }
}
```
**Time:** O(n), **Space:** O(1)

---

# #86 — Partition List

## 1. Problem Description
Partition list so nodes `< x` come before nodes `>= x`, preserving relative order.

## 2. Key Algorithmic Pattern
**Two dummy-headed lists, then concatenate**

## 6. Java Implementation
```java
class Solution {
    public ListNode partition(ListNode head, int x) {
        ListNode lh = new ListNode(0), gh = new ListNode(0), less = lh, great = gh;
        while (head != null) {
            if (head.val < x) { less.next = head; less = less.next; }
            else { great.next = head; great = great.next; }
            head = head.next;
        }
        great.next = null; less.next = gh.next;
        return lh.next;
    }
}
```
**Time:** O(n), **Space:** O(1)

---

# #146 — LRU Cache

## 1. Problem Description
LRU Cache with capacity: `get(key)` and `put(key, value)` in O(1). Evict least recently used when full.

## 2. Key Algorithmic Pattern
**HashMap + Doubly Linked List**

## 3. Intuition
HashMap: O(1) lookup. DLL: O(1) move-to-front and remove-from-tail. Most recent at head, least recent at tail. Sentinel head/tail nodes simplify pointer manipulation.

## 5. Time and Space Complexity
- **Time:** O(1) all ops, **Space:** O(capacity)

## 6. Java Implementation
```java
class LRUCache {
    class Node { int key, val; Node prev, next; Node(int k, int v) { key = k; val = v; } }
    Map<Integer, Node> map = new HashMap<>(); int cap;
    Node head = new Node(0, 0), tail = new Node(0, 0);

    public LRUCache(int cap) {
        this.cap = cap; head.next = tail; tail.prev = head;
    }
    public int get(int key) {
        if (!map.containsKey(key)) return -1;
        Node n = map.get(key); moveToHead(n); return n.val;
    }
    public void put(int key, int val) {
        if (map.containsKey(key)) { Node n = map.get(key); n.val = val; moveToHead(n); }
        else { Node n = new Node(key, val); map.put(key, n); addToHead(n);
            if (map.size() > cap) { Node r = tail.prev; remove(r); map.remove(r.key); } }
    }
    void addToHead(Node n) { n.next = head.next; n.prev = head; head.next.prev = n; head.next = n; }
    void remove(Node n) { n.prev.next = n.next; n.next.prev = n.prev; }
    void moveToHead(Node n) { remove(n); addToHead(n); }
}
```
# CATEGORY 9: Binary Tree — General

---

# #104 — Maximum Depth of Binary Tree

## 1. Problem Description
Return the maximum depth (longest root-to-leaf path node count) of a binary tree.

## 2. Key Algorithmic Pattern
**DFS (Recursive)**

## 3. Intuition
Depth of a node = 1 + max(depth of left subtree, depth of right subtree). Base case: null = 0.

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(h)

## 6. Java Implementation
```java
class Solution {
    public int maxDepth(TreeNode root) {
        if (root == null) return 0;
        return 1 + Math.max(maxDepth(root.left), maxDepth(root.right));
    }
}
```

---

# #100 — Same Tree

## 1. Problem Description
Return true if two binary trees are structurally identical with the same node values.

## 2. Key Algorithmic Pattern
**DFS**

## 6. Java Implementation
```java
class Solution {
    public boolean isSameTree(TreeNode p, TreeNode q) {
        if (p == null && q == null) return true;
        if (p == null || q == null) return false;
        return p.val == q.val && isSameTree(p.left, q.left) && isSameTree(p.right, q.right);
    }
}
```
**Time:** O(n), **Space:** O(h)

---

# #226 — Invert Binary Tree

## 1. Problem Description
Invert (mirror) a binary tree.

## 2. Key Algorithmic Pattern
**DFS**

## 6. Java Implementation
```java
class Solution {
    public TreeNode invertTree(TreeNode root) {
        if (root == null) return null;
        TreeNode temp = root.left;
        root.left = invertTree(root.right);
        root.right = invertTree(temp);
        return root;
    }
}
```
**Time:** O(n), **Space:** O(h)

---

# #101 — Symmetric Tree

## 1. Problem Description
Return true if a binary tree is a mirror of itself.

## 2. Key Algorithmic Pattern
**DFS (mirror pair comparison)**

## 3. Intuition
Two subtrees are mirrors if: roots equal, left.left mirrors right.right, and left.right mirrors right.left.

## 6. Java Implementation
```java
class Solution {
    public boolean isSymmetric(TreeNode root) { return isMirror(root.left, root.right); }
    private boolean isMirror(TreeNode a, TreeNode b) {
        if (a == null && b == null) return true;
        if (a == null || b == null) return false;
        return a.val == b.val && isMirror(a.left, b.right) && isMirror(a.right, b.left);
    }
}
```
**Time:** O(n), **Space:** O(h)

---

# #105 — Construct Binary Tree from Preorder and Inorder Traversal

## 1. Problem Description
Given `preorder` and `inorder` arrays, reconstruct the binary tree.

## 2. Key Algorithmic Pattern
**DFS + HashMap for O(1) index lookup**

## 3. Intuition
First element of preorder = root. Find in inorder — everything left is left subtree, right is right subtree. Recurse.

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(n)

## 6. Java Implementation
```java
class Solution {
    Map<Integer, Integer> inIdx = new HashMap<>(); int preIdx = 0;
    public TreeNode buildTree(int[] preorder, int[] inorder) {
        for (int i = 0; i < inorder.length; i++) inIdx.put(inorder[i], i);
        return build(preorder, 0, inorder.length - 1);
    }
    TreeNode build(int[] pre, int l, int r) {
        if (l > r) return null;
        int rootVal = pre[preIdx++]; TreeNode root = new TreeNode(rootVal);
        int mid = inIdx.get(rootVal);
        root.left = build(pre, l, mid - 1); root.right = build(pre, mid + 1, r);
        return root;
    }
}
```

---

# #106 — Construct Binary Tree from Inorder and Postorder Traversal

## 1. Problem Description
Given `inorder` and `postorder` arrays, reconstruct the binary tree.

## 2. Key Algorithmic Pattern
**DFS + HashMap** — same idea; root is the **last** element of postorder.

## 3. Intuition
Last element of postorder = root. Find in inorder to split subtrees. Process right subtree first (postorder is left-right-root).

## 6. Java Implementation
```java
class Solution {
    Map<Integer, Integer> inIdx = new HashMap<>(); int postIdx;
    public TreeNode buildTree(int[] inorder, int[] postorder) {
        for (int i = 0; i < inorder.length; i++) inIdx.put(inorder[i], i);
        postIdx = postorder.length - 1;
        return build(postorder, 0, inorder.length - 1);
    }
    TreeNode build(int[] post, int l, int r) {
        if (l > r) return null;
        int rootVal = post[postIdx--]; TreeNode root = new TreeNode(rootVal);
        int mid = inIdx.get(rootVal);
        root.right = build(post, mid + 1, r); root.left = build(post, l, mid - 1);
        return root;
    }
}
```
**Time:** O(n), **Space:** O(n)

---

# #117 — Populating Next Right Pointers in Each Node II

## 1. Problem Description
Populate each node's `next` pointer to the next right node on the same level (null if rightmost). Works for any binary tree.

## 2. Key Algorithmic Pattern
**O(1) space level-linking with dummy head**

## 3. Intuition
Process level by level using a dummy head for the next level. For each current-level node, wire its children to a growing "next level" chain via the dummy.

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public Node connect(Node root) {
        Node curr = root;
        while (curr != null) {
            Node dummy = new Node(0), tail = dummy;
            while (curr != null) {
                if (curr.left != null) { tail.next = curr.left; tail = tail.next; }
                if (curr.right != null) { tail.next = curr.right; tail = tail.next; }
                curr = curr.next;
            }
            curr = dummy.next;
        }
        return root;
    }
}
```

---

# #114 — Flatten Binary Tree to Linked List

## 1. Problem Description
Flatten binary tree in-place into a linked list in preorder order (right pointers as `next`, all left = null).

## 2. Key Algorithmic Pattern
**Reverse preorder (right → left → root)**

## 3. Intuition
Process right, then left, then root. Track `prev`. Set `root.right = prev`, `root.left = null`. Each node's right naturally points to the next preorder node.

## 6. Java Implementation
```java
class Solution {
    private TreeNode prev = null;
    public void flatten(TreeNode root) {
        if (root == null) return;
        flatten(root.right);
        flatten(root.left);
        root.right = prev; root.left = null; prev = root;
    }
}
```
**Time:** O(n), **Space:** O(h)

---

# #112 — Path Sum

## 1. Problem Description
Return true if any root-to-leaf path sums to `targetSum`.

## 2. Key Algorithmic Pattern
**DFS**

## 6. Java Implementation
```java
class Solution {
    public boolean hasPathSum(TreeNode root, int targetSum) {
        if (root == null) return false;
        if (root.left == null && root.right == null) return root.val == targetSum;
        return hasPathSum(root.left, targetSum - root.val) || hasPathSum(root.right, targetSum - root.val);
    }
}
```
**Time:** O(n), **Space:** O(h)

---

# #129 — Sum Root to Leaf Numbers

## 1. Problem Description
Each root-to-leaf path represents a number. Return total sum of all such numbers.

## 2. Key Algorithmic Pattern
**DFS with accumulated value**

## 6. Java Implementation
```java
class Solution {
    public int sumNumbers(TreeNode root) { return dfs(root, 0); }
    private int dfs(TreeNode node, int current) {
        if (node == null) return 0;
        current = current * 10 + node.val;
        if (node.left == null && node.right == null) return current;
        return dfs(node.left, current) + dfs(node.right, current);
    }
}
```
**Time:** O(n), **Space:** O(h)

---

# #124 — Binary Tree Maximum Path Sum

## 1. Problem Description
Return the maximum sum of any path in the binary tree (path need not pass through root).

## 2. Key Algorithmic Pattern
**DFS with global max**

## 3. Intuition
At each node: path through it = `node.val + max(0, leftGain) + max(0, rightGain)`. Update global max. Return `node.val + max(leftGain, rightGain, 0)` to parent (only one branch can extend upward).

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(h)

## 6. Java Implementation
```java
class Solution {
    private int maxSum = Integer.MIN_VALUE;
    public int maxPathSum(TreeNode root) { dfs(root); return maxSum; }
    private int dfs(TreeNode node) {
        if (node == null) return 0;
        int left = Math.max(0, dfs(node.left)), right = Math.max(0, dfs(node.right));
        maxSum = Math.max(maxSum, node.val + left + right);
        return node.val + Math.max(left, right);
    }
}
```

---

# #173 — Binary Search Tree Iterator

## 1. Problem Description
Implement BST iterator: `next()` returns smallest remaining, `hasNext()` checks if any remain. Average O(1) time, O(h) space.

## 2. Key Algorithmic Pattern
**Controlled Inorder with Stack**

## 3. Intuition
Simulate inorder lazily. Push entire left spine onto stack. `next()` pops a node, then pushes the left spine of its right subtree.

## 6. Java Implementation
```java
class BSTIterator {
    Deque<TreeNode> stack = new ArrayDeque<>();
    public BSTIterator(TreeNode root) { pushLeft(root); }
    public int next() { TreeNode node = stack.pop(); pushLeft(node.right); return node.val; }
    public boolean hasNext() { return !stack.isEmpty(); }
    private void pushLeft(TreeNode node) { while (node != null) { stack.push(node); node = node.left; } }
}
```
**Time:** O(1) amortized, **Space:** O(h)

---

# #222 — Count Complete Tree Nodes

## 1. Problem Description
Count nodes in a complete binary tree more efficiently than O(n).

## 2. Key Algorithmic Pattern
**Binary Search leveraging perfect tree shortcut**

## 3. Intuition
Compute left-depth and right-depth. If equal, it's a perfect tree: `2^h - 1` nodes. If unequal, recurse on both subtrees. Each recursion at least halves the tree.

## 5. Time and Space Complexity
- **Time:** O(log²n), **Space:** O(log n)

## 6. Java Implementation
```java
class Solution {
    public int countNodes(TreeNode root) {
        if (root == null) return 0;
        int lh = 0, rh = 0; TreeNode l = root, r = root;
        while (l != null) { lh++; l = l.left; }
        while (r != null) { rh++; r = r.right; }
        if (lh == rh) return (1 << lh) - 1;
        return 1 + countNodes(root.left) + countNodes(root.right);
    }
}
```

---

# #236 — Lowest Common Ancestor of a Binary Tree

## 1. Problem Description
Find the lowest common ancestor (deepest shared ancestor) of nodes `p` and `q`.

## 2. Key Algorithmic Pattern
**DFS (post-order)**

## 3. Intuition
If current node is `p` or `q`, return it. If both left and right return non-null, current is the LCA. Otherwise propagate the non-null result up.

## 6. Java Implementation
```java
class Solution {
    public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
        if (root == null || root == p || root == q) return root;
        TreeNode left = lowestCommonAncestor(root.left, p, q);
        TreeNode right = lowestCommonAncestor(root.right, p, q);
        if (left != null && right != null) return root;
        return left != null ? left : right;
    }
}
```
**Time:** O(n), **Space:** O(h)

---

# CATEGORY 10: Binary Tree — BFS

---

# #199 — Binary Tree Right Side View

## 1. Problem Description
Return values visible from the right side (last node of each level).

## 2. Key Algorithmic Pattern
**BFS (Level Order)**

## 6. Java Implementation
```java
class Solution {
    public List<Integer> rightSideView(TreeNode root) {
        List<Integer> res = new ArrayList<>(); if (root == null) return res;
        Queue<TreeNode> q = new LinkedList<>(); q.offer(root);
        while (!q.isEmpty()) {
            int sz = q.size();
            for (int i = 0; i < sz; i++) {
                TreeNode node = q.poll();
                if (i == sz - 1) res.add(node.val);
                if (node.left != null) q.offer(node.left);
                if (node.right != null) q.offer(node.right);
            }
        }
        return res;
    }
}
```
**Time:** O(n), **Space:** O(n)

---

# #637 — Average of Levels in Binary Tree

## 1. Problem Description
Return average value of nodes at each level.

## 2. Key Algorithmic Pattern
**BFS**

## 6. Java Implementation
```java
class Solution {
    public List<Double> averageOfLevels(TreeNode root) {
        List<Double> res = new ArrayList<>(); Queue<TreeNode> q = new LinkedList<>(); q.offer(root);
        while (!q.isEmpty()) {
            int sz = q.size(); double sum = 0;
            for (int i = 0; i < sz; i++) { TreeNode n = q.poll(); sum += n.val; if (n.left != null) q.offer(n.left); if (n.right != null) q.offer(n.right); }
            res.add(sum / sz);
        }
        return res;
    }
}
```
**Time:** O(n), **Space:** O(n)

---

# #102 — Binary Tree Level Order Traversal

## 1. Problem Description
Return level-order traversal (each level as a separate list).

## 2. Key Algorithmic Pattern
**BFS**

## 6. Java Implementation
```java
class Solution {
    public List<List<Integer>> levelOrder(TreeNode root) {
        List<List<Integer>> res = new ArrayList<>(); if (root == null) return res;
        Queue<TreeNode> q = new LinkedList<>(); q.offer(root);
        while (!q.isEmpty()) {
            int sz = q.size(); List<Integer> level = new ArrayList<>();
            for (int i = 0; i < sz; i++) { TreeNode n = q.poll(); level.add(n.val); if (n.left != null) q.offer(n.left); if (n.right != null) q.offer(n.right); }
            res.add(level);
        }
        return res;
    }
}
```
**Time:** O(n), **Space:** O(n)

---

# #103 — Binary Tree Zigzag Level Order Traversal

## 1. Problem Description
Level order traversal alternating direction each level.

## 2. Key Algorithmic Pattern
**BFS with direction flag**

## 3. Intuition
Use a `LinkedList` per level. Add to front or back depending on the `leftToRight` flag.

## 6. Java Implementation
```java
class Solution {
    public List<List<Integer>> zigzagLevelOrder(TreeNode root) {
        List<List<Integer>> res = new ArrayList<>(); if (root == null) return res;
        Queue<TreeNode> q = new LinkedList<>(); q.offer(root); boolean l2r = true;
        while (!q.isEmpty()) {
            int sz = q.size(); LinkedList<Integer> level = new LinkedList<>();
            for (int i = 0; i < sz; i++) { TreeNode n = q.poll(); if (l2r) level.addLast(n.val); else level.addFirst(n.val); if (n.left != null) q.offer(n.left); if (n.right != null) q.offer(n.right); }
            res.add(level); l2r = !l2r;
        }
        return res;
    }
}
```
**Time:** O(n), **Space:** O(n)

---

# CATEGORY 11: Binary Search Tree

---

# #530 — Minimum Absolute Difference in BST

## 1. Problem Description
Find minimum absolute difference between any two node values in BST.

## 2. Key Algorithmic Pattern
**Inorder Traversal** — gives sorted sequence; min diff is between consecutive values.

## 6. Java Implementation
```java
class Solution {
    int min = Integer.MAX_VALUE; Integer prev = null;
    public int getMinimumDifference(TreeNode root) { inorder(root); return min; }
    void inorder(TreeNode node) {
        if (node == null) return;
        inorder(node.left);
        if (prev != null) min = Math.min(min, node.val - prev);
        prev = node.val;
        inorder(node.right);
    }
}
```
**Time:** O(n), **Space:** O(h)

---

# #230 — Kth Smallest Element in a BST

## 1. Problem Description
Find the k-th smallest value in a BST.

## 2. Key Algorithmic Pattern
**Inorder Traversal (stop at k-th)**

## 6. Java Implementation
```java
class Solution {
    int count = 0, result = 0;
    public int kthSmallest(TreeNode root, int k) { inorder(root, k); return result; }
    void inorder(TreeNode node, int k) {
        if (node == null) return;
        inorder(node.left, k);
        if (++count == k) { result = node.val; return; }
        inorder(node.right, k);
    }
}
```
**Time:** O(h + k), **Space:** O(h)

---

# #98 — Validate Binary Search Tree

## 1. Problem Description
Determine whether a binary tree is a valid BST.

## 2. Key Algorithmic Pattern
**DFS with valid range (min, max)**

## 3. Intuition
Pass down `(min, max)` bounds. Each node must satisfy `min < node.val < max`. Left subtree: max becomes `node.val`. Right subtree: min becomes `node.val`.

## 6. Java Implementation
```java
class Solution {
    public boolean isValidBST(TreeNode root) { return validate(root, Long.MIN_VALUE, Long.MAX_VALUE); }
    boolean validate(TreeNode node, long min, long max) {
        if (node == null) return true;
        if (node.val <= min || node.val >= max) return false;
        return validate(node.left, min, node.val) && validate(node.right, node.val, max);
    }
}
```
**Time:** O(n), **Space:** O(h)

---

# CATEGORY 12: Graph — General

---

# #200 — Number of Islands

## 1. Problem Description
Count the number of islands (connected regions of '1') in a 2D grid.

## 2. Key Algorithmic Pattern
**DFS flood fill**

## 3. Intuition
For each unvisited '1', increment count and DFS to mark the entire connected land region as '0' (visited).

## 5. Time and Space Complexity
- **Time:** O(m*n), **Space:** O(m*n)

## 6. Java Implementation
```java
class Solution {
    public int numIslands(char[][] grid) {
        int count = 0;
        for (int r = 0; r < grid.length; r++)
            for (int c = 0; c < grid[0].length; c++)
                if (grid[r][c] == '1') { dfs(grid, r, c); count++; }
        return count;
    }
    void dfs(char[][] g, int r, int c) {
        if (r < 0 || r >= g.length || c < 0 || c >= g[0].length || g[r][c] != '1') return;
        g[r][c] = '0';
        dfs(g, r+1, c); dfs(g, r-1, c); dfs(g, r, c+1); dfs(g, r, c-1);
    }
}
```

---

# #130 — Surrounded Regions

## 1. Problem Description
Capture all 'O' regions not connected to the border (replace with 'X').

## 2. Key Algorithmic Pattern
**DFS from border**

## 3. Intuition
Border-connected 'O's can't be captured. DFS from all border 'O's, marking them safe ('S'). Then convert: 'O' → 'X', 'S' → 'O'.

## 6. Java Implementation
```java
class Solution {
    public void solve(char[][] board) {
        int m = board.length, n = board[0].length;
        for (int r = 0; r < m; r++) { dfs(board, r, 0); dfs(board, r, n-1); }
        for (int c = 0; c < n; c++) { dfs(board, 0, c); dfs(board, m-1, c); }
        for (int r = 0; r < m; r++) for (int c = 0; c < n; c++) board[r][c] = board[r][c] == 'S' ? 'O' : 'X';
    }
    void dfs(char[][] b, int r, int c) {
        if (r < 0 || r >= b.length || c < 0 || c >= b[0].length || b[r][c] != 'O') return;
        b[r][c] = 'S';
        dfs(b, r+1, c); dfs(b, r-1, c); dfs(b, r, c+1); dfs(b, r, c-1);
    }
}
```
**Time:** O(m*n), **Space:** O(m*n)

---

# #133 — Clone Graph

## 1. Problem Description
Deep copy a connected undirected graph.

## 2. Key Algorithmic Pattern
**DFS + HashMap (original → clone)**

## 6. Java Implementation
```java
class Solution {
    Map<Node, Node> visited = new HashMap<>();
    public Node cloneGraph(Node node) {
        if (node == null) return null;
        if (visited.containsKey(node)) return visited.get(node);
        Node clone = new Node(node.val); visited.put(node, clone);
        for (Node n : node.neighbors) clone.neighbors.add(cloneGraph(n));
        return clone;
    }
}
```
**Time:** O(V+E), **Space:** O(V)

---

# #399 — Evaluate Division

## 1. Problem Description
Given equations `A/B = k`, answer queries for `C/D`. Return -1.0 if undetermined.

## 2. Key Algorithmic Pattern
**Weighted directed graph + BFS**

## 3. Intuition
Build graph: `A→B` with weight `k`, `B→A` with weight `1/k`. For each query, BFS from source to destination multiplying weights.

## 5. Time and Space Complexity
- **Time:** O((V+E) * Q), **Space:** O(V+E)

## 6. Java Implementation
```java
class Solution {
    public double[] calcEquation(List<List<String>> eq, double[] vals, List<List<String>> queries) {
        Map<String, Map<String, Double>> g = new HashMap<>();
        for (int i = 0; i < eq.size(); i++) {
            String a = eq.get(i).get(0), b = eq.get(i).get(1);
            g.computeIfAbsent(a, k -> new HashMap<>()).put(b, vals[i]);
            g.computeIfAbsent(b, k -> new HashMap<>()).put(a, 1.0 / vals[i]);
        }
        double[] res = new double[queries.size()];
        for (int i = 0; i < queries.size(); i++) {
            String s = queries.get(i).get(0), d = queries.get(i).get(1);
            if (!g.containsKey(s) || !g.containsKey(d)) { res[i] = -1; continue; }
            if (s.equals(d)) { res[i] = 1; continue; }
            Queue<String> q = new LinkedList<>(); Map<String, Double> dist = new HashMap<>();
            q.offer(s); dist.put(s, 1.0); res[i] = -1;
            outer: while (!q.isEmpty()) {
                String cur = q.poll();
                for (Map.Entry<String, Double> e : g.get(cur).entrySet()) {
                    String next = e.getKey();
                    if (!dist.containsKey(next)) {
                        dist.put(next, dist.get(cur) * e.getValue());
                        if (next.equals(d)) { res[i] = dist.get(next); break outer; }
                        q.offer(next);
                    }
                }
            }
        }
        return res;
    }
}
```

---

# #207 — Course Schedule

## 1. Problem Description
Given prerequisites, determine if it's possible to finish all courses (detect cycle).

## 2. Key Algorithmic Pattern
**Topological Sort / Cycle Detection (DFS 3-state)**

## 3. Intuition
Build directed graph. DFS with states: 0=unvisited, 1=in-progress, 2=done. Reaching an in-progress node = cycle = impossible.

## 5. Time and Space Complexity
- **Time:** O(V+E), **Space:** O(V+E)

## 6. Java Implementation
```java
class Solution {
    public boolean canFinish(int n, int[][] pre) {
        List<List<Integer>> adj = new ArrayList<>();
        for (int i = 0; i < n; i++) adj.add(new ArrayList<>());
        for (int[] p : pre) adj.get(p[1]).add(p[0]);
        int[] state = new int[n];
        for (int i = 0; i < n; i++) if (hasCycle(adj, state, i)) return false;
        return true;
    }
    boolean hasCycle(List<List<Integer>> adj, int[] state, int node) {
        if (state[node] == 1) return true;
        if (state[node] == 2) return false;
        state[node] = 1;
        for (int next : adj.get(node)) if (hasCycle(adj, state, next)) return true;
        state[node] = 2; return false;
    }
}
```

---

# #210 — Course Schedule II

## 1. Problem Description
Return a valid course order (topological sort), or empty if impossible.

## 2. Key Algorithmic Pattern
**Topological Sort (DFS post-order)**

## 3. Intuition
Same cycle detection as above. Add nodes to order list after all their descendants are processed (post-order). Reverse at end for correct ordering.

## 6. Java Implementation
```java
class Solution {
    public int[] findOrder(int n, int[][] pre) {
        List<List<Integer>> adj = new ArrayList<>();
        for (int i = 0; i < n; i++) adj.add(new ArrayList<>());
        for (int[] p : pre) adj.get(p[1]).add(p[0]);
        int[] state = new int[n]; List<Integer> order = new ArrayList<>();
        for (int i = 0; i < n; i++) if (dfs(adj, state, i, order)) return new int[0];
        Collections.reverse(order);
        return order.stream().mapToInt(Integer::intValue).toArray();
    }
    boolean dfs(List<List<Integer>> adj, int[] state, int node, List<Integer> order) {
        if (state[node] == 1) return true;
        if (state[node] == 2) return false;
        state[node] = 1;
        for (int next : adj.get(node)) if (dfs(adj, state, next, order)) return true;
        state[node] = 2; order.add(node); return false;
    }
}
```
**Time:** O(V+E), **Space:** O(V+E)

---

# CATEGORY 13: Graph — BFS

---

# #909 — Snakes and Ladders

## 1. Problem Description
On an n×n boustrophedon-numbered board with snakes/ladders, find minimum dice rolls to reach square n².

## 2. Key Algorithmic Pattern
**BFS (shortest path on unweighted graph)**

## 3. Intuition
Each square is a node; dice rolls 1–6 create edges. BFS gives minimum rolls. Flatten the board to a 1D array first, respecting boustrophedon numbering.

## 5. Time and Space Complexity
- **Time:** O(n²), **Space:** O(n²)

## 6. Java Implementation
```java
class Solution {
    public int snakesAndLadders(int[][] board) {
        int n = board.length; int[] flat = new int[n * n + 1]; int idx = 1; boolean l2r = true;
        for (int r = n - 1; r >= 0; r--) {
            if (l2r) for (int c = 0; c < n; c++) flat[idx++] = board[r][c];
            else for (int c = n - 1; c >= 0; c--) flat[idx++] = board[r][c];
            l2r = !l2r;
        }
        Queue<Integer> q = new LinkedList<>(); boolean[] vis = new boolean[n * n + 1];
        q.offer(1); vis[1] = true; int moves = 0;
        while (!q.isEmpty()) {
            int sz = q.size();
            for (int i = 0; i < sz; i++) {
                int cur = q.poll(); if (cur == n * n) return moves;
                for (int d = 1; d <= 6; d++) {
                    int next = cur + d; if (next > n * n) break;
                    if (flat[next] != -1) next = flat[next];
                    if (!vis[next]) { vis[next] = true; q.offer(next); }
                }
            }
            moves++;
        }
        return -1;
    }
}
```

---

# #433 — Minimum Genetic Mutation

## 1. Problem Description
Find minimum single-character mutations to transform `startGene` to `endGene` using only bank sequences as intermediates.

## 2. Key Algorithmic Pattern
**BFS (Word Ladder variant)**

## 6. Java Implementation
```java
class Solution {
    public int minMutation(String start, String end, String[] bank) {
        Set<String> bankSet = new HashSet<>(Arrays.asList(bank));
        if (!bankSet.contains(end)) return -1;
        Queue<String> q = new LinkedList<>(); q.offer(start); int muts = 0;
        char[] genes = {'A', 'C', 'G', 'T'};
        while (!q.isEmpty()) {
            int sz = q.size();
            for (int i = 0; i < sz; i++) {
                String cur = q.poll(); if (cur.equals(end)) return muts;
                char[] arr = cur.toCharArray();
                for (int j = 0; j < 8; j++) {
                    char orig = arr[j];
                    for (char g : genes) { arr[j] = g; String next = new String(arr); if (bankSet.contains(next)) { bankSet.remove(next); q.offer(next); } }
                    arr[j] = orig;
                }
            }
            muts++;
        }
        return -1;
    }
}
```
**Time:** O(n * 32), **Space:** O(n)

---

# #127 — Word Ladder

## 1. Problem Description
Find length of shortest transformation sequence from `beginWord` to `endWord` (one letter change per step, each intermediate in wordList).

## 2. Key Algorithmic Pattern
**BFS** — try replacing each char position with all 26 letters.

## 3. Intuition
Each word = graph node. Two words connected if they differ by one char. BFS from `beginWord` to `endWord` gives shortest path. Remove visited words from set to avoid re-processing.

## 5. Time and Space Complexity
- **Time:** O(M² * N), **Space:** O(M * N)

## 6. Java Implementation
```java
class Solution {
    public int ladderLength(String begin, String end, List<String> wordList) {
        Set<String> wordSet = new HashSet<>(wordList);
        if (!wordSet.contains(end)) return 0;
        Queue<String> q = new LinkedList<>(); q.offer(begin); int level = 1;
        while (!q.isEmpty()) {
            int sz = q.size();
            for (int i = 0; i < sz; i++) {
                char[] word = q.poll().toCharArray();
                for (int j = 0; j < word.length; j++) {
                    char orig = word[j];
                    for (char c = 'a'; c <= 'z'; c++) {
                        word[j] = c; String next = new String(word);
                        if (next.equals(end)) return level + 1;
                        if (wordSet.remove(next)) q.offer(next);
                    }
                    word[j] = orig;
                }
            }
            level++;
        }
        return 0;
    }
}
```

---

# CATEGORY 14: Trie

---

# #208 — Implement Trie (Prefix Tree)

## 1. Problem Description
Implement a Trie with `insert(word)`, `search(word)`, `startsWith(prefix)`.

## 2. Key Algorithmic Pattern
**Trie** — tree where each edge represents a character; nodes share common prefixes.

## 3. Intuition
Each node has 26 children (lowercase letters) and an `isEnd` flag. Insert/search traverse char by char, creating nodes as needed.

## 5. Time and Space Complexity
- **Time:** O(m) per operation, **Space:** O(ALPHABET * N * M)

## 6. Java Implementation
```java
class Trie {
    class Node { Node[] ch = new Node[26]; boolean end; }
    Node root = new Node();
    public void insert(String w) { Node c = root; for (char ch : w.toCharArray()) { int i = ch - 'a'; if (c.ch[i] == null) c.ch[i] = new Node(); c = c.ch[i]; } c.end = true; }
    public boolean search(String w) { Node n = go(w); return n != null && n.end; }
    public boolean startsWith(String p) { return go(p) != null; }
    Node go(String s) { Node c = root; for (char ch : s.toCharArray()) { int i = ch - 'a'; if (c.ch[i] == null) return null; c = c.ch[i]; } return c; }
}
```

---

# #211 — Design Add and Search Words Data Structure

## 1. Problem Description
`addWord(word)` and `search(word)` where `'.'` matches any single letter.

## 2. Key Algorithmic Pattern
**Trie + DFS for wildcard**

## 3. Intuition
Use a Trie for storage. On `'.'`, try all 26 children recursively.

## 6. Java Implementation
```java
class WordDictionary {
    class Node { Node[] ch = new Node[26]; boolean end; }
    Node root = new Node();
    public void addWord(String w) { Node c = root; for (char ch : w.toCharArray()) { int i = ch - 'a'; if (c.ch[i] == null) c.ch[i] = new Node(); c = c.ch[i]; } c.end = true; }
    public boolean search(String w) { return dfs(w, 0, root); }
    boolean dfs(String w, int i, Node n) {
        if (i == w.length()) return n.end;
        char c = w.charAt(i);
        if (c == '.') { for (Node ch : n.ch) if (ch != null && dfs(w, i + 1, ch)) return true; return false; }
        return n.ch[c - 'a'] != null && dfs(w, i + 1, n.ch[c - 'a']);
    }
}
```
**Time:** O(M) avg, O(M*26^M) worst (all dots)

---

# #212 — Word Search II

## 1. Problem Description
Find all words from a list that exist in the board (connected adjacent cells, no reuse).

## 2. Key Algorithmic Pattern
**Trie + DFS Backtracking**

## 3. Intuition
Build Trie from all words. DFS from each cell, traverse Trie simultaneously. When a complete word node is reached, record it. Mark cells visited during DFS; restore on backtrack. Remove found words from Trie to avoid duplicates.

## 5. Time and Space Complexity
- **Time:** O(M*N * 4 * 3^(L-1)), **Space:** O(W*L)

## 6. Java Implementation
```java
class Solution {
    class Node { Node[] ch = new Node[26]; String word; }
    public List<String> findWords(char[][] board, String[] words) {
        Node root = new Node();
        for (String w : words) { Node c = root; for (char ch : w.toCharArray()) { int i = ch - 'a'; if (c.ch[i] == null) c.ch[i] = new Node(); c = c.ch[i]; } c.word = w; }
        List<String> res = new ArrayList<>();
        for (int r = 0; r < board.length; r++) for (int c = 0; c < board[0].length; c++) dfs(board, r, c, root, res);
        return res;
    }
    void dfs(char[][] b, int r, int c, Node n, List<String> res) {
        if (r < 0 || r >= b.length || c < 0 || c >= b[0].length) return;
        char ch = b[r][c]; if (ch == '#' || n.ch[ch - 'a'] == null) return;
        n = n.ch[ch - 'a']; if (n.word != null) { res.add(n.word); n.word = null; }
        b[r][c] = '#';
        dfs(b, r+1, c, n, res); dfs(b, r-1, c, n, res); dfs(b, r, c+1, n, res); dfs(b, r, c-1, n, res);
        b[r][c] = ch;
    }
}
```

---

# CATEGORY 15: Backtracking

---

# #17 — Letter Combinations of a Phone Number

## 1. Problem Description
Given digits 2–9, return all letter combinations from a phone keypad.

## 2. Key Algorithmic Pattern
**Backtracking**

## 6. Java Implementation
```java
class Solution {
    String[] keys = {"","","abc","def","ghi","jkl","mno","pqrs","tuv","wxyz"};
    public List<String> letterCombinations(String digits) {
        List<String> res = new ArrayList<>(); if (digits.isEmpty()) return res;
        bt(digits, 0, new StringBuilder(), res); return res;
    }
    void bt(String d, int i, StringBuilder cur, List<String> res) {
        if (i == d.length()) { res.add(cur.toString()); return; }
        for (char c : keys[d.charAt(i) - '0'].toCharArray()) { cur.append(c); bt(d, i+1, cur, res); cur.deleteCharAt(cur.length()-1); }
    }
}
```
**Time:** O(4^n * n), **Space:** O(n)

---

# #77 — Combinations

## 1. Problem Description
Return all combinations of k numbers from 1 to n.

## 2. Key Algorithmic Pattern
**Backtracking with pruning**

## 3. Intuition
At each step, only choose numbers ≥ `start`. Prune: if remaining needed > remaining available, stop early.

## 6. Java Implementation
```java
class Solution {
    public List<List<Integer>> combine(int n, int k) { List<List<Integer>> res = new ArrayList<>(); bt(n,k,1,new ArrayList<>(),res); return res; }
    void bt(int n, int k, int start, List<Integer> cur, List<List<Integer>> res) {
        if (cur.size() == k) { res.add(new ArrayList<>(cur)); return; }
        for (int i = start; i <= n - (k - cur.size()) + 1; i++) { cur.add(i); bt(n,k,i+1,cur,res); cur.remove(cur.size()-1); }
    }
}
```
**Time:** O(C(n,k)*k), **Space:** O(k)

---

# #46 — Permutations

## 1. Problem Description
Return all permutations of distinct integers.

## 2. Key Algorithmic Pattern
**Backtracking with used[] array**

## 6. Java Implementation
```java
class Solution {
    public List<List<Integer>> permute(int[] nums) { List<List<Integer>> res = new ArrayList<>(); bt(nums,new boolean[nums.length],new ArrayList<>(),res); return res; }
    void bt(int[] nums, boolean[] used, List<Integer> cur, List<List<Integer>> res) {
        if (cur.size() == nums.length) { res.add(new ArrayList<>(cur)); return; }
        for (int i = 0; i < nums.length; i++) { if (used[i]) continue; used[i]=true; cur.add(nums[i]); bt(nums,used,cur,res); cur.remove(cur.size()-1); used[i]=false; }
    }
}
```
**Time:** O(n!*n), **Space:** O(n)

---

# #39 — Combination Sum

## 1. Problem Description
Find all unique combinations summing to `target`. Each number may be used unlimited times.

## 2. Key Algorithmic Pattern
**Backtracking** — pass same index `i` (not `i+1`) to allow reuse.

## 6. Java Implementation
```java
class Solution {
    public List<List<Integer>> combinationSum(int[] cands, int target) {
        Arrays.sort(cands); List<List<Integer>> res = new ArrayList<>(); bt(cands,target,0,new ArrayList<>(),res); return res;
    }
    void bt(int[] c, int rem, int start, List<Integer> cur, List<List<Integer>> res) {
        if (rem == 0) { res.add(new ArrayList<>(cur)); return; }
        for (int i = start; i < c.length; i++) { if (c[i] > rem) break; cur.add(c[i]); bt(c,rem-c[i],i,cur,res); cur.remove(cur.size()-1); }
    }
}
```
**Time:** O(N^(T/M)), **Space:** O(T/M)

---

# #52 — N-Queens II

## 1. Problem Description
Count distinct solutions to the N-Queens problem.

## 2. Key Algorithmic Pattern
**Backtracking with bitmask column/diagonal tracking**

## 3. Intuition
Track which columns, diagonals (`row-col`), and anti-diagonals (`row+col`) are occupied. Try each column per row.

## 6. Java Implementation
```java
class Solution {
    int count = 0;
    public int totalNQueens(int n) { bt(0,n,new boolean[n],new boolean[2*n],new boolean[2*n]); return count; }
    void bt(int row, int n, boolean[] cols, boolean[] diag, boolean[] anti) {
        if (row == n) { count++; return; }
        for (int c = 0; c < n; c++) {
            if (cols[c] || diag[row-c+n] || anti[row+c]) continue;
            cols[c]=diag[row-c+n]=anti[row+c]=true; bt(row+1,n,cols,diag,anti); cols[c]=diag[row-c+n]=anti[row+c]=false;
        }
    }
}
```
**Time:** O(n!), **Space:** O(n)

---

# #22 — Generate Parentheses

## 1. Problem Description
Generate all combinations of well-formed parentheses for n pairs.

## 2. Key Algorithmic Pattern
**Backtracking** — add `(` if `open < n`; add `)` if `close < open`.

## 6. Java Implementation
```java
class Solution {
    public List<String> generateParenthesis(int n) { List<String> res = new ArrayList<>(); bt(n,0,0,new StringBuilder(),res); return res; }
    void bt(int n, int open, int close, StringBuilder cur, List<String> res) {
        if (cur.length() == 2*n) { res.add(cur.toString()); return; }
        if (open < n) { cur.append('('); bt(n,open+1,close,cur,res); cur.deleteCharAt(cur.length()-1); }
        if (close < open) { cur.append(')'); bt(n,open,close+1,cur,res); cur.deleteCharAt(cur.length()-1); }
    }
}
```
**Time:** O(4^n/√n) Catalan, **Space:** O(n)

---

# #79 — Word Search

## 1. Problem Description
Return true if `word` exists in the grid using adjacent cells (no cell reuse).

## 2. Key Algorithmic Pattern
**DFS Backtracking** — mark visited, restore on backtrack.

## 6. Java Implementation
```java
class Solution {
    public boolean exist(char[][] board, String word) {
        for (int r = 0; r < board.length; r++) for (int c = 0; c < board[0].length; c++) if (dfs(board,word,r,c,0)) return true;
        return false;
    }
    boolean dfs(char[][] b, String w, int r, int c, int i) {
        if (i == w.length()) return true;
        if (r < 0||r >= b.length||c < 0||c >= b[0].length||b[r][c] != w.charAt(i)) return false;
        char t = b[r][c]; b[r][c] = '#';
        boolean found = dfs(b,w,r+1,c,i+1)||dfs(b,w,r-1,c,i+1)||dfs(b,w,r,c+1,i+1)||dfs(b,w,r,c-1,i+1);
        b[r][c] = t; return found;
    }
}
```
**Time:** O(M*N*3^L), **Space:** O(L)

---

# CATEGORY 16: Divide & Conquer

---

# #108 — Convert Sorted Array to Binary Search Tree

## 1. Problem Description
Convert sorted array to height-balanced BST.

## 2. Key Algorithmic Pattern
**Divide & Conquer** — midpoint becomes root.

## 6. Java Implementation
```java
class Solution {
    public TreeNode sortedArrayToBST(int[] nums) { return build(nums, 0, nums.length - 1); }
    TreeNode build(int[] n, int l, int r) {
        if (l > r) return null; int m = l + (r-l)/2; TreeNode root = new TreeNode(n[m]);
        root.left = build(n,l,m-1); root.right = build(n,m+1,r); return root;
    }
}
```
**Time:** O(n), **Space:** O(log n)

---

# #148 — Sort List

## 1. Problem Description
Sort linked list in O(n log n) time.

## 2. Key Algorithmic Pattern
**Merge Sort on Linked List**

## 3. Intuition
Find midpoint (slow/fast pointers), split, sort both halves recursively, merge.

## 6. Java Implementation
```java
class Solution {
    public ListNode sortList(ListNode head) {
        if (head == null || head.next == null) return head;
        ListNode slow = head, fast = head.next;
        while (fast != null && fast.next != null) { slow = slow.next; fast = fast.next.next; }
        ListNode mid = slow.next; slow.next = null;
        return merge(sortList(head), sortList(mid));
    }
    ListNode merge(ListNode l1, ListNode l2) {
        ListNode d = new ListNode(0), c = d;
        while (l1 != null && l2 != null) { if (l1.val <= l2.val) { c.next = l1; l1 = l1.next; } else { c.next = l2; l2 = l2.next; } c = c.next; }
        c.next = l1 != null ? l1 : l2; return d.next;
    }
}
```
**Time:** O(n log n), **Space:** O(log n)

---

# #427 — Construct Quad Tree

## 1. Problem Description
Build a Quad Tree from an n×n binary grid. Each leaf covers a uniform region; internal nodes have four children.

## 2. Key Algorithmic Pattern
**Divide & Conquer**

## 6. Java Implementation
```java
class Solution {
    public Node construct(int[][] grid) { return build(grid, 0, 0, grid.length); }
    Node build(int[][] g, int r, int c, int sz) {
        if (isUniform(g, r, c, sz)) return new Node(g[r][c] == 1, true);
        int h = sz / 2;
        return new Node(true, false, build(g,r,c,h), build(g,r,c+h,h), build(g,r+h,c,h), build(g,r+h,c+h,h));
    }
    boolean isUniform(int[][] g, int r, int c, int sz) { int v = g[r][c]; for (int i=r;i<r+sz;i++) for (int j=c;j<c+sz;j++) if (g[i][j]!=v) return false; return true; }
}
```
**Time:** O(n² log n), **Space:** O(log n)

---

# #23 — Merge k Sorted Lists

## 1. Problem Description
Merge k sorted linked lists into one sorted list.

## 2. Key Algorithmic Pattern
**Min-Heap (Priority Queue)**

## 3. Intuition
Initialize heap with head of each list. Repeatedly extract minimum, add to result, push its successor into heap.

## 5. Time and Space Complexity
- **Time:** O(N log k), **Space:** O(k)

## 6. Java Implementation
```java
class Solution {
    public ListNode mergeKLists(ListNode[] lists) {
        PriorityQueue<ListNode> heap = new PriorityQueue<>((a,b) -> a.val - b.val);
        for (ListNode n : lists) if (n != null) heap.offer(n);
        ListNode dummy = new ListNode(0), curr = dummy;
        while (!heap.isEmpty()) { ListNode n = heap.poll(); curr.next = n; curr = curr.next; if (n.next != null) heap.offer(n.next); }
        return dummy.next;
    }
}
```

---

# CATEGORY 17: Kadane's Algorithm

---

# #53 — Maximum Subarray

## 1. Problem Description
Find contiguous subarray with the largest sum.

## 2. Key Algorithmic Pattern
**Kadane's Algorithm**

## 3. Intuition
At each position: extend current subarray OR start fresh. If running sum goes negative, it only hurts future subarrays — reset. Track global max throughout.

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public int maxSubArray(int[] nums) {
        int maxSum = nums[0], cur = nums[0];
        for (int i = 1; i < nums.length; i++) {
            cur = Math.max(nums[i], cur + nums[i]);
            maxSum = Math.max(maxSum, cur);
        }
        return maxSum;
    }
}
```

---

# #918 — Maximum Sum Circular Subarray

## 1. Problem Description
Find max subarray sum in a circular array (subarray may wrap around).

## 2. Key Algorithmic Pattern
**Kadane's + Total Sum trick**

## 3. Intuition
Two cases: (1) no wrap — standard Kadane's. (2) wrap — equals `totalSum - minSubarraySum`. Take max. Edge: if all negative, return case 1 result (case 2 would give 0).

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public int maxSubarraySumCircular(int[] nums) {
        int total = 0, maxSum = nums[0], curMax = 0, minSum = nums[0], curMin = 0;
        for (int n : nums) {
            curMax = Math.max(curMax + n, n); maxSum = Math.max(maxSum, curMax);
            curMin = Math.min(curMin + n, n); minSum = Math.min(minSum, curMin);
            total += n;
        }
        return maxSum > 0 ? Math.max(maxSum, total - minSum) : maxSum;
    }
}
```

---

# CATEGORY 18: Binary Search

---

# #35 — Search Insert Position

## 1. Problem Description
In sorted array, return index of `target` or where it would be inserted.

## 2. Key Algorithmic Pattern
**Binary Search** — `left` naturally lands at insertion point when target not found.

## 6. Java Implementation
```java
class Solution {
    public int searchInsert(int[] nums, int target) {
        int l = 0, r = nums.length - 1;
        while (l <= r) { int m = l+(r-l)/2; if (nums[m]==target) return m; else if (nums[m]<target) l=m+1; else r=m-1; }
        return l;
    }
}
```
**Time:** O(log n), **Space:** O(1)

---

# #74 — Search a 2D Matrix

## 1. Problem Description
Search for target in m×n matrix where rows are sorted and first element of each row > last of previous.

## 2. Key Algorithmic Pattern
**Binary Search treating matrix as 1D array** — map `mid` to `(mid/n, mid%n)`.

## 6. Java Implementation
```java
class Solution {
    public boolean searchMatrix(int[][] matrix, int target) {
        int m = matrix.length, n = matrix[0].length, l = 0, r = m*n-1;
        while (l <= r) { int mid = l+(r-l)/2, val = matrix[mid/n][mid%n]; if (val==target) return true; else if (val<target) l=mid+1; else r=mid-1; }
        return false;
    }
}
```
**Time:** O(log(m*n)), **Space:** O(1)

---

# #162 — Find Peak Element

## 1. Problem Description
Find any peak element (strictly greater than neighbors) in O(log n).

## 2. Key Algorithmic Pattern
**Binary Search** — if `nums[mid] < nums[mid+1]`, peak is to the right.

## 6. Java Implementation
```java
class Solution {
    public int findPeakElement(int[] nums) {
        int l = 0, r = nums.length - 1;
        while (l < r) { int m = l+(r-l)/2; if (nums[m]<nums[m+1]) l=m+1; else r=m; }
        return l;
    }
}
```
**Time:** O(log n), **Space:** O(1)

---

# #33 — Search in Rotated Sorted Array

## 1. Problem Description
Search for target in a rotated sorted array in O(log n).

## 2. Key Algorithmic Pattern
**Modified Binary Search** — one half is always sorted.

## 3. Intuition
At any `mid`, determine which half is sorted. If target lies in the sorted half, search there. Otherwise search the other half.

## 6. Java Implementation
```java
class Solution {
    public int search(int[] nums, int target) {
        int l = 0, r = nums.length - 1;
        while (l <= r) {
            int m = l+(r-l)/2; if (nums[m]==target) return m;
            if (nums[l]<=nums[m]) { if (target>=nums[l]&&target<nums[m]) r=m-1; else l=m+1; }
            else { if (target>nums[m]&&target<=nums[r]) l=m+1; else r=m-1; }
        }
        return -1;
    }
}
```
**Time:** O(log n), **Space:** O(1)

---

# #34 — Find First and Last Position of Element in Sorted Array

## 1. Problem Description
Find starting and ending position of target in sorted array. Return `[-1,-1]` if not found.

## 2. Key Algorithmic Pattern
**Binary Search twice** — once for leftmost, once for rightmost.

## 6. Java Implementation
```java
class Solution {
    public int[] searchRange(int[] nums, int target) { return new int[]{find(nums,target,true), find(nums,target,false)}; }
    int find(int[] nums, int target, boolean first) {
        int l=0, r=nums.length-1, res=-1;
        while (l<=r) { int m=l+(r-l)/2; if (nums[m]==target) { res=m; if(first) r=m-1; else l=m+1; } else if (nums[m]<target) l=m+1; else r=m-1; }
        return res;
    }
}
```
**Time:** O(log n), **Space:** O(1)

---

# #153 — Find Minimum in Rotated Sorted Array

## 1. Problem Description
Find minimum in a rotated sorted array (no duplicates) in O(log n).

## 2. Key Algorithmic Pattern
**Binary Search** — if `nums[mid] > nums[right]`, minimum is in right half.

## 6. Java Implementation
```java
class Solution {
    public int findMin(int[] nums) {
        int l = 0, r = nums.length - 1;
        while (l < r) { int m = l+(r-l)/2; if (nums[m]>nums[r]) l=m+1; else r=m; }
        return nums[l];
    }
}
```
**Time:** O(log n), **Space:** O(1)

---

# #4 — Median of Two Sorted Arrays

## 1. Problem Description
Find median of two sorted arrays in O(log(min(m,n))).

## 2. Key Algorithmic Pattern
**Binary Search on partition**

## 3. Intuition
Binary search on the smaller array to find the correct partition point such that `max(left parts) <= min(right parts)`. The partition divides both arrays so the left half has `(m+n+1)/2` total elements.

## 4. Step-by-Step Approach
1. Binary search on shorter array for partition `i`; derive `j = half - i`
2. Check if `maxLeft1 <= minRight2` and `maxLeft2 <= minRight1`
3. If yes, compute median from the boundary values
4. If not, adjust binary search bounds

## 5. Time and Space Complexity
- **Time:** O(log(min(m,n))), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public double findMedianSortedArrays(int[] n1, int[] n2) {
        if (n1.length > n2.length) return findMedianSortedArrays(n2, n1);
        int m = n1.length, n = n2.length, l = 0, r = m, half = (m+n+1)/2;
        while (l <= r) {
            int i = l+(r-l)/2, j = half-i;
            int ml1 = i==0?Integer.MIN_VALUE:n1[i-1], mr1 = i==m?Integer.MAX_VALUE:n1[i];
            int ml2 = j==0?Integer.MIN_VALUE:n2[j-1], mr2 = j==n?Integer.MAX_VALUE:n2[j];
            if (ml1<=mr2 && ml2<=mr1) {
                if ((m+n)%2==1) return Math.max(ml1,ml2);
                return (Math.max(ml1,ml2) + Math.min(mr1,mr2)) / 2.0;
            } else if (ml1>mr2) r=i-1; else l=i+1;
        }
        return 0;
    }
}
```

---

# CATEGORY 19: Heap

---

# #215 — Kth Largest Element in an Array

## 1. Problem Description
Find the k-th largest element (not k-th distinct).

## 2. Key Algorithmic Pattern
**Min-Heap of size k**

## 3. Intuition
Maintain a min-heap of size k. For each element, push it; if size exceeds k, pop the minimum. At end, the top is the k-th largest.

## 5. Time and Space Complexity
- **Time:** O(n log k), **Space:** O(k)

## 6. Java Implementation
```java
class Solution {
    public int findKthLargest(int[] nums, int k) {
        PriorityQueue<Integer> minHeap = new PriorityQueue<>();
        for (int n : nums) { minHeap.offer(n); if (minHeap.size() > k) minHeap.poll(); }
        return minHeap.peek();
    }
}
```

---

# #502 — IPO

## 1. Problem Description
Select up to k projects to maximize capital. Can only start a project if current capital ≥ its required capital.

## 2. Key Algorithmic Pattern
**Greedy + Two Heaps**

## 3. Intuition
Sort by capital. Use a max-heap of profits for affordable projects. At each step, unlock all affordable projects into max-heap, then pick the most profitable one.

## 5. Time and Space Complexity
- **Time:** O(n log n + k log n), **Space:** O(n)

## 6. Java Implementation
```java
class Solution {
    public int findMaximizedCapital(int k, int w, int[] profits, int[] capital) {
        int n = profits.length; int[][] p = new int[n][2];
        for (int i = 0; i < n; i++) p[i] = new int[]{capital[i], profits[i]};
        Arrays.sort(p, (a,b) -> a[0]-b[0]);
        PriorityQueue<Integer> maxP = new PriorityQueue<>(Collections.reverseOrder()); int idx = 0;
        for (int i = 0; i < k; i++) {
            while (idx < n && p[idx][0] <= w) maxP.offer(p[idx++][1]);
            if (maxP.isEmpty()) break; w += maxP.poll();
        }
        return w;
    }
}
```

---

# #373 — Find K Pairs with Smallest Sums

## 1. Problem Description
Given two sorted arrays, find k pairs (one from each) with smallest sums.

## 2. Key Algorithmic Pattern
**Min-Heap** — seed with `(nums1[0], nums2[j])` for each j. Pop min, push next candidate.

## 3. Intuition
The next candidate after `(i, j)` is `(i+1, j)`. Seed all `(0, j)` pairs to start. This explores candidates in increasing sum order.

## 5. Time and Space Complexity
- **Time:** O(k log k), **Space:** O(k)

## 6. Java Implementation
```java
class Solution {
    public List<List<Integer>> kSmallestPairs(int[] nums1, int[] nums2, int k) {
        List<List<Integer>> res = new ArrayList<>();
        if (nums1.length == 0 || nums2.length == 0) return res;
        PriorityQueue<int[]> heap = new PriorityQueue<>((a,b) -> a[0]-b[0]);
        for (int j = 0; j < Math.min(k, nums2.length); j++) heap.offer(new int[]{nums1[0]+nums2[j],0,j});
        while (k-- > 0 && !heap.isEmpty()) {
            int[] cur = heap.poll(); int i = cur[1], j = cur[2];
            res.add(Arrays.asList(nums1[i], nums2[j]));
            if (i+1 < nums1.length) heap.offer(new int[]{nums1[i+1]+nums2[j],i+1,j});
        }
        return res;
    }
}
```

---

# #295 — Find Median from Data Stream

## 1. Problem Description
Design data structure supporting `addNum` and `findMedian` at any time.

## 2. Key Algorithmic Pattern
**Two Heaps** (max-heap for lower half, min-heap for upper half)

## 3. Intuition
Keep `maxHeap` (lower half) size ≥ `minHeap` (upper half) size by at most 1. Median = `maxHeap.peek()` if odd total, else average of both tops.

## 5. Time and Space Complexity
- **Time:** O(log n) add, O(1) find, **Space:** O(n)

## 6. Java Implementation
```java
class MedianFinder {
    PriorityQueue<Integer> lo = new PriorityQueue<>(Collections.reverseOrder()); // max-heap
    PriorityQueue<Integer> hi = new PriorityQueue<>(); // min-heap
    public void addNum(int num) {
        lo.offer(num); hi.offer(lo.poll());
        if (hi.size() > lo.size()) lo.offer(hi.poll());
    }
    public double findMedian() {
        return lo.size() > hi.size() ? lo.peek() : (lo.peek() + hi.peek()) / 2.0;
    }
}
```

---

# CATEGORY 20: Bit Manipulation

---

# #67 — Add Binary

## 1. Problem Description
Given two binary strings, return their sum as a binary string.

## 2. Key Algorithmic Pattern
**Simulation with carry**

## 6. Java Implementation
```java
class Solution {
    public String addBinary(String a, String b) {
        StringBuilder sb = new StringBuilder(); int i = a.length()-1, j = b.length()-1, carry = 0;
        while (i >= 0 || j >= 0 || carry > 0) {
            int sum = carry;
            if (i >= 0) sum += a.charAt(i--)-'0';
            if (j >= 0) sum += b.charAt(j--)-'0';
            sb.append(sum%2); carry = sum/2;
        }
        return sb.reverse().toString();
    }
}
```
**Time:** O(max(m,n)), **Space:** O(max(m,n))

---

# #190 — Reverse Bits

## 1. Problem Description
Reverse bits of a 32-bit unsigned integer.

## 2. Key Algorithmic Pattern
**Bit Manipulation** — shift and OR.

## 6. Java Implementation
```java
public class Solution {
    public int reverseBits(int n) {
        int result = 0;
        for (int i = 0; i < 32; i++) { result = (result << 1) | (n & 1); n >>= 1; }
        return result;
    }
}
```
**Time:** O(1), **Space:** O(1)

---

# #191 — Number of 1 Bits

## 1. Problem Description
Return count of '1' bits (Hamming weight) in a positive integer.

## 2. Key Algorithmic Pattern
**Brian Kernighan's Algorithm** — `n & (n-1)` clears the lowest set bit.

## 6. Java Implementation
```java
public class Solution {
    public int hammingWeight(int n) {
        int count = 0; while (n != 0) { n &= (n-1); count++; } return count;
    }
}
```
**Time:** O(k) where k = number of 1-bits, **Space:** O(1)

---

# #136 — Single Number

## 1. Problem Description
Every element appears twice except one. Find it in O(n) with O(1) space.

## 2. Key Algorithmic Pattern
**XOR** — pairs cancel to 0; XOR with 0 = itself.

## 6. Java Implementation
```java
class Solution {
    public int singleNumber(int[] nums) { int r = 0; for (int n : nums) r ^= n; return r; }
}
```
**Time:** O(n), **Space:** O(1)

---

# #137 — Single Number II

## 1. Problem Description
Every element appears three times except one. Find it in O(n) with O(1) space.

## 2. Key Algorithmic Pattern
**Bit counting mod 3**

## 3. Intuition
For each of 32 bit positions, count how many numbers have a 1 there. If `count % 3 != 0`, the unique number has a 1 at that position.

## 6. Java Implementation
```java
class Solution {
    public int singleNumber(int[] nums) {
        int result = 0;
        for (int i = 0; i < 32; i++) {
            int bitSum = 0;
            for (int n : nums) bitSum += (n >> i) & 1;
            result |= (bitSum % 3) << i;
        }
        return result;
    }
}
```
**Time:** O(32n) = O(n), **Space:** O(1)

---

# #201 — Bitwise AND of Numbers Range

## 1. Problem Description
Return bitwise AND of all integers in range `[left, right]`.

## 2. Key Algorithmic Pattern
**Common prefix via bit shifting**

## 3. Intuition
Any bit that changes between `left` and `right` becomes 0 in the AND. Find the common prefix by right-shifting both until equal; left-shift back.

## 6. Java Implementation
```java
class Solution {
    public int rangeBitwiseAnd(int left, int right) {
        int shift = 0;
        while (left != right) { left >>= 1; right >>= 1; shift++; }
        return left << shift;
    }
}
```
**Time:** O(log n), **Space:** O(1)

---

# CATEGORY 21: Math

---

# #9 — Palindrome Number

## 1. Problem Description
Determine if an integer is a palindrome without converting to string.

## 2. Key Algorithmic Pattern
**Reverse half the number**

## 3. Intuition
Negative numbers and numbers ending in 0 (except 0) are never palindromes. Reverse the second half digit-by-digit and compare with the first half.

## 6. Java Implementation
```java
class Solution {
    public boolean isPalindrome(int x) {
        if (x < 0 || (x % 10 == 0 && x != 0)) return false;
        int rev = 0;
        while (x > rev) { rev = rev*10 + x%10; x /= 10; }
        return x == rev || x == rev/10;
    }
}
```
**Time:** O(log n), **Space:** O(1)

---

# #66 — Plus One

## 1. Problem Description
Given a number as a digit array, add one to it.

## 2. Key Algorithmic Pattern
**Linear scan with carry**

## 6. Java Implementation
```java
class Solution {
    public int[] plusOne(int[] digits) {
        for (int i = digits.length-1; i >= 0; i--) {
            if (digits[i] < 9) { digits[i]++; return digits; }
            digits[i] = 0;
        }
        int[] result = new int[digits.length+1]; result[0] = 1; return result;
    }
}
```
**Time:** O(n), **Space:** O(1) amortized

---

# #172 — Factorial Trailing Zeroes

## 1. Problem Description
Return number of trailing zeroes in `n!`.

## 2. Key Algorithmic Pattern
**Count factors of 5**

## 3. Intuition
Trailing zeroes come from 10 = 2 × 5. Factors of 2 always exceed factors of 5, so count multiples of 5, 25, 125... in `n!`.

## 6. Java Implementation
```java
class Solution {
    public int trailingZeroes(int n) { int count = 0; while (n >= 5) { n /= 5; count += n; } return count; }
}
```
**Time:** O(log n), **Space:** O(1)

---

# #69 — Sqrt(x)

## 1. Problem Description
Compute integer square root (floor) of `x` without `Math.sqrt`.

## 2. Key Algorithmic Pattern
**Binary Search**

## 6. Java Implementation
```java
class Solution {
    public int mySqrt(int x) {
        if (x < 2) return x;
        long l = 1, r = x/2;
        while (l <= r) { long m = l+(r-l)/2; if (m*m==x) return (int)m; else if (m*m<x) l=m+1; else r=m-1; }
        return (int)r;
    }
}
```
**Time:** O(log x), **Space:** O(1)

---

# #7 — Reverse Integer

## 1. Problem Description
Reverse digits of a 32-bit integer. Return 0 on overflow.

## 2. Key Algorithmic Pattern
**Math with overflow check**

## 3. Intuition
Check overflow before each multiply by 10: if `result > MAX/10` or `result == MAX/10` and digit `> 7`, it would overflow.

## 6. Java Implementation
```java
class Solution {
    public int reverse(int x) {
        int result = 0;
        while (x != 0) {
            int d = x % 10; x /= 10;
            if (result > Integer.MAX_VALUE/10 || (result==Integer.MAX_VALUE/10 && d>7)) return 0;
            if (result < Integer.MIN_VALUE/10 || (result==Integer.MIN_VALUE/10 && d<-8)) return 0;
            result = result*10 + d;
        }
        return result;
    }
}
```
**Time:** O(log x), **Space:** O(1)

---

# CATEGORY 22: 1D Dynamic Programming

---

# #70 — Climbing Stairs

## 1. Problem Description
Climb 1 or 2 steps at a time. How many distinct ways to reach step `n`?

## 2. Key Algorithmic Pattern
**1D DP (Fibonacci variant)**

## 3. Intuition
`dp[n] = dp[n-1] + dp[n-2]` — you reached step n from n-1 (one step) or n-2 (two steps).

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public int climbStairs(int n) {
        if (n <= 2) return n;
        int prev2 = 1, prev1 = 2;
        for (int i = 3; i <= n; i++) { int cur = prev1+prev2; prev2=prev1; prev1=cur; }
        return prev1;
    }
}
```

---

# #198 — House Robber

## 1. Problem Description
Rob non-adjacent houses to maximize total. Can't rob two adjacent houses.

## 2. Key Algorithmic Pattern
**1D DP**

## 3. Intuition
`dp[i] = max(dp[i-1], dp[i-2] + nums[i])` — rob current house (skipping previous) or skip current (keep previous best).

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public int rob(int[] nums) {
        int prev2 = 0, prev1 = 0;
        for (int n : nums) { int cur = Math.max(prev1, prev2+n); prev2=prev1; prev1=cur; }
        return prev1;
    }
}
```

---

# #139 — Word Break

## 1. Problem Description
Return true if string `s` can be segmented into dictionary words.

## 2. Key Algorithmic Pattern
**1D DP**

## 3. Intuition
`dp[i]` = true if `s[0..i-1]` can be segmented. For each `i`, check all `j < i`: if `dp[j]` and `s[j..i-1]` is in the dictionary, `dp[i] = true`.

## 4. Step-by-Step Approach
1. `dp[0] = true` (empty string)
2. For each `i` from 1 to n: for each `j` from 0 to i: if `dp[j]` and `s.substring(j,i)` in dict, `dp[i] = true`
3. Return `dp[n]`

## 5. Time and Space Complexity
- **Time:** O(n²), **Space:** O(n)

## 6. Java Implementation
```java
class Solution {
    public boolean wordBreak(String s, List<String> wordDict) {
        Set<String> dict = new HashSet<>(wordDict); int n = s.length();
        boolean[] dp = new boolean[n+1]; dp[0] = true;
        for (int i = 1; i <= n; i++)
            for (int j = 0; j < i; j++)
                if (dp[j] && dict.contains(s.substring(j,i))) { dp[i]=true; break; }
        return dp[n];
    }
}
```

---

# #322 — Coin Change

## 1. Problem Description
Minimum coins to make `amount`. Return -1 if impossible.

## 2. Key Algorithmic Pattern
**1D DP (unbounded knapsack)**

## 3. Intuition
`dp[i]` = min coins for amount `i`. For each coin: `dp[i] = min(dp[i], dp[i - coin] + 1)`. Initialize to infinity, `dp[0] = 0`.

## 5. Time and Space Complexity
- **Time:** O(amount * coins.length), **Space:** O(amount)

## 6. Java Implementation
```java
class Solution {
    public int coinChange(int[] coins, int amount) {
        int[] dp = new int[amount+1]; Arrays.fill(dp, amount+1); dp[0] = 0;
        for (int i = 1; i <= amount; i++)
            for (int coin : coins) if (coin <= i) dp[i] = Math.min(dp[i], dp[i-coin]+1);
        return dp[amount] > amount ? -1 : dp[amount];
    }
}
```

---

# #300 — Longest Increasing Subsequence

## 1. Problem Description
Return length of longest strictly increasing subsequence (not necessarily contiguous).

## 2. Key Algorithmic Pattern
**Binary Search + Patience Sorting** (O(n log n))

## 3. Intuition
Maintain `tails` array where `tails[i]` is the smallest possible tail for an increasing subsequence of length `i+1`. For each number, binary search for leftmost position ≥ number and replace (or extend if past end). This keeps tails as small as possible, maximizing future extension potential.

## 4. Step-by-Step Approach
1. For each `num`: binary search in `tails` for first value ≥ `num`
2. If found, replace it (smaller tail = more room to grow)
3. If not found, append (extends LIS length)
4. Return `tails.size()`

## 5. Time and Space Complexity
- **Time:** O(n log n), **Space:** O(n)

## 6. Java Implementation
```java
class Solution {
    public int lengthOfLIS(int[] nums) {
        List<Integer> tails = new ArrayList<>();
        for (int num : nums) {
            int l = 0, r = tails.size();
            while (l < r) { int m = l+(r-l)/2; if (tails.get(m)<num) l=m+1; else r=m; }
            if (l == tails.size()) tails.add(num);
            else tails.set(l, num);
        }
        return tails.size();
    }
}
```

---

# CATEGORY 23: Multidimensional Dynamic Programming

---

# #120 — Triangle

## 1. Problem Description
Find minimum path sum from top to bottom of triangle (each step to adjacent number in row below).

## 2. Key Algorithmic Pattern
**DP (bottom-up)**

## 3. Intuition
Work from second-to-last row upward. `dp[j] = triangle[i][j] + min(dp[j], dp[j+1])`. Answer is `dp[0]`.

## 5. Time and Space Complexity
- **Time:** O(n²), **Space:** O(n)

## 6. Java Implementation
```java
class Solution {
    public int minimumTotal(List<List<Integer>> triangle) {
        int n = triangle.size(); int[] dp = new int[n+1];
        for (int i = n-1; i >= 0; i--)
            for (int j = 0; j <= i; j++) dp[j] = triangle.get(i).get(j) + Math.min(dp[j], dp[j+1]);
        return dp[0];
    }
}
```

---

# #64 — Minimum Path Sum

## 1. Problem Description
Find path from top-left to bottom-right of grid (move only right or down) with minimum sum.

## 2. Key Algorithmic Pattern
**2D DP**

## 3. Intuition
`dp[i][j] = grid[i][j] + min(from above, from left)`. Modify grid in-place.

## 6. Java Implementation
```java
class Solution {
    public int minPathSum(int[][] grid) {
        int m = grid.length, n = grid[0].length;
        for (int i = 0; i < m; i++) for (int j = 0; j < n; j++) {
            if (i==0&&j==0) continue;
            else if (i==0) grid[i][j]+=grid[i][j-1];
            else if (j==0) grid[i][j]+=grid[i-1][j];
            else grid[i][j]+=Math.min(grid[i-1][j],grid[i][j-1]);
        }
        return grid[m-1][n-1];
    }
}
```
**Time:** O(m*n), **Space:** O(1)

---

# #62 — Unique Paths

## 1. Problem Description
Count unique paths from top-left to bottom-right of m×n grid (only right or down moves).

## 2. Key Algorithmic Pattern
**1D DP** (space-optimized 2D DP)

## 3. Intuition
`dp[j]` = paths to current cell = paths from above (`dp[j]`) + paths from left (`dp[j-1]`).

## 6. Java Implementation
```java
class Solution {
    public int uniquePaths(int m, int n) {
        int[] dp = new int[n]; Arrays.fill(dp, 1);
        for (int i = 1; i < m; i++) for (int j = 1; j < n; j++) dp[j] += dp[j-1];
        return dp[n-1];
    }
}
```
**Time:** O(m*n), **Space:** O(n)

---

# #63 — Unique Paths II

## 1. Problem Description
Same as Unique Paths but grid has obstacles (`1` = obstacle). Count paths avoiding obstacles.

## 6. Java Implementation
```java
class Solution {
    public int uniquePathsWithObstacles(int[][] g) {
        int m = g.length, n = g[0].length; int[] dp = new int[n]; dp[0] = 1;
        for (int i = 0; i < m; i++) for (int j = 0; j < n; j++) {
            if (g[i][j]==1) dp[j]=0; else if (j>0) dp[j]+=dp[j-1];
        }
        return dp[n-1];
    }
}
```
**Time:** O(m*n), **Space:** O(n)

---

# #516 — Longest Palindromic Subsequence

## 1. Problem Description
Find length of longest palindromic subsequence in string `s`.

## 2. Key Algorithmic Pattern
**2D DP**

## 3. Intuition
`dp[i][j]` = LPS length in `s[i..j]`. If `s[i]==s[j]`: `dp[i][j] = dp[i+1][j-1] + 2`. Else: `max(dp[i+1][j], dp[i][j-1])`. Fill diagonally by increasing length.

## 5. Time and Space Complexity
- **Time:** O(n²), **Space:** O(n²)

## 6. Java Implementation
```java
class Solution {
    public int longestPalindromeSubseq(String s) {
        int n = s.length(); int[][] dp = new int[n][n];
        for (int i = 0; i < n; i++) dp[i][i] = 1;
        for (int len = 2; len <= n; len++) for (int i = 0; i <= n-len; i++) {
            int j = i+len-1;
            if (s.charAt(i)==s.charAt(j)) dp[i][j] = dp[i+1][j-1]+2;
            else dp[i][j] = Math.max(dp[i+1][j], dp[i][j-1]);
        }
        return dp[0][n-1];
    }
}
```

---

# #72 — Edit Distance

## 1. Problem Description
Minimum operations (insert, delete, replace) to convert `word1` to `word2`.

## 2. Key Algorithmic Pattern
**2D DP (classic)**

## 3. Intuition
`dp[i][j]` = edit distance between `word1[0..i-1]` and `word2[0..j-1]`. If chars match: `dp[i-1][j-1]`. Else: `1 + min(replace, delete, insert)`.

## 5. Time and Space Complexity
- **Time:** O(m*n), **Space:** O(m*n)

## 6. Java Implementation
```java
class Solution {
    public int minDistance(String w1, String w2) {
        int m = w1.length(), n = w2.length(); int[][] dp = new int[m+1][n+1];
        for (int i = 0; i <= m; i++) dp[i][0] = i;
        for (int j = 0; j <= n; j++) dp[0][j] = j;
        for (int i = 1; i <= m; i++) for (int j = 1; j <= n; j++) {
            if (w1.charAt(i-1)==w2.charAt(j-1)) dp[i][j]=dp[i-1][j-1];
            else dp[i][j]=1+Math.min(dp[i-1][j-1],Math.min(dp[i-1][j],dp[i][j-1]));
        }
        return dp[m][n];
    }
}
```

---

# #123 — Best Time to Buy and Sell Stock III

## 1. Problem Description
At most 2 transactions. Maximize profit.

## 2. Key Algorithmic Pattern
**DP with 4-state machine**

## 3. Intuition
Track 4 states greedily: `buy1` (max profit after first buy), `sell1` (after first sell), `buy2` (after second buy), `sell2` (after second sell).

## 5. Time and Space Complexity
- **Time:** O(n), **Space:** O(1)

## 6. Java Implementation
```java
class Solution {
    public int maxProfit(int[] prices) {
        int buy1=Integer.MIN_VALUE, sell1=0, buy2=Integer.MIN_VALUE, sell2=0;
        for (int p : prices) {
            buy1=Math.max(buy1,-p); sell1=Math.max(sell1,buy1+p);
            buy2=Math.max(buy2,sell1-p); sell2=Math.max(sell2,buy2+p);
        }
        return sell2;
    }
}
```

---

# #188 — Best Time to Buy and Sell Stock IV

## 1. Problem Description
At most k transactions. Maximize profit.

## 2. Key Algorithmic Pattern
**DP with k states**

## 3. Intuition
If `k >= n/2`, unlimited transactions (greedy). Otherwise maintain arrays `buy[j]` and `sell[j]` for j-th transaction.

## 5. Time and Space Complexity
- **Time:** O(n*k), **Space:** O(k)

## 6. Java Implementation
```java
class Solution {
    public int maxProfit(int k, int[] prices) {
        int n = prices.length;
        if (k >= n/2) { int p=0; for (int i=1;i<n;i++) if (prices[i]>prices[i-1]) p+=prices[i]-prices[i-1]; return p; }
        int[] buy = new int[k], sell = new int[k];
        Arrays.fill(buy, Integer.MIN_VALUE);
        for (int p : prices) for (int j = 0; j < k; j++) {
            buy[j]=Math.max(buy[j],(j==0?0:sell[j-1])-p);
            sell[j]=Math.max(sell[j],buy[j]+p);
        }
        return sell[k-1];
    }
}
```

---

# #221 — Maximal Square

## 1. Problem Description
Find the largest square containing only '1's in a binary matrix. Return its area.

## 2. Key Algorithmic Pattern
**2D DP**

## 3. Intuition
`dp[i][j]` = side length of largest square with bottom-right corner at `(i,j)`. If `matrix[i][j]=='1'`: `dp[i][j] = min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1]) + 1`. The `min` is the bottleneck — all three neighbors must support the square.

## 5. Time and Space Complexity
- **Time:** O(m*n), **Space:** O(m*n)

## 6. Java Implementation
```java
class Solution {
    public int maximalSquare(char[][] matrix) {
        int m = matrix.length, n = matrix[0].length, maxSide = 0;
        int[][] dp = new int[m+1][n+1];
        for (int i = 1; i <= m; i++) for (int j = 1; j <= n; j++) {
            if (matrix[i-1][j-1]=='1') {
                dp[i][j]=Math.min(dp[i-1][j],Math.min(dp[i][j-1],dp[i-1][j-1]))+1;
                maxSide=Math.max(maxSide,dp[i][j]);
            }
        }
        return maxSide*maxSide;
    }
}
```

---

# #97 — Interleaving String

## 1. Problem Description
Return true if `s3` is formed by interleaving `s1` and `s2` (preserving relative order of each).

## 2. Key Algorithmic Pattern
**2D DP**

## 3. Intuition
`dp[i][j]` = true if `s3[0..i+j-1]` can be formed by interleaving `s1[0..i-1]` and `s2[0..j-1]`. At each cell, check if we can arrive from the left (using `s2[j-1]`) or from above (using `s1[i-1]`).

## 5. Time and Space Complexity
- **Time:** O(m*n), **Space:** O(m*n)

## 6. Java Implementation
```java
class Solution {
    public boolean isInterleave(String s1, String s2, String s3) {
        int m=s1.length(), n=s2.length();
        if (m+n!=s3.length()) return false;
        boolean[][] dp = new boolean[m+1][n+1]; dp[0][0]=true;
        for (int i=1;i<=m;i++) dp[i][0]=dp[i-1][0]&&s1.charAt(i-1)==s3.charAt(i-1);
        for (int j=1;j<=n;j++) dp[0][j]=dp[0][j-1]&&s2.charAt(j-1)==s3.charAt(j-1);
        for (int i=1;i<=m;i++) for (int j=1;j<=n;j++)
            dp[i][j]=(dp[i-1][j]&&s1.charAt(i-1)==s3.charAt(i+j-1))
                    ||(dp[i][j-1]&&s2.charAt(j-1)==s3.charAt(i+j-1));
        return dp[m][n];
    }
}
```
