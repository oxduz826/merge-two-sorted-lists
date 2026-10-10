# merge two sorted lists: the dummy node trick, edge cases that break your code, and keeping space at O(1)

You can solve this problem in about fifteen lines. Most people still fail it on the first try in an interview — not because the idea is hard, but because pointer bookkeeping rewards precision and punishes autopilot. This is LeetCode 21, and it's the single most common warm-up question in a linked list screen.

Here's the whole thing: two sorted singly linked lists, splice their nodes into one sorted list, return the head. No new nodes, no arrays. The examples and constraints:

- `list1 = [1,2,4]`, `list2 = [1,3,4]` → `[1,1,2,3,4,4]`
- `list1 = []`, `list2 = []` → `[]`
- `list1 = []`, `list2 = [0]` → `[0]`
- Node counts are tiny — 0 to 50 in the older version of the statement, 0 to 100 in the current one
- Node values sit between -100 and 100
- Both lists are sorted in non-decreasing order

That value range and list length matter more than they look. The official test data is so small that nearly anything passes, which is exactly why interviewers change the constraints on you: "now do it on a 10-million-node stream." The code you write against the tiny version often doesn't survive that.

## Why linear time is even possible

Merging two *sorted* sequences is a special case. At any moment, if there are unprocessed nodes left, the smallest one still remaining has to be the head of `list1` or the head of `list2`. Nothing else can be smaller, because every other node in each list sits behind its own head.

So you never need to look further than one node ahead in each list. Build the result by repeatedly taking whichever head is smaller, and the output stays sorted for free. That gives O(n + m) time, where n and m are the lengths of the two lists.

Compare that to the brute-force version, which is the honest answer if you blank in an interview: dump both lists into an array, sort it, rebuild the links. That's O((n + m) log(n + m)) time and O(n + m) extra space. It works, and mentioning it as a stepping stone is fine. Submitting it is not — you've thrown away the sortedness the problem handed you.

## The dummy node pattern

The one design decision that makes the code short is a sentinel node.

Without it you need a branch to decide what the merged head is, and that branch has to run before the loop. With a dummy node you start with `tail` pointing at a throwaway node, attach every chosen node behind it, and return `dummy.next` at the end. The first real node gets attached by the exact same line of code as the last one.

python
class Solution:
    def mergeTwoLists(self, list1, list2):
        dummy = ListNode(0)
        tail = dummy
        while list1 and list2:
            if list1.val <= list2.val:
                tail.next = list1
                list1 = list1.next
            else:
                tail.next = list2
                list2 = list2.next
            tail = tail.next
        tail.next = list1 if list1 else list2
        return dummy.next


Six moving pieces: `dummy`, `tail`, the two list cursors, the comparison, and the drain line. Worth tracing by hand once so the mechanics stop being abstract.

Start: `tail` → dummy. `list1` → 1, `list2` → 1.

| Step | Compare | Attach | Cursor moves to |
| --- | --- | --- | --- |
| 1 | 1 ≤ 1 | list1's 1 | list1 → 2 |
| 2 | 2 ≤ 1? no | list2's 1 | list2 → 3 |
| 3 | 2 ≤ 3 | list1's 2 | list1 → 4 |
| 4 | 4 ≤ 3? no | list2's 3 | list2 → 4 |
| 5 | 4 ≤ 4 | list1's 4 | list1 → None |

The loop exits because `list1` ran out. Whatever is left in the other list is already sorted and already larger than everything attached, so one assignment — `tail.next = list2` — finishes the job. Result: 1 → 1 → 2 → 3 → 4 → 4. Exactly the expected output.

Java, same shape:

java
class Solution {
    public ListNode mergeTwoLists(ListNode list1, ListNode list2) {
        ListNode dummy = new ListNode(0);
        ListNode tail = dummy;
        while (list1 != null && list2 != null) {
            if (list1.val <= list2.val) {
                tail.next = list1;
                list1 = list1.next;
            } else {
                tail.next = list2;
                list2 = list2.next;
            }
            tail = tail.next;
        }
        tail.next = (list1 != null) ? list1 : list2;
        return dummy.next;
    }
}


## The bugs that actually cost people offers

Most failed attempts at merge two sorted lists fail in one of about six places. Check these before you say "I think that's it."

**Returning the wrong thing.** `return dummy` gives the interviewer a sentinel node with an arbitrary value glued to the front. It's `dummy.next`.

