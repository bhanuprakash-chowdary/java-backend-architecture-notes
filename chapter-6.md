# Interview Survival Guide
## Chapter 6: Behavioral Mastery, DSA Patterns & Full Mock Interview Simulation
### (Weeks 3–4 & 23–26)

---

## PART 1 — DSA CODING SCREEN PATTERNS

The coding screen is a filter, not a showcase. The goal is to pass it efficiently and spend your energy on design rounds. You need 6 core patterns cold — the same underlying techniques appear in 90% of medium-difficulty problems.

---

### Pattern 1 — Two Pointers

**When to use:** Sorted arrays, finding pairs with a target sum, removing duplicates, palindrome checks.

**Template:**
```java
int left = 0, right = arr.length - 1;
while (left < right) {
    int sum = arr[left] + arr[right];
    if (sum == target) { /* found */ left++; right--; }
    else if (sum < target) left++;
    else right--;
}
```

**Problem: Container With Most Water**
```java
public int maxArea(int[] height) {
    int left = 0, right = height.length - 1;
    int maxWater = 0;

    while (left < right) {
        int water = Math.min(height[left], height[right]) * (right - left);
        maxWater = Math.max(maxWater, water);
        // Move the shorter side inward — only way to potentially increase area
        if (height[left] < height[right]) left++;
        else right--;
    }
    return maxWater;
}
// Time: O(n)  Space: O(1)
```

**Problem: 3Sum**
```java
public List<List<Integer>> threeSum(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    Arrays.sort(nums);  // sort enables two pointers

    for (int i = 0; i < nums.length - 2; i++) {
        if (i > 0 && nums[i] == nums[i - 1]) continue;  // skip duplicates

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
// Time: O(n²)  Space: O(1)
```

---

### Pattern 2 — Sliding Window

**When to use:** Subarray/substring problems with a contiguous constraint (max sum, longest with condition, minimum size).

**Fixed-size window template:**
```java
// Sum of every window of size k
int windowSum = 0;
for (int i = 0; i < k; i++) windowSum += arr[i];
int maxSum = windowSum;

for (int i = k; i < arr.length; i++) {
    windowSum += arr[i] - arr[i - k];  // slide: add new, remove old
    maxSum = Math.max(maxSum, windowSum);
}
```

**Variable-size window template (expand right, shrink left):**
```java
int left = 0;
Map<Character, Integer> freq = new HashMap<>();

for (int right = 0; right < s.length(); right++) {
    // Expand: add s[right] to window
    freq.merge(s.charAt(right), 1, Integer::sum);

    // Shrink: while window violates constraint, move left
    while (windowIsInvalid(freq)) {
        freq.merge(s.charAt(left), -1, Integer::sum);
        if (freq.get(s.charAt(left)) == 0) freq.remove(s.charAt(left));
        left++;
    }

    // Current window [left, right] is valid — check for answer
    maxLen = Math.max(maxLen, right - left + 1);
}
```

**Problem: Longest Substring Without Repeating Characters**
```java
public int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> lastSeen = new HashMap<>();
    int maxLen = 0;
    int left = 0;

    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        if (lastSeen.containsKey(c)) {
            // Jump left past the last occurrence (don't go backwards)
            left = Math.max(left, lastSeen.get(c) + 1);
        }
        lastSeen.put(c, right);
        maxLen = Math.max(maxLen, right - left + 1);
    }
    return maxLen;
}
// Time: O(n)  Space: O(min(n, charset))
```

**Problem: Minimum Window Substring**
```java
public String minWindow(String s, String t) {
    Map<Character, Integer> need = new HashMap<>();
    for (char c : t.toCharArray()) need.merge(c, 1, Integer::sum);

    int have = 0, required = need.size();
    int left = 0, minLen = Integer.MAX_VALUE, minStart = 0;
    Map<Character, Integer> window = new HashMap<>();

    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        window.merge(c, 1, Integer::sum);
        if (need.containsKey(c) && window.get(c).equals(need.get(c))) have++;

        while (have == required) {
            if (right - left + 1 < minLen) { minLen = right - left + 1; minStart = left; }
            char lc = s.charAt(left);
            window.merge(lc, -1, Integer::sum);
            if (need.containsKey(lc) && window.get(lc) < need.get(lc)) have--;
            left++;
        }
    }
    return minLen == Integer.MAX_VALUE ? "" : s.substring(minStart, minStart + minLen);
}
// Time: O(|s| + |t|)  Space: O(|t|)
```

---

### Pattern 3 — Binary Search (and its variants)

**When to use:** Sorted array, "find minimum X such that condition holds" (binary search on the answer).

**Standard template:**
```java
int left = 0, right = arr.length - 1;
while (left <= right) {
    int mid = left + (right - left) / 2;  // avoids overflow vs (left+right)/2
    if (arr[mid] == target) return mid;
    else if (arr[mid] < target) left = mid + 1;
    else right = mid - 1;
}
return -1;
```

**Binary search on the answer (most powerful variant):**
```java
// "Find minimum speed k such that all tasks complete within time T"
int left = minPossible, right = maxPossible;
while (left < right) {
    int mid = left + (right - left) / 2;
    if (canAchieve(mid)) right = mid;   // mid works — try smaller
    else left = mid + 1;                // mid fails — need larger
}
return left;  // left == right at termination — the minimum valid value
```

**Problem: Search in Rotated Sorted Array**
```java
public int search(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] == target) return mid;

        if (nums[left] <= nums[mid]) {  // left half is sorted
            if (nums[left] <= target && target < nums[mid]) right = mid - 1;
            else left = mid + 1;
        } else {  // right half is sorted
            if (nums[mid] < target && target <= nums[right]) left = mid + 1;
            else right = mid - 1;
        }
    }
    return -1;
}
// Time: O(log n)
```

