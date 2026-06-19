# Pattern 11 — Intervals
## Difficulty Level: CORE
## Interview Frequency: OFTEN
## Estimated time to master: 3 days
## Problems in this set: 11 problems

---

## THE GRANDMASTER'S HONEST TAKE

Intervals are deceptively simple. Everyone knows what an interval is. The trap is that there are five completely different interval problem types, each requiring a different approach, and they are easy to confuse with each other under interview pressure.

**The brutal truth: at your 1600 level, you can probably do Merge Intervals (LC 56). You cannot do Insert Interval (LC 57) cleanly in under 15 minutes, and you almost certainly cannot explain WHY the greedy for Non-overlapping Intervals sorts by END time and not start time. You have never written a line sweep from scratch.**

The gap between "I can merge intervals" and "I understand all five interval sub-patterns" is significant — and interviewers know exactly where the gap is. The three most commonly asked interview questions in this pattern (Merge, Meeting Rooms II, Non-overlapping) all look similar superficially but require entirely different techniques. Candidates who confuse them fail.

**After this document, you will:**
- Know the five interval sub-patterns and which technique applies to each — no more mixing them up
- Write the 3-phase Insert Interval template cleanly (Phase 1: no overlap before, Phase 2: merge overlapping, Phase 3: no overlap after)
- Explain the greedy proof for activity selection (sort by end, not start) in 30 seconds
- Know BOTH approaches for Meeting Rooms II: min-heap of end times AND line sweep — and know which is faster to code
- Implement line sweep with correct event-pair tie-breaking
- Write the two-pointer interval intersection without confusion

**The interview data:**
- Merge Intervals (LC 56) is asked as a warm-up at nearly every company — you must do it in under 5 minutes cold
- Meeting Rooms II (LC 253) is in the top 20 most asked problems at Amazon and Microsoft
- Non-overlapping Intervals (LC 435) appears in 15%+ of Google and Meta rounds as a "greedy" question
- Insert Interval (LC 57) is Microsoft's favorite "medium" — the 3-phase structure is the whole test
- Line sweep (My Calendar III, LC 732) appears at Google and Bloomberg for senior candidates

---

## ELI5 — THE CALENDAR METAPHORS

### Intervals as Calendar Blocks

An interval is just a time slot on a calendar. `[9, 11]` means "occupied from 9 AM to 11 AM." Everything in interval problems is about managing these time slots.

**Merge Intervals:** You have a messy calendar with overlapping appointments. A scheduling system should consolidate overlapping appointments into single blocks. `[9,11]` and `[10,13]` → `[9,13]` (one continuous block). That is merge intervals.

**Insert Interval:** Your sorted, clean calendar has no overlaps. You add a new appointment. It might overlap zero, one, or many existing ones. The question: what does the calendar look like after adding and merging the new appointment? Three phases: appointments before the new one (keep as-is), appointments overlapping the new one (merge all into one), appointments after the new one (keep as-is).

**Non-overlapping / Activity Selection:** You have a list of party invitations, each with a start and end time. You can only be at one party at a time. Maximize how many parties you attend. Greedy insight: always choose the party that ends soonest. Why? By leaving a party early, you maximize your remaining free time for future parties. The party that ends latest blocks the most future options — always reject it first when there's a conflict.

**Meeting Rooms II:** You have a meeting schedule and must assign rooms. A room becomes free when a meeting in it ends. At any moment, the number of rooms in use = number of meetings currently in progress. You need the peak. Line sweep or min-heap tell you this peak without simulating every moment.

### The Line Sweep Visualization

Imagine a vertical line sweeping from left to right across a timeline. Every time it crosses a meeting's START, a counter goes up by 1 (one more room needed). Every time it crosses a meeting's END, the counter goes down by 1 (one room freed). The counter's maximum value across the entire sweep is the number of rooms needed.

In code: convert each interval `[s, e]` into two events: `(s, +1)` and `(e, -1)`. Sort all events by time. Process them in order, maintaining a running sum. The maximum running sum is your answer.

**The tie-breaking rule matters:** If a meeting ends at time 10 and another starts at time 10, can the same room be used? In most LeetCode problems: YES (end is exclusive or "back-to-back is OK"). So process end events (-1) before start events (+1) at the same time. This happens automatically when you sort pairs because `(10, -1) < (10, +1)` since `-1 < 1`.

### Why Sort by END for Activity Selection

You have events:
- Event A: [1, 10] (runs all day)
- Event B: [2, 4] (morning slot)
- Event C: [5, 7] (afternoon slot)
- Event D: [8, 11] (evening slot)

Sort by END: B(ends 4), C(ends 7), A(ends 10), D(ends 11).
Greedy: pick B (ends 4). Next non-conflicting: C (starts 5 ≥ 4) → pick. Next: A (starts 1, conflicts with C ending 7) → skip. D (starts 8 ≥ 7) → pick. Total: 3 events.

If you sorted by START instead: A(starts 1), B(starts 2), C(starts 5), D(starts 8).
Greedy: pick A (starts 1). Next non-conflicting with A(ends 10): D (starts 8, but conflicts with A) → skip. Then after A, pick D. Total: 2 events.

Sorting by start is WRONG for maximizing count. The intuition: the event with the earliest end leaves the most runway for future events. The event that starts earliest might block the entire day.

---

## THE PATTERN — FORMALLY DEFINED

### The Five Interval Sub-Patterns:

**Sub-pattern 1: Merge Overlapping Intervals**
Input: unordered set of potentially overlapping intervals.
Goal: produce a set of non-overlapping intervals that covers the same range.
Technique: sort by start time, then sweep — extend the last merged interval's end if overlap.

**Sub-pattern 2: Insert Interval into a Sorted Non-overlapping List**
Input: a sorted, non-overlapping interval list + one new interval.
Goal: insert and merge to produce a new sorted, non-overlapping list.
Technique: 3-phase sweep — skip intervals ending before new starts, merge all overlapping, append remaining.

**Sub-pattern 3: Activity Selection / Non-overlapping Count**
Input: intervals, each with some "value" (usually we want to COUNT maximum non-overlapping or REMOVE minimum to make non-overlapping).
Goal: maximize the count of non-overlapping intervals kept (equivalently: minimize removals).
Technique: sort by END time, greedy selection.

**Sub-pattern 4: Resource Counting (Meeting Rooms)**
Input: intervals representing resource usage.
Goal: find peak simultaneous usage (minimum rooms, platforms, servers needed).
Technique A: min-heap of end times — O(n log n), practical for interviews.
Technique B: line sweep — O(n log n), more versatile, handles continuous time correctly.

**Sub-pattern 5: Interval Intersection / Two-List Operations**
Input: two sorted, non-overlapping interval lists.
Goal: find all pairwise intersections.
Technique: two pointers — always advance the pointer with the earlier-ending interval.

### The Overlap Condition (State This Precisely):

Two intervals `[a, b]` and `[c, d]` OVERLAP (touching counts as overlap) when:
$$\max(a, c) \leq \min(b, d)$$

Equivalently, they DO NOT overlap when: `b < c` OR `d < a`.

For "touching is overlap": use `<` as the non-overlap check (`a.end < b.start`).
For "touching is not overlap": use `<=` as the non-overlap check.
LeetCode interval problems almost always treat touching endpoints as overlapping.

---

## WHEN TO RECOGNIZE THIS PATTERN

**Signal 1:** "Merge all overlapping intervals" or "consolidate overlapping events" or "return a list with no overlapping ranges"
→ Sort by start time, then sweep with a single merge condition. The canonical merge intervals template. Sort once, one pass. O(n log n). This is the most asked interval problem — must be under 5 minutes.

**Signal 2:** "Insert a new interval into a sorted non-overlapping list" or "add an interval and return the updated non-overlapping list"
→ The 3-phase template. Do NOT re-sort the entire list (that would be O(n log n) and misses the point that the list is already sorted). Three pointer passes: skip non-conflicting intervals before the new one, merge all conflicting, append remaining. O(n).

**Signal 3:** "Minimum number of intervals to remove to make the rest non-overlapping" or "maximum number of events you can attend" or "minimum arrows to burst all balloons" (where each balloon is an interval)
→ Sort by END time, greedy selection. This is the activity selection problem. After sorting by end, always keep the current event and skip any future event that starts before the current end. O(n log n) for sort + O(n) for greedy sweep.

**Signal 4:** "Minimum number of meeting rooms" or "minimum platforms at a train station" or "maximum simultaneous calls at any moment"
→ Peak resource usage. Two approaches: (A) Sort by start, maintain a min-heap of end times — the heap size at the end is the answer. (B) Line sweep — create (time, ±1) event pairs, sort, sweep to find maximum running sum. Both are O(n log n). Use the heap approach when you need to know WHICH resource is freed; use line sweep when you only need the peak count.

