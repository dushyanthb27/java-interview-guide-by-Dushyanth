# Interview Question Patterns

## Recognize and Master Common Interview Question Patterns

Understanding common question patterns helps you prepare more effectively and recognize similar problems during interviews. This guide identifies the most common patterns and how to approach them.

## Technical Question Patterns

### Pattern 1: Two Pointer Problems

**Characteristics:**
- Array or string manipulation
- Need to find pairs or compare elements
- Often involves sorting or movement

**Common Questions:**
- "Find two numbers that sum to target"
- "Remove duplicates from sorted array"
- "Reverse a string/array"
- "Container with most water"
- "Merge sorted arrays"

**Approach:**
```
1. Sort the array (if needed)
2. Initialize two pointers: left = 0, right = n-1
3. Move pointers based on condition
4. Return result when condition is met

Example: Find two numbers that sum to target
int[] nums = [2, 7, 11, 15], target = 9
left = 0 (2), right = 3 (15)
2 + 15 = 17 > 9, move right--
2 + 11 = 13 > 9, move right--
2 + 7 = 9, found!
```

**Time Complexity:** O(n) or O(n log n) with sorting
**Space Complexity:** O(1)

**Practice Problems:**
- Two Sum II
- 3Sum
- Container with Most Water
- Trapping Rain Water

### Pattern 2: Sliding Window

**Characteristics:**
- Subarray or substring problems
- Need to find longest/shortest/count
- Window expands and contracts

**Common Questions:**
- "Longest substring without repeating characters"
- "Minimum window substring"
- "Maximum sum subarray of size k"
- "Fruit into baskets"

**Approach:**
```
1. Initialize left and right pointers
2. Expand window by moving right
3. Shrink window when condition is met
4. Track result at each step

Example: Longest substring without repeating
String s = "abcabcbb"
left = 0, right = 0, maxLen = 0
Expand: a, ab, abc (3 chars)
Expand: abca (duplicate 'a')
Shrink: bc (remove 'a')
Continue...
Result: 3 ("abc")
```

**Time Complexity:** O(n)
**Space Complexity:** O(min(m, n)) for character set

**Practice Problems:**
- Longest Substring Without Repeating Characters
- Minimum Window Substring
- Permutation in String
- Longest Repeating Character Replacement

### Pattern 3: HashMap/HashSet

**Characteristics:**
- Frequency counting
- Uniqueness checking
- Mapping problems

**Common Questions:**
- "Find duplicates"
- "Anagram problems"
- "Group anagrams"
- "Valid anagram"
- "Majority element"

**Approach:**
```
1. Create HashMap to store frequencies
2. Iterate through input
3. Update frequencies
4. Process frequencies to find answer

Example: Find duplicates
int[] nums = [1, 2, 2, 3]
Map: {1: 1, 2: 2, 3: 1}
Find key with count > 1: 2
```

**Time Complexity:** O(n)
**Space Complexity:** O(n)

**Practice Problems:**
- Contains Duplicate
- Valid Anagram
- Group Anagrams
- Majority Element
- Happy Number

### Pattern 4: Binary Search

**Characteristics:**
- Sorted array
- Need to find element or position
- Divide and conquer approach

**Common Questions:**
- "Search in rotated sorted array"
- "Find first/last position of element"
- "Peak element"
- "Search in 2D matrix"

**Approach:**
```
1. Initialize left = 0, right = n-1
2. While left <= right:
   - mid = (left + right) / 2
   - If target == nums[mid], return mid
   - If target < nums[mid], right = mid - 1
   - If target > nums[mid], left = mid + 1
3. Return -1 if not found

Time Complexity: O(log n)
```

**Practice Problems:**
- Binary Search
- Search in Rotated Sorted Array
- Find First and Last Position
- Peak Index in Mountain Array

### Pattern 5: Tree/Graph Traversal

**Characteristics:**
- Tree or graph problems
- Need to visit all nodes
- DFS or BFS approach

**Common Questions:**
- "Inorder/Preorder/Postorder traversal"
- "Level order traversal"
- "Path sum"
- "Lowest common ancestor"
- "Number of islands"

**Approach:**
```
DFS (Recursion):
void dfs(Node node) {
    if (node == null) return;
    process(node);
    dfs(node.left);
    dfs(node.right);
}

BFS (Queue):
Queue<Node> queue = new LinkedList<>();
queue.add(root);
while (!queue.isEmpty()) {
    Node node = queue.poll();
    process(node);
    if (node.left != null) queue.add(node.left);
    if (node.right != null) queue.add(node.right);
}
```

**Time Complexity:** O(n)
**Space Complexity:** O(h) for DFS, O(w) for BFS

