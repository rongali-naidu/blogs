## PROBLEM: Longest Substring Without Repeating Characters

(a.k.a. "The Window Cleaner" / session-replay framing)

Given a string `s`, find the length of the longest contiguous substring
that contains no repeating characters.

Input:  s = "grrbear"
Output: 4            (the answer is "bear")

Input:  s = "dvdf"
Output: 3            (the answer is "vdf")

-----------------------------------------------------------------------------
PATTERN: Sliding Window (variable-size window over a sequence)
-----------------------------------------------------------------------------
Recognize this pattern whenever a problem asks for the longest / shortest /
best CONTIGUOUS run in an array or string that satisfies some constraint,
where:
  - extending the window to the right can help,
  - but extending it too far can break a rule (here: introduces a repeat),
  - so you need to shrink from the left just enough to fix the violation.

Other problems that use this exact same shape:
  - Longest substring with at most K distinct characters
  - Minimum window substring (smallest window containing all chars of a
    target string)
  - Longest subarray with sum <= target
  - Max consecutive ones III (flip at most K zeros)
  - Fruit into baskets (longest subarray with at most 2 distinct types)

General template:
```
    left = 0
    for right in range(n):
        include arr[right] in the window
        while window violates the constraint:
            remove arr[left] from the window
            left += 1
        update the answer using (right - left + 1)
```
```
This particular problem is a slight variant of that template: instead of a
`while` loop shrinking one step at a time, we can jump `left` directly to
just past the conflicting duplicate, because a dict already tells us
exactly where it is. Same idea, just a shortcut for a single-duplicate
constraint.
```


```
def longest_unique_substr(s):
    """
    Sliding window solution.

    last_seen_index : dict, char -> most recent index where it appeared.
                       Being "in" this dict only means the character has
                       shown up SOMEWHERE earlier in the string -- it does
                       NOT by itself mean it's a duplicate of our CURRENT
                       window. That distinction is exactly what trips
                       people up, and exactly what the `if` condition
                       below is built to handle.

    window_start    : the start index of the current no-repeat window.
                       This is a running boundary, not a lookup table --
                       it accumulates the effect of every shrink decision
                       made so far and only ever moves forward. It's what
                       lets us tell a "real, current" duplicate apart from
                       "stale history from before this window began."
    """
    last_seen_index = {}
    window_start = 0
    max_length = 0

    for current_index, char in enumerate(s):
        # Two conditions, not one -- and both are required:
        #
        #   (1) char in last_seen_index
        #       -> "have I seen this character ANYWHERE before?"
        #          True doesn't necessarily mean it conflicts with our
        #          current window; it might be old history that's
        #          already behind window_start.
        #
        #   (2) last_seen_index[char] >= window_start
        #       -> "...and was that earlier sighting still INSIDE my
        #          current window?" Only if both are true is this a
        #          real, live duplicate that forces a shrink.
        #
        # Worked example showing why (2) can't be skipped:
        #   s = "xyyzabx"
        #   idx:  0123456
        #   At current_index=2 ('y' repeats 'y' at idx 1) -> window_start
        #   jumps from 0 to 2. Later at current_index=6 ('x'),
        #   last_seen_index['x'] = 0, which is BEFORE window_start(2).
        #   Condition (2) evaluates 0 >= 2 -> False, so we correctly
        #   leave window_start alone. If we only checked condition (1),
        #   we'd wrongly reset window_start to 0 + 1 = 1, which would
        #   re-admit the still-duplicated 'y' at index 1 into the window
        #   (producing the invalid window "yyzabx" -- 'y' appears twice).

        if char in last_seen_index and last_seen_index[char] >= window_start:
            # Real duplicate: jump window_start to just past the
            # previous occurrence. No need to discard the whole window,
            # only the part up to and including the old duplicate.
            window_start = last_seen_index[char] + 1

        # Always record the latest position for this character, whether
        # or not it just triggered a shrink.
        last_seen_index[char] = current_index

        # Window size = current_index - window_start + 1 (inclusive on
        # both ends). At every step this is guaranteed to be the best
        # possible no-repeat window ending at current_index, so the
        # running max across all steps is the correct final answer.
        max_length = max(max_length, current_index - window_start + 1)

    return max_length


if __name__ == "__main__":
    test_cases = [
        ("grrbear", 4),   # -> "bear"
        ("dvdf", 3),      # -> "vdf"
        ("xyyzabx", 5),   # new sequence, worked through in the comments above -> "yzabx"
        ("", 0),
        ("q", 1),
        ("mmmmmmm", 1),
    ]

    for s, expected in test_cases:
        result = longest_unique_substr(s)
        status = "PASS" if result == expected else "FAIL"
        print(f"{status}: longest_unique_substr({s!r}) = {result} (expected {expected})")
```
