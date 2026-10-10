# binary tree traversal: the four orders explained, plus the iterative templates that trip people up in interviews

Most people can recite the three depth-first orders. Ask them five minutes later which one to use for "find the kth smallest element in a BST" or "compute the diameter of a tree," and the recitation stops helping. The orders themselves are trivial. The skill interviewers are actually testing is knowing which traversal a problem is hiding behind, and being able to write that traversal without a compiler, with a stack you manage by hand.

This walks through all four orders, what each one is genuinely good for, the recursive and iterative versions, and where people usually lose the plot under time pressure.

## What traversal actually means

A traversal is a rule for visiting every node exactly once. Nothing more mystical than that. On a binary tree you get four standard orders, and the difference between them is only *when* the current node gets processed relative to its two subtrees.

| Order | Visit sequence | What it's typically for |
| --- | --- | --- |
| Preorder | root → left → right | Serializing or copying a tree, prefix notation, rebuilding a tree from a prefix string |
| Inorder | left → root → right | Sorted output from a BST, validating a BST, kth smallest, successor queries |
| Postorder | left → right → root | Anything that combines children's results first: height, diameter, subtree sums, deleting a tree |
| Level order (BFS) | top to bottom, left to right | Depth, right-side view, zigzag, connecting siblings on the same level |

All four run in O(n) time on a tree with n nodes. The space cost is where they diverge: the three depth-first orders use O(h) where h is the tree height, which is O(log n) for a balanced tree and O(n) for a path-shaped one. Level order uses O(w), where w is the widest level.

That last detail shows up more often than you'd expect. Recursion on a tree with 10^5 nodes that happens to be a straight line will blow the stack in plenty of languages, which is exactly the setup for the "do it iteratively" follow-up.

## The four orders, and when each one is the right answer

### Preorder: you're making a copy of the structure

Root first means the first thing you touch describes the whole tree. That's why serialization formats start with the root: `1,2,4,#,#,5,#,#,3,#,#` reads left to right and rebuilds the tree in a single pass. Copying a tree, emitting an expression in prefix notation, or flattening a tree into an array where a parent always precedes its children are all preorder problems wearing different clothes.

### Inorder: the BST trick

Inorder visits the left subtree, then the node, then the right subtree. On a plain binary tree that's just an order. On a binary search tree it's a sorted sequence, and that single fact solves a whole family of problems:

- **Validate BST.** Run inorder and check the sequence is strictly increasing with a `prev` pointer. No min/max bounds to track, no recursion-with-limits confusion.
- **Kth smallest element.** Inorder, decrement a counter, stop at zero. O(h) space if you do it iteratively.
- **Successor and predecessor queries.** Because the sorted order is implicit in the structure.

The common mistake is applying the sorted-output property to trees that aren't BSTs. Inorder on a plain binary tree gives you no ordering guarantee, and the "kth smallest" approach quietly breaks.

### Postorder: aggregates bottom-up

Postorder processes both subtrees before the node, which means when you're standing at a node, everything below it has already been reduced to whatever number you care about. Height is the classic case:


def height(node):
    if node is None:
        return -1
    return 1 + max(height(node.left), height(node.right))


The recursion doesn't *look* like a traversal, but the value is only combined after both recursive calls return, which is postorder. Same shape for subtree sums, counting good nodes, checking balance, computing tree diameter, and freeing a tree's memory. If the problem says "combine something from children," you want postorder even if the code never says the word.

### Level order: anything that's about depth

BFS with a queue, processing one level per round. Problems that lean on levels: maximum depth without recursion, right-side view, average of levels, zigzag traversal, minimum depth to a leaf, and level-order serialization. The queue mechanics also make it the natural fit for "process the tree layer by layer" problems on n-ary trees and graphs, which is where the same pattern reappears with a `visited` set bolted on.

## Recursion is the easy part. The stack is where interviews get interesting

Every candidate can write the recursive three-liner. The follow-up, almost every time, is: "now do it without recursion." That means owning an explicit stack.

Iterative preorder is the friendliest, because the root is processed the moment you see it:


def preorder(root):
    if not root:
        return []
    out, stack = [], [root]
    while stack:
        node = stack.pop()
        out.append(node.val)
        if node.right:
            stack.append(node.right)
        if node.left:
            stack.append(node.left)
    return out


