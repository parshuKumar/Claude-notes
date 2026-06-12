# Pattern 04 — Stack and Monotonic Stack
## Difficulty Level: FOUNDATION
## Interview Frequency: OFTEN
## Estimated time to master: 4-5 days
## Problems in this set: 11 problems

---

## THE GRANDMASTER'S HONEST TAKE

Stack is the pattern most people think is trivial because "it's just a stack." They do valid parentheses, pat themselves on the back, and move on. Then an interviewer asks Largest Rectangle in Histogram and they freeze completely. The reason: they treated the stack as a data structure to memorize instead of a thinking tool for solving a specific CLASS of problems.

The monotonic stack is one of the most powerful tools for competitive programming at the 1800-2000 level. It solves an entire class of "next greater / next smaller / how many elements can see X" problems in O(n) that look impossible without it. The contribution technique — where you use a monotonic stack to compute the sum of min/max over all subarrays — appears in Codeforces Div 2 D problems regularly. Most 1600-rated coders have never heard of it.

Here is the honest breakdown: basic stack takes 1 day to own. Monotonic stack for next-greater/smaller takes 1-2 days. The histogram problem and its variants (trapping rain water via stack, maximal rectangle) take 1-2 days. The contribution technique takes 1 full day to internalize. That's your 4-5 days. Don't rush the contribution technique — it pays off massively in contests.

---

## ELI5 — THE CORE IDEA

### Basic Stack
Imagine you're sorting stacks of dishes. Every time you put a dish down, the dish on top is the one you just placed. When you pick up a dish, you always take the top one. You never reach into the middle. This is LIFO — Last In, First Out.

Problems that fit this pattern: anything where the MOST RECENTLY seen element is the one most relevant to the current decision. Parentheses matching: the most recently opened bracket is the one the current closing bracket should close.

### Monotonic Stack
Now imagine you're watching a parade of skyscrapers roll by on a conveyor belt. You want to know, for each building, what's the NEXT TALLER building to its right.

You keep a stack of buildings you've seen but haven't found a "next taller" for yet. When a new building arrives:
- If it's taller than the top of the stack → that top building just found its "next taller." Pop it and record the answer. Keep comparing with the new top.
- If it's shorter than or equal to the top → this new building hasn't resolved anything yet. Push it.

At any moment, the stack contains buildings in DECREASING order of height (bottom is tallest, top is shortest). This is the monotonic invariant. The stack is always sorted. That sorted property is what makes it powerful — you can ask "what's the nearest element larger/smaller than X" in O(1) amortized.

---

## THE PATTERN — FORMALLY DEFINED

### What this pattern solves:

**Basic Stack:**
1. **Matching/validation:** Check balanced brackets, valid expressions
2. **Reversal:** Reverse sequences using stack's LIFO property
3. **Expression evaluation:** Postfix/infix evaluation
4. **Backtracking structure:** Undo operations (recent first)
5. **DFS iterative:** Simulate recursion explicitly

**Monotonic Stack:**
1. **Next Greater Element (NGE):** For each element, find the next element to the right that is strictly greater
2. **Next Smaller Element (NSE):** For each element, find the next element to the right that is strictly smaller
3. **Previous Greater Element (PGE):** Same but to the LEFT
4. **Previous Smaller Element (PSE):** Same but to the LEFT and smaller
5. **Span problems:** How many consecutive elements to the left/right are ≤ or ≥ current
6. **Histogram problems:** Largest rectangle, trapped water
7. **Contribution technique:** Sum of min/max over all subarrays

### The invariant of a monotonic stack:
At all times, the elements in the stack are in **monotonically increasing** OR **monotonically decreasing** order (depending on which you maintain). This sorted property is enforced by popping elements that violate the order when a new element arrives.

- **Increasing monotonic stack:** smallest on top. Pop when new element ≤ top. Used for Next Greater Element.
- **Decreasing monotonic stack:** largest on top. Pop when new element ≥ top. Used for Next Smaller Element.

**What "popping" means:** When you pop an element because the new element caused it to be popped, the new element IS the answer for the popped element. That's the insight.

### What the answer is extracted from:
- For NGE/NSE: recorded at pop time (the current element being added is the answer for the popped element)
- For histogram/trapped water: computed from the popped element's height and the width between left and right boundaries
- For contribution technique: computed from the distances between boundaries

---

## WHEN TO RECOGNIZE THIS PATTERN

### Signals that SCREAM "use a stack":

**Signal 1:** "Check if brackets/parentheses are balanced"
→ Basic stack. Open bracket → push. Close bracket → pop and verify match.

**Signal 2:** "Find the next greater/smaller element for each element"
→ Monotonic stack. Classic pattern.

**Signal 3:** "How many days until a warmer temperature" / "stock span" / "next taller building"
→ Monotonic stack. "Next X" family.

**Signal 4:** "Largest rectangle in histogram" / "Trapping rain water" / "Maximal rectangle"
→ Monotonic stack for histogram problems.

**Signal 5:** "Evaluate a mathematical expression" / "Decode a string" / "Simplify path"
→ Basic stack for expression parsing.

**Signal 6:** "Sum of (minimum/maximum) over all subarrays"
→ Contribution technique with monotonic stack.

**Signal 7:** "Remove K digits to make the smallest number" / "Remove invalid parentheses"
→ Monotonic stack for greedy construction.

### Signals that LOOK LIKE stack but are NOT:

**Fake signal 1:** "Find the minimum in every window of size K"
→ This needs a monotonic DEQUE (double-ended queue), not a stack. See Pattern 29. The key difference: in a window, old elements leave from the front; a stack can't pop from the front.

**Fake signal 2:** "Find all pairs (i, j) where nums[j] > nums[i] and j - i = 1"
→ Simple linear scan suffices. Stack is overkill for direct-neighbor comparisons.

**Fake signal 3:** "Sort elements"
→ Stack-based sorting (patience sort) exists but for most problems, just use `sort()`. Stack sort is Pattern 21 (LIS) territory.

---

## THE TEMPLATES (C++)

### Template 1: Basic Stack — Bracket Matching

```cpp
// Basic Stack Matching Template
// Time: O(n) | Space: O(n)
// Use for: balanced parentheses, nested structure validation

bool isValid(string s) {
    stack<char> st;
    
    for (char c : s) {
        if (c == '(' || c == '[' || c == '{') {
            st.push(c);          // open bracket → push
        } else {
            // close bracket → must match top
            if (st.empty()) return false;  // nothing to match
            char top = st.top(); st.pop();
            
            if (c == ')' && top != '(') return false;
            if (c == ']' && top != '[') return false;
            if (c == '}' && top != '{') return false;
        }
    }
    
    return st.empty(); // all brackets matched
}

// HOW TO ADAPT:
// 1. Change matching condition for different bracket types
// 2. For "minimum removals to make valid": count unmatched open + unmatched close
// 3. For "minimum insertions to make valid": track imbalance count
// 4. For "score of parentheses": track depth instead of match
```

### Template 2: Monotonic Stack — Next Greater Element (Core Template)

