# find all anagrams in a string: the sliding-window answer, why brute force times out, and how to spot the pattern before the clock runs out

Two strings go in, a list of starting indices comes out. LeetCode 438 looks like a small problem, which is exactly why it shows up so often in interviews — it separates people who can write a loop from people who can recognize a pattern and say the reason out loud.

The people searching this phrase usually want one of three things: a working solution, an explanation of why the obvious solution is too slow, or help seeing the pattern under time pressure. This covers all three, in that order.

## What the problem actually asks

Given strings `s` and `p`, return every start index in `s` where a substring is an anagram of `p`. Order of the returned indices doesn't matter.


Input:  s = "cbaebabacd", p = "abc"
Output: [0, 6]

"cba" at index 0 and "bac" at index 6 are both rearrangements of "abc".



Input:  s = "abab", p = "ab"
Output: [0, 1, 2]


Constraints worth noticing, because they rule out a whole class of solutions: both strings are at least length 1, up to 3 × 10⁴, and contain lowercase English letters only. That 30,000 upper bound is the part most candidates read past. Lowercase-only also matters — it means a 26-slot array is a legal alternative to a hash map, which is faster and simpler in Java or C++.

Two details that trip people up in the question itself: the answer returns *indices*, not substrings, and every anagram is exactly `p.length` characters long, so no window can ever be a different size.

## Why the brute-force version fails

The first version most people write checks each substring directly:

1. Slide a window of length `len(p)` across `s`.
2. For each window, compare `sorted(window)` to `sorted(p)` — or build two frequency maps and compare them.
3. Collect indices where they match.

Step 1 runs roughly `n − k + 1` times. Step 2 costs `O(k log k)` if you sort, or `O(k)` and a fresh allocation if you rebuild a map. With `n = 3 × 10⁴` and `k` close to `n`, you're looking at something in the neighborhood of 10⁸ operations, plus a new map or sorted list allocated per index. On LeetCode that's a time-limit failure; in an interview it's a hint that you haven't found the invariant yet.

The fix is not to check each window from scratch. It's to notice that consecutive windows differ by exactly two characters: one leaves on the left, one enters on the right. Nothing else changes. So the count structure you built for window `i` is already 95% of the answer for window `i + 1` — you just need to update it instead of rebuilding it.

That observation is the whole problem.

## The sliding window idea, stated plainly

Maintain one fixed-size window and two frequency structures:

- `need` — character counts of `p`. Built once, never changes.
- `window` — character counts of the current `k` characters of `s`.

Then for each new position: add the character entering on the right, subtract the character leaving on the left, and compare. Because the window never changes size, the comparison is meaningful at every step after the first window is filled.

Walking through `s = "cbaebabacd"`, `p = "abc"` (`need` = a:1, b:1, c:1):

| Window start | Characters | Counts | Match? |
| --- | --- | --- | --- |
| 0 | cba | a:1, b:1, c:1 | yes → record 0 |
| 1 | bae | a:1, b:1, e:1 | no |
| 2 | aeb | a:1, b:1, e:1 | no |
| 3 | eba | a:1, b:1, e:1 | no |
| 4 | bab | a:1, b:2 | no |
| 5 | aba | a:2, b:1 | no |
| 6 | bac | a:1, b:1, c:1 | yes → record 6 |

Seven comparisons for a 10-character string, and the counts move by one character per step instead of being rebuilt.

**A detail that matters more than it looks:** when a character's count drops to zero, delete the key rather than leaving it at 0. If you leave `e: 0` sitting in the map, the equality check can fail or succeed depending on which language and version you're on. That's not hypothetical — AlgoMonster's discussion thread for this exact problem has a user reporting a solution that returns `[0, 6]` on LeetCode and `[0]` on the platform, traced to a Python 3.10 change in how `Counter` equality treats missing versus zero-valued keys. Deleting zeroed keys removes the whole class of bug.

## Python solution

python
from collections import Counter

def find_anagrams(s: str, p: str) -> list[int]:
    n, k = len(s), len(p)
    if k > n:
        return []

    need = Counter(p)
    window = Counter(s[:k])
    res = []

    if window == need:
        res.append(0)

    for i in range(k, n):
        window[s[i]] += 1
        left = s[i - k]
        window[left] -= 1
        if window[left] == 0:
            del window[left]
        if window == need:
            res.append(i - k + 1)

    return res


