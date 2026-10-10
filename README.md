# implement trie prefix tree: build the Trie class from scratch in Python, get search vs startsWith right, and survive the follow-up questions

LeetCode 208 looks harmless. Four methods, one class, "medium" difficulty. Then you submit and it fails on the case where `search("app")` returns `false` right after you inserted `"apple"`, and you spend twenty minutes wondering why your perfectly good tree is lying to you.

That failure isn't a coding bug. It's a design question the problem is quietly testing: do you know the difference between *a path exists* and *a word ends here*? Everything else in this problem follows from that.

Here's what the problem actually asks, how to build it so it holds up, and where the same structure shows up in the harder problems that come after it.

## What LeetCode 208 is really asking for

The problem statement gives you four operations:

- `Trie()` — initialize the object
- `void insert(String word)` — add a word
- `boolean search(String word)` — is this exact word in the trie?
- `boolean startsWith(String prefix)` — is there any inserted word that begins with this?

Constraints: `word` and `prefix` are between 1 and 2000 characters, lowercase English letters only, and there are at most 3 × 10⁴ total calls.

The canonical test case makes the trap obvious:


Trie trie = new Trie();
trie.insert("apple");
trie.search("apple");   // true
trie.search("app");     // false  ← the part that catches people
trie.startsWith("app"); // true
trie.insert("app");
trie.search("app");     // true


Notice the third line. After inserting `"apple"`, the path `a → p → p` exists in the tree. But `"app"` was never inserted as a word, so `search` has to say no while `startsWith` says yes. A trie that only stores characters has no way to express that distinction. It needs one extra bit per node.

## The node: children plus an "end of word" flag

A trie node holds two things:

1. Where you can go next — the children
2. Whether a word terminates *at this node*

That second field is the whole trick. In a trie, characters live on the edges, so a node sitting at depth 3 after `a → p → p` means "some inserted word has the prefix `app`", not "app is a word". Marking `is_end = True` is what turns a prefix into a word.

For children, you have two reasonable implementations, and the problem accepts both:

**A 26-slot array.** `children = [None] * 26`, indexed by `ord(ch) - ord('a')`. Constant-time lookup, no hashing, and it matches the constraint that inputs are lowercase letters. The cost is memory: every node carries 26 pointers whether it uses one or twenty-six of them.

**A hash map.** `children = {}` keyed by character. Allocates lazily, so sparse branches stay cheap, and it extends to uppercase, digits or Unicode without changes. Slightly slower per lookup due to hashing.

The array version is what interviewers usually expect when the problem states lowercase-only. The map version is more honest about real-world tries, where the alphabet is rarely 26 wide. If you're asked which you'd pick, the answer depends on the alphabet size and how dense the tree is — arrays win for dense small alphabets, maps win when the branching factor is low.

## Full implementation

Here's the Python version using a hash map, with the shared traversal pulled into a helper. LeetCode's Python harness instantiates the class directly, so no wrapper is needed.

python
class Trie:

    def __init__(self):
        self.children = {}
        self.is_end = False

    def insert(self, word: str) -> None:
        node = self
        for ch in word:
            if ch not in node.children:
                node.children[ch] = Trie()
            node = node.children[ch]
        node.is_end = True

    def _walk(self, s: str):
        """Return the node at the end of s, or None if the path breaks."""
        node = self
        for ch in s:
            if ch not in node.children:
                return None
            node = node.children[ch]
        return node

    def search(self, word: str) -> bool:
        node = self._walk(word)
        return node is not None and node.is_end

    def startsWith(self, prefix: str) -> bool:
        return self._walk(prefix) is not None


The `_walk` helper exists because `search` and `startsWith` do the same traversal and differ in exactly one place. Writing them as one function with a boolean flag (a common pattern: `search(word, prefixMatch=True)`) works too, but a default parameter is easier to misread later than two three-line methods sharing a helper.

The array-based variant swaps the child access:

python
class Trie:
    def __init__(self):
        self.children = [None] * 26
        self.is_end = False

    def insert(self, word: str) -> None:
        node = self
        for ch in word:
            i = ord(ch) - ord('a')
            if node.children[i] is None:
                node.children[i] = Trie()
            node = node.children[i]
        node.is_end = True


Note that `insert` never allocates a node for a character that already has one. That's what makes prefix sharing work: inserting `"apple"` then `"app"` reuses the four existing nodes for `a`, `p`, `p` and simply flips `is_end` on the node for the second `p`. Six nodes total, not nine. If your solution builds a fresh branch each time, it still passes LeetCode's tests but you've written a very expensive set.

## Complexity, and why it looks too good

Every operation is bounded by the length of the string you pass in, not by how many words the trie holds.

| Operation | Time | Space |
| --- | --- | --- |
| `insert(word)` | O(m), m = word length | O(m) new nodes worst case |
| `search(word)` | O(m) | O(1) |
| `startsWith(prefix)` | O(m) | O(1) |

Overall storage is O(T) where T is the total number of trie nodes — worst case, the total number of characters across all inserted words, and less whenever words share prefixes[1][4].

The part worth saying out loud in an interview: lookups are independent of n, the number of stored words. A hash set gives you O(1) average lookup for an exact word, which beats O(m) — so if exact matching is all you need, a set is the better structure. The trie earns its keep on *prefix* queries, where a hash set has nothing to offer short of scanning every entry, and at that point you're at O(n·k) instead of O(k).