```cpp
// Monotonic Stack — Next Greater Element Template
// Time: O(n) amortized — each element pushed and popped at most once
// Space: O(n) — stack holds at most n elements

// INVARIANT: stack contains indices of elements for which we
//            have NOT yet found the "next greater element"
//            The elements at these indices are in DECREASING order
//            (bottom is largest, top is smallest)

vector<int> nextGreaterElement(vector<int>& nums) {
    int n = nums.size();
    vector<int> result(n, -1);  // -1 = no next greater element
    stack<int> st;              // stack of INDICES (not values)
    
    for (int i = 0; i < n; i++) {
        // While current element > top of stack → top found its answer
        while (!st.empty() && nums[i] > nums[st.top()]) {
            int idx = st.top(); st.pop();
            result[idx] = nums[i];  // nums[i] is the NGE for nums[idx]
        }
        st.push(i);
    }
    // Elements remaining in stack have no NGE → result[idx] stays -1
    
    return result;
}

// KEY DESIGN CHOICES:
// 1. Push INDICES not values → allows computing distances
// 2. Pop condition is STRICT > (for strictly greater)
//    Use >= for "next greater or equal"
// 3. Default value -1 handles "no next greater" automatically
// 4. Process LEFT to RIGHT for "next to the right"
//    Process RIGHT to LEFT for "previous to the left" (or iterate backwards)

// VARIANT: Next Greater Element in circular array (LC 503):
vector<int> nextGreaterCircular(vector<int>& nums) {
    int n = nums.size();
    vector<int> result(n, -1);
    stack<int> st;
    
    // Process 2n elements to simulate circular array
    for (int i = 0; i < 2 * n; i++) {
        while (!st.empty() && nums[i % n] > nums[st.top()]) {
            result[st.top()] = nums[i % n];
            st.pop();
        }
        if (i < n) st.push(i);  // only push indices from first pass
    }
    return result;
}
```

### Template 3: All Four Directions — PSE, NSE, PGE, NGE

```cpp
// The Four Boundary Templates
// These four arrays form the building blocks of histogram/contribution problems

// ─────────────────────────────────────────────────
// PREVIOUS SMALLER ELEMENT (PSE) — index of nearest smaller to the LEFT
// Stack is INCREASING (smallest on top)
// ─────────────────────────────────────────────────
vector<int> previousSmaller(vector<int>& nums) {
    int n = nums.size();
    vector<int> left(n, -1);   // -1 = no smaller element to the left
    stack<int> st;
    
    for (int i = 0; i < n; i++) {
        while (!st.empty() && nums[st.top()] >= nums[i]) st.pop();
        left[i] = st.empty() ? -1 : st.top();
        st.push(i);
    }
    return left;
}

// ─────────────────────────────────────────────────
// NEXT SMALLER ELEMENT (NSE) — index of nearest smaller to the RIGHT
// Stack is INCREASING (smallest on top)
// ─────────────────────────────────────────────────
vector<int> nextSmaller(vector<int>& nums) {
    int n = nums.size();
    vector<int> right(n, n);   // n = virtual "past the end" index
    stack<int> st;
    
    for (int i = n - 1; i >= 0; i--) {   // process RIGHT to LEFT
        while (!st.empty() && nums[st.top()] >= nums[i]) st.pop();
        right[i] = st.empty() ? n : st.top();
        st.push(i);
    }
    return right;
}

// ─────────────────────────────────────────────────
// PREVIOUS GREATER ELEMENT (PGE) — index of nearest greater to the LEFT
// Stack is DECREASING (largest on top)
// ─────────────────────────────────────────────────
vector<int> previousGreater(vector<int>& nums) {
    int n = nums.size();
    vector<int> left(n, -1);
    stack<int> st;
    
    for (int i = 0; i < n; i++) {
        while (!st.empty() && nums[st.top()] <= nums[i]) st.pop();
        left[i] = st.empty() ? -1 : st.top();
        st.push(i);
    }
    return left;
}

// ─────────────────────────────────────────────────
// NEXT GREATER ELEMENT (NGE) — index of nearest greater to the RIGHT
// Stack is DECREASING (largest on top)
// ─────────────────────────────────────────────────
vector<int> nextGreater(vector<int>& nums) {
    int n = nums.size();
    vector<int> right(n, n);
    stack<int> st;
    
    for (int i = n - 1; i >= 0; i--) {
        while (!st.empty() && nums[st.top()] <= nums[i]) st.pop();
        right[i] = st.empty() ? n : st.top();
        st.push(i);
    }
    return right;
}

// ═══════════════════════════════════════════════════════════
// CRITICAL: Handling DUPLICATES in boundary arrays
// ═══════════════════════════════════════════════════════════
// When you have duplicates, be careful about strict vs non-strict:
//
// For contribution technique (sum of min/max over all subarrays):
// To avoid counting subarrays TWICE when there are duplicate elements,
// use STRICT inequality on one side and NON-STRICT on the other:
//
// left[i]  = previous index j where nums[j] <  nums[i]  (strictly less)
// right[i] = next     index j where nums[j] <= nums[i]  (less or equal)
//
// This ensures each subarray is counted with exactly ONE "minimum" element.
```

### Template 4: Largest Rectangle in Histogram

```cpp
// Largest Rectangle in Histogram — The Classic Monotonic Stack Problem
// Time: O(n) | Space: O(n)
// Key insight: for each bar, find how far left and right it can extend as the minimum height

int largestRectangleArea(vector<int>& heights) {
    int n = heights.size();
    stack<int> st;   // monotonic increasing stack of indices
    int maxArea = 0;
    
    // Process each bar + one sentinel (height 0) at the end
    // The sentinel flushes all remaining bars from the stack
    for (int i = 0; i <= n; i++) {
        int h = (i == n) ? 0 : heights[i];   // sentinel height = 0
        
        while (!st.empty() && heights[st.top()] > h) {
            int height = heights[st.top()]; st.pop();
            
            // Width: from (new top + 1) to (i - 1)
            // If stack is empty, the bar extends all the way to index 0
            int width = st.empty() ? i : (i - st.top() - 1);
            
            maxArea = max(maxArea, height * width);
        }
        
        st.push(i);
    }
    
    return maxArea;
}

// TRACING THROUGH THE LOGIC:
// heights = [2, 1, 5, 6, 2, 3]
//
// When we pop bar at index `top` because heights[i] < heights[top]:
//   - heights[i] is the FIRST smaller element to the right of top → right boundary
//   - st.top() (new top after pop) is the last element still in stack → left boundary
//   - Width = right boundary - left boundary - 1 = i - st.top() - 1
//   - If stack is empty after pop: bar extends to index 0, width = i
//
// WHY the sentinel (height = 0) at the end:
// Ensures all bars still in the stack get processed.
// Without it, bars that never found a smaller element to the right
// would remain unprocessed.
```

### Template 5: Trapping Rain Water — Stack Approach

```cpp
// Trapping Rain Water — Stack Approach
// Time: O(n) | Space: O(n)
// Alternative to two-pointer approach (see Pattern 02)
// Key insight: water is trapped in "valleys" between taller walls

int trap(vector<int>& height) {
    stack<int> st;   // monotonic decreasing stack (stores indices)
    int water = 0;
    
    for (int i = 0; i < height.size(); i++) {
        // Process "valley" formations
        while (!st.empty() && height[i] > height[st.top()]) {
            int bottom = st.top(); st.pop();   // the valley floor
            
            if (st.empty()) break;  // no left wall → no water can be trapped
            
            int left = st.top();               // left wall index
            int right = i;                     // right wall index (current)
            
            // Water height = min(left wall, right wall) - valley floor
            int waterHeight = min(height[left], height[right]) - height[bottom];
            int width = right - left - 1;
            
            water += waterHeight * width;
        }
        
        st.push(i);
    }
    
    return water;
}

// HOW IT WORKS:
// Stack maintains indices in DECREASING order of height (decreasing monotonic).
// When we see a taller bar:
//   - The top of stack is a "valley bottom" (shorter than current)
//   - The element below it in the stack is the "left wall"
//   - Current index is the "right wall"
//   - Compute water in this layer: (min(left, right) - bottom) * width
//   - This "layer by layer" filling approach handles complex profiles correctly.
```

