# search in rotated sorted array: the one-half-sorted rule, a clean binary search template, and the edge cases that quietly break it

Getting stuck on this problem is rarely about binary search itself. Most people can write `while left <= right` in their sleep. What trips them up is the moment the array stops being fully sorted and the mental model falls apart: which half do I keep, which comparison do I trust, and why does my code return `-1` for a target that's obviously sitting at index 4?

The fix is one sentence long: **at every step of the search, at least one of the two halves is still sorted, so you can decide which half to throw away by looking at boundaries instead of at the middle value alone.** Everything else is bookkeeping.

Below is the full reasoning, a walkthrough you can follow by hand, working code, the edge cases that show up in real interviews, and where this problem sits inside a broader practice plan if you want to stop guessing your way through binary search variants.

## What a rotated sorted array actually is

Start with a normally sorted array. LeetCode 33 uses `[0,1,2,4,5,6,7]`. Now rotate it at some pivot you aren't told about, and you might get `[4,5,6,7,0,1,2]`. The task: given that array and a target, return the index of the target, or `-1` if it isn't there. The array has distinct elements, and your algorithm has to run in `O(log n)` time.

Two properties matter, and they're easy to overlook:

- The array is not sorted, but it's **two sorted runs glued together**. In `[4,5,6,7,0,1,2]` the first run is `4,5,6,7` and the second is `0,1,2`.
- Every element in the first run is larger than every element in the second run, because the rotation moved a block of small values from the back to the front.

That second point is what makes the boundary comparisons meaningful.

## Why you can't hand this to the built-in binary search

Languages give you a library binary search, and it will happily lie to you here. `Arrays.binarySearch` or `bisect` assumes a fully sorted input. On `[4,5,6,7,0,1,2]` the sortedness assumption is false, so the algorithm may discard the half that contains your target and return a negative result. The array being "almost sorted" is exactly the trap: it looks close enough to sorted that a naive call seems reasonable, and it fails on a subset of inputs rather than on all of them.

You have to write the search yourself.

## The insight that makes the problem manageable

Picture the current window `nums[left..right]`, with `mid` somewhere inside it. Whatever the rotation looks like, the window you're searching is itself a rotated sorted subarray, which means it also splits into at most two sorted runs.

If `nums[left] <= nums[mid]`, the left half from `left` to `mid` is sorted. There's no break point inside it, because a break would force `nums[left]` to be greater than `nums[mid]`.

If that condition fails, then `nums[left] > nums[mid]`, and the pivot sits inside the left half. The right half from `mid` to `right` must then be sorted.

So each iteration gives you one sorted subarray for free. Once you know which side is sorted, you can check whether the target falls inside its value range using two comparisons:

- Left half sorted: does `nums[left] <= target < nums[mid]` hold?
- Right half sorted: does `nums[mid] < target <= nums[right]` hold?

If the target is inside the sorted range, search there. If it isn't, it has to be in the other half. Either way, half the window disappears.

An important detail: check `nums[mid] == target` and return immediately before branching. Many buggy submissions skip that and rely on the range conditions to converge, which works only sometimes.

## A dry run you can verify by hand

Take `nums = [4,5,6,7,0,1,2]` and `target = 0`.

| Iteration | left | right | mid | nums[mid] | Sorted half | Decision |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 0 | 6 | 3 | 7 | left (`4 <= 7`) | target 0 is not in `[4, 7]` → go right, `left = 4` |
| 2 | 4 | 6 | 5 | 1 | left (`0 <= 1`) | 0 is in `[0, 1]` → go left, `right = 4` |
| 3 | 4 | 4 | 4 | 0 | match | return `4` |

Three iterations for seven elements. Now the harder case, where the target is absent: same array, `target = 3`.

| Iteration | left | right | mid | nums[mid] | Sorted half | Decision |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 0 | 6 | 3 | 7 | left | 3 is not in `[4, 7]` → `left = 4` |
| 2 | 4 | 6 | 5 | 1 | left | 3 is not in `[0, 1]` → `left = 6` |
| 3 | 6 | 6 | 6 | 2 | left | 3 is not in `[2, 2]` → `left = 7` |

Now `left > right`, the loop exits, and you return `-1`. Watch iteration 3 closely: `nums[6] = 2` is the last element, and the code treats the single-element window as a sorted left half. That's fine, because the range check still fails and the window shrinks by exactly one. If your code grew the window instead of shrinking it, you'd be in an infinite loop. That's the most common reason a submission times out rather than returning a wrong answer.