**Signal 5:** "Find all intervals that are present in both list A and list B" or "intersection of two sorted interval lists"
→ Two pointers. Two indices, one per list. Compute the intersection of the current pair. Advance the pointer whose current interval ends sooner (the other list's current interval might intersect the NEXT interval in this list, but this interval is done).

**Signal 6:** "Count the number of ongoing events at any time" or "track total covered length" or "find maximum concurrent bookings"
→ Line sweep with a difference array or sorted event map. Create events at each start (+1) and end (-1), sort them, sweep to find maximums or totals.

**Signal 7:** "Design a calendar class that supports adding events and querying overlaps"
→ Sorted set (C++ `std::map` or `std::set`) with binary search for overlap detection. Insert the new interval, find the neighbors in sorted order, check if they overlap. O(log n) per insertion using lower_bound.

**Signal 8:** "Given a stream of intervals, maintain a sorted set of non-overlapping intervals"
→ Online merge with sorted map. For each new interval, find all existing intervals that overlap with it (using lower_bound), merge them all into one, update the map. Amortized O(log n) per interval if each interval is inserted and removed at most once.

### Signals that LOOK like Intervals but are NOT:

**Fake signal 1:** "Find if any element appears in both array A and array B"
→ Hash set, not intervals. The "range" aspect is about the element values, not positions that can be merged.

**Fake signal 2:** "Sum of all elements in range [l, r] for multiple queries"
→ Prefix sums (Pattern 06), not interval merging. The intervals here are query ranges, not intervals to be merged.

**Fake signal 3:** "Rotate an array by K positions"
→ Two pointers or reversal (Pattern 02). This looks like it involves ranges but is fundamentally about array manipulation.

---

## THE TEMPLATES (C++)

### Template 1: Merge Intervals — Sort by Start + Sweep

```cpp
// LC 56: Merge Intervals
// Prerequisite: sort by start time. Then one forward pass.
// Overlap check: result.back()[1] >= next[0] → they overlap, extend the end.

vector<vector<int>> merge(vector<vector<int>>& intervals) {
    // Sort by start time (default vector comparison is lexicographic: compares [0] first)
    sort(intervals.begin(), intervals.end());
    
    vector<vector<int>> result;
    
    for (auto& interval : intervals) {
        if (result.empty() || result.back()[1] < interval[0]) {
            // No overlap: current interval starts strictly AFTER last merged interval ends
            result.push_back(interval);
        } else {
            // Overlap (touching included): extend the end if needed
            result.back()[1] = max(result.back()[1], interval[1]);
        }
    }
    
    return result;
}

// THE OVERLAP CONDITION:
// result.back()[1] >= interval[0] → overlap (they share at least one point)
// result.back()[1] < interval[0]  → no overlap (gap between them)
//
// Using `<` for non-overlap: [1,3] and [3,5] are merged → result is [1,5]
// (touching counts as overlapping — standard LeetCode behavior)
//
// WHY sort by start, not end?
// If we sort by end: [1,4] and [2,3] → [2,3] comes first, but [1,4] CONTAINS [2,3].
// Processing [2,3] first, then [1,4]: last merged end = 3, but [1,4] starts at 1 (before 3)
// and ends at 4 (after 3) → we'd miss the merge. Sort by start ensures we never encounter
// an interval whose start is BEFORE the current merged interval's start.
//
// TRACE for [[1,3],[2,6],[8,10],[15,18]]:
// After sort: [[1,3],[2,6],[8,10],[15,18]] (already sorted)
// Process [1,3]: result=[[1,3]]
// Process [2,6]: 3 >= 2 → overlap. back[1] = max(3,6)=6. result=[[1,6]]
// Process [8,10]: 6 < 8 → no overlap. result=[[1,6],[8,10]]
// Process [15,18]: 10 < 15 → no overlap. result=[[1,6],[8,10],[15,18]]
// Output: [[1,6],[8,10],[15,18]] ✓

// HOW TO ADAPT for "total covered length":
// After merging, sum up (interval[1] - interval[0]) for each merged interval.
```

---

### Template 2: Insert Interval — 3-Phase Sweep

```cpp
// LC 57: Insert Interval (sorted non-overlapping list, insert + merge)
// NO re-sorting: the list is already sorted. Use a 3-phase O(n) pass.
//
// Phase 1: Collect all intervals that FULLY PRECEDE the new interval (no overlap)
//          Condition: intervals[i][1] < newInterval[0] (current ends BEFORE new starts)
// Phase 2: Merge all intervals that OVERLAP with the new interval
//          Condition: intervals[i][0] <= newInterval[1] (current starts BEFORE new ends)
//          Action: extend newInterval to cover all overlapping intervals
// Phase 3: Collect all remaining intervals that FULLY FOLLOW the new interval

vector<vector<int>> insert(vector<vector<int>>& intervals, vector<int> newInterval) {
    vector<vector<int>> result;
    int i = 0, n = intervals.size();
    
    // Phase 1: Intervals ending strictly before the new one starts
    while (i < n && intervals[i][1] < newInterval[0]) {
        result.push_back(intervals[i++]);
    }
    
    // Phase 2: Merge all intervals overlapping with newInterval
    while (i < n && intervals[i][0] <= newInterval[1]) {
        newInterval[0] = min(newInterval[0], intervals[i][0]);
        newInterval[1] = max(newInterval[1], intervals[i][1]);
        i++;
    }
    result.push_back(newInterval);  // push the merged interval
    
    // Phase 3: Intervals starting strictly after the new one ends
    while (i < n) result.push_back(intervals[i++]);
    
    return result;
}

// WHY Phase 1 condition is `intervals[i][1] < newInterval[0]`:
// We want intervals that END strictly before the new interval STARTS.
// "Strictly" because touching endpoints (e.g., interval ends at 5, new starts at 5)
// should be merged in LeetCode problems. Touching → overlap → handled in Phase 2.
//
// WHY Phase 2 condition is `intervals[i][0] <= newInterval[1]`:
// An interval overlaps with newInterval if it STARTS at or before newInterval ends.
// `<=` includes the touching case (current starts at exactly newInterval[1] → merge them).
//
// TRACE for intervals=[[1,2],[3,5],[6,7],[8,10],[12,16]], newInterval=[4,8]:
// Phase 1: intervals[0][1]=2 < 4 → push [1,2]. i=1.
//          intervals[1][1]=5 >= 4 → stop Phase 1. i=1.
// Phase 2: intervals[1][0]=3 <= 8 → merge. new=[min(4,3),max(8,5)]=[3,8]. i=2.
//          intervals[2][0]=6 <= 8 → merge. new=[min(3,6),max(8,7)]=[3,8]. i=3.
//          intervals[3][0]=8 <= 8 → merge. new=[min(3,8),max(8,10)]=[3,10]. i=4.
//          intervals[4][0]=12 > 8 → stop Phase 2.
//          Push [3,10].
// Phase 3: Push [12,16].
// Output: [[1,2],[3,10],[12,16]] ✓

// EDGE CASES the 3-phase handles automatically:
// - newInterval doesn't overlap anything: Phase 2 loop doesn't execute.
//   The newInterval is inserted between Phase 1 and Phase 3 results. ✓
// - newInterval overlaps all: Phase 1 collects nothing, Phase 2 merges everything,
//   Phase 3 collects nothing. Output: [newInterval]. ✓
// - Empty intervals list: all three phases produce nothing. Output: [newInterval]. ✓
```

---

### Template 3: Non-overlapping Intervals — Greedy Sort by End

```cpp
// LC 435: Non-overlapping Intervals — minimum number of removals to make non-overlapping
// Equivalently: maximum number of non-overlapping intervals to KEEP, then remove the rest.
//
// GREEDY INSIGHT: sort by END time. Always keep the interval with the earliest end.
// Why? The interval ending soonest leaves the maximum remaining timeline for future intervals.
// Any interval ending later blocks more future intervals — it is the first to be removed
// when a conflict arises.
//
// PROOF SKETCH (exchange argument):
// Suppose we have an optimal solution that does NOT include interval X (ends at e_X)
// but includes interval Y (ends at e_Y > e_X) where X and Y start at the same time.
// Swap Y with X in the solution: X doesn't conflict with anything Y didn't,
// AND X ends earlier, potentially fitting one MORE interval after it. ✓
// Therefore, always preferring the earlier-ending interval cannot be suboptimal.

int eraseOverlapIntervals(vector<vector<int>>& intervals) {
    // Sort by END time (not start time — this is the key)
    sort(intervals.begin(), intervals.end(), [](auto& a, auto& b) {
        return a[1] < b[1];
    });
    
    int removals = 0;
    int prevEnd = INT_MIN;  // end time of the last KEPT interval
    
    for (auto& interval : intervals) {
        if (interval[0] >= prevEnd) {
            // No conflict: this interval starts at or after the last kept one ends
            prevEnd = interval[1];  // keep this interval, update boundary
        } else {
            // Conflict: remove THIS interval (it ends later because we sorted by end,
            // so the one we already "kept" with earlier end is the right choice)
            removals++;
        }
    }
    
    return removals;
}

// NOTE: interval[0] >= prevEnd uses >=, meaning:
// If the previous kept interval ends at 5 and this one starts at 5: NO conflict (keep it).
// [1,5] and [5,8] → they are back-to-back, not overlapping → keep both.
// This is the standard interpretation for this problem.

// ALTERNATIVE FRAMING: "Maximum non-overlapping intervals to KEEP"
// = total - eraseOverlapIntervals(intervals)
// The same code, but return intervals.size() - removals.

// TRACE for [[1,2],[2,3],[3,4],[1,3]]:
// Sort by end: [[1,2],[2,3],[1,3],[3,4]]
// [1,2]: 1 >= INT_MIN → keep. prevEnd=2.
// [2,3]: 2 >= 2 → keep. prevEnd=3.
// [1,3]: 1 < 3 → remove. removals=1.
// [3,4]: 3 >= 3 → keep. prevEnd=4.
// Return 1 (remove [1,3]). ✓
```

---

### Template 4A: Meeting Rooms II — Min-Heap of End Times

```cpp
// LC 253: Meeting Rooms II — minimum number of rooms required
// Insight: a room can be reused if it's free (the meeting in it has ended)
//          by the time the next meeting starts.
// Strategy: sort meetings by start time. For each meeting, check if the
//            earliest-ending ongoing meeting has ended. If so, reuse that room.
//            Otherwise, open a new room. The heap size = rooms currently in use.

int minMeetingRooms(vector<vector<int>>& intervals) {
    sort(intervals.begin(), intervals.end());  // sort by start time
    
    // Min-heap of END TIMES of currently ongoing meetings
    // The top = the meeting ending soonest (the one we'd reuse first)
    priority_queue<int, vector<int>, greater<int>> minHeap;
    
    for (auto& interval : intervals) {
        if (!minHeap.empty() && minHeap.top() <= interval[0]) {
            // The earliest-ending meeting has ended at or before this meeting starts
            minHeap.pop();  // "free" that room — remove old end time
        }
        // Assign this meeting a room (either the freed one, or a new one)
        minHeap.push(interval[1]);  // room in use until interval[1]
    }
    
    return minHeap.size();  // rooms still in use = total rooms needed
}

// WHY minHeap.top() <= interval[0] and not < ?
// If the previous meeting ends at time 5 and the new one starts at 5:
// The room is free (meeting ended at 5). The new meeting starts at 5. Reuse the room.
// So: ended at 5 AND starts at 5 → NOT a conflict → reuse → use <=.
// (If problem said end is inclusive and rooms need time to be cleaned, use <.)

// TRACE for intervals=[[0,30],[5,10],[15,20]]:
// Sort: [[0,30],[5,10],[15,20]] (already sorted by start)
// [0,30]: heap empty, push 30. heap=[30]. Rooms=1.
// [5,10]: heap.top()=30, 30 > 5 → NO free room. Push 10. heap=[10,30]. Rooms=2.
// [15,20]: heap.top()=10, 10 <= 15 → FREE! Pop 10, push 20. heap=[20,30]. Rooms=2.
// Return heap.size() = 2 ✓
// (Meeting [5,10] frees its room for meeting [15,20] — they share a room.)

// IMPORTANT: we do NOT pop and then check if the room can be reused.
// We check heap.top() BEFORE popping. If top() <= start, pop (free) THEN push (assign).
// If top() > start, just push (new room needed) — do NOT pop.
```

---

### Template 4B: Meeting Rooms II — Line Sweep

```cpp
// Alternative: line sweep. More generalizable to non-integer timestamps and
// problems where you need the full utilization curve (not just the peak).

int minMeetingRooms_Sweep(vector<vector<int>>& intervals) {
    // Create events: +1 at meeting start, -1 at meeting end
    vector<pair<int,int>> events;
    for (auto& interval : intervals) {
        events.push_back({interval[0], 1});   // meeting starts: +1 room needed
        events.push_back({interval[1], -1});  // meeting ends:   -1 room freed
    }
    
    // Sort events by time. At the SAME time, process END events (-1) before START events (+1).
    // Pair comparison: (t, -1) < (t, +1) since -1 < 1 → end events come first. ✓
    sort(events.begin(), events.end());
    
    int maxRooms = 0, currentRooms = 0;
    for (auto& [time, delta] : events) {
        currentRooms += delta;
        maxRooms = max(maxRooms, currentRooms);
    }
    return maxRooms;
}

// WHY end events before start events at the same time?
// If meeting A ends at 10 and meeting B starts at 10: they can share a room.
// Processing the end event first: currentRooms decreases, THEN increases for B.
// The peak during "time 10" correctly reflects 1 room (not 2).
// If we processed starts before ends: currentRooms would momentarily reach 2 (wrong).
//
// AUTOMATIC TIE-BREAKING: pair {10, -1} < {10, 1} since -1 < 1.
// std::sort on pairs handles this correctly without custom comparator. ✓
//
// TRACE for [[0,30],[5,10],[15,20]]:
// Events (unsorted): {0,+1},{30,-1},{5,+1},{10,-1},{15,+1},{20,-1}
// Sorted: {0,+1},{5,+1},{10,-1},{15,+1},{20,-1},{30,-1}
//   (Note: {10,-1} < {10,+1} but +1 doesn't exist at 10 here)
// Sweep: 0→1, 5→2, 10→1, 15→2, 20→1, 30→0
// maxRooms = 2 ✓
```

---

### Template 5: Interval List Intersections — Two Pointers

```cpp
// LC 986: Interval List Intersections
// Given two sorted, non-overlapping interval lists A and B,
// find all pairs (A[i], B[j]) that intersect, and return the intersection intervals.
//
// KEY INSIGHT: after computing the intersection of A[i] and B[j],
// advance the pointer whose interval ends EARLIER.
// Why? That interval cannot intersect with any LATER interval in the other list
// (the other list is sorted, and later intervals start no earlier than the current).

vector<vector<int>> intervalIntersection(vector<vector<int>>& A, vector<vector<int>>& B) {
    vector<vector<int>> result;
    int i = 0, j = 0;
    
    while (i < (int)A.size() && j < (int)B.size()) {
        // Compute the intersection: [max of starts, min of ends]
        int lo = max(A[i][0], B[j][0]);
        int hi = min(A[i][1], B[j][1]);
        
        if (lo <= hi) {
            // Valid intersection (lo <= hi means they actually overlap)
            result.push_back({lo, hi});
        }
        
        // Advance the pointer with the earlier-ending interval
        if (A[i][1] < B[j][1]) i++;  // A[i] ends sooner → it can't intersect B[j+1], B[j+2], ...
        else j++;                     // B[j] ends sooner (or tied) → advance B
    }
    
    return result;
}

// WHY advance the pointer with the earlier end?
// Suppose A[i] = [1, 5] and B[j] = [3, 8]. Intersection = [3, 5].
// A[i] ends at 5. B[j] ends at 8. A[i] ends sooner → advance i.
// Can A[i] = [1, 5] intersect B[j+1]? B[j+1] starts at >= 8 (B is sorted, non-overlapping).
// 8 > 5 → A[i] cannot reach B[j+1]. Advancing i is safe. ✓
// Can B[j] = [3, 8] intersect A[i+1]? Yes, possibly — A[i+1] starts after 1, might start ≤ 8.
// We correctly keep j and advance i.
//
// TRACE for A=[[0,2],[5,10],[13,23],[24,25]], B=[[1,5],[8,12],[15,24],[25,26]]:
// i=0,j=0: A[0]=[0,2],B[0]=[1,5]. lo=max(0,1)=1,hi=min(2,5)=2. lo<=hi → push [1,2].
//          A[0].end=2 < B[0].end=5 → i++. i=1.
// i=1,j=0: A[1]=[5,10],B[0]=[1,5]. lo=5,hi=5. lo<=hi → push [5,5].
//          A[1].end=10 > B[0].end=5 → j++. j=1.
// i=1,j=1: A[1]=[5,10],B[1]=[8,12]. lo=8,hi=10. push [8,10].
//          A[1].end=10 < B[1].end=12 → i++. i=2.
// ... continues → Result: [[1,2],[5,5],[8,10],[15,23],[24,24],[25,25]] ✓
```

---

### Template 6: Line Sweep — General Pattern (My Calendar / Booking Problems)

```cpp
// General line sweep for "how many events overlap at any point"
// Works for any integer or coordinate-compressed timeline.
// Use an ordered map for sparse timelines (intervals with large range but few events).

class MyCalendarThree {
    map<int, int> timeline;  // time → net change (can be negative)
    
public:
    int book(int start, int end) {
        timeline[start]++;   // event starts: +1 ongoing
        timeline[end]--;     // event ends: -1 ongoing (half-open interval [start, end))
        
        int maxBookings = 0, ongoing = 0;
        for (auto& [time, delta] : timeline) {
            ongoing += delta;
            maxBookings = max(maxBookings, ongoing);
        }
        return maxBookings;
    }
};

// WHY `map<int, int>` and not vector?
// The range of start/end values can be huge (up to 10^9 in LeetCode).
// Map only stores points where something actually changes (sparse representation).
// std::map iterates in sorted order → correctly processes events in chronological order.
//
// WHY timeline[end]-- and not timeline[end-1]--?
// This uses HALF-OPEN intervals: [start, end) means from start (inclusive) to end (exclusive).
// A booking [10, 20] is "active" during [10, 11, 12, ..., 19] but NOT at 20.
// At time 20, it's gone. So we decrement at end (not end-1).
// For closed intervals [start, end] where end is the last active point:
// use timeline[end+1]--.
//
// HOW TO ADAPT for "check if any overlap exists" (My Calendar I, LC 729):
// After adding start++ and end--, sweep: if any point has sum > 1, there's a double-booking.
// But for online queries, use a sorted set approach instead (O(log n) per query).
// Using the sweep for every query is O(n) per query → O(n²) total → too slow for large n.

// FASTER approach for "check if new interval overlaps existing" (My Calendar I):
// Use an ordered map storing {start → end} for booked intervals.
// For a new interval [s, e): find the first existing interval with start >= s.
//   - If it exists and its start < e → new interval overlaps it → reject.
// Also check the interval just before the new one (start < s):
//   - If its end > s → overlaps the new interval → reject.

class MyCalendarOne {
    map<int, int> booked;  // start → end (half-open intervals)
    
public:
    bool book(int start, int end) {
        auto it = booked.lower_bound(start);  // first interval with start >= new start
        
        // Check the interval AFTER new interval: does it start before new ends?
        if (it != booked.end() && it->first < end) return false;  // overlap
        
        // Check the interval BEFORE new interval: does it end after new starts?
        if (it != booked.begin()) {
            --it;
            if (it->second > start) return false;  // overlap
        }
        
        booked[start] = end;  // no conflict, book it
        return true;
    }
};
```

---

### Template 7: Employee Free Time — Merge All + Find Gaps

```cpp
// LC 759: Employee Free Time
// Given K employees, each with a sorted schedule (list of intervals),
// find ALL time slots when ALL employees are simultaneously free.
// = find the gaps between the merged union of all intervals.

// Using the Interval struct (LC 759 definition):
struct Interval { int start, end; };

vector<Interval> employeeFreeTime(vector<vector<Interval>> schedule) {
    // Step 1: Collect all intervals from all employees into one flat list
    vector<Interval> all;
    for (auto& emp : schedule) for (auto& iv : emp) all.push_back(iv);
    
    // Step 2: Sort by start time
    sort(all.begin(), all.end(), [](const Interval& a, const Interval& b) {
        return a.start < b.start;
    });
    
    // Step 3: Merge all overlapping intervals (Template 1)
    vector<Interval> merged;
    for (auto& iv : all) {
        if (merged.empty() || merged.back().end < iv.start) {
            merged.push_back(iv);
        } else {
            merged.back().end = max(merged.back().end, iv.end);
        }
    }
    
    // Step 4: Gaps between consecutive merged intervals ARE the free times
    vector<Interval> result;
    for (int i = 1; i < (int)merged.size(); i++) {
        result.push_back({merged[i-1].end, merged[i].start});
    }
    
    return result;
}

// ALTERNATIVE: Merge K sorted lists using a heap (O(N log K) vs O(N log N)):
// Initialize a min-heap with the first interval of each employee's schedule.
// Pop the minimum, extend merged or start new, push the next interval from that employee.
// Same approach as Pattern 10's "merge K sorted lists" but for intervals.
// Useful when K is small and each employee's list is long.

// NOTE: there is no "free time before the first interval" or "after the last interval"
// in the standard problem (assumes free time is only between work intervals).
// If the problem asks for global free time including before/after, add those explicitly.
```

---

## THE VARIANTS — DEEP DIVE INTO EACH SUB-PATTERN

### VARIANT 1: Minimum Number of Arrows to Burst Balloons (LC 452) — Greedy on Interval Coverage

```cpp
// Each balloon is an interval [start, end]. An arrow at position x bursts all balloons
// containing x (i.e., start <= x <= end). Minimize arrows.
// Same as: maximum non-overlapping intervals (different framing, same greedy).

int findMinArrowShots(vector<vector<int>>& points) {
    sort(points.begin(), points.end(), [](auto& a, auto& b) {
        return a[1] < b[1];  // sort by END (same as activity selection)
    });
    
    int arrows = 0;
    int arrowPos = INT_MIN;  // position of the last arrow shot
    
    for (auto& balloon : points) {
        if (balloon[0] > arrowPos) {
            // This balloon is not yet burst — shoot a new arrow at its END
            // (shooting at the end maximizes the chance of also hitting future overlapping balloons)
            arrowPos = balloon[1];
            arrows++;
        }
        // else: this balloon is already burst by the previous arrow
    }
    
    return arrows;
}

// KEY DIFFERENCE from LC 435 (Non-overlapping Intervals):
// LC 435 uses balloon[0] >= prevEnd (>= means touching is NOT overlapping)
// LC 452 uses balloon[0] > arrowPos (strictly > means touching IS overlapping — 
//   an arrow at position 5 DOES burst a balloon starting at 5)
// The comparison operator changes because of the problem's definition of "overlap."
// This is a common source of off-by-one errors between these two problems.
```

---

### VARIANT 2: Remove Covered Intervals (LC 1288) — Sort + Greedy Coverage Check

```cpp
// An interval [a, b] is "covered" by [c, d] if c <= a AND b <= d.
// Remove all covered intervals and return the count of remaining intervals.

int removeCoveredIntervals(vector<vector<int>>& intervals) {
    // Sort: by start ascending, then by end DESCENDING for equal starts.
    // (If two intervals start at the same time, the longer one "covers" the shorter.)
    sort(intervals.begin(), intervals.end(), [](auto& a, auto& b) {
        if (a[0] != b[0]) return a[0] < b[0];
        return a[1] > b[1];  // longer interval first for same start
    });
    
    int count = 0;
    int maxEnd = 0;  // the maximum end seen so far (the "cover" boundary)
    
    for (auto& interval : intervals) {
        if (interval[1] > maxEnd) {
            // This interval extends beyond the current cover → NOT covered
            count++;
            maxEnd = interval[1];
        }
        // else: interval[1] <= maxEnd → this interval is fully covered by a previous one
    }
    
    return count;
}

// WHY sort by start asc, end desc for equal starts?
// [1,4] and [1,2]: if [1,4] comes first, maxEnd=4 after processing it.
// Then [1,2]: 2 <= 4 → covered (skipped). Correct: [1,2] is covered by [1,4]. ✓
// If [1,2] comes first: maxEnd=2, count=1. Then [1,4]: 4 > 2 → count=2. Wrong!
// [1,4] is NOT covered by [1,2]. Sorting longer intervals first for same start fixes this.
```

---

### VARIANT 3: Data Stream as Disjoint Intervals (LC 352) — Online Interval Merge

```cpp
// Maintain a sorted set of non-overlapping intervals. Support:
// - addNum(val): add value val as an interval [val, val], merge if overlapping
// - getIntervals(): return the sorted list of non-overlapping intervals

class SummaryRanges {
    map<int, int> intervals;  // start → end (sorted by start, non-overlapping)
    
public:
    void addNum(int val) {
        int lo = val, hi = val;
        
        // Find the first interval that starts AFTER val
        auto it = intervals.upper_bound(val);
        
        // Check the interval BEFORE it: does it overlap or touch [val, val]?
        if (it != intervals.begin()) {
            auto prev = std::prev(it);
            if (prev->second >= val - 1) {  // >= val-1 means it touches or overlaps
                lo = min(lo, prev->first);
                hi = max(hi, prev->second);
                intervals.erase(prev);
            }
        }
        
        // Check the interval AT it: does it overlap or touch [val, val]?
        if (it != intervals.end() && it->first <= val + 1) {  // <= val+1 means it touches
            hi = max(hi, it->second);
            intervals.erase(it);
        }
        
        intervals[lo] = hi;  // insert the merged interval
    }
    
    vector<vector<int>> getIntervals() {
        vector<vector<int>> result;
        for (auto& [start, end] : intervals) result.push_back({start, end});
        return result;
    }
};

// TIME: addNum is O(log n) amortized. Each interval is inserted once and erased at most once.
// SPACE: O(n) where n = number of unique values added.
```

---

### VARIANT 4: Meeting Rooms I (LC 252) — Simple Overlap Check

```cpp
// Can one person attend all meetings? = Are there any overlapping intervals?
// Sort by start; check if any consecutive pair overlaps.

bool canAttendMeetings(vector<vector<int>>& intervals) {
    sort(intervals.begin(), intervals.end());
    for (int i = 1; i < (int)intervals.size(); i++) {
        if (intervals[i][0] < intervals[i-1][1]) {  // strictly less: start < end = conflict
            return false;
        }
    }
    return true;
}
// If meeting ends at 5 and next starts at 5: intervals[i][0]=5 is NOT < intervals[i-1][1]=5
// → no conflict. Back-to-back meetings are OK. ✓
// If meeting ends at 5 and next starts at 4: 4 < 5 → conflict. ✗
```

---

### VARIANT 5: My Calendar I (LC 729) — O(log n) Overlap Check with Sorted Map

Already covered in Template 6 (MyCalendarOne). The key operations:
- `lower_bound(start)` finds the first booked interval starting AT or AFTER the new start
- Check if this interval starts before the new interval ends (would overlap)
- Check the previous interval (if any) to see if it ends after the new start (would overlap)
- If neither check triggers: no overlap, book the interval

---

## TIME AND SPACE COMPLEXITY — COLD RECITATION

| Operation | Time | Space | Why |
|-----------|------|-------|-----|
| Merge Intervals | O(n log n) | O(n) | Sort + one pass; result stores all merged intervals |
| Insert Interval | O(n) | O(n) | Input already sorted; 3-phase linear sweep |
| Non-overlapping (Activity Selection) | O(n log n) | O(1) | Sort + one greedy pass, no extra storage |
| Meeting Rooms II (min-heap) | O(n log n) | O(n) | Sort + heap; heap holds up to n end times |
| Meeting Rooms II (line sweep) | O(n log n) | O(n) | Sort 2n events + one sweep |
| Interval Intersections | O(n + m) | O(1) | Two pointers; each interval visited once |
| Employee Free Time (sort) | O(N log N) | O(N) | N = total intervals; merge all then find gaps |
| Employee Free Time (heap) | O(N log K) | O(K) | K = employees; merge K sorted lists |
| Online Merge (sorted map) | O(log n) amortized | O(n) | Each interval inserted and erased once |
| Line Sweep (map-based) | O(n log n) | O(n) | n events in a sorted map |

### The O(n) Fact to State Out Loud:

"Insert Interval is O(n), NOT O(n log n). The input list is already sorted and non-overlapping — a given invariant. The 3-phase approach sweeps through the list exactly once. There's no re-sorting. Candidates who say 'sort then merge' for Insert Interval are solving the wrong problem — they're solving Merge Intervals (LC 56) instead of Insert Interval (LC 57)."

---

## COMMON MISTAKES AT THIS PATTERN (At YOUR 1600 Rating Level)

---

### Mistake 1: Using `<=` vs `<` Incorrectly in the Overlap/Non-overlap Condition

**What they do:** They use `result.back()[1] <= interval[0]` as the non-overlap check in Merge Intervals, treating touching endpoints as non-overlapping. Or they use `result.back()[1] < interval[0]` in the wrong direction.

**What goes wrong:** `[1,3]` and `[3,5]` should merge to `[1,5]` in the standard LeetCode interpretation (touching = overlapping). With `<= interval[0]` as the non-overlap check: `3 <= 3` is true → they're "not overlapping" → pushed as separate intervals → `[[1,3],[3,5]]` instead of `[[1,5]]`. Wrong answer.

**The fix:** For non-overlap in Merge Intervals: `result.back()[1] < interval[0]` (strictly less than). The interval `[a, b]` and `[c, d]` do NOT overlap only when `b < c` (the first ends strictly before the second starts).

```cpp
// WRONG: treats touching as non-overlapping
if (result.back()[1] <= interval[0]) {
    result.push_back(interval);      // [1,3] and [3,5] → NOT merged. Bug!
}

// CORRECT: treats touching as overlapping (standard LeetCode behavior)
if (result.back()[1] < interval[0]) {
    result.push_back(interval);      // Only truly non-overlapping intervals are separate
} else {
    result.back()[1] = max(result.back()[1], interval[1]);  // merge
}
```

---

### Mistake 2: Sorting by START Instead of END for Non-overlapping Interval Greedy

**What they do:** For "minimum removals to make non-overlapping" (LC 435), they sort by start time instead of end time, applying the same sort as Merge Intervals.

**What goes wrong:** Sorting by start and keeping intervals until a conflict produces incorrect results. The greedy choice of "earliest start" does not guarantee the maximum number of non-conflicting intervals. Example: interval [1, 100] starts earliest but blocks all other intervals. The correct greedy (sort by end) would skip [1, 100] in favor of shorter intervals.

**The fix:** Sort by END time for the activity selection greedy. The intuition: keeping the interval with the earliest end leaves the most room for future intervals.

```cpp
// WRONG: sort by start time (correct for merge, WRONG for non-overlapping)
sort(intervals.begin(), intervals.end());  // sorts by [0] = start time
int removals = 0, prevEnd = INT_MIN;
for (auto& iv : intervals) {
    if (iv[0] < prevEnd) removals++;  // conflict → remove
    else prevEnd = iv[1];
}
// Fails on [[1,100],[2,3],[4,5]] — sort by start: [1,100],[2,3],[4,5]
// Keep [1,100], but [2,3] and [4,5] both conflict → 2 removals. WRONG!
// Optimal: remove [1,100], keep [2,3] and [4,5] → 1 removal.

// CORRECT: sort by END time
sort(intervals.begin(), intervals.end(), [](auto& a, auto& b) {
    return a[1] < b[1];
});
// Sort by end: [2,3],[4,5],[1,100]
// Keep [2,3] (end=3), keep [4,5] (start=4 >= 3), [1,100] conflicts (1 < 5) → 1 removal. ✓
```

---

### Mistake 3: Meeting Rooms II — Popping the Heap Without Checking if It's Actually Free

**What they do:** They pop from the min-heap unconditionally for each interval and only push if a "new room is needed." Logic: pop the earliest-ending meeting, see if the room is free, push back if not free.

**What goes wrong:** If they pop and then push back (because the room isn't free), they've done unnecessary work. Worse: if they pop first and then check, they may pop a meeting that's still ongoing — and then they've "freed" a room that wasn't free, losing the tracking of that room's end time.

**The fix:** Check `heap.top() <= interval[0]` BEFORE popping. Only pop (free the room) if the condition is satisfied. Never pop and push back.

```cpp
// WRONG: pop unconditionally, push back if not free
for (auto& interval : intervals) {
    if (!minHeap.empty()) {
        int earliestEnd = minHeap.top(); minHeap.pop();  // unconditional pop!
        if (earliestEnd <= interval[0]) {
            // Room is free — don't push back (room is being reused)
        } else {
            minHeap.push(earliestEnd);  // room not free — push back
        }
    }
    minHeap.push(interval[1]);
}
// This is functionally equivalent to the correct solution but wasteful.
// The real bug: if minHeap becomes empty after the unconditional pop, the first push-back
// condition is never checked, and you've unnecessarily modified the heap state.

// CORRECT: conditional pop — only pop if the room is actually free
for (auto& interval : intervals) {
    if (!minHeap.empty() && minHeap.top() <= interval[0]) {
        minHeap.pop();  // room is free, "release" it
    }
    minHeap.push(interval[1]);  // assign room (new or freed) until interval[1]
}
```

---

### Mistake 4: Insert Interval — Phase 2 Condition Off-by-One (Using < Instead of <=)

**What they do:** In Phase 2 of Insert Interval, they use `intervals[i][0] < newInterval[1]` (strictly less than) instead of `intervals[i][0] <= newInterval[1]`.

**What goes wrong:** The touching case is missed. For `intervals = [[3,5],[6,7]]` and `newInterval = [1,6]`: interval `[6,7]` starts at 6 and newInterval ends at 6. With `< 6`: `6 < 6` is false → `[6,7]` is not merged → result includes `[1,6],[6,7]` as separate (overlapping!) intervals. The output is NOT non-overlapping. Correct: `[6,7]` starts at 6 ≤ 6 (newInterval's end) → should be merged → result is `[1,7]`.

**The fix:** Phase 2 condition must be `intervals[i][0] <= newInterval[1]` to handle the touching case correctly.

```cpp
// WRONG: < instead of <=, misses touching intervals
while (i < n && intervals[i][0] < newInterval[1]) {  // BUG: < misses start == end
    newInterval[0] = min(newInterval[0], intervals[i][0]);
    newInterval[1] = max(newInterval[1], intervals[i][1]);
    i++;
}
// For intervals=[[1,5],[6,8]], newInterval=[3,6]:
// Phase 2: intervals[1][0]=6 < 6? 6 < 6 is FALSE → [6,8] not merged
// Result: [[1,6],[6,8]] — WRONG! These overlap (6 == 6).

// CORRECT: <= to include touching intervals in Phase 2
while (i < n && intervals[i][0] <= newInterval[1]) {  // <= is correct
    newInterval[0] = min(newInterval[0], intervals[i][0]);
    newInterval[1] = max(newInterval[1], intervals[i][1]);
    i++;
}
// Phase 2: intervals[1][0]=6 <= 6? TRUE → merge. new=[min(3,6),max(6,8)]=[3,8]. ✓
```

---

### Mistake 5: Line Sweep — Wrong Tie-breaking: Processing Start Events Before End Events

**What they do:** They sort events with `(time, +1)` before `(time, -1)` at the same timestamp — either by using a custom comparator that puts starts first, or by using `(time, 1)` as the end event and `(time, 0)` as the start event for the value field.

**What goes wrong:** If a meeting ends at time T and another starts at time T, they CAN share a room. Processing the start event (+1) before the end event (-1) at time T: the room count momentarily goes UP before going DOWN. This makes the max count 1 higher than the actual peak — reporting one more room than needed.

**The fix:** Process end events (-1) before start events (+1) at the same timestamp. Use `(time, -1)` for end and `(time, +1)` for start as event pairs. Pair comparison is lexicographic: `(10, -1) < (10, 1)` since `-1 < 1`. The sort handles this automatically without a custom comparator.

```cpp
// WRONG: start events before end events at same time
events.push_back({interval[1], 1});   // end: marked with +1 ← BUG
events.push_back({interval[0], 0});   // start: marked with 0 ← BUG
// Sorted: {0,+1} for starts come after {0,0} for starts? No...
// Actually this specific encoding is confusing — just a wrong approach.

// Also WRONG: explicit sort that puts +1 before -1 at same time
sort(events.begin(), events.end(), [](auto& a, auto& b) {
    if (a.first == b.first) return a.second > b.second;  // +1 before -1 at same time — BUG
    return a.first < b.first;
});

// CORRECT: end events (-1) before start events (+1) at same time (automatic with pair sort)
events.push_back({interval[0], +1});  // start
events.push_back({interval[1], -1});  // end
sort(events.begin(), events.end());   // pair: (t,-1) < (t,+1) → ends processed first ✓
```

---

### Mistake 6: Confusing Merge Intervals Template with Insert Interval — Re-sorting the Already-Sorted List

**What they do:** For Insert Interval (LC 57), they add the new interval to the vector, sort the entire vector, and then call the Merge Intervals function. They treat Insert Interval as "just Merge Intervals with one more interval."

**What goes wrong:** Re-sorting is O(n log n). The correct Insert Interval is O(n) because the input is already sorted and non-overlapping. More importantly, sorting + merging loses the specific structure the 3-phase approach exploits: "the list is sorted, so we know exactly where the new interval fits." The re-sort approach is technically correct but:
(1) It's the wrong time complexity (O(n log n) when O(n) is achievable).
(2) Interviewers ask Insert Interval specifically to test the 3-phase approach — giving the O(n log n) answer signals you don't know the intended solution.

**The fix:** Use the 3-phase O(n) approach that leverages the sorted order of the input. Never re-sort an already-sorted input.

```cpp
// WRONG: re-sort approach (O(n log n), shows you treat this like Merge Intervals)
vector<vector<int>> insert_Wrong(vector<vector<int>>& intervals, vector<int> newInterval) {
    intervals.push_back(newInterval);
    sort(intervals.begin(), intervals.end());  // O(n log n) — wasteful!
    return merge(intervals);                    // then merge — double the work
}

// CORRECT: 3-phase O(n) approach (exploits the sorted + non-overlapping invariant)
vector<vector<int>> insert_Correct(vector<vector<int>>& intervals, vector<int> newInterval) {
    vector<vector<int>> result;
    int i = 0, n = intervals.size();
    while (i < n && intervals[i][1] < newInterval[0])  result.push_back(intervals[i++]);
    while (i < n && intervals[i][0] <= newInterval[1]) {
        newInterval[0] = min(newInterval[0], intervals[i][0]);
        newInterval[1] = max(newInterval[1], intervals[i][1]);
        i++;
    }
    result.push_back(newInterval);
    while (i < n) result.push_back(intervals[i++]);
    return result;
}
```

---

## THE PROBLEM SET — SOLVE IN THIS EXACT ORDER

---

### WARMUP — 2 problems (build interval syntax reflexes)

**1. Meeting Rooms — LC 252 — [Sort by start; check consecutive pair overlap]**

Why this problem: The simplest possible interval problem. Sort by start time, then check if any consecutive pair overlaps (`intervals[i][0] < intervals[i-1][1]`). Forces you to write your first interval sort and understand the overlap condition. If you can't do this in 3 minutes, your interval foundation is not ready.

What to prove: Sorting by start time is sufficient — if there's any overlap in the entire list, it will appear as a consecutive overlap after sorting. (If `[a,b]` overlaps `[c,d]` with `c > a`, then after sorting, they're adjacent, and the check catches it.) The overlap condition: `interval[i].start < interval[i-1].end` (strictly less: start at end time means back-to-back, which is allowed).

Target time: 3 minutes.

Edge cases: Empty intervals (return true). One interval (return true — no pairs to conflict). All intervals touching end-to-start (allowed — return true).

---

**2. Merge Intervals — LC 56 — [Template 1: sort by start + one-pass merge]**

Why this problem: The canonical merge intervals template. Must be mechanical. Sort by `intervals[0]` (start time), then sweep: either push a new interval (no overlap) or extend the current one's end (overlap). The `max()` in the merge is non-trivial — one interval can be fully contained in another, so you can't just take the current's end.

What to prove: After sorting by start, the `result.back()[1] < interval[0]` condition correctly detects non-overlap. The `max()` for the end handles the containment case: `[1,10]` and `[2,3]` → `max(10, 3) = 10` → `[1,10]` is correctly preserved.

Target time: 5 minutes.

Edge cases: Single interval (no merging). All intervals disjoint (output = input). All intervals contained within one large interval (output = 1 interval). All intervals identical (output = 1 interval).

---

### CORE — 4 problems (the five essential techniques)

**3. Insert Interval — LC 57 — [Template 2: 3-phase O(n) sweep on sorted non-overlapping list]**

Why this problem: Tests whether you know Insert Interval is DIFFERENT from Merge Intervals. The key: the input is already sorted and non-overlapping — do NOT re-sort. The 3-phase approach (skip non-conflicting before, merge overlapping, append after) is the tested skill. Microsoft asks this frequently as their "medium" interval question.

What to prove: Phase 1 terminates when `intervals[i][1] >= newInterval[0]`. Phase 2 uses `intervals[i][0] <= newInterval[1]` (not `<`). Phase 2 merges by taking `min` of starts and `max` of ends. After Phase 2, push the merged `newInterval`. Phase 3 pushes the rest. All three phases together are O(n) — linear in the input size.

Target time: 12 minutes.

Edge cases: New interval starts before all existing (Phase 1 collects nothing). New interval ends after all existing (Phase 3 collects nothing). New interval doesn't overlap anything (Phase 2 executes zero iterations; newInterval is inserted at the correct position). Empty intervals list (all phases collect nothing; output is just newInterval).

---

**4. Non-overlapping Intervals — LC 435 — [Template 3: sort by END; greedy keep-earliest-ending]**

Why this problem: The "activity selection" greedy. Forces you to know WHY you sort by end and not start. If an interviewer asks "why sort by end?" you must answer with the exchange argument: "keeping the interval ending soonest leaves maximum room for future intervals." Any interval ending later that conflicts should be removed first.

What to prove: Sort by end. Greedy loop: keep the interval with the earliest end (the first one, then any subsequent that starts at or after `prevEnd`). Count the ones NOT kept (the removals). The overlap check: `interval[0] < prevEnd` (strictly less: touching endpoints are NOT a conflict here — `interval[0] >= prevEnd` means the interval starts at prevEnd or after → no conflict).

Target time: 12 minutes.

Edge cases: No overlapping intervals (removals = 0). All intervals identical (remove n-1 of them). Two intervals where one contains the other (the shorter one is kept because it ends sooner; the containing one is removed).

---

**5. Meeting Rooms II — LC 253 — [Template 4A: min-heap of end times; OR Template 4B: line sweep]**

Why this problem: Top 20 most asked at Amazon and Microsoft. Two valid approaches: heap (more intuitive, shows data structure thinking) and line sweep (more general, shows algorithmic thinking). An interviewer may follow up asking for the alternative after you give the first. Know BOTH.

What to prove: Heap: after sorting by start, for each interval, pop the heap if `heap.top() <= interval[0]` (earliest-ending room is free). Push the current interval's end. Answer: `heap.size()`. Line sweep: events `{start, +1}` and `{end, -1}`, sorted (automatic tie-break: `-1 < +1`), running sum's maximum.

Target time: 15 minutes (first approach). 8 more minutes for the second approach.

Edge cases: No meetings (0 rooms). One meeting (1 room). All meetings at the same time (n rooms). All meetings back-to-back (1 room — they can share because `heap.top() <= start`).

---

**6. Interval List Intersections — LC 986 — [Template 5: two pointers; advance the earlier-ending interval]**

Why this problem: The two-pointer technique specifically designed for two sorted interval lists. Forces you to understand the "advance the earlier-ending pointer" logic. Compute intersection as `[max(starts), min(ends)]`; push if `lo <= hi`.

What to prove: The "advance the earlier-ending pointer" is correct because the current interval from the earlier-ending list cannot intersect any future interval from the other list (future intervals start at least at the current interval's start, so they don't extend backward to overlap the already-ended interval). The two-pointer approach is O(n + m) — optimal.

Target time: 12 minutes.

Edge cases: No intersections (one list ends before the other begins — all `lo > hi`). One interval fully contains an interval from the other list (intersection = the smaller interval). Both lists have a single interval (one intersection or none).

---

### STRETCH — 3 problems

**7. Minimum Number of Arrows to Burst Balloons — LC 452 — [Greedy sort by end; shoot at the end of the first unshot balloon]**

Why this problem: Looks like LC 435 but has a subtle difference: the overlap condition uses `>` instead of `>=`. A balloon at `[5, 7]` and an arrow at position 5 DOES burst it (`start <= arrowPos`). So the check for "already burst" is `balloon[0] <= arrowPos` (not `< arrowPos`). Getting the comparison right under pressure is the test.

What to prove: Sort by end. For each balloon not yet burst (start > current arrow position), shoot a new arrow at its end position. The arrow at the end maximizes overlap with future balloons (shooting earlier would miss balloons that start between current and end). Number of arrows = number of times we shoot = answer.

Target time: 15 minutes.

Edge cases: Single balloon (1 arrow). All balloons identical (1 arrow). All balloons disjoint (one arrow per balloon).

---

**8. Employee Free Time — LC 759 — [Merge all intervals from all employees; find gaps between merged intervals]**

Why this problem: Tests whether you can handle K sorted lists of intervals elegantly. The naive approach (flatten all to one list, sort, merge, find gaps) is O(N log N) and correct. The heap-based approach (merge K sorted lists, like Pattern 10 Template 5) is O(N log K) and demonstrates Pattern 10 + 11 integration. Both approaches should be known.

What to prove: After merging all intervals: any point NOT covered by a merged interval is "free." The free times are exactly the GAPS between consecutive merged intervals. There are no "free time before the first interval" or "after the last interval" in the standard problem (infinite workdays on both ends are assumed to be occupied — only gaps between given intervals count).

Target time: 20 minutes.

Edge cases: One employee with no breaks (no free time). All employees have non-overlapping schedules with the same gap (one free time period).

---

**9. Remove Covered Intervals — LC 1288 — [Sort by start asc, end desc for equal starts; count intervals with new maximum end]**

Why this problem: A variation of the containment/coverage check. An interval is "covered" if there's another interval that contains it entirely. Forces you to reason about the sorting comparator: for equal start times, longer intervals must come first (they are the "covers"). After sorting, an interval is NOT covered if and only if its end exceeds all previous ends seen.

What to prove: The sorting invariant: for intervals with the same start, the longest comes first. After sorting, maintain `maxEnd` = maximum end seen so far. An interval `[a, b]` is NOT covered iff `b > maxEnd` (it extends beyond all previous intervals). Count these uncovered intervals.

Target time: 15 minutes.

Edge cases: No covered intervals (all are distinct with no containment). All intervals except one are covered (answer = 1). A chain of nested intervals like [1,10],[2,8],[3,6] (only [1,10] survives after sorting and checking).

---

### CONTEST — 2 problems

**10. My Calendar III — LC 732 — [Line sweep with sorted map; return max concurrent bookings]**

Why this is contest level: Requires the ordered-map line sweep pattern (Template 6). For each new booking `[start, end)`: `timeline[start]++; timeline[end]--`. After each booking, sweep the map to find the maximum running sum. This is O(n) per booking (sweep is linear in current bookings), giving O(n²) total. For large inputs, a segment tree with lazy propagation is needed for O(n log n). The map approach is the interview-viable solution.

What to prove: `timeline[end]--` uses the half-open interval convention: the booking is NOT active at time `end`. The map's sorted order guarantees the sweep processes events chronologically. The maximum running sum across all map entries = maximum concurrent bookings.

Target time: 20 minutes.

Edge cases: Non-overlapping bookings (max is always 1). Completely overlapping bookings (max = n). Bookings that only touch at one endpoint ([1,5) and [5,10): they don't overlap — touching is not overlapping with half-open intervals).

---

**11. Data Stream as Disjoint Intervals — LC 352 — [Online merge with sorted map; O(log n) amortized per addNum]**

Why this is contest level: Requires combining a sorted map with interval merging logic — an online problem where you can't batch-sort. Each `addNum(val)` must merge the new `[val, val]` with any adjacent or overlapping existing intervals. Uses `upper_bound` to find the next interval and `prev()` to find the previous one. The amortized O(log n) per operation requires careful reasoning: each interval is inserted once and erased at most once.

What to prove: The `prev - 1` and `next + 1` checks handle the "touching" case: if an existing interval ends at `val - 1`, it should be extended to include `val`. The `upper_bound(val)` gives the first interval starting AFTER val — and its predecessor (if any) ends at or before val. After touching/overlapping checks, erase conflicting intervals and insert the merged one.

Target time: 30 minutes.

Edge cases: Adding a value already in an existing interval (no change to the set). Adding a value that bridges two existing intervals (merge all three: left, new, right). Adding values in decreasing order (the map handles unsorted insertions naturally — it maintains sorted order).

---

## PATTERN CONNECTIONS

**Connects to Sorting (Pattern 01 / General):**
Interval problems are almost entirely sorting problems in disguise. The choice of sort key (start or end) determines the algorithm. Merge → sort by start. Non-overlapping greedy → sort by end. Meeting rooms (heap) → sort by start. Most interval bugs come from using the wrong sort key. This pattern is primarily a sorting pattern that operates on pairs.

**Connects to Heap (Pattern 10):**
Meeting Rooms II (min-heap of end times) is a direct application of Pattern 10's min-heap template. The heap tracks "which room becomes available next" — always the one with the smallest end time. Employee Free Time can use a heap to merge K sorted interval lists (O(N log K) instead of O(N log N)). If you've mastered Pattern 10, Meeting Rooms II's heap solution should take under 8 minutes.

**Connects to Binary Search (Pattern 05):**
Insert Interval's Phase 1 and Phase 3 can be optimized with binary search (instead of linear scan) if you only need the boundaries: find the last interval ending before the new one starts (lower_bound on end times), and find the first interval starting after the new one ends. The standard 3-phase template is O(n) but binary search optimization gives O(log n) to find the boundaries + O(n) to output the result (no asymptotic improvement, but reduces constant factors).

**Connects to Prefix Sums / Difference Arrays (Pattern 06):**
Line sweep IS a difference array applied to a timeline. `timeline[start]++; timeline[end]--` is the difference array update operation from Pattern 06. The "sweep" is computing prefix sums over the difference array. If you know Pattern 06's difference array technique, line sweep is an immediate extension to a sorted map (for non-dense timelines).

**Connects to Graphs — Shortest Path (Pattern 14):**
Interval scheduling with precedence constraints (one task must finish before another starts) models as a DAG. The minimum time to complete all dependent tasks = longest path in the DAG. This connection shows up in project scheduling problems: intervals represent tasks, overlap/dependency defines edges.

**Connects to Greedy (Pattern 34):**
Activity selection (non-overlapping intervals greedy) is THE canonical greedy example. The exchange argument proof — "swapping any other choice with the greedy choice doesn't improve the solution" — is a greedy proof template used in Pattern 34. Every other greedy interval problem (balloon arrows, weighted job scheduling) can be analyzed with the same exchange argument framework.

---

## INTERVIEW SIMULATION QUESTIONS

---

### Q1: "Given an array of intervals, merge all overlapping intervals and return the resulting non-overlapping intervals."

**Company type:** Every company — Amazon, Google, Microsoft, Meta, all Indian unicorns. This is the most asked warm-up interval problem.
**Expected time:** 5-8 minutes (if it takes longer, your interval fundamentals are not interview-ready)
**What they are testing:** The canonical merge template, sort key choice, the overlap condition, handling the `max()` for merged end

**Red flags that get candidates rejected:**
- Not sorting first — attempting to merge without sorting leads to missed merges (e.g., `[3,5]` before `[1,4]` in the input)
- Sorting by end instead of start — produces incorrect merges for containing intervals
- Using `<=` for the non-overlap check instead of `<` — misses merging of touching intervals
- Forgetting `max()` in the merge: `result.back()[1] = interval[1]` instead of `result.back()[1] = max(result.back()[1], interval[1])` — fails when one interval fully contains another
- Not handling empty input — accessing `result.back()` on an empty vector causes undefined behavior

**What they want you to say:**
"I'll sort by start time. Then one pass: for each interval, if it starts after the last merged interval ends — no overlap, push as a new interval. If it starts at or before the last merged interval's end — overlap, extend the last interval's end using `max`. The `max` is important because the new interval might be fully contained within the previous one."

**The follow-up:** "What if the intervals represent time slots where you need to schedule work, and you want to maximize total covered time?" → Sum `(interval[1] - interval[0])` for each merged interval. Then: "What if you can remove at most K intervals to maximize covered time?" → Sort by contribution, greedy removal (now connects to activity selection).

---

### Q2: "You are given a list of meeting time intervals. Find the minimum number of conference rooms required."

**Company type:** Amazon (top 15 most asked), Microsoft, Bloomberg
**Expected time:** 15 minutes (first approach). May ask for second approach.
**What they are testing:** Resource counting pattern, min-heap of end times, correct `<=` comparison for "same time is OK," ability to explain the line sweep alternative

**Red flags that get candidates rejected:**
- Using a counter without a heap — just counting overlaps without tracking WHICH rooms are free leads to wrong answers for non-trivial inputs
- Popping the heap unconditionally for every interval instead of conditionally (`if heap.top() <= interval.start`)
- Using `< interval.start` instead of `<= interval.start` — fails to reuse a room for back-to-back meetings (a meeting ending at 5 could free a room for one starting at 5)
- Forgetting to sort by start time first — without sorting, the "check if a room is free" logic breaks because meetings aren't processed in chronological order
- Not knowing the line sweep alternative — after giving the heap solution, an interviewer often asks "is there another way?" Inability to discuss line sweep limits the candidate

**What they want you to say:**
"I'll sort meetings by start time. I maintain a min-heap of end times — the heap represents rooms that are currently in use, with each element being the time that room becomes free. For each meeting: if the heap's top (earliest-ending meeting) finishes at or before this meeting starts, pop it — that room is now free. Then push this meeting's end time (assign it a room, either the freed one or a new one). The heap size at the end is the answer."

**The follow-up:** "What if meetings can last for different amounts of time and we want to MINIMIZE total idle time between meetings?" → This is a scheduling optimization problem, not just counting. It connects to Pattern 34 (greedy scheduling). Then: "What's the time complexity of your solution?" → O(n log n) for sort + O(n log n) for heap operations = O(n log n) overall.

---

### Q3: "Given a sorted list of non-overlapping intervals and a new interval, insert the new interval and return the resulting sorted list of non-overlapping intervals."

**Company type:** Microsoft (favorite medium interval problem), Amazon, Apple
**Expected time:** 12-15 minutes
**What they are testing:** The 3-phase O(n) approach (NOT re-sorting), correct Phase 1/Phase 2 boundary conditions, handling all edge cases

**Red flags that get candidates rejected:**
- Re-sorting the array after inserting the new interval — O(n log n) instead of O(n), and signals the candidate doesn't see the input invariant
- Getting the Phase 1 condition wrong: using `intervals[i][1] <= newInterval[0]` (should be `<` — touching should NOT stop Phase 1, it should be merged in Phase 2)
- Getting the Phase 2 condition wrong: using `intervals[i][0] < newInterval[1]` (should be `<=` — an interval starting exactly at newInterval's end should be merged)
- Not handling the edge case where the new interval doesn't overlap anything — the 3-phase handles this naturally, but candidates who hardcode logic for "overlapping case" often miss the "no overlap" case
- Not updating `newInterval[0]` during Phase 2 — only updating `newInterval[1]` misses cases where the current interval starts before the new one

**What they want you to say:**
"Since the input is already sorted and non-overlapping, I DON'T need to re-sort — that's the key insight that makes this O(n) instead of O(n log n). Three phases: Phase 1: collect intervals whose end is strictly before the new interval's start — they're fully to the left, no overlap. Phase 2: merge all intervals overlapping with the new interval — an interval overlaps if its START is at or before the new interval's END. I update newInterval's start and end with the min/max. Phase 3: collect the remaining intervals that start after the new interval ends."

**The follow-up:** "Can you do this in O(log n) time?" → Finding where the new interval belongs is O(log n) with binary search. But outputting the result requires O(n) to shift elements. So the read/locate step is O(log n), but the write/output step remains O(n). If the output is allowed to be implicit (e.g., give the answer as a reference to the modified array), then O(log n) is achievable — but modifying a sorted vector inherently requires O(n) for insertion in the worst case.

---

## MENTAL MODEL CHECKPOINT

**Answer all 7 from memory before attempting sign-off.**

**Q1:** For Merge Intervals, why do you sort by START time and not END time? What breaks if you sort by end?
> **A:** Sorting by start ensures that when we process intervals in order, no interval we encounter can START before the current merged interval. If we sorted by end: consider `[1,10]` and `[2,3]`. Sorted by end: `[2,3]` comes first (ends at 3 < 10). We process `[2,3]` and set our current interval to `[2,3]`. Then `[1,10]` starts at 1 — BEFORE the current merged interval's start of 2. Our overlap check `result.back()[1] < interval[0]` computes `3 < 1` = false → we'd merge, but the merge produces `[min(2,1), max(3,10)] = [1,10]`. This happens to work here, but the key point is that sorting by start guarantees we process intervals in the natural left-to-right order and never encounter a start before the current merged start. Sorting by end destroys this property.

**Q2:** For the Non-overlapping Intervals greedy (activity selection), give the exchange argument for why sorting by END time is optimal.
> **A:** Suppose an optimal solution S includes interval Y (ends at time `e_Y`) but NOT interval X (ends at `e_X < e_Y`) where X and Y both start at the same time. Swap Y with X in S: the resulting S' has X instead of Y. Since X ends earlier (`e_X < e_Y`), any interval that was compatible with Y (started after `e_Y`) is still compatible with X (started after `e_Y > e_X` → also after `e_X`). And X might allow ADDITIONAL intervals that started between `e_X` and `e_Y` — intervals that Y blocked. So swapping Y for X never decreases the count of compatible intervals. Therefore, always choosing the earliest-ending interval is at least as good as any other choice, proving the greedy is optimal.

**Q3:** In the 3-phase Insert Interval template, what are the exact conditions for Phase 1 and Phase 2? Why does one use `<` and the other use `<=`?
> **A:** Phase 1 condition: `intervals[i][1] < newInterval[0]` — collect intervals whose END is strictly less than the new interval's start. These are fully to the LEFT. We use `<` because if an interval's end EQUALS the new interval's start (they TOUCH), they should be merged — this is a Phase 2 case, not Phase 1. Phase 2 condition: `intervals[i][0] <= newInterval[1]` — merge intervals whose START is at or before the new interval's END. We use `<=` because if an interval's start EQUALS the new interval's end (they touch), they should still be merged. Both Phase 1 and Phase 2 conditions handle the "touching = merging" convention consistently.

**Q4:** For Meeting Rooms II (min-heap approach), explain exactly what the heap represents at any point during the algorithm, and why the heap's final size is the answer.
> **A:** The min-heap at any point contains the END TIMES of all meetings that are CURRENTLY ONGOING — one entry per meeting room in use. The heap's top is the meeting ending soonest (the "most likely to free up" room). When we process a new meeting: if the earliest-ending meeting (heap top) has ended (≤ new meeting start), we pop it (room is now free) and push the new meeting's end time (room is reused). If no meeting has ended, we push the new meeting's end time (a new room is opened). At the end, every element in the heap represents a room that still has an ongoing meeting (or whose last meeting hasn't ended yet from our perspective). The heap size = number of rooms that were allocated = minimum rooms needed to accommodate all meetings without conflict.

**Q5:** Describe the line sweep technique. What is an "event"? How do you handle ties (same timestamp) correctly?
> **A:** Line sweep models an interval `[s, e]` as two events: a "start" event at time `s` with value `+1`, and an "end" event at time `e` with value `-1`. We sort all events by time. Then we sweep from left to right, maintaining a running sum: add each event's value as we encounter it. The running sum at any point = the number of intervals currently active. The maximum running sum = the peak (e.g., minimum rooms needed). **Tie-breaking:** if a meeting ends at time T and another starts at time T, they can share a room — so we want to count the room as freed before it's reused. Process END events (-1) before START events (+1) at the same time T. In C++, using `pair<int, int>` with `(time, -1)` for ends and `(time, +1)` for starts, sorting pairs lexicographically: `(T, -1) < (T, +1)` since `-1 < 1`. End events are automatically processed first. No custom comparator needed.

**Q6:** In Interval List Intersections (two pointers), you always advance the pointer whose interval ends sooner. Prove this is correct — i.e., that the interval you advance cannot have any more intersections.
> **A:** Suppose we're at `A[i]` and `B[j]`, and `A[i].end < B[j].end`. We advance `i`. Claim: `A[i]` cannot intersect with `B[j+1]`, `B[j+2]`, etc. Proof: since `B` is sorted and non-overlapping, `B[j+1].start >= B[j].end > A[i].end`. Since `B[j+1]` starts AFTER `A[i]` ends, `A[i]` and `B[j+1]` cannot overlap (the overlap condition requires `max(A[i].start, B[j+1].start) <= min(A[i].end, B[j+1].end)` → `B[j+1].start <= A[i].end` → but `B[j+1].start > A[i].end` → no overlap). Similarly for all future `B[j+k]`. So advancing `i` is safe — we don't miss any intersections involving `A[i]`.

**Q7:** What is the difference between Merge Intervals (LC 56) and Insert Interval (LC 57) in terms of input guarantees and optimal time complexity?
> **A:** Merge Intervals (LC 56): input is an arbitrary, possibly unsorted list of overlapping intervals. No invariant about the input. Must sort first: O(n log n). Insert Interval (LC 57): input is a sorted, non-overlapping list of intervals (a maintained invariant). One new interval to insert. Since the input is already sorted and non-overlapping, we can use the 3-phase O(n) linear scan instead of re-sorting. Insert Interval's optimal complexity is O(n) time, not O(n log n). The key difference: Insert Interval exploits the sorted + non-overlapping invariant of the input; Merge Intervals cannot (no such invariant exists). Using O(n log n) for Insert Interval is "correct" but misses the point of the problem.

---

## C++ INTERVALS CHEAT SHEET

```cpp
// ============================================================
// SORT KEYS — WHICH PROBLEM, WHICH SORT
// ============================================================
// Merge Intervals (LC 56):            sort by START (default vec<vec<int>> sort) ✓
// Insert Interval (LC 57):            NO SORT — input already sorted, use 3-phase
// Non-overlapping / Activity (LC 435): sort by END time
// Meeting Rooms I (LC 252):           sort by START, check consecutive overlap
// Meeting Rooms II (LC 253):          sort by START (for min-heap approach)
// Interval Intersections (LC 986):    inputs already sorted, two pointers
// Arrows (LC 452):                    sort by END time (same as activity selection)
// Employee Free Time (LC 759):        sort all by START, then merge

// ============================================================
// MERGE INTERVALS — TEMPLATE (O(n log n))
// ============================================================
sort(intervals.begin(), intervals.end());  // by start
vector<vector<int>> result;
for (auto& iv : intervals) {
    if (result.empty() || result.back()[1] < iv[0])  // non-overlap: < (not <=)
        result.push_back(iv);
    else
        result.back()[1] = max(result.back()[1], iv[1]);  // merge: take max end
}

// ============================================================
// INSERT INTERVAL — 3-PHASE (O(n))
// ============================================================
// Phase 1: intervals[i][1] < newInterval[0]  → push as-is
// Phase 2: intervals[i][0] <= newInterval[1] → merge (update newInterval min/max)
// Phase 3: remaining → push as-is
// Then push newInterval between Phase 2 and Phase 3.

// ============================================================
// ACTIVITY SELECTION (NON-OVERLAPPING) — TEMPLATE (O(n log n))
// ============================================================
sort(intervals.begin(), intervals.end(), [](auto& a, auto& b) { return a[1] < b[1]; });
int prevEnd = INT_MIN, removals = 0;
for (auto& iv : intervals) {
    if (iv[0] >= prevEnd) prevEnd = iv[1];  // >= means touching is allowed
    else removals++;
}

// ============================================================
// MEETING ROOMS II — MIN-HEAP (O(n log n))
// ============================================================
sort(intervals.begin(), intervals.end());
priority_queue<int, vector<int>, greater<int>> minHeap;
for (auto& iv : intervals) {
    if (!minHeap.empty() && minHeap.top() <= iv[0]) minHeap.pop();  // free a room
    minHeap.push(iv[1]);
}
return minHeap.size();

// ============================================================
// LINE SWEEP — TEMPLATE (O(n log n))
// ============================================================
vector<pair<int,int>> events;
for (auto& iv : intervals) {
    events.push_back({iv[0], +1});   // start event
    events.push_back({iv[1], -1});   // end event
}
sort(events.begin(), events.end()); // (t,-1) < (t,+1) → ends before starts at same t ✓
int maxCount = 0, curr = 0;
for (auto& [t, d] : events) { curr += d; maxCount = max(maxCount, curr); }

// ============================================================
// INTERVAL INTERSECTIONS — TWO POINTERS (O(n + m))
// ============================================================
int i = 0, j = 0;
while (i < A.size() && j < B.size()) {
    int lo = max(A[i][0], B[j][0]), hi = min(A[i][1], B[j][1]);
    if (lo <= hi) result.push_back({lo, hi});
    if (A[i][1] < B[j][1]) i++;  // advance the earlier-ending
    else j++;
}

// ============================================================
// OVERLAP CHECK SUMMARY
// ============================================================
// [a,b] and [c,d] OVERLAP if max(a,c) <= min(b,d)       → touching = overlap
// [a,b] and [c,d] NOT OVERLAP if b < c OR d < a          → strict gap required
// "Back-to-back OK" (end == next start = no conflict): use >= in keep condition
// "Touching = conflict":                                  use >  in keep condition

// ============================================================
// SORTED MAP FOR ONLINE QUERIES (My Calendar, Data Stream)
// ============================================================
map<int, int> booked;  // start → end
// Find first interval with start >= new_start:
auto it = booked.lower_bound(new_start);
// Check it: if it != end && it->first < new_end → OVERLAP
// Check prev(it): if it != begin && prev(it)->second > new_start → OVERLAP
// If no overlap: booked[new_start] = new_end;
```

---

## WHAT TO DO AFTER COMPLETING THIS PATTERN

1. Write the Merge Intervals template from memory three times — sorting step, overlap condition (`< interval[0]`), merge using `max()`. Under 5 minutes each time with zero bugs.

2. Write the Insert Interval 3-phase template from memory — all three while-loop conditions, the push of `newInterval` between Phase 2 and Phase 3. Trace through `[[1,2],[3,5],[6,7]]` + `newInterval=[4,8]` manually step by step.

3. Deliberately implement Non-overlapping Intervals by sorting by START (wrong) and observe how it fails on `[[1,100],[2,3],[4,5]]`. Then implement the correct sort-by-end version. The deliberate failure cements the "sort by end" intuition permanently.

4. In REVISION-TRACKER.md, add these for weekly revisit:
   - LC 57 (Insert Interval) — 3-phase approach, Phase 1/2 conditions
   - LC 253 (Meeting Rooms II) — both heap and line sweep approaches
   - LC 986 (Interval Intersections) — "advance earlier-ending" justification

5. Connecting forward to Pattern 12 (Graphs BFS/DFS): interval problems in 2D (overlap of rectangles, coverage in a grid) are graph problems where each cell or region is a node and overlapping regions are edges. Pattern 11 is 1D; Pattern 12 extends to 2D grids.

6. Connecting forward to Pattern 14 (Shortest Path): interval scheduling with deadlines and profits (weighted job scheduling) is a DP+binary-search problem. The "compatible intervals" lookup uses the same binary search on end times that appears in the activity selection sort.

7. Connecting forward to Pattern 27 (Segment Tree): the "My Calendar III" problem in O(n log n) with segment tree (lazy propagation) requires knowing Pattern 11's line sweep concept and Pattern 27's range-update technique. Pattern 11's map-based line sweep is the prerequisite.

---

## SIGN-OFF CRITERIA

**Tier 1 (Non-negotiable):**
- [ ] Merge Intervals template from memory — correct sort key, correct overlap condition `<`, correct `max()` merge — under 5 minutes
- [ ] Can state the exact Phase 1 and Phase 2 conditions for Insert Interval (`< newInterval[0]` and `<= newInterval[1]`) and explain WHY one is `<` and the other is `<=`
- [ ] Can explain in 30 seconds why activity selection sorts by END time (exchange argument)

**Tier 2:**
- [ ] LC 56 (Merge Intervals) — under 5 minutes, no bugs
- [ ] LC 57 (Insert Interval) — 3-phase O(n), not O(n log n) re-sort, correct conditions
- [ ] LC 253 (Meeting Rooms II) — min-heap approach with correct `<=` comparison

**Tier 3:**
- [ ] LC 435 (Non-overlapping Intervals) — sort by end, greedy proof stated
- [ ] LC 986 (Interval Intersections) — two-pointer with "advance earlier-ending" justified
- [ ] Line sweep template from memory — event pairs, pair tie-breaking, running max

**Tier 4 (attempt before sign-off):**
- [ ] LC 452 (Minimum Arrows) — note the `>` vs `>=` difference from LC 435
- [ ] LC 352 (Data Stream as Disjoint Intervals) — sorted map with touching interval handling

**Sign-off test:** I give you one unseen interval problem. You must:
1. Identify which of the 5 interval sub-patterns it belongs to
2. State the correct sort key (start or end) and justify it
3. Write complete C++ solution
4. State time and space complexity

Type `SIGN OFF P11` when ready.

---

*Pattern 11 Complete — Intervals*
*Next: Pattern 12 — Graphs: BFS and DFS*