### Template 6: Contribution Technique — Sum of Subarray Minimums

```cpp
// Contribution Technique Template
// For "sum of min/max of all subarrays"
// Time: O(n) | Space: O(n)
// Key insight: instead of enumerating subarrays, ask:
//   "How many subarrays have nums[i] as their minimum?"
//   Then total contribution of nums[i] = nums[i] * (count of such subarrays)

int sumSubarrayMins(vector<int>& arr) {
    int n = arr.size();
    const int MOD = 1e9 + 7;
    
    // For each element, find:
    // left[i] = distance to previous SMALLER element (or start of array)
    // right[i] = distance to next SMALLER OR EQUAL element (or end of array)
    // Using strict on left and non-strict on right avoids double-counting duplicates
    
    vector<int> left(n), right(n);
    stack<int> st;
    
    // Previous smaller: strict <
    for (int i = 0; i < n; i++) {
        while (!st.empty() && arr[st.top()] >= arr[i]) st.pop();
        left[i] = st.empty() ? i + 1 : i - st.top();
        // left[i] = number of elements that can extend LEFT with arr[i] as min
        // = i - (index of previous smaller) = i - st.top()
        // If no previous smaller: left[i] = i + 1 (extends to index 0)
        st.push(i);
    }
    
    // Clear stack for reuse
    while (!st.empty()) st.pop();
    
    // Next smaller or equal: non-strict <=
    for (int i = n - 1; i >= 0; i--) {
        while (!st.empty() && arr[st.top()] > arr[i]) st.pop();
        right[i] = st.empty() ? n - i : st.top() - i;
        // right[i] = number of elements that can extend RIGHT with arr[i] as min
        // = (index of next smaller or equal) - i = st.top() - i
        // If no next smaller or equal: right[i] = n - i
        st.push(i);
    }
    
    // Contribution of arr[i] = arr[i] * left[i] * right[i]
    // Explanation: arr[i] is minimum in left[i] * right[i] subarrays
    // (left[i] choices for left boundary × right[i] choices for right boundary)
    
    long long result = 0;
    for (int i = 0; i < n; i++) {
        result = (result + (long long)arr[i] * left[i] * right[i]) % MOD;
    }
    
    return (int)result;
}

// UNDERSTANDING left[i] and right[i]:
// left[i] = how many consecutive elements to the LEFT (including i itself)
//           form subarrays where arr[i] is still the minimum
// right[i] = how many consecutive elements to the RIGHT (including i itself)
//            form subarrays where arr[i] is still the minimum
//
// Total subarrays where arr[i] is minimum = left[i] * right[i]
// (one choice of left boundary for each of the left[i] options,
//  combined with one choice of right boundary for each of the right[i] options)
//
// Example: arr = [3, 1, 2, 4], for i=1 (value=1):
// left[1] = 2 (can extend to include index 0, since arr[0]=3 > 1)
// right[1] = 3 (can extend to include indices 1, 2, 3 since all ≥ 1)
// Subarrays with 1 as minimum: 2*3 = 6
// They are: [1], [1,2], [1,2,4], [3,1], [3,1,2], [3,1,2,4] ✓
```

### Template 7: Expression Evaluation Stack

```cpp
// Stack-Based Expression Evaluation
// For: Evaluate Reverse Polish Notation, Basic Calculator variants

// ─────────────────────────────────────────────────
// Reverse Polish Notation (Postfix) Evaluation:
// ─────────────────────────────────────────────────
int evalRPN(vector<string>& tokens) {
    stack<long long> st;
    
    for (const string& token : tokens) {
        if (token == "+" || token == "-" || token == "*" || token == "/") {
            long long b = st.top(); st.pop();  // second operand
            long long a = st.top(); st.pop();  // first operand
            
            if (token == "+") st.push(a + b);
            else if (token == "-") st.push(a - b);
            else if (token == "*") st.push(a * b);
            else st.push(a / b);  // truncation toward zero for int division
        } else {
            st.push(stoll(token));
        }
    }
    
    return (int)st.top();
}

// ─────────────────────────────────────────────────
// Infix Evaluation with parentheses (Basic Calculator II):
// No parentheses version: track sign and accumulated value
// ─────────────────────────────────────────────────
int calculate(string s) {
    stack<int> st;
    int num = 0;
    char sign = '+';  // sign BEFORE current number
    
    for (int i = 0; i < s.size(); i++) {
        char c = s[i];
        
        if (isdigit(c)) {
            num = num * 10 + (c - '0');  // build multi-digit number
        }
        
        // Process when: operator found, OR end of string
        if ((!isdigit(c) && c != ' ') || i == s.size() - 1) {
            if (sign == '+') st.push(num);
            else if (sign == '-') st.push(-num);
            else if (sign == '*') { int top = st.top(); st.pop(); st.push(top * num); }
            else if (sign == '/') { int top = st.top(); st.pop(); st.push(top / num); }
            
            sign = c;  // update sign for next number
            num = 0;   // reset number accumulator
        }
    }
    
    int result = 0;
    while (!st.empty()) { result += st.top(); st.pop(); }
    return result;
}

// ─────────────────────────────────────────────────
// With Parentheses (Basic Calculator I):
// When '(' → push current result and sign, reset
// When ')' → pop and combine with saved result
// ─────────────────────────────────────────────────
int calculateWithParens(string s) {
    stack<int> st;  // stores [result, sign] pairs alternately
    int result = 0, num = 0, sign = 1;
    
    for (char c : s) {
        if (isdigit(c)) {
            num = num * 10 + (c - '0');
        } else if (c == '+') {
            result += sign * num; num = 0; sign = 1;
        } else if (c == '-') {
            result += sign * num; num = 0; sign = -1;
        } else if (c == '(') {
            st.push(result);   // save current result
            st.push(sign);     // save sign before '('
            result = 0;        // reset for inner expression
            sign = 1;
        } else if (c == ')') {
            result += sign * num; num = 0;
            result *= st.top(); st.pop();  // apply sign before '('
            result += st.top(); st.pop();  // add saved result
        }
    }
    result += sign * num;
    return result;
}
```

### Template 8: Min Stack — Tracking Minimum with Auxiliary Stack

```cpp
// Min Stack Template — O(1) minimum access always
// Two approaches: auxiliary stack or encoding in single stack

// Approach 1: Auxiliary stack (cleaner, preferred)
class MinStack {
    stack<int> main;  // stores all elements
    stack<int> minSt; // stores running minimums
public:
    void push(int val) {
        main.push(val);
        if (minSt.empty() || val <= minSt.top()) {
            minSt.push(val);  // new minimum (or equal minimum)
        }
    }
    
    void pop() {
        if (main.top() == minSt.top()) {
            minSt.pop();  // the minimum itself is being removed
        }
        main.pop();
    }
    
    int top() { return main.top(); }
    int getMin() { return minSt.top(); }
};

// WHY push equal values to minSt:
// If you have [3, 1, 1] and pop twice, you need minSt to still have 1.
// Using <= ensures duplicates of the minimum are tracked.

// Approach 2: Single stack with encoding (O(1) space for min tracking)
// Store pairs (value, current_min_at_this_point)
class MinStackV2 {
    stack<pair<int,int>> st; // (value, min_so_far)
public:
    void push(int val) {
        int currentMin = st.empty() ? val : min(val, st.top().second);
        st.push({val, currentMin});
    }
    void pop() { st.pop(); }
    int top() { return st.top().first; }
    int getMin() { return st.top().second; }
};
```