## The template

Here's the standard iterative version in Python. It's the shape most interviewers expect, and it maps directly onto Java or C++ with the same comparisons.

python
def search(nums, target):
    left, right = 0, len(nums) - 1
    while left <= right:
        mid = left + (right - left) // 2
        if nums[mid] == target:
            return mid
        if nums[left] <= nums[mid]:
            # left half is sorted
            if nums[left] <= target < nums[mid]:
                right = mid - 1
            else:
                left = mid + 1
        else:
            # right half is sorted
            if nums[mid] < target <= nums[right]:
                left = mid + 1
            else:
                right = mid - 1
    return -1


Three things worth noting about the code itself:

- `mid = left + (right - left) // 2` avoids the overflow that `(left + right) / 2` causes in fixed-width integer languages. In Python it's cosmetic, but write it anyway.
- Both range checks use strict inequality on the `mid` side, because you already handled `nums[mid] == target` above. Mixing `<=` on both ends of both checks is the second most common source of wrong answers.
- The `<=` in `nums[left] <= nums[mid]` is deliberate. When `left == mid` (a one- or two-element window), the left half is trivially sorted.

**The two-pass alternative.** You can also find the rotation point first, then binary search the correct sorted segment. Find the pivot with `nums[mid] > nums[right]` comparisons, and the search space splits into `nums[0..pivot-1]` and `nums[pivot..n-1]`, each of which is genuinely sorted. Both approaches are `O(log n)`. The single-pass version above is shorter and easier to get right under pressure, which is why it's the one taught in most pattern-based curricula.

## Edge cases that decide the outcome

Nearly all of these are handled by the template without extra branches. Knowing that is the point.

**Single element.** `nums = [1]`, target `1` or `5`. `left = right = 0`, `mid = 0`, the target check fires or the range check fails, the loop exits. No special case needed.

**The array isn't rotated at all.** `[1,2,3,4,5]`. Every window has a sorted left half, so the code reduces to plain binary search.

**Target is the first or last element.** Both are covered by the range checks, since those elements define the boundaries of a sorted half at some iteration.

**Rotation near either end.** Rotating by 0 or by `n-1` produces the same two-run structure, just highly unbalanced. The search discards one half per iteration regardless of balance, so the complexity doesn't change.

**Duplicate values.** This is where the problem changes character. `search in rotated sorted array II` (LeetCode 81) drops the distinct-elements guarantee, and the worst case is no longer `O(log n)`. When `nums[left] == nums[mid] == nums[right]`, the sortedness test can't tell you which half holds the target, and you're forced to shrink by one element (`left += 1`, `right -= 1`) to make progress. An array like `[1,1,1,1,1,3,1]` can degrade to `O(n)`. Interviewers ask this variant precisely to see whether you notice that your earlier assumption has been pulled out from under you.

**Related variants worth knowing.** `Find Minimum in Rotated Sorted Array` (153) compares `nums[mid]` against `nums[right]` rather than `nums[left]`, because comparing to the left boundary is ambiguous about which side the minimum lives on. Comparing to the right boundary always points you the right direction. If you've internalized 33, 153 is a 10-minute variation, and 154 (minimum with duplicates) adds the same shrink-by-one fallback as 81.

## Complexity, stated plainly

- Time: `O(log n)` with distinct elements, since each iteration discards half the remaining window. With duplicates, `O(n)` worst case.
- Space: `O(1)` iteratively. The recursive version carries `O(log n)` stack frames, which interviewers sometimes ask about.

## Where to drill this, and how to not re-learn it four times

The reason binary search variants feel slippery is that each one comes with its own tutorial, its own variable names, and its own boundary convention. People learn 33 in isolation, then hit 153, see a different comparison in the editorial, and conclude that binary search is a pile of special cases to memorize. It isn't. It's a small number of decisions applied to different structures.

That's the argument AlgoMonster makes with its pattern-based approach: instead of presenting 33 as a standalone puzzle, it teaches binary search as a pattern with a reusable template, a decision flowchart for choosing the right variant, and a set of practice problems that drill the pattern rather than the specific problem. The decision flowchart is the piece worth caring about for this particular problem. Its whole purpose is to get you from a problem description to the right algorithm in under a minute, and "is the input sorted, or sorted and rotated?" is one of the questions it walks you through.