**Problem: Koko Eating Bananas (binary search on answer)**
```java
public int minEatingSpeed(int[] piles, int h) {
    int left = 1, right = Arrays.stream(piles).max().getAsInt();

    while (left < right) {
        int mid = left + (right - left) / 2;
        long hours = 0;
        for (int pile : piles) hours += (pile + mid - 1) / mid;  // ceil division

        if (hours <= h) right = mid;   // can eat slower
        else left = mid + 1;           // need to eat faster
    }
    return left;
}
// Time: O(n log(max_pile))
```

---

### Pattern 4 — Trees (DFS / BFS)

**DFS recursive template:**
```java
public void dfs(TreeNode node) {
    if (node == null) return;
    // PRE-ORDER: process node here
    dfs(node.left);
    // IN-ORDER: process node here
    dfs(node.right);
    // POST-ORDER: process node here
}
```

**DFS iterative (avoids stack overflow on deep trees):**
```java
public List<Integer> inorderIterative(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    Deque<TreeNode> stack = new ArrayDeque<>();
    TreeNode curr = root;

    while (curr != null || !stack.isEmpty()) {
        while (curr != null) { stack.push(curr); curr = curr.left; }
        curr = stack.pop();
        result.add(curr.val);
        curr = curr.right;
    }
    return result;
}
```

**BFS template (level-order):**
```java
public List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> result = new ArrayList<>();
    if (root == null) return result;
    Queue<TreeNode> queue = new ArrayDeque<>();
    queue.offer(root);

    while (!queue.isEmpty()) {
        int levelSize = queue.size();  // snapshot size = nodes at this level
        List<Integer> level = new ArrayList<>();
        for (int i = 0; i < levelSize; i++) {
            TreeNode node = queue.poll();
            level.add(node.val);
            if (node.left != null) queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }
        result.add(level);
    }
    return result;
}
```

**Problem: Validate BST**
```java
public boolean isValidBST(TreeNode root) {
    return validate(root, Long.MIN_VALUE, Long.MAX_VALUE);
}

private boolean validate(TreeNode node, long min, long max) {
    if (node == null) return true;
    if (node.val <= min || node.val >= max) return false;
    return validate(node.left, min, node.val)
        && validate(node.right, node.val, max);
}
// Time: O(n)  Space: O(h) — h = height
```

**Problem: Lowest Common Ancestor**
```java
public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    if (root == null || root == p || root == q) return root;

    TreeNode left = lowestCommonAncestor(root.left, p, q);
    TreeNode right = lowestCommonAncestor(root.right, p, q);

    // If both sides found something, root is the LCA
    if (left != null && right != null) return root;
    return left != null ? left : right;
}
// Time: O(n)
```

**Problem: Maximum Path Sum in Binary Tree**
```java
private int globalMax = Integer.MIN_VALUE;

public int maxPathSum(TreeNode root) {
    dfsMaxPath(root);
    return globalMax;
}

private int dfsMaxPath(TreeNode node) {
    if (node == null) return 0;
    // Don't take negative branches
    int left = Math.max(0, dfsMaxPath(node.left));
    int right = Math.max(0, dfsMaxPath(node.right));
    // Best path through this node (can go both directions)
    globalMax = Math.max(globalMax, node.val + left + right);
    // Return best single-direction path (for parent's consideration)
    return node.val + Math.max(left, right);
}
// Time: O(n)
```

---

### Pattern 5 — Graphs (DFS / BFS / Union-Find)

**Graph DFS template (adjacency list):**
```java
public void dfs(int node, Map<Integer, List<Integer>> graph, boolean[] visited) {
    visited[node] = true;
    for (int neighbor : graph.getOrDefault(node, Collections.emptyList())) {
        if (!visited[neighbor]) dfs(neighbor, graph, visited);
    }
}
```

**Graph BFS template (shortest path in unweighted):**
```java
public int bfs(int start, int target, Map<Integer, List<Integer>> graph) {
    Queue<Integer> queue = new ArrayDeque<>();
    Set<Integer> visited = new HashSet<>();
    queue.offer(start);
    visited.add(start);
    int steps = 0;

    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            int curr = queue.poll();
            if (curr == target) return steps;
            for (int neighbor : graph.getOrDefault(curr, Collections.emptyList())) {
                if (!visited.contains(neighbor)) {
                    visited.add(neighbor);
                    queue.offer(neighbor);
                }
            }
        }
        steps++;
    }
    return -1;
}
```

**Problem: Number of Islands (grid DFS)**
```java
public int numIslands(char[][] grid) {
    int count = 0;
    for (int r = 0; r < grid.length; r++) {
        for (int c = 0; c < grid[0].length; c++) {
            if (grid[r][c] == '1') {
                count++;
                sink(grid, r, c);  // DFS flood-fill marks visited
            }
        }
    }
    return count;
}

private void sink(char[][] grid, int r, int c) {
    if (r < 0 || r >= grid.length || c < 0 || c >= grid[0].length
            || grid[r][c] == '0') return;
    grid[r][c] = '0';  // mark visited (in-place)
    sink(grid, r + 1, c); sink(grid, r - 1, c);
    sink(grid, r, c + 1); sink(grid, r, c - 1);
}
// Time: O(m×n)
```