**Skipping the drain line.** Leave it out and the merged list gets cut off the moment either input runs dry. This is the most common silent failure — the code looks clean and produces `[1,1,2,3]` instead of `[1,1,2,3,4,4]`.

**Looping on `or` instead of `and`.** `while list1 or list2` then `.val` inside means you'll dereference `None` the moment one list empties. Python raises `AttributeError`; Java throws a null pointer.

**Forgetting `tail = tail.next`.** You overwrite the same `next` pointer repeatedly and end up with a two-node list. Every iteration needs the advance.

**Allocating new nodes.** The statement says to splice the existing nodes. A `new ListNode(...)` per element changes the memory profile and, if node equality is checked by identity anywhere in the harness, changes correctness.

**Tuple-assignment cleverness.** Idioms like `list1, tail = list1.next, list1` are correct in Python and read well to some reviewers. Mixed with `cur.next = list1` on the previous line they're also a reliable source of confusion under pressure. Write it the boring way.

## Complexity, and the space number people get wrong

Time is O(n + m) — each node from each list is visited once and attached once. The loop runs at most n + m times, and the drain is O(1).

Space is O(1) auxiliary. This is the part candidates fumble. You're rearranging existing `next` pointers, not copying data into a new structure, so the only extra memory is a fixed number of references: `dummy`, `tail`, and the two cursors. The dummy node itself is one allocation that gets discarded — constant, not proportional to input.

If someone asks "O(1)? You created a node." — one node, regardless of input size, is still O(1).

## Recursive version, and when it bites

The recursive form is three lines and reads like the definition of the answer.

python
class Solution:
    def mergeTwoLists(self, list1, list2):
        if not list1:
            return list2
        if not list2:
            return list1
        if list1.val <= list2.val:
            list1.next = self.mergeTwoLists(list1.next, list2)
            return list1
        list2.next = self.mergeTwoLists(list1, list2.next)
        return list2


The logic: the smaller head must come first, and everything after it is the merge of the leftover lists. Same O(n + m) time.

The catch is space. Every recursive call sits on the stack until its caller returns, so you pay O(n + m) stack frames. With LeetCode's 50-to-100-node limits that's invisible. On the 10-million-node follow-up an interviewer likes to invent, Python's default recursion limit blows up around a thousand frames. The iterative version is what ships. Say that out loud rather than hoping nobody asks.

## Ties: `<=` or `<`?

Both produce a correctly sorted list. The difference is what happens to equal values. Using `<=` attaches `list1`'s node first when the values match, which preserves the original relative order of equal elements — a stable merge, the same convention merge sort relies on. Using `<` sends ties to `list2` first. Pick one deliberately and be able to say why.

Also note `1 <= 1` and `1 < 1` differ on the trace above, so if you compare your output against a walkthrough and it diverges on the duplicate, check this line before rewriting anything.

## Edge cases worth naming before you code

Interviewers listen for whether you find these yourself:

- Both lists empty → the loop never runs, `dummy.next` is `None`, you return `None`
- One list empty → return the other list immediately; the recursive version does this with its base cases, the iterative version with the drain line
- Single node each → one comparison, one attach, one drain
- Duplicate values across lists → covered above
- All of `list1` smaller than all of `list2` → the loop exits after `list1` is exhausted, drain attaches the rest in one step — no O(n) copying

## Where this problem shows up again

Merge two sorted lists is rarely asked in isolation. It's a component of three bigger questions, and knowing that changes how you should study it.

**Merge Sorted Array (LeetCode 88)** is the same idea on arrays, except `nums1` has m + n slots and you must merge in place. Compare from the *back* — the largest remaining element goes into the last free slot, so you never overwrite a value you haven't read. O(m + n) time, O(1) space. If your comparison is destructive on arrays, this is why.

**Merge k Sorted Lists (LeetCode 23)** is the natural escalation. Merging k lists pairwise one at a time costs O(N·k) for N total nodes. A heap of list heads gets you O(N log k). Divide and conquer — merge pairs, then merge the results — also lands at O(N log k) and reuses the exact function you just wrote. Expect the follow-up.

**Sort List (LeetCode 148)** is merge sort on a linked list, where splitting the list with slow and fast pointers is the hard part and your merge function is the easy half.