### Template 9: Monotonic Stack for Greedy Construction

```cpp
// Greedy Construction with Monotonic Stack
// For: "Remove K digits to get smallest number"
//      "Build lexicographically smallest/largest sequence"

string removeKdigits(string num, int k) {
    stack<char> st;  // maintains increasing order (smallest possible prefix)
    
    for (char c : num) {
        // If current digit < top and we still have removals left:
        // Remove the top (it creates a larger number by staying)
        while (!st.empty() && k > 0 && st.top() > c) {
            st.pop();
            k--;
        }
        st.push(c);
    }
    
    // If k > 0: remove from the top (which is the largest remaining digits)
    while (k > 0) { st.pop(); k--; }
    
    // Build result from stack (stack is bottom-to-top = left-to-right of result)
    string result;
    while (!st.empty()) { result += st.top(); st.pop(); }
    reverse(result.begin(), result.end());
    
    // Remove leading zeros
    int start = 0;
    while (start < result.size() - 1 && result[start] == '0') start++;
    return result.substr(start);
}

// Pattern: Whenever you see "build optimal string/array by choosing whether to keep or remove"
// → Think monotonic stack with greedy pop condition
```

---

## THE VARIANTS — DEEP DIVE INTO EACH SUB-PATTERN

---

### VARIANT 1: Valid Parentheses Family — All Variations

**The base case:** Push open brackets. When you see a close bracket, check if top matches.

**Variations and their extensions:**

```cpp
// Variation 1: Count minimum removals to make valid (LC 921)
// At the end, stack has only unmatched '(' → they each need 1 removal
// During processing, unmatched ')' can't be pushed → count them separately
int minAddToMakeValid(string s) {
    int open = 0;    // count of unmatched '('
    int close = 0;   // count of unmatched ')'
    
    for (char c : s) {
        if (c == '(') {
            open++;
        } else {
            if (open > 0) open--;  // matched with existing '('
            else close++;          // unmatched ')'
        }
    }
    return open + close;
}

// Variation 2: Minimum insertions to make valid with rule "()" = valid, but also allows "())" (LC 1541)
// ...)... → more complex rules

// Variation 3: Valid parenthesis string where '*' can be '(', ')' or '' (LC 678)
// Greedy with two bounds (min and max possible open count):
bool checkValidString(string s) {
    int lo = 0, hi = 0;  // min and max possible open count
    for (char c : s) {
        if (c == '(') { lo++; hi++; }
        else if (c == ')') { lo--; hi--; }
        else { lo--; hi++; }  // '*': lo decreases (treat as ')'), hi increases (treat as '(')
        if (hi < 0) return false;  // impossible even with best choice
        lo = max(lo, 0);           // lo can't go below 0
    }
    return lo == 0;
}

// Variation 4: Longest valid parentheses substring (LC 32)
// Stack stores indices of unmatched brackets
int longestValidParentheses(string s) {
    stack<int> st;
    st.push(-1);  // sentinel: base for length computation
    int maxLen = 0;
    
    for (int i = 0; i < s.size(); i++) {
        if (s[i] == '(') {
            st.push(i);
        } else {
            st.pop();  // try to match with top
            if (st.empty()) {
                st.push(i);  // unmatched ')' becomes new base
            } else {
                maxLen = max(maxLen, i - st.top());  // length from base to current
            }
        }
    }
    return maxLen;
}
```

---

### VARIANT 2: Next Greater / Smaller — All Four + Span Problems

**The full framework:**

```cpp
// Online Stock Span (LC 901): classic span problem
// Span of day i = number of consecutive days up to and including i where price ≤ price[i]
class StockSpanner {
    stack<pair<int,int>> st;  // (price, span)
public:
    int next(int price) {
        int span = 1;
        // Merge spans of all days with price <= current
        while (!st.empty() && st.top().first <= price) {
            span += st.top().second;  // absorb their span
            st.pop();
        }
        st.push({price, span});
        return span;
    }
};
// WHY absorb spans: if days A, B, C all had price <= current day D's price,
// and B's span absorbed A, we don't need A anymore.
// B.span already includes the count for A.
// So we just add B.span to our span and pop B.

// Daily Temperatures (LC 739): next warmer day
vector<int> dailyTemperatures(vector<int>& T) {
    int n = T.size();
    vector<int> result(n, 0);
    stack<int> st;
    
    for (int i = 0; i < n; i++) {
        while (!st.empty() && T[i] > T[st.top()]) {
            int idx = st.top(); st.pop();
            result[idx] = i - idx;  // distance to next warmer day
        }
        st.push(i);
    }
    return result;
}

// Next Greater Element II - circular (LC 503)
// Process array TWICE to simulate circular behavior
vector<int> nextGreaterElementsCircular(vector<int>& nums) {
    int n = nums.size();
    vector<int> result(n, -1);
    stack<int> st;
    
    for (int i = 0; i < 2 * n; i++) {
        while (!st.empty() && nums[i % n] > nums[st.top()]) {
            result[st.top()] = nums[i % n];
            st.pop();
        }
        if (i < n) st.push(i);
    }
    return result;
}
```

---

### VARIANT 3: Largest Rectangle in Histogram — Deep Analysis

This is the most important monotonic stack problem for interviews. You must understand it deeply enough to explain the width calculation.

**The "left-right boundary" interpretation:**

```
For each bar at index i with height h[i]:
- The largest rectangle using h[i] as the HEIGHT extends from:
  - Left boundary: the first bar to the left that is SHORTER than h[i]
    (everything up to that point is ≥ h[i], so the rectangle fits)
  - Right boundary: the first bar to the right that is SHORTER than h[i]

- Width = right_boundary_index - left_boundary_index - 1
- Area = h[i] * width
```

**Why the stack approach works:**

```
The monotonic INCREASING stack maintains bars we haven't found a "right smaller" for.
When h[i] < h[stack.top()]:
  - h[i] is the RIGHT SMALLER ELEMENT for stack.top()
  - After popping stack.top(), the NEW top is the LEFT SMALLER ELEMENT for stack.top()
  - We now have BOTH boundaries → compute area → done with this bar

This is why it's O(n): each bar is pushed once and popped at most once.
```

**The two common approaches (know both):**

```cpp
// Approach 1: Sentinel at end (shown in Template 4)
// Add height 0 at end → flushes all remaining bars

// Approach 2: Precompute left[] and right[] arrays then one pass
int largestRectangleV2(vector<int>& heights) {
    int n = heights.size();
    vector<int> left(n), right(n);
    stack<int> st;
    
    // Previous smaller element (left boundary)
    for (int i = 0; i < n; i++) {
        while (!st.empty() && heights[st.top()] >= heights[i]) st.pop();
        left[i] = st.empty() ? 0 : st.top() + 1;
        st.push(i);
    }
    
    while (!st.empty()) st.pop();
    
    // Next smaller element (right boundary)
    for (int i = n - 1; i >= 0; i--) {
        while (!st.empty() && heights[st.top()] >= heights[i]) st.pop();
        right[i] = st.empty() ? n - 1 : st.top() - 1;
        st.push(i);
    }
    
    int maxArea = 0;
    for (int i = 0; i < n; i++) {
        maxArea = max(maxArea, heights[i] * (right[i] - left[i] + 1));
    }
    return maxArea;
}
```

---

### VARIANT 4: Maximal Rectangle — Histogram Per Row

