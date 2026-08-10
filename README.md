# markmcnees-com

## Count guard

The data center tracker states a number of states in many places across the
site: the tracker itself, the home page, the press desk, the reporter FAQ, and
the speaker page. When the tracker adds a state, every one of those has to move
together. This grep is the check. Run it from the repo root.

```
grep -rniE "(thirteen|fourteen|1[34])[^<]{0,20}states?" --include=*.html . | grep -v 'count-guard-ok'
```

Three things to know about it.

**The pattern is count-specific and must be updated when the count changes.**
It names the current number and the one before it on purpose, so a stale value
still surfaces. When the tracker goes to fifteen, the alternation becomes
`(fourteen|fifteen|1[45])` and this section gets updated in the same commit.
A pattern that matched any number would return the whole site and get ignored.

**Both spelled-out and numeric forms are matched.** Prose uses "fourteen
states" and "thirteen-state reference"; other places use "13 states plus D.C."
An earlier word-only pattern missed every numeric reference, including a PJM
line in the tracker that had gone unreviewed for months.

**Markers must state their reason.** A bare `<!-- count-guard-ok -->` reads as
"this line is exempt" and stops the next person from looking. Write what makes
it exempt:

```html
<!-- count-guard-ok: PJM footprint, not tracker count -->
<!-- count-guard-ok: tracks tracker total minus Florida -->
<!-- count-guard-ok: tracker total, must move with the count -->
```

The third of those is not an exemption at all. It marks a line that does move
with the count but that the pattern cannot match, so the marker is the only
thing telling you it is coupled. A bare marker on that same line previously hid
a stale count on a live press page through a full review.

### Known limits

**No matching across tag boundaries.** The `[^<]{0,20}` window stops at the
next `<`, so a count split by markup is invisible. `thirteen <b>states</b>`
will not match. Counts inside a sentence broken by a link or a `<span>` have to
be caught by reading.

**No marker can go inside an `application/ld+json` block.** JSON has no comment
syntax, and an HTML comment inside the script element breaks the parse and
destroys the structured data. `press/faq.html` has two PJM references inside
its FAQPage block, currently near lines 179 and 299, that the pattern matches
and that can never be silenced. Check them by reading and move on. Do not add a
marker to make the output clean.

**Dates and markup produce false positives.** A date like "July 14, 2026,
pauses state environmental permits" matches, because "state" falls within the
character window after "14". So does an SVG attribute like `font-size="13">State`.
Both are expected. Neither is a count.

Because of those last two, the guard is a prompt to review, not a pass or fail
gate. A clean run means nothing was missed by the pattern. It does not mean
nothing was missed.

### Related

The tracker's own "Last verified" date moves only when verification actually
happened. Adding a state or fixing a link is not verification, and the date
should not move for it.
