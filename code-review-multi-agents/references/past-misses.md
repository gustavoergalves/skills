# Past misses

Patterns human reviewers caught that a multi-agent review missed or underrated. Agents 1 and 2 check each one against
the change. Add an entry whenever it happens again: the pattern, how to spot it, why it was missed.

## Committing early on a value that arrives later

**Pattern**: a new rule applies only when some input has a given value (e.g. ship country = US). While that input is
still blank, the request falls through to the older rules. One of them takes an action that is not undone later. When
the value arrives, the new rule finds the earlier action already done and cannot keep its promise.

**Spot it**: for every new condition, ask what happens to the same entity while the input is null or blank, and which
older rule handles it then. Check whether any of those older rules ignores that input entirely (e.g. "fires for any
country"). Then trace the second event, when the value arrives.

**Fix shape**: while the input is unknown, do nothing for entities the new rule covers, if an existing event re-runs
the decision once the value arrives. Check that such an event exists.

**Why it was missed**: the review saw the case and filed it as a MEDIUM test gap ("add a test for a blank country"),
and it was lost among 28 findings. It was not reclassified as an invariant break, and nobody traced the case where the
rest of the order could not follow.