```cpp
// Maximal Rectangle (LC 85) — build histogram row by row
// For each row, the "histogram heights" are the consecutive 1s above this row
// Then apply Largest Rectangle in Histogram on each row's histogram

int maximalRectangle(vector<vector<char>>& matrix) {
    if (matrix.empty()) return 0;
    int m = matrix.size(), n = matrix[0].size();
    vector<int> heights(n, 0);  // histogram heights
    int maxArea = 0;
    
    for (int row = 0; row < m; row++) {
        // Update histogram for this row
        for (int col = 0; col < n; col++) {
            heights[col] = (matrix[row][col] == '1') ? heights[col] + 1 : 0;
            // If '1': extend the bar upward
            // If '0': bar is broken, reset to 0
        }
        
        // Apply largest rectangle in histogram on current heights
        maxArea = max(maxArea, largestRectangleArea(heights));
    }
    
    return maxArea;
}

// Time: O(m * n) — each row takes O(n) for histogram update + O(n) for stack
// Space: O(n) — histogram array + stack
```

---

### VARIANT 5: Contribution Technique — Sum of Min/Max Over All Subarrays

**This is the most advanced sub-pattern. Most 1600-rated coders have never seen it.**

**The idea:**

Instead of iterating over all O(n²) subarrays:
```
sum(min of all subarrays) = Σ over all i: arr[i] × (# subarrays where arr[i] is minimum)
```

```
# subarrays where arr[i] is minimum = (# valid left extensions) × (# valid right extensions)
```

**Valid extensions:** arr[i] can be the minimum of a subarray as long as there is no smaller element within the subarray boundaries.

```cpp
// Sum of Subarray Minimums (LC 907) — FULL WORKED EXAMPLE
// arr = [3, 1, 2, 4]

// For i=0 (value=3):
//   Left: no element smaller to left → can extend left to index 0 → left[0] = 1
//   Right: next smaller element at index 1 (value=1) → right[0] = 1
//   Subarrays: 1 * 1 = 1 → subarray [3]
//   Contribution: 3 * 1 = 3

// For i=1 (value=1):
//   Left: no element smaller to left → left[1] = 2 (can include indices 0 and 1)
//   Right: no element smaller to right → right[1] = 3 (can include indices 1, 2, 3)
//   Subarrays: 2 * 3 = 6
//   Contribution: 1 * 6 = 6

// For i=2 (value=2):
//   Left: previous smaller at index 1 → left[2] = 1
//   Right: no element smaller to right → right[2] = 2
//   Subarrays: 1 * 2 = 2 → subarrays [2], [2,4]
//   Contribution: 2 * 2 = 4

// For i=3 (value=4):
//   Left: previous smaller at index 2 → left[3] = 1
//   Right: no element smaller → right[3] = 1
//   Subarrays: 1 * 1 = 1 → subarray [4]
//   Contribution: 4 * 1 = 4

// Total = 3 + 6 + 4 + 4 = 17
// Verify: min([3])=3, min([1])=1, min([2])=2, min([4])=4,
//         min([3,1])=1, min([1,2])=1, min([2,4])=2,
//         min([3,1,2])=1, min([1,2,4])=1, min([3,1,2,4])=1
// Sum = 3+1+2+4+1+1+2+1+1+1 = 17 ✓

// Sum of Subarray Ranges (LC 2104):
// = Sum of subarray maximums - Sum of subarray minimums
// Apply contribution technique twice: once for max, once for min
long long subArrayRanges(vector<int>& nums) {
    int n = nums.size();
    long long sumMax = 0, sumMin = 0;
    
    // Compute sum of subarray maximums
    {
        vector<int> left(n), right(n);
        stack<int> st;
        // Previous SMALLER for max: use PGE (previous greater)
        for (int i = 0; i < n; i++) {
            while (!st.empty() && nums[st.top()] <= nums[i]) st.pop();
            left[i] = st.empty() ? i + 1 : i - st.top();
            st.push(i);
        }
        while (!st.empty()) st.pop();
        for (int i = n-1; i >= 0; i--) {
            while (!st.empty() && nums[st.top()] < nums[i]) st.pop();
            right[i] = st.empty() ? n - i : st.top() - i;
            st.push(i);
        }
        for (int i = 0; i < n; i++) sumMax += (long long)nums[i] * left[i] * right[i];
    }
    
    // Compute sum of subarray minimums
    {
        vector<int> left(n), right(n);
        stack<int> st;
        for (int i = 0; i < n; i++) {
            while (!st.empty() && nums[st.top()] >= nums[i]) st.pop();
            left[i] = st.empty() ? i + 1 : i - st.top();
            st.push(i);
        }
        while (!st.empty()) st.pop();
        for (int i = n-1; i >= 0; i--) {
            while (!st.empty() && nums[st.top()] > nums[i]) st.pop();
            right[i] = st.empty() ? n - i : st.top() - i;
            st.push(i);
        }
        for (int i = 0; i < n; i++) sumMin += (long long)nums[i] * left[i] * right[i];
    }
    
    return sumMax - sumMin;
}
```

---

### VARIANT 6: Decode String — Stack for Nested Structures

```cpp
// Decode String (LC 394): "3[a2[c]]" → "accaccacc"
// Stack stores (current_string, repeat_count) at each nesting level

string decodeString(string s) {
    stack<pair<string, int>> st;  // (accumulated string, repeat count)
    string current = "";
    int k = 0;
    
    for (char c : s) {
        if (isdigit(c)) {
            k = k * 10 + (c - '0');  // build multi-digit number
        } else if (c == '[') {
            st.push({current, k});    // save current state
            current = "";             // reset for inner content
            k = 0;
        } else if (c == ']') {
            auto [prev, repeat] = st.top(); st.pop();
            string inner = current;
            current = prev;           // restore previous string
            for (int i = 0; i < repeat; i++) current += inner;  // repeat inner
        } else {
            current += c;
        }
    }
    
    return current;
}
// The stack represents the "nesting depth" — each '[ ' pushes the current
// context, and each ']' pops and combines.
```

---

### VARIANT 7: Iterative DFS Using Stack

```cpp
// Tree traversals iteratively using a stack
// Important for interview: "implement DFS without recursion"

// Preorder: Root → Left → Right
vector<int> preorderIterative(TreeNode* root) {
    if (!root) return {};
    vector<int> result;
    stack<TreeNode*> st;
    st.push(root);
    
    while (!st.empty()) {
        TreeNode* node = st.top(); st.pop();
        result.push_back(node->val);
        if (node->right) st.push(node->right);  // push RIGHT first
        if (node->left)  st.push(node->left);   // push LEFT second (processed first)
    }
    return result;
}

// Inorder: Left → Root → Right
vector<int> inorderIterative(TreeNode* root) {
    vector<int> result;
    stack<TreeNode*> st;
    TreeNode* curr = root;
    
    while (curr || !st.empty()) {
        while (curr) {            // go as far left as possible
            st.push(curr);
            curr = curr->left;
        }
        curr = st.top(); st.pop();
        result.push_back(curr->val);
        curr = curr->right;       // visit right subtree
    }
    return result;
}

// Postorder: Left → Right → Root
// Trick: reversed preorder (Root → Right → Left) → reverse result
vector<int> postorderIterative(TreeNode* root) {
    if (!root) return {};
    vector<int> result;
    stack<TreeNode*> st;
    st.push(root);
    
    while (!st.empty()) {
        TreeNode* node = st.top(); st.pop();
        result.push_back(node->val);
        if (node->left)  st.push(node->left);   // push LEFT first
        if (node->right) st.push(node->right);  // push RIGHT second (processed first)
    }
    
    reverse(result.begin(), result.end());  // Root→Right→Left reversed = Left→Right→Root
    return result;
}
```