The `if k > n: return []` guard is not decoration. Without it, `Counter(s[:k])` silently returns a short counter and you start comparing windows that don't exist.

## Java solution

java
class Solution {
    public List<Integer> findAnagrams(String s, String p) {
        List<Integer> res = new ArrayList<>();
        int n = s.length(), k = p.length();
        if (k > n) return res;

        int[] need = new int[26];
        int[] window = new int[26];

        for (int i = 0; i < k; i++) need[p.charAt(i) - 'a']++;

        for (int i = 0; i < n; i++) {
            window[s.charAt(i) - 'a']++;
            if (i >= k) window[s.charAt(i - k) - 'a']--;
            if (i >= k - 1 && Arrays.equals(window, need)) res.add(i - k + 1);
        }
        return res;
    }
}


Because the input is lowercase-only, the two 26-slot arrays carry the same information as the hash maps with none of the allocation. `Arrays.equals` on two `int[26]` arrays is a fixed 26-element scan, which is why the loop can stay this small.

## Complexity, and how to say it in an interview

The map version: **O(n) time, O(1) space**. The loop touches each character of `s` once, and each comparison is bounded by the size of the alphabet — 26, a constant. Space is `O(1)` for the same reason, since the counters can't grow beyond 26 keys regardless of input size. If you want to be pedantic about it, the standard phrasing is `O(n · Σ)` time where Σ is the alphabet size, which is a constant here.

The array version: identical, `O(n)` time and `O(1)` space.

Say the last part explicitly. Interviewers ask this problem to hear you explain that the fixed alphabet is what turns an apparently per-window comparison into a constant-time operation. If you say "O(n·k)" and leave it there, you've thrown away the point of your own solution.

## The mistakes that actually cost people offers

- **Rebuilding counts for every window.** Functionally correct, times out. This is the single most common failure mode.
- **Comparing `window` to `need` before the window is full.** In the Python version the first comparison happens after seeding the whole window, and in the Java version it's guarded by `i >= k - 1`. Get this wrong and you'll report spurious indices for partial windows.
- **Leaving zero-count keys in the map.** Covered above; it causes version-dependent false negatives.
- **Confusing `i - k + 1` with `i - k`.** Off by one here gives an answer that looks plausible and is wrong on every case with more than one match.
- **Using a variable-size window.** Sliding window as a general technique often means "expand until invalid, then shrink." This problem doesn't want that. The window is *always* `len(p)` — the constraint defines the size, so you never decide when to shrink, you just move. Candidates who reach for the shrink-while loop usually end up writing something harder to reason about for no gain.
- **Forgetting the `k > n` early return.** A short `p` inside a long `s` is fine; an empty or oversized `p` is not.

Checking a solution: `"abab"` with `"ab"` should produce three consecutive indices, `"aaaa"` with `"aa"` should produce `[0, 1, 2]`, and a `p` longer than `s` should produce `[]`. If your code passes the first and fails either of the other two, the guard or the window bounds are wrong.

## This problem is never asked alone

LeetCode 438 is one node in a cluster, and interviewers rotate through the cluster:

- **567. Permutation in String** — the same sliding window, but you return `true` on the first match instead of collecting every index.
- **76. Minimum Window Substring** — variable-size window with a shrink condition. Same counters, harder invariant, much harder to get right under time pressure.
- **3. Longest Substring Without Repeating Characters** — sliding window where the validity condition is on the window's own contents.
- **242. Valid Anagram** — the primitive underneath all of them: comparing character frequencies.

If you can write 438 cleanly, you're most of the way to 567. If you can't explain *why* 76 uses a different window mechanic than 438, that's the gap interviewers probe.

The reason this specific problem has a reputation is that it tests recognition, not implementation. The code is fifteen lines. Choosing to write those fifteen lines instead of the nested-loop version — within the first two minutes, while someone watches — is the actual skill being measured.