**Practice Problems:**
- Binary Tree Inorder Traversal
- Level Order Traversal
- Path Sum
- Lowest Common Ancestor
- Number of Islands

### Pattern 6: Dynamic Programming

**Characteristics:**
- Optimization problems
- Overlapping subproblems
- Optimal substructure

**Common Questions:**
- "Climbing stairs"
- "Coin change"
- "Longest increasing subsequence"
- "Edit distance"
- "Knapsack problems"

**Approach:**
```
1. Define subproblem
2. Find recurrence relation
3. Implement with memoization or tabulation

Example: Climbing stairs
dp[i] = number of ways to reach step i
dp[i] = dp[i-1] + dp[i-2]

Base cases:
dp[0] = 1 (one way to stay at ground)
dp[1] = 1 (one way to reach step 1)
```

**Time Complexity:** O(n)
**Space Complexity:** O(n)

**Practice Problems:**
- Climbing Stairs
- Coin Change
- Longest Increasing Subsequence
- Edit Distance
- Partition Equal Subset Sum

### Pattern 7: Backtracking

**Characteristics:**
- Permutation/combination problems
- Need to explore all possibilities
- Recursive approach with pruning

**Common Questions:**
- "Permutations"
- "Combinations"
- "N-Queens"
- "Sudoku solver"
- "Word search"

**Approach:**
```
void backtrack(List<Integer> current, int[] nums) {
    // Base case: found a solution
    if (current.size() == nums.length) {
        result.add(new ArrayList<>(current));
        return;
    }
    
    // Try each possibility
    for (int i = 0; i < nums.length; i++) {
        if (!used[i]) {
            current.add(nums[i]);
            used[i] = true;
            
            backtrack(current, nums);
            
            current.remove(current.size() - 1);
            used[i] = false;
        }
    }
}
```

**Time Complexity:** O(n!)
**Space Complexity:** O(n)

**Practice Problems:**
- Permutations
- Combinations
- N-Queens
- Word Search
- Sudoku Solver

### Pattern 8: Linked List

**Characteristics:**
- Linked list manipulation
- Pointer operations
- Node insertion/deletion

**Common Questions:**
- "Reverse linked list"
- "Merge sorted lists"
- "Detect cycle"
- "Remove nth node"
- "Reorder list"

**Approach:**
```
// Reverse linked list
ListNode prev = null;
ListNode curr = head;
while (curr != null) {
    ListNode next = curr.next;
    curr.next = prev;
    prev = curr;
    curr = next;
}
return prev;
```

**Time Complexity:** O(n)
**Space Complexity:** O(1)

**Practice Problems:**
- Reverse Linked List
- Merge Two Sorted Lists
- Linked List Cycle
- Remove Nth Node From End
- Reorder List

## Behavioral Question Patterns

### Pattern 1: Leadership/Initiative

**Characteristics:**
- Taking charge
- Making decisions
- Influencing others

**Common Questions:**
- "Tell me about a time you led a team"
- "Describe a situation where you took initiative"
- "Tell me about a time you motivated others"

**STAR Approach:**
```
Situation: Team was stuck on a problem
Task: As senior engineer, I needed to help
Action: I organized a brainstorming session, 
        proposed a solution, got team buy-in
Result: We solved the problem and shipped on time
```

### Pattern 2: Teamwork/Collaboration

**Characteristics:**
- Working with others
- Communication
- Compromise

**Common Questions:**
- "Tell me about a time you worked with a difficult person"
- "Describe a situation where you had to compromise"
- "Tell me about a time you helped a colleague"

**STAR Approach:**
```
Situation: Colleague had different approach
Task: Need to align on solution
Action: I listened to their perspective, 
        found common ground, compromised
Result: Better solution through collaboration
```

### Pattern 3: Challenge/Problem-Solving

**Characteristics:**
- Overcoming obstacles
- Learning from mistakes
- Persistence

**Common Questions:**
- "Tell me about your biggest failure"
- "Describe a time you solved a complex problem"
- "Tell me about a time you had to learn something new"

**STAR Approach:**
```
Situation: Faced a complex technical problem
Task: Had to solve it quickly
Action: Broke it down, researched, tried different approaches
Result: Found solution, learned valuable lesson
```

### Pattern 4: Conflict/Disagreement

**Characteristics:**
- Handling disagreement
- Professional communication
- Finding solutions

**Common Questions:**
- "Tell me about a time you disagreed with your manager"
- "Describe a time you had to give bad news"
- "Tell me about a conflict with a colleague"

**STAR Approach:**
```
Situation: Disagreed with manager's approach
Task: Need to express concern professionally
Action: Gathered data, presented alternative, listened
Result: Found compromise, maintained relationship
```

## System Design Question Patterns