---

## TIME AND SPACE COMPLEXITY — COLD RECITATION

| Operation | Time | Space | Why |
|-----------|------|-------|-----|
| Valid Parentheses | O(n) | O(n) | Each char pushed/popped at most once |
| Next Greater Element | O(n) | O(n) | Each element pushed and popped at most once |
| Daily Temperatures | O(n) | O(n) | Same as NGE |
| Largest Rectangle | O(n) | O(n) | Each bar pushed/popped at most once |
| Trapping Rain Water (stack) | O(n) | O(n) | Each element processed once |
| Sum Subarray Mins (contribution) | O(n) | O(n) | Two separate O(n) stack passes |
| Monotonic stack (all variants) | O(n) amortized | O(n) | Each element enters and exits stack once |
| Min Stack push/pop/getMin | O(1) | O(n) | All operations O(1) |
| Decode String | O(output size) | O(n) | Each char processed once |
| Evaluate RPN | O(n) | O(n) | Each token processed once |

### The O(n) amortized argument for ALL monotonic stack solutions:
"Each element is **pushed** to the stack exactly once when it is first processed. Each element is **popped** from the stack at most once — when a larger/smaller element causes it to be removed. Total pushes + total pops ≤ 2n. Therefore, the total work across all iterations of the inner while loop combined = O(n). The inner while loop is NOT O(n) per outer iteration — it's O(n) TOTAL."

This is the argument you state in an interview. Practice saying it out loud until it's fluent.

---

## COMMON MISTAKES AT THIS PATTERN (At YOUR 1600 Rating Level)

---

### Mistake 1: Using values instead of indices in the stack

**What they do:** Push `nums[i]` (the value) into the stack instead of `i` (the index).

**What goes wrong:** When computing width in histogram problems, you need the INDEX to compute `right - left - 1`. If you stored values, you lost the position information. You get wrong width → wrong area.

**The fix:** Always push INDICES into the monotonic stack. Access the value via `nums[st.top()]` when needed.

```cpp
// WRONG:
while (!st.empty() && heights[i] > st.top()) {
    int h = st.top(); st.pop();
    // width = ??? you can't compute this without positions
}

// CORRECT:
while (!st.empty() && heights[i] > heights[st.top()]) {
    int idx = st.top(); st.pop();
    int width = st.empty() ? i : i - st.top() - 1;  // can compute width
    maxArea = max(maxArea, heights[idx] * width);
}
```

---

### Mistake 2: Forgetting the sentinel in Largest Rectangle in Histogram

**What they do:** Process only the actual bars, not adding a zero-height bar at the end.

**What goes wrong:** Bars that are monotonically increasing (like [1, 2, 3, 4, 5]) never get popped — the loop ends with the stack still holding all elements. Their areas are never computed. Result: wrong answer (often 0 for these cases).

**The fix:** Iterate to `i <= n` (one past end) with `h = 0` at `i == n`. This flushes all remaining elements.

```cpp
// WRONG: misses bars remaining in stack
for (int i = 0; i < n; i++) { ... }

// CORRECT: sentinel at end flushes everything
for (int i = 0; i <= n; i++) {
    int h = (i == n) ? 0 : heights[i];  // sentinel height = 0
    ...
}
```

Alternatively: process the stack after the loop ends. Both approaches work.

---

### Mistake 3: Wrong duplicate handling in contribution technique

**What they do:** Use strict inequality on BOTH sides (left: strict <, right: strict <).

**What goes wrong:** Subarrays with duplicate minimums are counted TWICE. Example: arr = [1, 1], subarray [1, 1] has minimum 1. Both indices claim to be the minimum and both count this subarray → result is 2 + 2 = 4 but correct answer is 2.

**The fix:** Use STRICT on left, NON-STRICT on right (or vice versa). This gives each subarray exactly ONE "representative minimum."

```cpp
// For sum of subarray MINIMUMS:
// Left boundary: strictly less than (breaks ties by giving ownership to the RIGHTMOST duplicate)
while (!st.empty() && arr[st.top()] >= arr[i]) st.pop();  // >= means strict on left

// Right boundary: strictly greater than (only pop if strictly greater)
while (!st.empty() && arr[st.top()] > arr[i]) st.pop();   // > means non-strict on right

// For sum of subarray MAXIMUMS: reverse the strict/non-strict
```

---

### Mistake 4: Off-by-one in width computation for Largest Rectangle

**What they do:** Compute `width = i - st.top()` instead of `i - st.top() - 1`.

**What goes wrong:** The width calculation is off by 1. For a bar popped at index `top`, the left boundary is `st.top()` (the new top, which is the first smaller element to the left). The right boundary is `i` (the first smaller element to the right). The rectangle spans from `st.top() + 1` to `i - 1`, which is `i - st.top() - 1` columns wide.

**The fix:** Memorize the formula. `width = i - st.top() - 1` when stack is not empty. `width = i` when stack is empty (bar extends to the very beginning).

```cpp
int width = st.empty() ? i : (i - st.top() - 1);
// +0 would be wrong: the boundaries at st.top() and i are NOT part of the rectangle
```

---

### Mistake 5: Wrong pop condition for min vs max

**What they do:** Use `>` when they need `>=` or vice versa, causing the stack to not maintain strict monotonicity with duplicates.

**What goes wrong:** 
- For NGE (next greater): `while nums[i] > nums[st.top()]` — strict. Non-strict would find "greater or equal" which is a different problem.
- For NSE (next smaller): `while nums[i] < nums[st.top()]` — strict.
- For contribution: `>=` on one side ensures correct duplicate handling.

**The fix:** Before writing the pop condition, ask: "Am I looking for strictly greater, or greater-or-equal?" Choose the operator accordingly. For contribution technique, always use the asymmetric strict/non-strict convention.

---

## THE PROBLEM SET — SOLVE IN THIS EXACT ORDER

---

### WARMUP — 2 problems (solve in under 15 minutes each)

**1. Valid Parentheses — LC 20 — [Basic stack matching]**

Why this problem: The simplest possible stack application. Trains the open→push, close→pop-and-verify reflex. Must be solvable in 3 minutes.

What to prove: You handle all three bracket types correctly. You return `false` when the stack is empty on a close bracket (nothing to match). You return `false` if the stack is NOT empty at the end (unmatched opens remain).

Target time: 3 minutes.

Edge cases: empty string (true), single bracket (false), `"(]"` (false — type mismatch), `"([)]"` (false — interleaved).

**2. Min Stack — LC 155 — [Auxiliary stack tracking running minimum]**

Why this problem: Trains the "auxiliary stack for tracking aggregate" thinking. This exact pattern (maintaining an extra stack for some property) appears in many more complex problems.

What to prove: You know that auxiliary min-stack only needs to push when the new value is ≤ current minimum (not just <). Equality matters when there are duplicates.

Target time: 10 minutes.

---

### CORE — 4 problems (the real learning happens here)

**3. Next Greater Element I — LC 496 — [Monotonic stack, first exposure]**

What this tests: The first exposure to the monotonic stack. You precompute NGE for one array and look up answers for another.

Key insight: Process the `nums2` array with a monotonic stack to build a mapping: `next_greater[value] = next_greater_value`. Then answer queries for `nums1` using this mapping.

The thing to practice: write the NGE template cleanly from memory without hesitation. This is the template that everything else builds on.