**Union-Find (Disjoint Set Union) — for connectivity problems:**
```java
class UnionFind {
    private final int[] parent, rank;

    public UnionFind(int n) {
        parent = new int[n];
        rank = new int[n];
        for (int i = 0; i < n; i++) parent[i] = i;
    }

    public int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]);  // path compression
        return parent[x];
    }

    public boolean union(int x, int y) {
        int px = find(x), py = find(y);
        if (px == py) return false;  // already connected
        if (rank[px] < rank[py]) { int tmp = px; px = py; py = tmp; }
        parent[py] = px;
        if (rank[px] == rank[py]) rank[px]++;
        return true;
    }

    public boolean connected(int x, int y) { return find(x) == find(y); }
}

// Number of Islands using Union-Find
public int numIslands(char[][] grid) {
    int m = grid.length, n = grid[0].length;
    UnionFind uf = new UnionFind(m * n);
    int islands = 0;
    int[][] dirs = {{1,0},{-1,0},{0,1},{0,-1}};

    for (int r = 0; r < m; r++) {
        for (int c = 0; c < n; c++) {
            if (grid[r][c] == '1') {
                islands++;
                for (int[] d : dirs) {
                    int nr = r + d[0], nc = c + d[1];
                    if (nr >= 0 && nr < m && nc >= 0 && nc < n && grid[nr][nc] == '1') {
                        if (uf.union(r * n + c, nr * n + nc)) islands--;
                    }
                }
            }
        }
    }
    return islands;
}
```

**Topological Sort (DAG — dependency ordering):**
```java
// Kahn's Algorithm (BFS-based)
public int[] topologicalSort(int n, int[][] edges) {
    int[] indegree = new int[n];
    List<List<Integer>> adj = new ArrayList<>();
    for (int i = 0; i < n; i++) adj.add(new ArrayList<>());

    for (int[] edge : edges) { adj.get(edge[0]).add(edge[1]); indegree[edge[1]]++; }

    Queue<Integer> queue = new ArrayDeque<>();
    for (int i = 0; i < n; i++) if (indegree[i] == 0) queue.offer(i);

    int[] order = new int[n];
    int idx = 0;
    while (!queue.isEmpty()) {
        int node = queue.poll();
        order[idx++] = node;
        for (int neighbor : adj.get(node)) {
            if (--indegree[neighbor] == 0) queue.offer(neighbor);
        }
    }
    return idx == n ? order : new int[0];  // empty if cycle detected
}
```

---

### Pattern 6 — Dynamic Programming

**The DP mindset:** Define `dp[i]` as "the answer for the subproblem ending at i (or of size i)." Find the recurrence: `dp[i] = f(dp[i-1], dp[i-2], ...)`. Fill bottom-up.

**1D DP:**
```java
// Maximum Subarray (Kadane's)
public int maxSubArray(int[] nums) {
    int maxSum = nums[0], currentSum = nums[0];
    for (int i = 1; i < nums.length; i++) {
        currentSum = Math.max(nums[i], currentSum + nums[i]);
        maxSum = Math.max(maxSum, currentSum);
    }
    return maxSum;
}

// Climbing Stairs (Fibonacci DP)
public int climbStairs(int n) {
    if (n <= 2) return n;
    int prev2 = 1, prev1 = 2;
    for (int i = 3; i <= n; i++) {
        int curr = prev1 + prev2;
        prev2 = prev1; prev1 = curr;
    }
    return prev1;
}

// House Robber
public int rob(int[] nums) {
    int prev2 = 0, prev1 = 0;
    for (int num : nums) {
        int curr = Math.max(prev1, prev2 + num);
        prev2 = prev1; prev1 = curr;
    }
    return prev1;
}
```

**2D DP:**
```java
// Longest Common Subsequence
public int longestCommonSubsequence(String text1, String text2) {
    int m = text1.length(), n = text2.length();
    int[][] dp = new int[m + 1][n + 1];

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (text1.charAt(i - 1) == text2.charAt(j - 1))
                dp[i][j] = dp[i - 1][j - 1] + 1;
            else
                dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
        }
    }
    return dp[m][n];
}

// 0/1 Knapsack
public int knapsack(int[] weights, int[] values, int capacity) {
    int n = weights.length;
    int[][] dp = new int[n + 1][capacity + 1];

    for (int i = 1; i <= n; i++) {
        for (int w = 0; w <= capacity; w++) {
            dp[i][w] = dp[i - 1][w];  // don't take item i
            if (weights[i - 1] <= w)
                dp[i][w] = Math.max(dp[i][w], dp[i - 1][w - weights[i - 1]] + values[i - 1]);
        }
    }
    return dp[n][capacity];
}

// Coin Change (unbounded knapsack variant)
public int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1);  // infinity sentinel
    dp[0] = 0;

    for (int i = 1; i <= amount; i++) {
        for (int coin : coins) {
            if (coin <= i) dp[i] = Math.min(dp[i], dp[i - coin] + 1);
        }
    }
    return dp[amount] > amount ? -1 : dp[amount];
}
```

**Interval DP:**
```java
// Merge Intervals
public int[][] merge(int[][] intervals) {
    Arrays.sort(intervals, (a, b) -> a[0] - b[0]);
    List<int[]> merged = new ArrayList<>();

    for (int[] interval : intervals) {
        if (merged.isEmpty() || merged.get(merged.size() - 1)[1] < interval[0]) {
            merged.add(interval);
        } else {
            merged.get(merged.size() - 1)[1] =
                Math.max(merged.get(merged.size() - 1)[1], interval[1]);
        }
    }
    return merged.toArray(new int[0][]);
}
```

---

### Pattern 7 — Heap / Priority Queue

**When to use:** Top-K problems, K-th largest/smallest, merge K sorted lists, streaming median.