For a pattern-first review, 👉 [the linked list and two-pointer tracks on AlgoMonster](https://algo.monster?fpr=li69) walk through node manipulation with the same template-then-practice structure, which is useful if your failure mode is knowing the idea but fumbling the pointers live.

## Practicing this efficiently instead of grinding it

The honest problem with a question this small is that repetition stops helping fast. You'll solve it, feel good, then meet Merge k Lists two weeks later and freeze because you never learned the *pattern*, only one instance of it.

That's the specific thing AlgoMonster is built around: it names its patterns, gives a code template per pattern, and maps each practice problem back to the pattern it belongs to, rather than leaving you to infer the categories from a few hundred random problems. The site's own description of the product: 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials, unlimited AI assistant use, and Speedrun mode for timed review. Third-party reviews consistently describe the curriculum as 48 named patterns and roughly 231 mapped problems, plus a pattern-identification flowchart for deciding which approach a new problem wants.

The tradeoffs are worth stating plainly, because they decide whether it fits you:

- Everything is text and interactive diagrams. There are no video walkthroughs. If you learn by watching someone code, this is the wrong format.
- The problem bank is far smaller than LeetCode's 3,000+. Most reviewers describe pairing the two: learn the pattern here, get reps elsewhere.
- System design content is introductory. For senior loops with a dedicated design round, budget for something else.
- No live mock interviews. Pattern recognition under a senior engineer watching you is a separate skill.
- It's for people who can already write working code. If you're still shaky on what a pointer is, start earlier.

## Current plans and prices

AlgoMonster doesn't do monthly billing. It's a free tier, an annual plan, or a one-time lifetime purchase. Pricing moves with promotions, so check the subscribe page for the live number before you decide.

| Plan | What you get | Price | Billing | Link |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns, lessons and problems | $0 | — | [Start with the free tier](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems unlocked; unlimited AI assistant; unlimited Speedrun; 500+ lessons across 6 courses; member discount on live bootcamps | $99/year ($8.25/mo billed annually) | Recurring, 12 months of access, cancel or upgrade anytime | [Get Pro Yearly](https://algo.monster?fpr=li69) |
| Pro — Lifetime | Everything in the annual plan plus lifetime updates and permanent access to future content | $189 | One-time payment | [Get Pro Lifetime](https://algo.monster?fpr=li69) |

Which one makes sense depends on your timeline. If you have an interview in six weeks and one prep cycle ahead of you, the yearly plan covers it and the lifetime premium buys you access you won't use this year. If you expect to interview again in two or three years — the normal pattern for engineers who change jobs — the lifetime plan costs less than two years of the annual price and includes future content.

The free tier is genuinely there, and it's the right first move if you're unsure whether you like reading your algorithm explanations instead of watching them. 👉 [Browse the free content before paying](https://algo.monster?fpr=li69) and see whether the diagram-heavy format clicks for you.

## FAQ

**Is merge two sorted lists actually easy?**
The idea is easy; the implementation is where people lose points. It's rated Easy, it appears constantly in phone screens, and a meaningful share of candidates still produce a truncated list because they forgot to drain the remaining nodes.

**What's the time and space complexity?**
O(n + m) time, where n and m are the two list lengths. O(1) auxiliary space for the iterative version, because you only rearrange existing pointers. The recursive version uses O(n + m) stack space.

**Iterative or recursive in an interview?**
Write iterative. Recursion is elegant and worth mentioning, but it costs a stack frame per element and fails on large inputs. Explaining the tradeoff scores better than only knowing one version.

**What if the problem asks for a new list instead of splicing?**
Then allocate nodes and copy values, and the space becomes O(n + m). Read the constraint — LeetCode 21 explicitly asks you to splice existing nodes, and reviewers notice when you don't.

**Does it matter which language I use?**
Python, Java, C++, Go and any others all express the same five-step loop. AlgoMonster provides templates in six languages, which is handy if you're switching languages close to an interview.

**Do I need to build my own notes for this?**
One tracing table like the one above is worth more than re-solving the problem five times. Write down what the cursor does at each step until you can reproduce it from memory without hesitating on the drain line.

The version of merge two sorted lists that gets you hired isn't the one that passes the tiny test cases. It's the one you can modify on request — to work in place on arrays, to handle k lists, to survive a stream that doesn't fit in memory — while explaining why each pointer moves where it does.