Push right before left, because the stack gives you back the last thing you pushed. Get the order backwards and you produce a perfectly valid traversal of the mirror image, which is a fun way to fail a screen.

Iterative inorder is the one worth memorizing properly. The trick is that you can't visit a node until you've exhausted its left spine, so you walk down pushing everything, then pop, record, and jump right:


def inorder(root):
    out, stack, cur = [], [], root
    while cur or stack:
        while cur:
            stack.append(cur)
            cur = cur.left
        cur = stack.pop()
        out.append(cur.val)
        cur = cur.right
    return out


Iterative postorder is the awkward middle child. The clean workaround: do a modified preorder that pushes left-then-right (giving root → right → left) and reverse the output at the end. Same complexity, far fewer bugs than trying to track whether you're returning from a left or right child.

There's also Morris traversal, which gets you O(n) time and O(1) extra space by temporarily rewiring the tree's null right pointers into threads back to the inorder successor, then undoing them. Rarely required, occasionally asked by interviewers who want to see whether you know it exists.

## Level order is a queue and a level marker

The one structural detail people fumble: separating levels. Snapshot the queue length before each round, and the inner loop gives you exactly one level:


def level_order(root):
    if not root:
        return []
    from collections import deque
    q, out = deque([root]), []
    while q:
        level = []
        for _ in range(len(q)):
            node = q.popleft()
            level.append(node.val)
            if node.left:  q.append(node.left)
            if node.right: q.append(node.right)
        out.append(level)
    return out


That `for _ in range(len(q))` is the whole trick. Skip it and you get a flat list, which is a different (and easier) problem than the one usually asked.

## Reconstructing a tree from its traversals

A frequent follow-up: given preorder and inorder, rebuild the tree. Preorder's first element is always the root. Find that value's position in the inorder array, and everything to its left is the left subtree, everything to its right is the right subtree. Recurse. Hash the inorder indices once and you're at O(n) instead of O(n²).

Two caveats worth stating out loud in an interview, because they're the actual point of the question: the values must be unique, and preorder + postorder alone is *not* enough to reconstruct a binary tree. You need an order that separates left from right. Preorder plus inorder does; preorder plus postorder doesn't.

## The complexity answers that earn points

O(n) time for all four orders is the expected answer, but the space discussion is where you can distinguish yourself:

- Recursive DFS: O(h) stack frames. Balanced tree, O(log n). Degenerate tree, O(n). This is a real crash risk, not a theoretical one.
- Iterative DFS: same O(h), but on the heap instead of the call stack, so it survives deep trees.
- Level order: O(w), the maximum width. On a complete binary tree that's O(n/2), which is much worse than the DFS numbers despite costing the same in time.
- Morris: O(1) extra space, at the cost of temporarily mutating the tree.

If a problem adds a constraint like "the tree may have up to 10^6 nodes and may be skewed," the expected answer is iterative, and saying why is most of the credit.

## Where traversal learning usually breaks down

The pattern is recognizable. You read the explanation of inorder, it makes sense, you do three problems, and a week later you're staring at a problem that needs postorder with no idea why the code isn't working.

Three things tend to cause that:

1. **The orders get memorized as sequences rather than purposes.** "Left, root, right" is a fact you can forget. "Inorder is sorted output, so it's the BST order" is a hook that survives forgetting.
2. **Practice volume is spent on problems that don't force the distinction.** Ten DFS problems that all happen to be preorder don't build pattern recognition.
3. **The iterative versions never get written from scratch.** Reading iterative inorder once feels like understanding it. Writing it cold, three weeks later, is a different test.

This is where the structure of a platform like AlgoMonster matters for this specific topic. It's built as a pattern course rather than a problem bank, and tree traversal lives inside its DFS/BFS tree patterns. Each pattern comes with an explanation of when the pattern applies, a reusable code template, and then a set of problems that use it, which is aimed directly at the "I understood it last week" failure mode. The decision flowchart is the part most relevant to traversal: it walks through questions like whether the input is a tree or graph, or whether you need results from children before processing a node, and points at the pattern that fits.