```java
// Top K Frequent Elements
public int[] topKFrequent(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int n : nums) freq.merge(n, 1, Integer::sum);

    // Min-heap of size k — keeps top k elements
    PriorityQueue<Integer> heap = new PriorityQueue<>(Comparator.comparingInt(freq::get));

    for (int num : freq.keySet()) {
        heap.offer(num);
        if (heap.size() > k) heap.poll();  // remove least frequent
    }

    return heap.stream().mapToInt(Integer::intValue).toArray();
}
// Time: O(n log k)

// K-th Largest Element (QuickSelect O(n) average)
public int findKthLargest(int[] nums, int k) {
    return quickSelect(nums, 0, nums.length - 1, nums.length - k);
}

private int quickSelect(int[] nums, int left, int right, int k) {
    int pivotIdx = partition(nums, left, right);
    if (pivotIdx == k) return nums[pivotIdx];
    return pivotIdx < k
        ? quickSelect(nums, pivotIdx + 1, right, k)
        : quickSelect(nums, left, pivotIdx - 1, k);
}

private int partition(int[] nums, int left, int right) {
    int pivot = nums[right], i = left;
    for (int j = left; j < right; j++) {
        if (nums[j] <= pivot) { int tmp = nums[i]; nums[i++] = nums[j]; nums[j] = tmp; }
    }
    int tmp = nums[i]; nums[i] = nums[right]; nums[right] = tmp;
    return i;
}

// Merge K Sorted Lists
public ListNode mergeKLists(ListNode[] lists) {
    PriorityQueue<ListNode> heap = new PriorityQueue<>(Comparator.comparingInt(n -> n.val));
    for (ListNode list : lists) if (list != null) heap.offer(list);

    ListNode dummy = new ListNode(0), curr = dummy;
    while (!heap.isEmpty()) {
        curr.next = heap.poll();
        curr = curr.next;
        if (curr.next != null) heap.offer(curr.next);
    }
    return dummy.next;
}
// Time: O(N log k)  where N = total nodes, k = number of lists
```

---

### DSA Quick-Reference: Time/Space Complexities

| Operation | Data Structure | Time |
|-----------|---------------|------|
| HashMap get/put | `HashMap` | O(1) avg |
| TreeMap get/put | `TreeMap` (Red-Black) | O(log n) |
| Heap insert/poll | `PriorityQueue` | O(log n) |
| Binary search | Sorted array | O(log n) |
| DFS/BFS on graph | Adjacency list | O(V + E) |
| Merge sort | Array | O(n log n) |
| Quick select | Array | O(n) avg, O(n²) worst |
| Union-Find find | Path-compressed | O(α(n)) ≈ O(1) |
| Trie insert/search | Trie | O(L) — L = key length |

---

### Coding Interview Communication Protocol

Use this exact structure for every problem:

**Step 1 — Clarify (2 min):**
"Before I code, let me make sure I understand the problem. [restate in your own words]. A few quick questions: Can input be empty? Are there duplicates? What's the input size range — I want to know if O(n²) is acceptable."

**Step 2 — Think aloud, state approach (2 min):**
"My first instinct is [brute force — always mention it]. That's O(n²) / O(n) space. I can do better with [your chosen pattern] because [one-sentence reason]. Let me walk through the logic before coding."

**Step 3 — Code with narration (15 min):**
Write left-to-right, top-to-bottom. Name variables clearly. Say out loud what each block does as you write it. Never be silent for more than 30 seconds.

**Step 4 — Trace through example (3 min):**
"Let me trace through the example to verify. [walk through input step by step, show state changes]"

**Step 5 — State complexity (1 min):**
"Time complexity is O(n log n) because [reason]. Space is O(n) for [the structure]. If we needed to reduce space, we could [optimization]."

**Step 6 — Edge cases (1 min):**
"Edge cases I'd handle: empty input, single element, all duplicates, integer overflow on sum/product."

---

## PART 2 — ADDITIONAL STAR STORIES

---

### STAR Story 6: Handling Ambiguous Requirements

**Situation:** Mid-project, the product team asked for "real-time payment status updates on the dashboard." No further specification. The backend team interpreted "real-time" differently — someone assumed polling every 30 seconds, I assumed WebSocket push. Two days of parallel development later, we realized we'd built two different systems.

**Task:** Align the team and deliver a single agreed-upon implementation without losing both teams' work.

**Action:** I called a 30-minute sync and started with the user story: "A partner should see a payment move from 'Processing' to 'Completed' without manually refreshing." I then drew the latency spectrum on a whiteboard: polling every 30s = average 15s delay, polling every 5s = high DB load, WebSocket push = < 1s delay with low overhead at our scale. I presented a decision matrix: implementation effort, server load impact, and user experience for each approach. WebSocket won on user experience; polling was simpler but objectively worse. We adopted WebSocket, I integrated the polling fallback the other developer had built as a reconnection recovery mechanism (chapter 4 story), and we shipped a better product than either team had individually designed. I also proposed creating a lightweight "technical requirement glossary" for terms like "real-time," "near-real-time," and "eventual" to avoid this ambiguity in future sprints.

**Result:** Shipped a WebSocket implementation with polling fallback in 5 days (3 days after alignment). The glossary became a team document that prevented two similar misalignments in the following quarter.

---

### STAR Story 7: Proactively Preventing a Problem

**Situation:** During a code review, I noticed that a new API endpoint for bulk lead assignment accepted an array of lead IDs with no upper bound limit. The endpoint did synchronous DB writes for each lead.

**Task:** The PR would have shipped to production by the next morning if approved as-is. I needed to flag the risk and propose a fix without blocking the developer's work unnecessarily.

**Action:** I ran a quick load simulation in my local environment: 500 leads submitted synchronously → 47 seconds response time → Tomcat request thread blocked for 47 seconds → thread pool saturated after 10 concurrent requests → all other API endpoints 503. I added a detailed comment in the PR with the reproduction steps and a specific failure scenario: "If a partner submits 500 leads during peak hours, this endpoint will saturate the thread pool and take down the entire payment API for 30+ seconds." I proposed three options ranked by implementation effort: (1) Add a hard limit of 50 leads per request (30 minutes), (2) Return a job ID and process async (half a day), (3) Process in parallel batches of 10 using CompletableFuture (1 hour). The developer chose option 3 (parallel batching) — slightly more work but better UX than option 1's hard limit.

**Result:** The fix was implemented before the PR was merged. Six weeks later, a partner submitted a 300-lead bulk assignment — it completed in 4.2 seconds instead of the projected 47. No outage. The developer later said this was the most useful code review comment they'd received — it came with a reproduction path, not just a concern.