**4. Daily Temperatures — LC 739 — [Monotonic decreasing stack, compute distances]**

What this tests: NGE applied to real-world problem. The "pop and compute distance" pattern where the answer is `i - popped_index`.

Key insight: Store indices (not values) so you can compute the distance. When you pop index `j` because `T[i] > T[j]`, the answer for day `j` is `i - j`.

Common mistake: pushing values instead of indices → can't compute distance.

**5. Evaluate Reverse Polish Notation — LC 150 — [Stack-based evaluation]**

What this tests: Stack as a computation device. Operands go on the stack, operators pop two and push one.

Key insight: This is NOT a monotonic stack. It's a pure stack. The test is whether you can handle the "pop TWO, compute, push ONE" cycle cleanly. And getting division right: `a / b` where `a = st.top()` AFTER popping `b`. Pop order matters — `b` is popped first (it was pushed last), then `a`.

```cpp
long long b = st.top(); st.pop();  // SECOND operand
long long a = st.top(); st.pop();  // FIRST operand
// a op b, NOT b op a
```

**6. Largest Rectangle in Histogram — LC 84 — [Monotonic increasing stack, THE hard stack problem]**

What this tests: The full power of monotonic stack for histogram problems. This is THE interview-level hard stack problem. Every senior engineer knows it.

Key insight: Pop when current bar is shorter than top. At pop time, the popped bar's right boundary is current index, left boundary is new top. Width formula: `i - st.top() - 1` (or `i` if stack is empty).

Time limit: 40 minutes. If you can't solve this cleanly in 40 minutes, re-study Template 4 and trace through the example manually step by step.

Why this matters: It directly leads to Maximal Rectangle (LC 85) which is asked at Google and Amazon. Solve this first.

---

### STRETCH — 3 problems

**7. Trapping Rain Water — Stack Approach — LC 42**

What makes this hard: There are THREE different approaches to this problem (two-pointer, prefix arrays, stack). The stack approach processes water "layer by layer" rather than "column by column." Understanding all three approaches shows deep understanding.

The stack approach insight: maintains a decreasing stack. When a taller bar arrives, it creates a "right wall." The top of the stack is the "valley floor," and the new top is the "left wall." Compute the water in this horizontal layer.

Note: You already know the two-pointer solution from Pattern 02. Compare the two approaches. Know both for interviews.

**8. Sum of Subarray Minimums — LC 907 — [Contribution technique]**

What makes this hard: Requires seeing the contribution technique — the insight that you should compute "for each element, how many subarrays have it as the minimum" instead of "for each subarray, what is the minimum."

This insight is what separates 1800+ from 1600 coders. It appears in contest problems regularly.

Time limit: 45 minutes for the insight to arrive. If it doesn't, the hint is: "Think about how many subarrays have nums[i] as the minimum. How can you count that efficiently?"

**9. Online Stock Span — LC 901 — [Span computation + absorbing accumulated spans]**

What makes this hard: The span compression — instead of storing individual elements, you store (price, span) pairs and accumulate spans when popping. This is an optimization that requires understanding WHY it works.