Two structural facts about the platform are worth knowing before you pay for anything. It's text and diagrams, not video: reviews of it are consistent that there are no video walkthroughs, which suits people who learn by reading and stepping through visualizations and frustrates people who need someone coding on screen. And the problem count is smaller than a competitive programming site's, which several reviews flag as the main limitation for anyone who needs raw volume after the patterns click. The common recommendation in those same reviews is to learn the patterns there and drill volume elsewhere.

## AlgoMonster's plans and current pricing

AlgoMonster runs on a simple model with no monthly billing. You're choosing between annual access, a one-time lifetime purchase, or staying on the free tier with a subset of the content.

| Plan | What you get | Price | Billing period | Sign up |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of lessons and problems, enough to test the format | $0 | No billing | [Start with the free tier](https://algo.monster?fpr=li69) |
| Pro Yearly | All patterns, lessons and problems unlocked; unlimited AI assistant; unlimited Speedrun; real questions from top tech companies updated monthly; 500+ lessons across 6 courses; 1,000+ illustrations; 3,000+ editorials; member discount on live bootcamps | $99/year (shown as $8.25/month billed annually) | Billed once a year, 12 months of access, cancel or upgrade anytime | [Get the Pro yearly plan](https://algo.monster?fpr=li69) |
| Pro Lifetime | Everything in the annual plan plus lifetime updates | $189 | One-time payment | [Buy Pro lifetime access](https://algo.monster?fpr=li69) |

Two notes on those numbers. The $99 and $189 figures are the prices currently displayed on the subscribe page, and a coupon tracker that verifies deals monthly lists the underlying list prices as $198 per year and $378 one-time, with the current numbers described as a roughly 50% promotion. So the honest read is that these are discounted prices that can move back up when the promotion ends. Some older third-party reviews still quote list figures in the $299 to $459 range, which is a sign of how much the pricing has shifted over time rather than evidence of a hidden extra tier.

Either way, the free tier is the right first move if you want to see how the traversal and DFS/BFS tree material is written before deciding. 👉 [Compare all AlgoMonster plans here](https://algo.monster?fpr=li69)

## A practical order of study for traversal specifically

If tree traversal is the thing you're actually trying to fix, this sequence works better than working through problems in whatever order a list gives them:

1. Write all four orders recursively from a blank file, without reference. If you can't, stop and drill that first.
2. Write iterative preorder, inorder, and postorder. Postorder via the reversed-preorder trick is fine.
3. Do Validate BST and Kth Smallest with inorder only, using a `prev` pointer. These force you to rely on the BST property rather than intuition.
4. Do Maximum Depth, Diameter, and Balanced Binary Tree. All three are postorder aggregates, and recognizing that on your own is the milestone worth hitting.
5. Do Level Order, Right Side View, and Zigzag. All queue plus level snapshot.
6. Finish with Serialize and Deserialize, then reconstruct-from-preorder-and-inorder. These two combine the orders rather than testing one.

Six sessions, maybe two hours each. That's a more useful allocation than fifty random tree problems, most of which will test the same order you already know.

## Questions people actually ask

**Do I need to memorize the iterative postorder?** No. Memorize the reverse-of-modified-preorder trick and you'll produce correct code. The version with a `last_visited` pointer is worth reading once so you can follow it if an interviewer shows it.

**Can I always use BFS instead of DFS?** On trees, yes in the sense that both enumerate all nodes, but they're not interchangeable for problems. Computing height bottom-up with BFS is possible but clumsy; getting level boundaries with DFS requires tracking depth explicitly and grouping results, which is usually more code for no benefit.

**Is recursive inorder actually O(1) space?** No. The call stack counts, and interviewers count it. "O(1) if you ignore the recursion stack" is a phrase that invites a follow-up you don't want.

**Does traversal order matter for LeetCode-style evaluation?** For problems returning a list of node values, yes, the order is the answer. For problems returning a single number, the order is an implementation detail, and choosing the wrong one usually turns a clean solution into a bug hunt.

The orders are four lines each. What separates people who pass the tree questions from people who don't is knowing which four lines the problem is asking for, and writing the iterative version without panicking. 👉 [Learn the DFS/BFS tree patterns at AlgoMonster](https://algo.monster?fpr=li69)