👉 [Work through the sliding-window pattern with AlgoMonster's template and problem ladder](https://algo.monster?fpr=li69)

## How to train the recognition part

Understanding the solution once is not the same as producing it under pressure. The gap is pattern identification, and it has to be drilled.

AlgoMonster is built around that gap. It's a text-and-diagram based interview prep platform from ex-Google engineers, organized into pattern lessons rather than a flat problem bank. For this problem specifically, the relevant structure looks like this:

- **Sliding window pattern lesson** with a language-agnostic template — the expand/shrink skeleton — plus the fixed-size variant this problem uses. Templates are available across multiple languages, so a Java developer can see the Python form side by side.
- **Pattern selection before coding.** You pick the pattern first and get told whether it's right and why. Choosing "sliding window" for a substring problem returns an explanation of why the pattern fits, which is essentially rehearsal for saying it out loud to an interviewer.
- **Complexity analysis as a required step,** not an afterthought. The platform prompts for time and space complexity after each solution, which is exactly the habit that makes the difference between "I solved it" and "I explained it."
- **Speedrun mode** — multiple-choice pattern questions instead of full implementations, so you can push through a large number of problems quickly. Useful when your bottleneck is recognition speed rather than typing speed.
- **A flowchart-based decision tree** for routing an unfamiliar problem to a candidate pattern (sorted input, contiguous subarray, counting constraints, and so on).
- **Curated practice lists** — Blind 75 and the Monster 50 — organized by category, with sliding window entries marked by what each one adds: introducing the pattern, adding a replacement budget, tightening shrink conditions. Practicing those lists is free with an account.

The limitation worth knowing up front: there are **no video walkthroughs at all**. Everything is written explanation, code templates, and interactive diagrams. Reviews of the platform consistently flag this, along with thin system design coverage for senior-level loops and no live coaching or mock interviews. If you learn best by watching someone code, this format will feel like work.

For a 4–8 week prep window with an interview already scheduled, the format is efficient. For a 2-week sprint before a loop, no course fixes retention.

## AlgoMonster pricing: what's actually on the page

AlgoMonster doesn't sell a monthly plan. The pricing page lists a free tier, an annual plan, and a one-time lifetime plan. Current listed prices:

| Plan | What you get | Price | Billing | Sign up |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of lessons and problems; Blind 75 and Monster 50 practice lists usable | $0 | — | [Start with the free tier](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems unlocked; 325 lessons and problems, 678 illustrations, 3,000+ editorials; unlimited AI assistant; unlimited Speedrun; company questions updated monthly; 3-day free trial | $99/year (listed at 50% off, $198 standard) | Billed annually, ~$8.25/mo | [See the yearly plan](https://algo.monster?fpr=li69) |
| Pro — Lifetime | Everything in the yearly plan, plus lifetime updates and an internal referral to a top tech company for qualified candidates | $189 one-time (listed at 50% off, $378 standard) | Billed once | [See the lifetime plan](https://algo.monster?fpr=li69) |

A few honest notes about those numbers. Discounts on this platform rotate — older reviews cite different lifetime figures, and the company's own forum has threads from users who missed a cheaper lifetime promotion and asked for it back. Treat the pricing page as the source of truth at the moment you buy, not this table. And the content counts differ across sources too: several reviews published earlier describe roughly 48 patterns and around 230 problems, while the current pricing page advertises 325 lessons and problems. The official page is the one to trust for what's live now.

The value question comes down to how many times you'll do this. If you have one interview cycle in front of you, the yearly plan covers it for less than a year of LeetCode Premium at $159. If you expect to be interviewing again in two or three years — the normal pattern for engineers who switch companies — the lifetime plan costs less than two annual renewals and stops the subscription math entirely. Neither plan makes sense if your interview is in ten days; patterns need repetition before they stick.

👉 [Check the current AlgoMonster discount and plan details](https://algo.monster?fpr=li69)

## Quick answers

**Is the sliding window answer always better than sorting each window?** For this problem, yes — the fixed window size means you update counts incrementally instead of rebuilding them, which is the difference between O(n) and O(n·k log k). For a one-off anagram check between two strings of equal length, sorting is fine and shorter to write.

**Does this problem need a hash map, or are arrays enough?** Arrays are enough because the constraints say lowercase English letters. If the problem statement allowed arbitrary Unicode, you'd need the hash map version, and the space complexity would become O(Σ) with a much larger Σ — still bounded by the alphabet, but no longer trivially constant.

**What if `p` is longer than `s`?** Return an empty list. No substring of `s` can be a permutation of a strictly longer pattern.

**Does the order of the returned indices matter?** The problem says any order. In practice, both solutions above emit indices in increasing order, which is what an interviewer expects to see.

**Where does this problem come up outside interviews?** Substring frequency counting, log scanning for permutations of a token, and any fixed-size sliding aggregate over a stream. The window-update trick is the reusable part, not the anagram framing.