---

### STAR Story 8: Delivering Under Pressure

**Situation:** Two days before a demo to a major enterprise prospect, the client's CTO specifically asked to see "real-time IoT device telemetry with multi-tenant isolation" in the demo environment. Our ThingsBoard instance had multi-tenant auth, but the telemetry pipeline was configured for a single-tenant demo setup — all device data was visible to all tenants.

**Task:** Fix the tenant data isolation in the telemetry pipeline within 48 hours, in a demo environment, without breaking the existing multi-tenant auth work.

**Action:** I analyzed the telemetry pipeline and found the gap: the device data was flowing through a shared Kafka topic without tenant context in the message headers. The consumer wrote to a shared time-series table without a `tenant_id` partition. I couldn't change the Kafka pipeline in 48 hours, so I took a targeted approach: add `tenant_id` to the time-series table, backfill it from the device registry, and add the Hibernate `@Filter` for tenant isolation on the telemetry query path. This was a localized change — 4 files, 2 new DB migrations. I wrote a migration script, tested it on a clone of the demo environment, and validated the isolation by simulating two tenants with overlapping device names. Demo day: Tenant A saw only their sensors, Tenant B saw only theirs. The CTO specifically called out the isolation model as "production-grade."

**Result:** We won the enterprise contract. The targeted fix took 14 hours. I documented it as a "partial isolation" — the Kafka pipeline still needed proper tenant headers for a complete production implementation, which I added in the following sprint.

---

### STAR Story 9: Disagreeing with a Senior Decision

**Situation:** Our tech lead proposed migrating our monolith payment service to microservices — splitting into 6 separate services — as a Q3 priority. This would be a 3-month engineering effort with no new features delivered during that period.

**Task:** I had a genuine technical concern: our service had ~ 8K requests/day. The operational complexity of microservices — distributed tracing, service discovery, inter-service authentication, eventual consistency — was not justified at this scale. I needed to make this case without being dismissive of the tech lead's vision.

**Action:** I prepared a 15-minute analysis. I quantified: at 8K requests/day, our p99 latency was 180ms on a single service with 3 instances. I pulled stats on typical microservices overhead: 30–50ms per service hop, 3x operational cost (monitoring, deployment, on-call). I showed that the top 3 pain points the team had filed in the last 6 months — slow deploys, test environment flakiness, and a God class with 2000 lines — were solvable with modular monolith patterns: internal module boundaries, improved CI pipeline, and refactoring the God class. I wasn't saying "never microservices." I was saying "not now, at this scale, with these pain points." I also acknowledged the tech lead's valid concern: the team had been burned before by a monolith that became unmaintainable, and the instinct to prevent that recurrence was sound. I proposed a compromise: implement strict module boundaries within the monolith (separate packages, enforced via ArchUnit tests), revisit microservices if we crossed 100K requests/day or added a second team.

**Result:** The tech lead agreed to the modular monolith approach. We spent 3 weeks refactoring into clear modules and adding ArchUnit boundary enforcement. Two quarters later, the codebase was measurably cleaner. The microservices discussion was tabled but not closed — at 80K requests/day six months later, we revisited it with actual scale data to drive the decision.

---

## PART 3 — COMPANY-SPECIFIC RESEARCH FRAMEWORK

---

### How to Research a Company Before an Interview

Run this framework for every company. Takes 90 minutes. Produces 5 talking points per round.

---

**Step 1 — Engineering Blog (30 min)**

Search: `site:engineering.{company}.com` or `{company} engineering blog`.