### Pattern 1: URL Shortening Service

**Key Components:**
- Hash function for encoding
- Database for mapping
- Cache for hot URLs
- Analytics service

**Approach:**
```
1. Clarify requirements
2. Estimate scale
3. Design database schema
4. Design API endpoints
5. Add caching layer
6. Discuss scalability
```

### Pattern 2: Social Media Feed

**Key Components:**
- User graph
- Feed generation
- Real-time updates
- Ranking algorithm

**Approach:**
```
1. Understand feed requirements
2. Design user relationships
3. Design feed generation
4. Add caching
5. Handle real-time updates
```

### Pattern 3: Distributed Cache

**Key Components:**
- Hash ring for distribution
- Replication strategy
- Eviction policy
- Consistency protocol

**Approach:**
```
1. Understand cache requirements
2. Design distribution strategy
3. Handle failures
4. Ensure consistency
```

## Recognition Strategy

### How to Recognize Patterns During Interview

1. **Listen for Keywords**
   - "Find two numbers" → Two pointer
   - "Longest/shortest substring" → Sliding window
   - "Duplicates" → HashMap
   - "Sorted array" → Binary search
   - "Tree/graph" → DFS/BFS
   - "Optimal solution" → Dynamic programming

2. **Identify Problem Type**
   - Array/String → Two pointer, sliding window, HashMap
   - Tree/Graph → DFS, BFS, recursion
   - Optimization → Dynamic programming
   - Permutation/Combination → Backtracking
   - Linked List → Pointer manipulation

3. **Apply Known Pattern**
   - Use template approach
   - Adapt to specific problem
   - Implement solution

## Pattern Mastery Checklist

### Two Pointer
- [ ] Understand when to use
- [ ] Know basic template
- [ ] Can solve 5+ problems
- [ ] Can explain trade-offs

### Sliding Window
- [ ] Understand window concept
- [ ] Know expansion/contraction
- [ ] Can solve 5+ problems
- [ ] Can optimize space

### HashMap/HashSet
- [ ] Understand frequency counting
- [ ] Know when to use
- [ ] Can solve 5+ problems
- [ ] Can handle collisions

### Binary Search
- [ ] Understand divide and conquer
- [ ] Know template
- [ ] Can solve 5+ problems
- [ ] Can handle edge cases

### Tree/Graph Traversal
- [ ] Understand DFS vs BFS
- [ ] Know both approaches
- [ ] Can solve 5+ problems
- [ ] Can optimize space

### Dynamic Programming
- [ ] Understand subproblems
- [ ] Know recurrence relation
- [ ] Can solve 5+ problems
- [ ] Can optimize space

### Backtracking
- [ ] Understand exploration
- [ ] Know pruning strategies
- [ ] Can solve 5+ problems
- [ ] Can optimize time

### Linked List
- [ ] Understand pointer operations
- [ ] Know common operations
- [ ] Can solve 5+ problems
- [ ] Can handle edge cases

## Practice Strategy

### Week 1: Foundation
- [ ] Learn pattern templates
- [ ] Solve 2-3 problems per pattern
- [ ] Understand time/space complexity

### Week 2: Depth
- [ ] Solve 5-10 problems per pattern
- [ ] Understand variations
- [ ] Practice explaining

### Week 3: Mastery
- [ ] Solve 15+ problems per pattern
- [ ] Mix patterns in practice
- [ ] Do timed practice

### Week 4: Refinement
- [ ] Do mock interviews
- [ ] Practice under pressure
- [ ] Get feedback

## Quick Reference: Pattern Identification

| Keywords | Pattern | Approach |
|----------|---------|----------|
| Two numbers, pair | Two pointer | Sort, move pointers |
| Longest/shortest substring | Sliding window | Expand/contract window |
| Duplicates, frequency | HashMap | Count occurrences |
| Sorted array, search | Binary search | Divide and conquer |
| Tree, graph, visit | DFS/BFS | Recursion or queue |
| Optimal, minimum, maximum | Dynamic programming | Subproblems |
| Permutation, combination | Backtracking | Explore all |
| Linked list, reverse | Pointer manipulation | Update pointers |

## Final Tips

1. **Learn patterns deeply** - Don't just memorize
2. **Practice variations** - Same pattern, different problems
3. **Recognize patterns** - Identify during interview
4. **Adapt templates** - Customize for specific problem
5. **Understand trade-offs** - Know when to use each pattern
6. **Practice under pressure** - Time yourself
7. **Get feedback** - Review solutions
8. **Keep learning** - New patterns emerge

---

**Remember: Pattern recognition is a skill that improves with practice. The more problems you solve, the faster you'll recognize patterns during interviews.**

Last Updated: 2024