Key insight: If days A, B, C all have price ≤ current day D, and B already absorbed A's span, you only need to store B (with its accumulated span). When D absorbs B, it gets B's full span (which includes A's). This is lazy aggregation.

---

### CONTEST LEVEL — 2 problems

**10. Maximal Rectangle — LC 85 — [Histogram per row + Largest Rectangle]**

Why this is contest level: Combines the histogram technique with dynamic histogram building row by row. The insight that "build a histogram of consecutive 1s above each row" reduces a 2D problem to a sequence of 1D problems.

Key insight: For each row, update heights: if `matrix[row][col] == '1'`, `heights[col]++`; else `heights[col] = 0`. Then apply Largest Rectangle in Histogram. If you can't solve Largest Rectangle (problem 6), you cannot solve this. Do problem 6 first.

**11. Sum of Subarray Ranges — LC 2104 — [Contribution technique for both max and min]**

Why this is contest level: Range of a subarray = max - min. So sum of ranges = sum of maximums - sum of minimums. Requires applying contribution technique TWICE (once for max, once for min) and being careful about duplicate handling in each case (strict/non-strict on different sides).

Key insight: The duplicate handling convention differs between computing sum-of-max and sum-of-min. Make sure you're consistent.

---

## PATTERN CONNECTIONS

### How Stack and Monotonic Stack connects to other patterns:

**Leads directly to Monotonic Deque (Pattern 29):**
Monotonic stack handles "within the entire array seen so far." Monotonic deque handles "within a sliding window of size K." The deque adds the ability to pop from the FRONT (old elements leaving the window). If you understand monotonic stack, monotonic deque is a small extension.

**Combines with Sliding Window (Pattern 03):**
The sliding window maximum problem (LC 239) is solved by a monotonic deque. Understanding monotonic stack first makes the deque intuition natural.

**Foundation for Segment Tree / Sparse Table:**
Range minimum/maximum queries can be solved with O(1) after O(n log n) preprocessing using sparse tables. But for "nearest smaller element" specifically, monotonic stack is both simpler and faster.

**Often confused with:**
- **Two Pointers (Trapping Rain Water):** The same problem has two solutions. Two pointers is O(1) space. Stack is O(n) space but more "template-derivable." Know both; use two pointers in interviews for extra credit.
- **Monotonic Deque (Pattern 29):** Same invariant (monotonic order), different data structure. Stack = fixed boundary problems. Deque = sliding window problems.

---

## INTERVIEW SIMULATION QUESTIONS

---

### Q1: "Given an array, find the number of days you have to wait for a warmer temperature. Return 0 if there is no future warmer day."

**Company type:** Amazon, Microsoft, Google (LC 739 — very common)
**Expected time:** 15 minutes
**What they are testing:** Monotonic stack + compute index difference

**Red flags that get candidates rejected:**
- O(n²) nested loop brute force (acceptable only as a first pass, must then optimize)
- Pushing values instead of indices (can't compute distances)
- Not initializing result to 0 (the default answer for "no warmer day")
- Confusion about which direction the stack is monotonic

**The follow-up they WILL ask:** "What if temperatures can repeat (duplicates)?" → Same code handles it. Strict `>` means equal temperatures DON'T trigger a pop → correct.

**The follow-up that distinguishes candidates:** "Walk me through what the stack contains at each step for input [73, 74, 75, 71, 69, 72, 76, 73]."

```
i=0: push 0. stack=[0]
i=1: 74>73 → pop 0, result[0]=1. push 1. stack=[1]
i=2: 75>74 → pop 1, result[1]=1. push 2. stack=[2]
i=3: 71<75 → push 3. stack=[2,3]
i=4: 69<71 → push 4. stack=[2,3,4]
i=5: 72>69 → pop 4, result[4]=1. 72>71 → pop 3, result[3]=2. push 5. stack=[2,5]
i=6: 76>72 → pop 5, result[5]=1. 76>75 → pop 2, result[2]=4. push 6. stack=[6]
i=7: 73<76 → push 7. stack=[6,7]
End: result=[1,1,4,2,1,1,0,0]
```

---

### Q2: "Given a heights array representing a histogram, find the area of the largest rectangle."

**Company type:** FAANG (Google, Amazon, Microsoft) — direct interview question
**Expected time:** 30 minutes
**What they are testing:** Full monotonic stack for histogram — the HARD stack problem

**Red flags:**
- O(n²) solution with two pointer scan for each bar (shows lack of pattern knowledge)
- Computing incorrect width (off-by-one)
- Forgetting sentinel (missing bars at the end of processing)
- Can't explain WHY the popped bar's right boundary is the current index

**What they want you to say when explaining:**
"When I encounter a bar shorter than the top of the stack, the top bar found its right boundary. The element below it in the stack (the new top after popping) is its left boundary. I compute width as rightIndex - leftIndex - 1, and area as height × width."

**The follow-up:** "What if the histogram has equal-height adjacent bars?" → Handle by treating equal-height bars with non-strict comparison OR strict comparison consistently. The result is the same as long as you're consistent. Walk through an example with [2, 2] to show it works.

---

### Q3: "Design a stack that supports push, pop, top, and also returns the minimum element in O(1) time."

**Company type:** Startups, mid-stage companies, Amazon (system design introduction)
**Expected time:** 10 minutes
**What they are testing:** Auxiliary stack thinking

**Red flags:**
- Sorting after every push (O(n log n) per push)
- Scanning the entire stack on `getMin()` call (O(n) per query)
- Only tracking one global min without handling when that min is popped

**The follow-up:** "What if you want O(1) space overhead (not O(n) for the auxiliary stack)?"
→ Use the "encode in single stack" trick: push `(value, current_min)` pairs. Or push the value and the difference from current min (advanced trick to avoid storing min separately). Discuss tradeoffs.

**Second follow-up:** "Can you implement a Max Stack with O(log n) operations that also supports `popMax()` — pop the maximum element?"
→ This requires a sorted set (balanced BST) + doubly linked list. See LC 716. This is the advanced version that only L5+ candidates would be expected to solve.

---

## MENTAL MODEL CHECKPOINT

**Answer all 7 from memory before attempting sign-off.**

1. **What is the invariant of a monotonic increasing stack?**
   → At all times, elements in the stack are in increasing order from bottom to top. When a new element is pushed, all elements ≥ new element are first popped (to maintain the invariant). The popping tells you the new element is the next smaller element for whatever got popped.

2. **What are the two signals that tell you to use a monotonic stack instead of a regular stack?**
   → (a) The problem asks "for each element, find the nearest element that is larger/smaller to the left/right." (b) The problem involves a histogram and asks for maximum rectangle or trapped area — these require knowing the nearest shorter bar on each side.

3. **What is the time complexity of the monotonic stack approach and WHY?**
   → O(n) amortized. Each element is pushed exactly once and popped at most once. Total operations ≤ 2n regardless of how many times the inner while loop runs. The while loop does O(n) total work, not O(n) per outer iteration.

4. **What is the most common mistake in Largest Rectangle in Histogram?**
   → Forgetting the sentinel (height = 0 at end), which causes bars that are never "resolved" by a shorter bar to the right to be skipped. Also: computing `i - st.top()` instead of `i - st.top() - 1` for width.

5. **What is the contribution technique and when do you use it?**
   → Instead of computing min/max for each subarray and summing, compute for each element "how many subarrays have this element as their min/max" and multiply by the element value. Use when the problem asks for sum of min/max over ALL subarrays. Use monotonic stack to find left/right boundaries in O(n).

6. **When using the contribution technique with duplicates, what convention do you use to avoid double-counting?**
   → Strict inequality on one side, non-strict on the other. Convention: left boundary uses strict < (previous strictly smaller), right boundary uses non-strict ≤ (next smaller or equal). This gives each subarray exactly one "representative minimum."

7. **For sum of subarray ranges (max - min), what two problems do you reduce it to?**
   → (1) Sum of subarray maximums + (2) Sum of subarray minimums (negated). Apply the contribution technique separately for max (using decreasing stack) and min (using increasing stack). Answer = sumMax - sumMin.

---

## C++ STL CHEAT SHEET FOR THIS PATTERN

```cpp
// ═══════════════════════════════════════════
// stack<T> operations:
// ═══════════════════════════════════════════
stack<int> st;

st.push(x);         // push to top — O(1)
st.pop();           // remove top — O(1), DOES NOT RETURN VALUE
st.top();           // peek top — O(1), does NOT remove
st.empty();         // true if empty — O(1)
st.size();          // number of elements — O(1)

// CRITICAL: st.pop() returns void. To get value + remove:
int val = st.top(); st.pop();  // always do this in two steps

// ═══════════════════════════════════════════
// Common monotonic stack patterns:
// ═══════════════════════════════════════════

// NGE (monotonic decreasing): pop when new element is LARGER
while (!st.empty() && nums[i] > nums[st.top()]) {
    // st.top() found its NGE = nums[i]
    st.pop();
}

// NSE (monotonic increasing): pop when new element is SMALLER
while (!st.empty() && nums[i] < nums[st.top()]) {
    // st.top() found its NSE = nums[i]
    st.pop();
}

// Histogram (increasing stack, pop when shorter):
while (!st.empty() && heights[i] < heights[st.top()]) {
    int h = heights[st.top()]; st.pop();
    int w = st.empty() ? i : i - st.top() - 1;
    maxArea = max(maxArea, h * w);
}

// ═══════════════════════════════════════════
// Integer overflow watch:
// ═══════════════════════════════════════════
// Sum of subarray mins: can overflow int when n = 3*10^4, values up to 3*10^4
// max sum ≈ n² * maxVal ≈ 9*10^8 * 3*10^4 → overflow!
// Always use long long for contribution technique sums.
long long result = 0;
result = (result + (long long)arr[i] * left[i] * right[i]) % MOD;

// ═══════════════════════════════════════════
// String-to-integer conversion (for expression evaluation):
// ═══════════════════════════════════════════
stoi(str)       // string to int
stoll(str)      // string to long long
to_string(val)  // int/long to string

// ═══════════════════════════════════════════
// Pair in stack (for decode string / stock span):
// ═══════════════════════════════════════════
stack<pair<int,int>> st;
st.push({value, span});
auto [v, s] = st.top(); st.pop();  // C++17 structured bindings
```

---

## WHAT TO DO AFTER COMPLETING THIS PATTERN

1. **Solve all 11 problems in order.** For problem 6 (Largest Rectangle), trace through the algorithm manually on paper first.
2. **Write the four boundary templates (PSE, NSE, PGE, NGE) from memory** until you can produce all four correctly in 5 minutes total.
3. **For LC 907 (Sum of Subarray Mins):** Implement it twice — once with the left[]/right[] precomputation approach, once with the inline single-pass approach. Understand why both give the same answer.
4. **Revision schedule:** Redo problems 6, 8, 10 after 3 days, 7 days, and 14 days. These are CHALLENGE problems.
5. **When you can solve LC 84 (Largest Rectangle) in under 15 minutes cold** — that's when you own this pattern.

---

## THE SIGN-OFF CRITERIA

When you type `SIGN OFF P04`, I will give you one unseen problem. You must:
1. Identify whether it needs basic stack, monotonic stack, or contribution technique
2. Name the specific sub-variant (NGE / NSE / histogram / contribution / matching)
3. Write correct C++ solution with indices stored in stack (not values)
4. State time/space complexity with the amortized O(n) justification
5. Handle edge cases: empty array, single element, all equal elements, all increasing, all decreasing

---

*Now go solve. Start with LC 20 (Valid Parentheses) and LC 155 (Min Stack). Then go directly to LC 739 (Daily Temperatures) — that's your first exposure to the core monotonic stack template. When you reach LC 84 (Largest Rectangle in Histogram) — take your time, trace it manually, and get it right. That problem alone is worth more than 5 easy problems. Paste solutions with `SOLVE` for 5-dimension review.*
