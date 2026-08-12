# markmcnees-com

## Count guard

The data center tracker states a number of states in many places across the
site: the tracker itself, the home page, the press desk, the reporter FAQ, and
the speaker page. When the tracker adds a state, every one of those has to move
together. This grep is the check. Run it from the repo root.

```
NUM='(^|[^0-9.])1[34]([^0-9.]|$)'
grep -rniE "(thirteen|fourteen|$NUM).{0,25}states?|states?.{0,25}(thirteen|fourteen|$NUM)" \
  --include=*.html . | grep -v 'count-guard-ok'
```

Five things to know about it.

**The pattern is count-specific and must be updated when the count changes.**
It names the current number and the one before it on purpose, so a stale value
still surfaces. When the tracker goes to fifteen, the word alternation becomes
`(fourteen|fifteen)` and `NUM` becomes `(^|[^0-9.])1[45]([^0-9.]|$)`, and this
section gets updated in the same commit.
A pattern that matched any number would return the whole site and get ignored.

**Both spelled-out and numeric forms are matched.** Prose uses "fourteen
states" and "thirteen-state reference"; other places use "13 states plus D.C."
An earlier word-only pattern missed every numeric reference, including a PJM
line in the tracker that had gone unreviewed for months.

**The numeral form is bounded, and the noun can come first.** `1[34]` on its
own also matches inside "140" and "13.5", so it is wrapped in `(^|[^0-9.])`
and `([^0-9.]|$)`. The pattern is also run in both orders, so "all 14 of them"
after the word states is caught as well as "14 states" before it. Verified
against a fixed case list: the earlier pattern was wrong on 7 of 13 cases, this
one on 2, and the remaining 2 are the date and SVG cases below, which no regex
can separate from a real count.

**It must stay ERE.** The obvious way to write the digit bounds is a PCRE
lookaround, but `grep -P` is unavailable in the Git Bash build used here and
fails with "supports only unibyte and UTF-8 locales". Everything above is
POSIX ERE so `grep -E` alone is enough.

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

**Matching across tag boundaries now works, on one line.** The window used to
be `[^<]{0,20}`, which stopped at the next `<`, so `thirteen <b>states</b>`
was invisible and had to be caught by reading. The window is now `.{0,25}` and
crosses tags. It still cannot cross a newline, because grep is line based, so a
count split across two lines is still only findable by reading.

**No marker can go inside an `application/ld+json` block.** JSON has no comment
syntax, and an HTML comment inside the script element breaks the parse and
destroys the structured data. `press/faq.html` has two PJM references inside
its FAQPage block, currently near lines 179 and 299, that the pattern matches
and that can never be silenced. Check them by reading and move on. Do not add a
marker to make the output clean.

**Dates and markup produce false positives.** A date like "July 14, 2026,
pauses state environmental permits" matches, because "state" falls within the
character window after "14". So does an SVG attribute like `font-size="13">State`.
Neither is a count. These are the only two irreducible false positives on the
site as of August 12, 2026, and both now carry a marker naming the reason, so
the guard's output is the JSON-LD pair and genuine count statements only. If you
add a date or an SVG label that trips the pattern, mark it the same way rather
than leaving it to be re-diagnosed.

Because of those last two, the guard is a prompt to review, not a pass or fail
gate. A clean run means nothing was missed by the pattern. It does not mean
nothing was missed.

### Related

The tracker's own "Last verified" date moves only when verification actually
happened. Adding a state or fixing a link is not verification, and the date
should not move for it.

## Repo conventions

### Line endings

Line endings are not uniform across this repo, and `.gitattributes` does not
normalize them. It only marks `*.pdf` as binary. As of August 10, 2026, of the
15 HTML files, 12 use CRLF and 3 use LF:

- `press/data-center-tracker.html`
- `press/cost-of-serving-large-load.html`
- `resources/index.html`

No file currently mixes the two, and none should.

**Content inserted by a script has to match the file it is going into, not a
repo-wide assumption.** A multi-line string authored with LF and written into a
CRLF file produces mixed endings silently. Nothing errors, the page still
renders, and the damage only shows up as a diff that claims the whole region
changed. Check the target first:

```
node -e "const s=require('fs').readFileSync(process.argv[1],'utf8');const c=(s.match(/\r\n/g)||[]).length,l=(s.match(/\n/g)||[]).length;console.log(c===0?'LF':c===l?'CRLF':'MIXED')" press/faq.html
```

If the target is CRLF, convert the inserted text with `.replace(/\n/g, "\r\n")`
before writing.

Anchor strings have the same problem in reverse. An anchor containing
`</div>\n</div>` will not match a CRLF file, where the bytes are
`</div>\r\n</div>`, so the replacement finds nothing and writes nothing. That
failure is silent unless you check. Count the matches for every replacement and
refuse to write the file when a count is not what you expected.

### Structured data

Several pages carry an `application/ld+json` block whose `dateModified` is
separate from the visible "Last verified" line, and the two drift. When a
page's visible date moves, move `dateModified` with it.

`press/faq.html` goes further: each answer's text is mirrored inside its
FAQPage block. An edit to a visible answer has to land in both copies, or the
structured data will keep asserting something the page no longer says. Anchor
on a sentence that appears in both and expect two matches, rather than editing
the visible copy alone.