That's also why the standard advice is to preprocess a word list into a trie when you'll be doing many prefix lookups. The build costs O(total characters) once; each query after that is O(k).

## The mistakes that actually cost submissions

**Forgetting to mark the end.** `insert` walks the word and then... does nothing. Every `search` returns false. This is the single most common 208 failure.

**Treating `startsWith` as `search`.** Both walk the same path. `search` must check `is_end` on the final node; `startsWith` must not. Swapping them produces the exact `search("app") == false` symptom from the example.

**Storing words at nodes instead of characters.** Saving the full string at the last node works, and LeetCode accepts it, but it throws away the memory advantage that makes a trie a trie. Expect a follow-up asking why you'd do that.

**Creating nodes eagerly instead of checking first.** `node.children[ch] = Trie()` unconditionally overwrites existing branches. Always check before assigning.

**Recursive implementations without thinking about depth.** Word length can reach 2000, which is fine for Python's default limit, but an iterative loop has no depth risk at all and is just as short.

**Assuming the alphabet.** If you hardcode 26 slots, `ord(ch) - ord('a')` throws on anything outside a–z. The problem guarantees lowercase only, so it passes — but a reviewer asking "what if the input has digits?" is asking whether you understood why the array is safe here.

## Where this shows up after 208

The follow-ups matter more than the problem itself, because a trie is a building block rather than a one-off exercise:

- **211. Add and Search Word** — same node, but `.` matches any single character, so `search` becomes a small DFS over branches.
- **212. Word Search II** — a trie of the dictionary plus backtracking on a grid. This is where writing `insert` cleanly pays off.
- **648. Replace Words** — shortest-prefix lookup, which is `_walk` with an early `is_end` check.
- **642. Design Search Autocomplete System** — the autocomplete application the problem statement itself names[2].
- **Maximum XOR problems** — the same node shape over bits instead of characters, which is a genuinely different-looking problem that falls out of the same structure.

If you've solved 208 once and it felt like a typing exercise, that's usually because the `is_end` reasoning never clicked and you pattern-matched your way through. Recognition is the skill being tested across all of these, not individual solutions.

## Drilling the pattern instead of re-reading your own code

The awkward part of a trie is that the code is short enough to memorize and just subtle enough to get wrong under pressure. Re-reading a solution you already wrote doesn't fix that; solving 211 and 212 does, because they force the same node design through a different set of operations.

AlgoMonster's flowchart has a branch for exactly this decision: *"Prefix or pattern matching? → Trie / String Matching / Rolling Hash"*[3]. That's the recognition step — before writing a line of code, you decide whether the problem is asking for prefix structure or something else. The trie section sits alongside the rest of the curriculum, which the site lists as 500+ lessons across 6 courses, 1,000+ illustrations and 3,000+ editorials, all unlocked by a subscription, with unlimited AI assistant usage and Speedrun drills for timed practice. Third-party pages covering the site, including the Tech Interview Handbook's trie topic page, list it as a recommended course for this specific topic[4].

Two things about the format are worth knowing before you buy. Everything is text and diagrams — there are no video walkthroughs, so if you learn by watching someone type, this will feel like a mismatch. And the content assumes you already write code fluently; it teaches pattern selection, not programming fundamentals.

## Plans and current pricing

The subscription page currently shows two paid tiers on top of a limited free tier, with no monthly billing option at all:

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of lessons and problems — enough to judge the text-and-diagram format | $0 | — | [Start with the free tier](https://algo.monster/subscribe?fpr=li69) |
| Pro – Yearly | All patterns, lessons and problems; unlimited AI assistant usage; unlimited Speedrun; company-tagged questions updated monthly; 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials; member discount on live bootcamps | $69.99/year ($5.84/mo billed annually) | Billed yearly, cancel or upgrade anytime | [Check the Pro yearly price](https://algo.monster/subscribe?fpr=li69) |
| Pro – Lifetime | Everything in the annual plan, plus lifetime updates and access to future courses at no extra cost, including the Foundation Course and Build an LLM Course | $89 | Billed once | [Get lifetime access](https://algo.monster/subscribe?fpr=li69) |

One thing to be aware of: if you search around, you'll find third-party roundups quoting annual prices from $89 to $299 and lifetime prices up to $459. Those numbers come from older sales and seasonal promotions that the sites haven't updated. Prices have moved, and the only figure that matters is what the checkout page shows you on the day you buy. Confirm the amount before paying rather than trusting a blog table — including this one.

Read the two paid tiers against each other and the choice is close to trivial. Lifetime is $89 once against $69.99 per year. If you'll be touching interview prep in more than one hiring cycle — and most engineers who job-hop every two to three years will — the one-time option costs barely more than a single year and doesn't expire.

Whether it's worth paying at all depends on the gap you're trying to close. If you can already write a trie from scratch and the code just needs reps, free problem sets will do the job. If you keep solving 208-type problems without being able to say *why* a trie and not a hash set, the structured walkthrough of pattern selection is the part you're paying for. And if you're two weeks from an onsite, no course fixes a retention problem — you need volume on problems you already understand.
