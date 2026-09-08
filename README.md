# Dota deviation report

A single file browser tool that reads a Dota 2 player's public match history from
the OpenDota API and reports which conditions correlate with winning, treating the
question as an instrumentation problem rather than a coaching one.

Open `index.html` in any browser. There is no build step, no server, no login and
no API key. React and SheetJS load from a CDN; everything else is in the file.

## Running it so that it fetches automatically

The tool needs to make two requests to OpenDota. Where it can, it does that on its
own and you never see it. Where it cannot, it falls back to a paste path that gets
you the same report by hand.

| How you open it | Fetches automatically |
|---|---|
| GitHub Pages | Yes |
| Local file, `index.html` opened directly | Yes |
| Local web server, e.g. `python3 -m http.server` | Yes |
| Published Claude artifact | No, the viewer forbids a page from making its own requests |

GitHub Pages is the easiest way to get an automatic version you can open from any
machine, including a phone. In the repository go to Settings, then Pages, set
Source to "Deploy from a branch", Branch to `main`, folder to `/ (root)`, and Save.
About a minute later it is live at
`https://<your-username>.github.io/DotaGameHistoryAnalyser/` and everything works
without pasting anything.

## Known slow endpoint

Looking an account up by display name uses OpenDota's `/search`, which scans a very
large table and times out reasonably often. The tool gives up on it after thirty
seconds rather than hanging, and tells you to use your numeric friend ID instead,
which goes to a different and much faster endpoint. A search timeout says nothing
about whether your account or your data is fine.

## What it measures

The player's own win rate across the filtered sample is the datum. Every analysis
asks whether a bucket sits far enough off that line to be distinguishable from
sampling noise.

For categorical splits the standard error on the proportion is
`sqrt(p * (1 - p) / k)`, where `p` is the datum and `k` the bucket size. The
reported sigma is how many of those standard errors the bucket sits from the datum.
A bucket must clear 1.5 sigma and hold at least 8 games before a finding is
written. Buckets below that threshold still appear in the table, muted, so it is
visible what was measured without inviting anyone to act on it.

For the wins against losses comparisons the two populations are compared with
Welch's method, which assumes neither equal variance nor equal size. A finding
needs 1.8 sigma.

Thirteen analyses run at once, so some buckets will clear 1.5 sigma by chance
alone. The tool says so on its own front page. A finding is a hypothesis to test
against the next hundred games, not a conclusion.

## Analyses

Win rate splits, each measured against your own overall win rate as the datum:
hero, match length bands, time of day in three hour bands on local time, day of
week, position within a play session, previous game result as a tilt check,
Radiant against Dire, solo against stack size, lane role (off by default, since
it needs parsed matches) and form by calendar month.

Contribution rankings, each a mean per hero measured against your own overall
figure: hero damage per minute, weighted KDA being kills plus three tenths of
assists over deaths, and stun seconds applied per match.

The two families are headlined separately. Contribution metrics separate far
harder than win rate splits because they are partly structural, a mid laner out
damages a hard support by construction, so a single merged ranking by sigma
would contain nothing but hero damage rows and would bury every behavioural
finding.

Comparing your statistics in winning games against losing games is deliberately
absent. Winning hands you towers, gold and a lower death count by construction,
so those comparisons separate at eight to ten sigma and report only that wins
looked like wins.

A bucket holding a single match is left out of every table. A row reading 0% or
100% next to an n of one is noise wearing a number. The counts are still in the
exported workbook, and every block header states how many were hidden.

## Data handling

Matches shorter than five minutes are discarded as abandons. A win is determined
by comparing `player_slot < 128` against `radiant_win`. Sessions are cut where more
than three hours pass between the end of one match and the start of the next.

OpenDota only populates parsed fields on a minority of matches, `lane_role` above
all, and it frequently returns a null `party_size`. Every field is read defensively
and nothing is inferred to fill a gap. Unparsed and unknown are reported as their
own buckets and never produce a finding.

## Account ID

The input accepts a 32 bit friend ID, a 17 bit Steam ID which is converted with
BigInt so no precision is lost, or a pasted profile URL from Steam, Dotabuff or
OpenDota. A `steamcommunity.com/id/name` vanity URL cannot be resolved, because
that requires a Steam Web API key called from a server. The Steam Web API sends no
CORS headers, so a browser page cannot call it at all regardless of key.

An empty response from OpenDota almost always means the player has not enabled
Expose Public Match Data in their Dota 2 privacy settings. The tool says this
explicitly rather than showing an empty report. Fewer than ten matches surviving
the filters is refused rather than analysed.

## Export

The export button builds an xlsx workbook with SheetJS containing a summary sheet
with run metadata and every written finding, one sheet per analysis with the
underlying counts and sigma values, and a raw sheet with one row per match under an
autofilter.

When the page is opened as a local file the workbook downloads through an ordinary
anchor element. When it is served inside the Claude artifact viewer, which does not
permit a page to start its own download, the file is handed over through the
viewer's downloads capability instead.
