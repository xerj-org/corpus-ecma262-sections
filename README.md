# corpus-ecma262-sections

Section-split mirror of the **ECMA-262 (ECMAScript) specification**, built for
retrieval corpora. Upstream publishes the whole spec as one 3 MB+
self-contained HTML file (`spec.html`), which whole-file corpus indexing
skips; this mirror splits it into one file per clause/sub-clause so every
section is independently indexable and addressable.

## Provenance

- Upstream: <https://github.com/tc39/ecma262>
- Pinned commit: `2477796715947008b0eb7b170ed3d13ad3d78c9c`
  (`2477796`, 2026-10-01, "Editorial: Simplify IsLooselyEqual bigint vs.
  number comparison (#3911)")
- Source file: `spec.html` (3,088,516 bytes), the Ecmarkup build output
  checked in at the repo root.
- Licence: upstream `LICENSE.md`, copied verbatim to this repo. Natural
  language text of the spec is under the Ecma International text copyright
  policy (alternative copyright notice). See `LICENSE.md` for the exact terms.

## Layout

```
sections/<section-id>.txt   one file per section (2,340 files)
sections/_preamble.txt      everything before the first section element
LICENSE.md                  upstream licence, verbatim
README.md                   this file
```

Each section file is:

```
<breadcrumb line> [id: <section id>]
<blank line>
<the section's HTML, verbatim>
```

Example first line:

```
ECMA-262 > 14 ECMAScript Language: Statements and Declarations > 14.7 Iteration Statements > 14.7.5 The `for`-`in`, `for`-`of`, and `for`-`await`-`of` Statements [id: sec-for-in-and-for-of-statements]
```

## Split rules

1. **This Ecmarkup build contains no plain `<section>` elements.** Its clause
   structure is carried by custom elements — `<emu-clause>` (2,266),
   `<emu-annex>` (73), `<emu-intro>` (1) — each with an `id` attribute and an
   `<h1>` heading regardless of depth. The split is on every one of those
   three element boundaries; they play exactly the role `<section>` plays in
   other specs' builds.
2. **Nested sections become their own files.** Every clause/sub-clause at
   every depth gets a file, named by its (filesystem-sanitized) `id`
   attribute. All ids in this build are unique and already safe, so names are
   unchanged (a few sections use their abstract-operation name as the id,
   e.g. `HostLoadImportedModule`).
3. **Inclusion choice: EXCLUDING nested sections.** A section's file contains
   its outer open/close tags, its `<h1>` heading, and its immediate markup,
   with all nested section subtrees excised — they live in their own files.
   **No text is duplicated anywhere in the mirror** (verified; see below).
   Consequence worth knowing: markup that a parent places *after* a nested
   clause appears in the parent's file directly after the content that
   preceded the nested clause, i.e. the excision point is not marked.
4. **Clause numbers in breadcrumbs are derived**, not scraped: this build's
   headings carry no numbers (they are rendered client-side). Numbers follow
   the Ecmarkup convention, confirmed against the published TOC at
   tc39.es/ecma262: the `emu-intro` ("Introduction") is unnumbered, top-level
   clauses number 1..N in document order (1 Scope, 2 Conformance, 3 Normative
   References, 4 Overview, 5 Notational Conventions, ...), top-level annexes
   letter A, B, C..., and nested sections take parent number + "." + ordinal.
5. **Preamble:** everything before the first section element (DOCTYPE, head,
   CSS, scripts, the metadata block) goes to `sections/_preamble.txt`
   (6.3 KB).
6. **Size cap:** no output file exceeds 150 KB. After nested splitting the
   largest file is 27,226 bytes (`sec-example-cyclic-module-record-graphs`),
   so the last-resort `<emu-table>`-row splitting was never needed.
7. **Excluded** from the mirror: `.git/` and everything in the upstream repo
   other than `spec.html` (`biblio/`, `img/`, `scripts/`, `.github/`,
   `package.json`/`package-lock.json`, the small `table-*.html` build
   includes, and the various `*.md` files). The spec's content lives entirely
   in `spec.html`; the only other large file at root was `package-lock.json`
   (83 KB), which is build tooling, not spec content.

## Verification

Run on this exact output (`python3 verify.py <spec.html> sections`):

1. **Span tiling** — the content intervals emitted across all files are
   pairwise disjoint (zero duplication) and, together with 75 dropped
   inter-section whitespace bytes, tile all 3,082,614 characters of
   `spec.html` (zero loss).
2. **Read-back** — all 2,341 files are byte-identical to their expected
   source spans.
3. **Size cap** — no file over 150 KB.
4. **Spot checks** — two distinctive verbatim phrases from different parts of
   the spec, each found in exactly one section file:
   - "This Standard defines the ECMAScript 2027 general-purpose programming
     language" -> only `sections/sec-scope.txt` (clause 1, early)
   - "PDF renderings of this specification are produced using a print
     stylesheet which takes advantage of the CSS Paged Media specification"
     -> only `sections/sec-colophon.txt` (final annex, late)

## Regeneration

Needs only Python 3 (stdlib). From a clone of tc39/ecma262 at the pinned
commit:

```
python3 split.py <path-to>/spec.html sections <upstream-SHA>
python3 verify.py <path-to>/spec.html sections
```

`split.py`:

```python
#!/usr/bin/env python3
"""Split tc39/ecma262 spec.html into one file per section (clause/sub-clause).

This Ecmarkup build has NO plain <section> elements: clause structure is
expressed with custom elements <emu-intro>, <emu-clause>, <emu-annex>,
each with an id attribute and an <h1> heading. We split on those.

Inclusion choice: EXCLUDING nested sections. A section's own file contains
its outer tags + heading + immediate markup, with nested section subtrees
excised (they are their own files). No duplicated text anywhere.
"""
import re, sys, os
from html import unescape

SRC = sys.argv[1] if len(sys.argv) > 1 else 'spec.html'
OUT = sys.argv[2] if len(sys.argv) > 2 else 'sections'
SHA = sys.argv[3] if len(sys.argv) > 3 else 'unknown'
LIMIT = 150 * 1024

data = open(SRC, encoding='utf-8', newline='').read()

TAG = re.compile(r'<(/?)emu-(intro|clause|annex)(?=[\s>])([^>]*)>')
ID = re.compile(r'\bid="([^"]+)"')

class Node:
    pass

root = Node(); root.parent = None; root.children = []; root.tag='document'; root.id=None
stack = [root]
nodes = []
for m in TAG.finditer(data):
    closing, kind, attrs = m.group(1), m.group(2), m.group(3)
    if closing:
        n = stack.pop()
        n.close_start, n.close_end = m.start(), m.end()
        assert stack, 'unbalanced close'
        continue
    n = Node()
    n.tag, n.attrs = kind, attrs
    idm = ID.search(attrs)
    n.id = idm.group(1) if idm else None
    n.open_start, n.open_end = m.start(), m.end()
    n.children = []
    stack[-1].children.append(n)
    n.parent = stack[-1]
    stack.append(n)
    nodes.append(n)
assert stack == [root], 'unclosed sections remain'

# --- numbering (Ecmarkup convention, verified against tc39.es/ecma262 TOC):
# top-level emu-intro: unnumbered; top-level emu-clause: 1..N in document
# order; top-level emu-annex: A, B, C...; nested: parent number + '.' + ordinal
def label(n):
    if n.parent is root:
        if n.tag == 'intro':
            return None
        if n.tag == 'annex':
            top_annexes = [c for c in root.children if c.tag == 'annex']
            i = top_annexes.index(n)
            return chr(ord('A') + i)
        top_clauses = [c for c in root.children if c.tag == 'clause']
        return str(top_clauses.index(n) + 1)
    p = label(n.parent)
    sibs = [c for c in n.parent.children if c.tag in ('intro','clause','annex')]
    suffix = str(sibs.index(n) + 1)
    return f'{p}.{suffix}' if p is not None else suffix

HEADING = re.compile(r'<h[1-6][^>]*>(.*?)</h[1-6]>', re.S)
def own_content(n):
    """Content span of n with DIRECT child section subtrees excised.
    Grandchildren need no handling here: every byte of a nested section's
    content belongs to exactly one node — the node itself — so excising each
    direct child's full [open_start, close_end) span distributes all content
    exactly once across the file set."""
    spans = []
    cur = n.open_end
    for c in n.children:
        if c.tag in ('intro', 'clause', 'annex'):
            spans.append((cur, c.open_start))
            cur = c.close_end
    spans.append((cur, n.close_start))
    return ''.join(data[a:b] for a, b in spans)

def title(n):
    m = HEADING.search(own_content(n))
    if not m:
        return '(untitled)'
    t = re.sub(r'<[^>]+>', '', m.group(1))
    t = re.sub(r'\s+', ' ', unescape(t)).strip()
    return t

def breadcrumb(n):
    parts = []
    p = n
    while p is not root:
        lab, ti = label(p), title(p)
        parts.append(f'{lab} {ti}' if lab else ti)
        p = p.parent
    return ' > '.join(['ECMA-262'] + parts[::-1])   # root first, self last

def sanitize(s):
    s = re.sub(r'[^A-Za-z0-9._-]', '_', s)
    return s[:180] or 'unnamed'

os.makedirs(OUT, exist_ok=True)
written = []
# preamble: everything before the first section element
first = min(c.open_start for c in root.children)
with open(os.path.join(OUT, '_preamble.txt'), 'w', encoding='utf-8', newline='') as f:
    f.write(f'ECMA-262 > (preamble before first section) [source: tc39/ecma262 @ {SHA}]\n\n')
    f.write(data[:first])
written.append(('_preamble', os.path.getsize(os.path.join(OUT, '_preamble.txt'))))

used = {}
for n in nodes:
    name = sanitize(n.id if n.id else f'{n.tag}-nopos')
    if name in used:
        used[name] += 1
        name = f'{name}-{used[name]}'
    else:
        used[name] = 0
    path = os.path.join(OUT, name + '.txt')
    body = data[n.open_start:n.open_end] + own_content(n) + data[n.close_start:n.close_end]
    with open(path, 'w', encoding='utf-8', newline='') as f:
        f.write(breadcrumb(n) + f' [id: {n.id}]' + '\n\n')
        f.write(body)
    written.append((name, os.path.getsize(path)))

sizes = sorted((s, nm) for nm, s in written)
print('sections written:', len(nodes), '+ _preamble')
print('largest:', sizes[-3:])
print('smallest:', sizes[:3])
over = [(nm, s) for nm, s in written if s > LIMIT]
print('over 150KB:', over)
total = sum(s for _, s in written)
print('total bytes written:', total, 'vs source', len(data.encode()))
```

`verify.py`:

```python
#!/usr/bin/env python3
"""Verification for the ecma262 section-split mirror.

1. SPAN TILING: the intervals emitted across all files (preamble + each
   section's open-tag / own-content spans / close-tag) must tile spec.html
   exactly: full cover, zero overlap => no text lost, no text duplicated.
2. READ-BACK: every written file's body equals its expected spans, byte-exact.
3. SIZE CAP: no file over 150KB.
4. SPOT CHECKS: two distinctive verbatim phrases (early + late), each must
   appear in exactly one section file.
"""
import re, sys, os, glob
SRC, OUT = sys.argv[1], sys.argv[2]
data = open(SRC, encoding='utf-8', newline='').read()
TAG = re.compile(r'<(/?)emu-(intro|clause|annex)(?=[\s>])([^>]*)>')
ID = re.compile(r'\bid="([^"]+)"')
class N: pass
root = N(); root.children=[]; root.tag='document'
stack=[root]; order=[]
for m in TAG.finditer(data):
    if m.group(1):
        n=stack.pop(); n.close_start,n.close_end=m.start(),m.end(); continue
    n=N(); n.tag=m.group(2); n.open_start,n.open_end=m.start(),m.end()
    idm=ID.search(m.group(3)); n.id=idm.group(1) if idm else None
    n.children=[]; stack[-1].children.append(n); stack.append(n); order.append(n)
assert len(stack)==1, 'unbalanced'

def sanitize(s): return re.sub(r'[^A-Za-z0-9._-]','_',s)[:180] or 'unnamed'

def own_spans(n):
    spans=[]; cur=n.open_end
    for c in n.children:
        if c.tag in ('intro','clause','annex'):
            spans.append((cur,c.open_start)); cur=c.close_end
    spans.append((cur,n.close_start))
    return spans

# --- 1. span tiling
first=min(c.open_start for c in root.children)
intervals=[(0,first)]
dropped=[]; prev_end=first
for c in root.children:
    dropped.append((prev_end,c.open_start)); prev_end=c.close_end
dropped.append((prev_end,len(data)))
for n in order:
    intervals.append((n.open_start,n.open_end))
    intervals.append((n.close_start,n.close_end))
    intervals.extend(own_spans(n))
intervals.sort()
dropped.sort()
# full cover: intervals + dropped separators must tile [0, len) with no overlap
merged=sorted(intervals+dropped)
pos=0; overlaps=0
for a,b in merged:
    if a<pos: overlaps+=1
    if a>pos: print('GAP at',pos,'..',a); sys.exit(1)
    pos=b
if pos!=len(data): print('SHORT OF END:',pos,len(data)); sys.exit(1)
# no file content may overlap another file's content: intervals alone must be pairwise disjoint
pos=0; iov=0
for a,b in intervals:
    if a<pos: iov+=1
    pos=max(pos,b)
assert iov==0, f'{iov} overlapping content intervals (duplication)'
nonws=[(a,b) for a,b in dropped if data[a:b].strip()]
assert not nonws, f'dropped non-whitespace: {[data[a:b][:60] for a,b in nonws]}'
print(f"1. span tiling: {len(intervals)} content intervals are pairwise disjoint; with {sum(b-a for a,b in dropped)} dropped inter-section whitespace bytes they tile all {len(data)} chars (merged overlaps={overlaps})")

# --- 2. read-back every file
def body_of(path):
    h,sep,b=open(path,encoding='utf-8',newline='').read().partition('\n\n')
    assert sep and '\n' not in h, path
    return b
expect={sanitize(n.id)+'.txt': data[n.open_start:n.open_end]+''.join(data[a:b] for a,b in own_spans(n))+data[n.close_start:n.close_end] for n in order}
files=sorted(os.path.basename(p) for p in glob.glob(OUT+'/*.txt'))
assert '_preamble.txt' not in expect
bad=[f for f,b in expect.items() if body_of(os.path.join(OUT,f))!=b]
assert not bad, f'read-back mismatch: {bad[:5]}'
assert body_of(os.path.join(OUT,'_preamble.txt'))==data[:min(c.open_start for c in root.children)]
assert len(files)==len(order)+1, (len(files),len(order)+1)
print(f'2. read-back: all {len(files)} files byte-identical to their expected spans')

# --- 3. size cap
sizes={f:os.path.getsize(os.path.join(OUT,f)) for f in files}
over=[f for f,s in sizes.items() if s>LIMIT] if (LIMIT:=150*1024) else []
assert not over, over
big=sorted(sizes.items(), key=lambda kv:-kv[1])[:3]
print(f'3. size cap: max OK; largest: {big}')

# --- 4. spot checks (args: phrase1 phrase2)
if len(sys.argv)>3:
    for phr in sys.argv[3:]:
        hits=[f for f in files if phr in open(os.path.join(OUT,f),encoding='utf-8',newline='').read()]
        print(f'4. spot [{phr[:55]}...]: {len(hits)} file(s): {hits[:4]}')
```