👉 [See how AlgoMonster teaches the binary search pattern](https://algo.monster?fpr=li69)

The platform is text and diagram-based rather than video, created by ex-Google engineers and competitive programmers. For a problem whose entire difficulty lives in conditional logic, text plus a dry run you can step through beats watching someone type code. There are interactive illustrations showing pointer movement, an inline editor with AI that will debug a failing boundary condition, and a Speedrun mode for cycling through problems fast before an interview.

The honest limits, since they matter for the decision: there are no video walkthroughs at all, so if you learn by watching someone code, this format will feel dry. The problem bank is small relative to LeetCode, on the order of a couple hundred mapped problems, which is enough to internalize patterns and not enough to build raw speed. System design coverage is introductory and won't carry a senior-level loop on its own. And the content assumes you can already write working code in one language, so it isn't a place to learn programming fundamentals.

## What it costs right now

AlgoMonster has no monthly plan and no formal free trial. You get a free tier with limited access to a subset of the content, or you pay once a year or once for life. Prices move with their frequent sales, so what's below reflects the current promotional pricing rather than a fixed list price.

| Plan | What you get | Price | Billing | Start |
| --- | --- | --- | --- | --- |
| Free | Limited access to sample patterns and problems | $0 | — | [Browse the free tier](https://algo.monster?fpr=li69) |
| Pro Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; company question sets updated monthly; 500+ lessons across 6 courses, 1,000+ illustrations and 3,000+ editorials; 12 months of full access; member discount on live bootcamps | $99/year (about $8.25/month billed annually; typically listed at $198) | Annual, cancel or upgrade anytime | [Get Pro Yearly](https://algo.monster?fpr=li69) |
| Pro Lifetime | Everything in the annual plan plus lifetime updates, one payment | $189 one-time (typically listed at $378) | One-time | [Get Pro Lifetime](https://algo.monster?fpr=li69) |

Two bonus courses are bundled with a subscription at no extra cost: a fundamentals course covering data structures and algorithms for absolute beginners, and a course on building a large language model from scratch. Some specialized partner courses, such as Quant Interview Prep, are sold separately.

## Which plan, given that you're here for one problem

If you searched for this problem because an interview is in two weeks and binary search keeps eating your time, the yearly plan at $99 is the rational choice. You'd cover binary search, two pointers, sliding window and the rest of the high-frequency patterns inside the access window, and you're not betting on returning to interview prep in three years.

The lifetime plan at $189 is a better deal for a narrower group: engineers who interview every couple of years for promotions or external offers, and who would rather not pay attention to renewals. It breaks even against two years of the annual plan, so the question is simply whether you expect to use it twice.

And if you're not sure the text-and-diagram style suits you, the free tier answers that question in an afternoon for nothing. Work through the binary search intro with the flowchart open, and you'll know whether the format clicks before spending anything.

👉 [Compare the AlgoMonster plans yourself](https://algo.monster?fpr=li69)

## Questions that come up a lot

**Is there an `O(log n)` solution when the array has duplicates?** No. The distinct-elements guarantee in problem 33 is doing real work. Once equal values are allowed, `nums[left] == nums[mid] == nums[right]` gives you no information about which half holds the target, and the worst case is `O(n)` for the duplicate variants.

**Does recursion change anything?** It's the same logic with the same complexity. Iteration is preferred because it's `O(1)` space and because the boundary conditions are easier to read in a loop.

**Do I need to find the rotation pivot first?** No, it's optional. The pivot-first approach is two binary searches over a known-sorted segment; the single-pass approach discards a half per iteration using the sorted-half observation. Both pass, and the single-pass version is less code to break.

**Which problems should I do right after this one?** In roughly increasing difficulty: `Binary Search` (704) to confirm you have the base template, `Find Minimum in Rotated Sorted Array` (153) and `Search in Rotated Sorted Array II` (81) for the two obvious variations, then `Koko Eating Bananas` (875) or `Search a 2D Matrix` (74) to practice binary search on something that isn't a plain array. The last two are where the pattern concept earns its keep, because the sortedness is hidden in the problem's structure rather than stated in the description.

Get the one-half-sorted rule down, and the family of rotated-array problems stops being a set of separate tricks. That's the whole reason this question keeps showing up in interviews: it tests whether you can reason about an invariant under a small structural change, which is a fair proxy for debugging production code you didn't write.