What to look for:
- Recent system design posts (shows what problems they're solving)
- Tech stack choices (what languages, frameworks, databases)
- Scale numbers (helps you calibrate your examples)
- Engineering culture signals (on-call culture, blameless postmortems, open-source)

**Questions to extract:**
- "I read your post on [specific article] — you mentioned [specific decision]. Can you tell me more about how that plays out in practice?"
- "I noticed you moved from X to Y in [article] — was that primarily a reliability or a performance decision?"

---

**Step 2 — LinkedIn (15 min)**

Search the engineering team, especially your potential manager and interviewers.

Look for:
- Tenure (long tenures = stable team, short = churn — ask about it)
- Background of engineers (ex-FAANG = high bar, domain specialists = values depth)
- Open roles (what are they hiring? Where are they growing?)

---

**Step 3 — Glassdoor / Levels.fyi Interview Reports (15 min)**

Search "{company} software engineer interview."

Extract:
- Specific interview rounds (coding screen, system design, behavioral)
- Questions that appeared multiple times
- Red flags in experience reports

---

**Step 4 — Products & Domain (15 min)**

Use the product. Read the pricing page. Read the API docs if it's a developer tool.

What to understand:
- Who are their customers? (B2B, B2C, enterprise, SMB)
- What's the core technical problem they're solving?
- What failure modes would a bug or outage cause for their customers?

This directly feeds your behavioral answers. "I'm particularly excited about the reliability problem you're solving for — the combination of payment processing and IoT data pipelines is exactly the intersection I've been working in."

---

**Step 5 — Prepare 3 Specific Questions per Round (15 min)**

For each round type, prepare 3 questions. Ask 1–2 per round. Never ask about salary, benefits, or vacation.

**For system design interviewers:**
- "What's the hardest scaling problem the team has encountered in the past year?"
- "Are there any architectural decisions the team wishes it had made differently?"
- "How do you approach the trade-off between operational simplicity and technical sophistication?"

**For behavioral interviewers:**
- "What does a typical first 90 days look like for someone in this role?"
- "How does the team handle on-call? Is there a blameless postmortem culture?"
- "What's an example of a technical decision the team made that you're proud of?"

**For engineering managers:**
- "How do you support engineers moving toward Senior / Staff level?"
- "How does the team balance feature development with technical debt?"
- "What does success look like in this role at the 6-month mark?"

---

### Company Research Template (fill this out for each interview)

```
Company: ___________________________
Interview Date: ___________________________

Tech Stack (from job description + blog):
  Languages: ___________________________
  Databases: ___________________________
  Infrastructure: ___________________________

Their Core Engineering Challenges (from blog/articles):
  1. ___________________________
  2. ___________________________

How My Experience Applies:
  Project → Their Challenge:
  1. ___________________________
  2. ___________________________

Questions I'll Ask:
  System Design round: ___________________________
  Behavioral round: ___________________________
  Hiring Manager: ___________________________

One Specific Story to Lead With:
  ___________________________
```

---

## PART 4 — FULL MOCK INTERVIEW SIMULATION

---

### Round 1: Coding Screen (45 minutes)

**Problem:** Given a list of payment transactions, each with `[accountId, amount, timestamp]`, find all accounts where the total amount transferred within any 30-minute window exceeds a given threshold. Flag them for fraud review.

**Walkthrough approach:**

"Let me clarify: we want accounts where ANY 30-minute window's sum exceeds the threshold — not the total sum. And 'window' means a sliding window, not fixed buckets, right?

For each account, I need to find the maximum sum of transactions within any 30-minute window. I'll group transactions by account, sort by timestamp, then use a sliding window with a running sum.

```java
public List<String> flagFraudulentAccounts(
        List<int[]> transactions, int threshold) {
    // Group by accountId
    Map<String, List<int[]>> byAccount = new TreeMap<>();
    for (int[] tx : transactions) {
        byAccount.computeIfAbsent(String.valueOf(tx[0]), k -> new ArrayList<>()).add(tx);
    }

    List<String> flagged = new ArrayList<>();

    for (Map.Entry<String, List<int[]>> entry : byAccount.entrySet()) {
        List<int[]> txns = entry.getValue();
        txns.sort(Comparator.comparingInt(t -> t[2]));  // sort by timestamp

        // Sliding window: keep transactions within 30-min window
        Deque<int[]> window = new ArrayDeque<>();
        long windowSum = 0;

        for (int[] tx : txns) {
            window.addLast(tx);
            windowSum += tx[1];

            // Remove transactions outside 30-minute window
            while (!window.isEmpty()
                   && tx[2] - window.peekFirst()[2] > 1800) {  // 1800 seconds
                windowSum -= window.pollFirst()[1];
            }

            if (windowSum > threshold) {
                flagged.add(entry.getKey());
                break;  // already flagged this account
            }
        }
    }
    return flagged;
}
```

Time complexity: O(n log n) — dominated by sorting per account. Space: O(n).

Edge cases: transactions with the same timestamp, accounts with a single large transaction (threshold comparison handles this), empty input."

---

### Round 2: System Design (60 minutes)

**Problem:** "Design a real-time fraud detection system for payments."

**Your structured response:**

**Step 1 — Requirements (5 min):**
"Let me clarify a few things. Are we building the rules engine or the infrastructure to apply rules? Is this real-time (< 100ms decision during payment processing) or near-real-time (flag after the fact)? What's the scale — 1000 TPS? How complex are the rules — simple thresholds or ML model inference?"

*Assume: real-time, 1000 TPS, rules-based + optional ML, decision in < 50ms*

**Step 2 — Capacity:**
```
1000 TPS
Each check: fetch last 30 days of account history
Account history: 10 transactions/day × 30 days = 300 records × 500 bytes = 150KB per account
Hot accounts (top 10K): 10K × 150KB = 1.5GB — fits in Redis
```

**Step 3 — Architecture:**

```
Payment API
    │ synchronous, < 50ms budget
    ▼
Fraud Check Service
    │
    ├── Rule Engine (fast, configurable)
    │     ├── Velocity check: > N transactions in last M minutes
    │     ├── Amount threshold: single tx > X
    │     ├── Geographic anomaly: IP country ≠ card country
    │     └── Blacklist check: known fraudulent accounts
    │
    ├── Redis (pre-warmed account feature store)
    │     └── tx_count:{accountId}:{bucket}  — time-bucketed counters
    │
    └── Decision: ALLOW / DENY / REVIEW (async human review queue)

Async enrichment (post-decision):
    Kafka "payment-events" → Feature Store Updater → Redis/ClickHouse
    Kafka "fraud-decisions" → ML Training Pipeline → Model Registry
```

**Deep dive — velocity check in < 5ms:**
```java
// Redis time-bucketed counter — 1-minute buckets
public int getTransactionCount(String accountId, int windowMinutes) {
    long now = System.currentTimeMillis() / 60_000;  // current minute bucket
    
    List<String> keys = new ArrayList<>();
    for (int i = 0; i < windowMinutes; i++) {
        keys.add("tx_count:" + accountId + ":" + (now - i));
    }
    
    List<Object> counts = redisTemplate.opsForValue().multiGet(keys);
    return counts.stream()
        .filter(Objects::nonNull)
        .mapToInt(c -> Integer.parseInt(c.toString()))
        .sum();
}

// Increment on each transaction
public void recordTransaction(String accountId) {
    String key = "tx_count:" + accountId + ":" + (System.currentTimeMillis() / 60_000);
    redisTemplate.opsForValue().increment(key);
    redisTemplate.expire(key, Duration.ofMinutes(60));  // keep 60 min of history
}
```

**Failure scenarios:**
- Redis down: fall back to ClickHouse (slower, ~50ms). Circuit breaker wraps Redis call.
- Fraud service down: payment API defaults to ALLOW with async review flag (availability > false blocking).
- High FP rate: rule tuning pipeline, A/B test new rules in shadow mode before production cutover.

---

### Round 3: Java Deep Dive (45 minutes)

**Likely questions for your background — verbatim answers:**

**Q: How does `@Transactional` work internally in Spring?**

"When Spring processes a class annotated with `@Transactional`, the `AnnotationAwareAspectJAutoProxyCreator` BeanPostProcessor creates a proxy for the bean — either a CGLIB subclass or a JDK dynamic proxy, depending on whether the bean implements an interface. The proxy intercepts method calls and, before delegating to the real object, calls `TransactionInterceptor`, which in turn calls the `PlatformTransactionManager`. The transaction manager calls `getTransaction()` on the underlying `DataSource` — specifically it calls `DataSource.getConnection()` and disables auto-commit. After the method returns, the interceptor calls `commit()`. If an unchecked exception propagates, it calls `rollback()`. The entire thing falls apart on self-invocation because `this.method()` bypasses the proxy — you're calling the real object directly."

**Q: What is the difference between `Callable` and `Runnable`?**

"`Runnable.run()` returns void and cannot throw checked exceptions — it's fire-and-forget. `Callable<V>.call()` returns a value and can throw checked exceptions. `ExecutorService.submit(callable)` returns `Future<V>`, which you can use to retrieve the result or catch exceptions from the async computation. The practical implication: if you need a result from an async task, or you need to propagate a checked exception across a thread boundary, use `Callable`. For pure side-effect tasks with no result, `Runnable` is simpler."

**Q: What happens when a Spring bean has both `@Cacheable` and `@Transactional`?**

"Both annotations work via AOP proxies. The order of advice matters: by default, `@Transactional` has lower precedence (higher order number) than `@Cacheable`. So the cache interceptor fires first: if cache hit, it returns without ever entering the transaction. If cache miss, the cache interceptor delegates to the transaction interceptor, which starts a transaction, runs the method, commits, then the cache interceptor stores the result. The subtle issue: if the `@Transactional` method throws an exception that triggers a rollback, but the cache interceptor has already stored the result — you'd cache a value that was rolled back. In practice, Spring's cache abstraction doesn't cache on exception (`unless` condition), so this is safe by default. But it's a real concern if you've customized the cache error handler."

---

### Round 4: Behavioral / Managerial (45 minutes)

**New questions + answers beyond Chapter 4:**

**Q: Tell me about a time you failed.**

"The most significant failure was during the initial deployment of the idempotent webhook system. I designed a 5-layer system and was confident it was correct — I had unit tests for each layer. What I didn't test was the integration between layers under a specific race condition: when two Kafka consumer threads processed the same event simultaneously on startup (before the Redis dedup key was written by the first thread). In that 10-millisecond window, both threads would pass the SETNX check. We caught it in staging, not production — but only because I had specifically added high-concurrency integration tests as an afterthought.

The failure wasn't the bug itself — that's engineering. The failure was my confidence level: I said 'this is production-ready' after unit tests alone. The lesson: for any system where correctness is financial-critical, the definition of 'tested' must include concurrency tests at realistic thread counts, not just functional correctness at single-thread. I now have a personal checklist for concurrency-sensitive code: 1. Does it work with 1 thread? 2. Does it work with 10 threads hammering the same resource? 3. Does it work correctly after a crash at each step?"

**Q: How do you stay current with technology?**

"I focus on depth over breadth. I subscribe to a small set of high-signal sources: Martin Fowler's blog for architecture patterns, the High Scalability newsletter for production war stories, and Java Inside Out for JVM internals. I read one technical book per quarter — most recently 'Designing Data-Intensive Applications' by Kleppmann, which directly improved how I reason about distributed systems consistency.

The most useful practice for me is reading post-mortems from companies like Cloudflare, GitHub, and Stripe. They're free, they're real, and they show you failure modes that you'd never see in documentation. I also try to contribute to at least one open-source project per year — not to build a portfolio, but because reading others' production code is the fastest way to learn idioms that books don't cover."

**Q: What's the hardest technical concept you've had to learn?**

"Distributed consistency. I can write a single-node transactional system with confidence — the rules are clear, the tools are well-understood. But when you move to distributed systems, 'transaction' becomes a spectrum rather than a binary. The hardest part wasn't understanding what eventual consistency means in theory — it was developing intuition for *when* it's acceptable. Reading a slightly stale dashboard balance is fine. Crediting a payment twice because two nodes saw different states is not fine.

The mental model that unlocked it for me was 'what's the financial consequence of stale data?' If the answer is 'minor UX inconvenience,' eventual consistency is fine. If the answer is 'money moves incorrectly,' you need strong consistency or a compensation mechanism. That filter made every subsequent distributed systems design decision easier to reason about."

---

## PART 5 — THE 30-DAY INTERVIEW SPRINT PLAN

Use this when you have an active pipeline and interviews scheduled in 4–6 weeks.

---

### Week 1 — Foundations (ensure no gaps)

| Day | Activity | Time |
|-----|----------|------|
| Mon | Re-read Chapter 1 — JVM, Spring Boot startup | 2 hrs |
| Tue | Re-read Chapter 2 — Concurrency deep dive | 2 hrs |
| Wed | LeetCode: 5 Two Pointer + Sliding Window mediums | 2 hrs |
| Thu | Speak Chapter 3 Project Narratives aloud — record yourself | 1 hr |
| Fri | LeetCode: 5 Binary Search mediums | 2 hrs |
| Sat | Mock design: design a URL shortener from scratch, time-boxed to 45 min | 45 min |
| Sun | Review weak areas from the week | 1 hr |

---

### Week 2 — Design Depth

| Day | Activity | Time |
|-----|----------|------|
| Mon | Re-read Chapter 4 — HLD Framework + CAP/Saga/Outbox | 2 hrs |
| Tue | Re-read Chapter 5 — Twitter feed + job scheduler deep dive | 2 hrs |
| Wed | LeetCode: 5 Tree DFS/BFS mediums | 2 hrs |
| Thu | Mock design: design a notification system with 10M users | 45 min |
| Fri | LeetCode: 5 DP mediums | 2 hrs |
| Sat | Full behavioral run — speak all 9 STAR stories in sequence | 90 min |
| Sun | Company research for nearest scheduled interview | 1.5 hrs |

---

### Week 3 — Mock Interviews

| Day | Activity | Time |
|-----|----------|------|
| Mon | Full mock coding screen with a peer (or Pramp/Interviewing.io) | 45 min |
| Tue | Full mock system design with a peer | 60 min |
| Wed | LeetCode: 5 Graph mediums | 2 hrs |
| Thu | Full mock behavioral round — record and review | 45 min |
| Fri | Java deep dive Q&A — all Quick-Reference sections from Chapters 1-4 | 1.5 hrs |
| Sat | Mock design: design a rate limiter for 10M users | 45 min |
| Sun | Rest and light review | 30 min |

---

### Week 4 — Final Preparation

| Day | Activity | Time |
|-----|----------|------|
| Mon–Tue | Full loop simulation: coding → design → Java → behavioral, back-to-back | 4 hrs |
| Wed | Target company research — fill out the company template | 1.5 hrs |
| Thu | Re-read resume bullets (Chapter 5 Part 7) — practice verbal bridges | 1 hr |
| Fri | Day before interview: light review only, no new material | 1 hr |
| Sat | Interview day — apply the frameworks, not perfection |  |

---

### Day-Before Checklist

```
□ Read the job description one more time — note the exact language they use
□ Review your 3 competitive differentiators (README.md)
□ Run through 3 STAR stories mentally — Situation summary only
□ Prepare your 3 questions per round
□ Confirm interview format, number of rounds, interviewers if shared
□ Test tech setup if virtual (mic, camera, screen share for coding)
□ Sleep 7+ hours
```

### During the Interview — Mental Anchors

**Coding:** "I'm going to state my approach before I code." → never code silently.

**System Design:** "Let me start with requirements." → always start with Step 1.

**Behavioral:** "Let me give you a specific example." → always concrete, never abstract.

**When stuck:** "Let me think for a moment." → 30 seconds of silence is fine. Going blank is not. Verbalize the thinking: "I know I need to handle X, I'm trying to decide between Y and Z..."

**When you don't know the answer:** "I haven't worked with that specific technology, but here's how I'd reason about it based on [related knowledge]..." → show reasoning ability, not just recall.

---

## QUICK-REFERENCE — Final Cheat Sheet

### Java Senior Filter — One-Line Answers

| Question | One-Line Answer |
|----------|----------------|
| Why is `ArrayList` faster than `LinkedList` for reads? | Contiguous memory → cache-friendly random access O(1); LinkedList needs pointer traversal O(n) |
| What is a memory leak in Java? | Object referenced (directly or via static/ThreadLocal) that can never be GC'd — common with classloaders, ThreadLocals, listeners |
| When does `hashCode()` matter? | HashMap/HashSet use it for bucket placement — if two equal objects have different hashCodes, you break the contract and get incorrect behavior |
| What is a `WeakReference`? | GC can collect the referenced object even if WeakReference exists — used for caches where entries should be collectible under memory pressure |
| What does `transient` mean? | Field is excluded from Java serialization |
| `Comparable` vs `Comparator`? | `Comparable` = natural ordering defined in the class itself (`compareTo`). `Comparator` = external comparison logic, passed to sort methods. Use `Comparator` when you don't own the class or need multiple orderings |

### Spring Senior Filter — One-Line Answers

| Question | One-Line Answer |
|----------|----------------|
| What is a `FactoryBean`? | A bean that creates other beans — `getObject()` returns the bean the factory produces. Used by Spring internally for JPA EntityManagerFactory, etc. |
| `@Component` vs `@Bean`? | `@Component` is class-level — Spring auto-detects it. `@Bean` is method-level in `@Configuration` — you control instantiation, useful for third-party classes you can't annotate |
| What is `@Scope("prototype")`? | New instance per injection point (vs singleton = one shared instance). Use for stateful beans or non-thread-safe objects |
| What does `@Async` require? | `@EnableAsync` on a config class, a separate executor bean, and the method must be called from outside the bean (proxy issue — same as `@Transactional`) |
| What is `ApplicationContext` refresh? | The lifecycle method that instantiates all singleton beans, runs BeanPostProcessors, and brings the application from "configured" to "running" |

### System Design Senior Filter — One-Line Answers

| Question | One-Line Answer |
|----------|----------------|
| SQL vs NoSQL — how to choose? | SQL for relational data, ACID transactions, complex queries. NoSQL for flexible schemas, horizontal write scale, specific access patterns (wide-column, document, graph) |
| What is a CDN? | Edge network of geographically distributed servers that cache static content closer to users — reduces origin server load and reduces latency |
| What is idempotency? | Property where an operation produces the same result regardless of how many times it is applied — critical for retry safety |
| What is back pressure? | Mechanism where a slow consumer signals a fast producer to slow down — prevents unbounded queue growth. `CallerRunsPolicy` in ThreadPoolExecutor is back pressure |
| What is a hot partition? | A shard or partition receiving disproportionate traffic — causes performance degradation. Prevent with random shard key suffixes or consistent hashing with virtual nodes |
| What is tail latency? | The latency at high percentiles (p99, p999) — more actionable than averages because averages hide outliers that real users experience |

---

> **Guide Complete.** You now have a definitive reference covering the full interview loop: Java & Spring internals (Ch 1), Concurrency & Databases (Ch 2), LLD & Design Patterns (Ch 3), HLD Frameworks & Behavioral (Ch 4), Advanced HLD & Project Narratives (Ch 5), DSA Patterns & Mock Interview Simulation (Ch 6). Every chapter is anchored to your three real projects. Use the 30-day sprint plan when interviews are scheduled.
