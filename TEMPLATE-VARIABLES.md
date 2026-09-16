# AstraLink Connect — Report Emails: Template Variable Contract

| Template | File | Duplicate copy |
|---|---|---|
| Weekly Report | `index.html` | `weekly-report-static.html` |
| Monthly Report | `monthly.html` | `monthly-report-static.html` |

**Status: the HTML files are static and stay static.** They carry the
design-approved sample values, not placeholders. This document is the specification
of which of those values are dynamic, what to call each one, and how to resolve the
conditional states — the backend team inserts the placeholders when they wire the
files into their renderer.

Suggested placeholder syntax throughout is **Handlebars / Mustache dotted paths**
(`{{report.site_name}}`), which Jinja2, Django, Liquid and Thymeleaf also read
unchanged — but the naming is a proposal, not a constraint. §13 is the complete
insertion map: every value to replace, with its exact line number in each file and
the static text that currently sits there.

---

## 1. Ground rules for the backend

1. **Keep the templates free of logic.** Every colour, icon URL, arrow glyph and
   formatted number should be resolved server-side and delivered as a plain string.
   Reason: these values sit *inside* `style="..."` attributes and Outlook-specific
   table markup. A mis-nested `{{#if}}` inside an inline style silently breaks layout
   in Outlook/Word, and most ESP template engines (SES, SendGrid, Mailchimp) handle
   nested conditionals inconsistently. §6 shows how every conditional state can be
   expressed as data instead of markup.
2. **If your renderer is a plain string-replace** with no nested-object support,
   flatten each path with underscores — `{{stats.threat_detections.value}}` becomes
   `{{stats_threat_detections_value}}`. Whatever you choose, tell the frontend, so
   both files get the same treatment rather than drifting apart.
3. **HTML-escape every interpolated value** (`& < > " '`) with two documented
   exceptions, both of which carry raw HTML entities by design:
   - `glance.*.change.glyph` — `&#9650;` / `&#9660;` / `&mdash;`
   - `glance.*.value` — carries `&nbsp;` between number and unit
   Everything else must be escaped. Site names, port lists and hostnames come from
   network telemetry and must never be able to inject markup into the email.
4. **Numbers are formatted by the backend, not the template.** Rules in §7.
5. **Never leave a placeholder unresolved.** An unrendered `{{...}}` ships to the
   customer. Every token has a required value; §6 gives the neutral/empty value for
   each state where there is nothing to show.

---

## 2. Shared variables (both templates)

W = `index.html` (weekly), M = `monthly.html`.

| Variable | Example value | W line | M line | Notes |
|---|---|---|---|---|
| `subject` | `Your weekly network report — 01-07 Sep` | — | — | Not in the HTML. Set on the send. Keep at or under 60 chars. |
| `preheader` | `Here's what protected your network this week — detected, blocked and monitored.` | 212 | 232 | Hidden inbox preview line. 40-100 chars. Should not repeat the subject verbatim. |
| `report.type_label` | `Weekly Report` / `Monthly Report` | 254 | 274 | Meta bar, left. Fixed per template — make it a variable only if one renderer serves both files. |
| `report.site_name` | `Tilbury House` | 266 | 286 | Meta bar. Cell is `white-space:nowrap` — **max ~20 chars** or the meta bar overflows 640px. Truncate longer names server-side with an ellipsis. |
| `report.period_range` | `01 Sep - 07 Sep` (W) · `01 Sep - 01 Oct` (M) | 283 | 303 | Format `DD Mmm - DD Mmm`, hyphen-minus with single spaces (matches the design). Use the **site's local timezone**, not UTC. |
| `report.period_year` | `2026` | 294 | 314 | Sits after a 1px divider. Send the year the range *ends* in; if the range spans a year boundary use `2026-27`. |
| `report.comparison_label` | `VS PREVIOUS WEEK` / `VS PREVIOUS MONTH` | 351, 389, 427 | 378, 419, 460, 514, 555 | Once per stat card — 3x weekly, 5x monthly. Uppercase. Same value in every occurrence. |
| `report.intro_line` | `Here's what your network handled over the past seven days.` | 320 | 344 | Sub-heading under "Detected, blocked, monitored". **Open item:** the monthly file's original copy read "over the previous month" — confirm final monthly wording with design. |

### Links

| Variable | Example | W line | M line | Notes |
|---|---|---|---|---|
| `report.url` | `https://app.astralinkconnect.com/reports/2026-w36` | 477, 480 | 726, 729 | "VIEW FULL REPORT" button. **Two occurrences** — the text and the arrow image are separate `<a>` elements. Both must get the same value. |
| `report.all_reports_url` | `https://app.astralinkconnect.com/reports` | — | 581, 588 | Monthly only — the 6th "View all reports" tile. Also **two occurrences** (text + round arrow). |
| `recipient.unsubscribe_url` | `https://www.astralinkconnect.com/u/<signed-token>` | 676 | 925 | Per-recipient. See §8. |

Two links are left hardcoded because they never vary — the footer band
(`https://astralinkconnect.com`, W 662 / M 911) and Privacy
(`.../legal/privacy-policy`, W 678 / M 927). Tokenize them only if that changes.

---

## 3. Stat cards

Weekly has **3** cards. Monthly has **6** — the same 3 plus two more; the 6th slot is
the static "View all reports" tile and carries no data.

Each data card takes four values, where `<key>` is the card key below:

| Field | Example | Notes |
|---|---|---|
| `stats.<key>.value` | `17,059` | Formatted display string, already comma-grouped / unit-suffixed. See §7. |
| `stats.<key>.trend.pct` | `7%` | **Absolute** value, no sign — the arrow carries the direction. Integer + `%`. |
| `stats.<key>.trend.icon_url` | `https://d12sb60thikcyd.cloudfront.net/static-assets/email/icon-red-up.png` | Full absolute URL, not a filename. Resolved per §6.1. |
| `stats.<key>.trend.color` | `#ee0004` | Hex for the percentage text. Resolved per §6.1. |

The two-line card label (`Threats<br>detections`) is left static — it is design copy,
and the `<br>` forces the intended wrap in the 135px measure. Raise a change request
if it needs to vary.

### Card keys

| # | Card key | Label as designed | Icon (static) | W | M | Trend line (W/M) | Value line (W/M) |
|---|---|---|---|---|---|---|---|
| 1 | `threat_detections` | THREATS / DETECTIONS | `icon-threats.png` | yes | yes | 347 / 371 | 361 / 388 |
| 2 | `malicious_domains_blocked` | MALICIOUS / DOMAINS BLOCKED | `icon-domains.png` | yes | yes | 385 / 412 | 399 / 429 |
| 3 | `devices_on_network` | DEVICES / ON NETWORK | `icon-devices.png` | yes | yes | 423 / 453 | 437 / 470 |
| 4 | `most_severe_detections` | MOST-SEVERE / DETECTIONS | `icon-detenctions.png` | — | yes | — / 507 | — / 524 |
| 5 | `traffic_inspected` | TRAFFIC / INSPECTED | `icon-traffic.png` | — | yes | — / 548 | — / 565 |
| 6 | *(static CTA tile)* | VIEW ALL REPORTS | `icon-round-arrow.png` | — | yes | — | — / 581 |

All three trend fields (`icon_url`, `color`, `pct`) sit on the single trend line for
that card.

> The asset filename `icon-detenctions.png` carries a typo on the CDN. It is
> referenced exactly as it exists there. Do not "correct" it without renaming the
> file on CloudFront in the same change.

---

## 4. "This month at a glance" table — monthly only

7 rows, fixed order, `monthly.html` lines 653-687. Heading and sub-line at 616 / 621
are static.

**The metric labels are static too** — deliberately. Only the value and the change
should be variable, keyed by metric name, so a label can never end up next to another
metric's number.

| Field | Example | Notes |
|---|---|---|
| `glance.<key>.value` | `68.3&nbsp;TB` | **Use `&nbsp;` between number and unit**, not a plain space. Delivered unescaped (§1.3). |
| `glance.<key>.change.glyph` | `&#9650;` / `&#9660;` / `&mdash;` | Text glyph, not an image. Up / down / no-change. Delivered unescaped (§1.3). |
| `glance.<key>.change.pct` | `12%` | Absolute, no sign. Empty string in the no-change and no-baseline states. |
| `glance.<key>.change.color` | `#008a17` | Per §6.1. **The green here is `#008a17`, not the card green `#04c023`** — a different tone against the light-grey card. Do not share one constant between the two. |

The composed change cell renders as `{{glyph}}&nbsp;{{pct}}`. In the no-change state
`pct` is empty, so the cell renders an em dash followed by a non-breaking space —
invisible in a left-aligned cell, which is why the markup needs no conditional.

| Row | Static label (fixed) | `glance` key | Sample value | Sample change | Value / change line |
|---|---|---|---|---|---|
| 1 | Traffic inspected | `traffic_inspected` | 68.3 TB | down 12% | 655 / 656 |
| 2 | Threat detections | `threat_detections` | 62,317 | down 18% | 660 / 661 |
| 3 | Most-severe detections | `most_severe_detections` | 28,441 | up 9% | 665 / 666 |
| 4 | Malicious domains blocked | `malicious_domains_blocked` | 9,465 | up 46% | 670 / 671 |
| 5 | Name lookups handled | `name_lookups_handled` | 447,054 | up 16% | 675 / 676 |
| 6 | Devices on the network | `devices_on_network` | 52 | up 13% | 680 / 681 |
| 7 | Appliance restarts | `appliance_restarts` | 0 | — | 685 / 686 |

Row 7 is the only row whose cells carry no `padding-bottom:10px`. That is already
correct in the markup — it only matters if someone later converts these 7 rows into a
loop, in which case the padding must be applied to all but the last row.

> `glance.threat_detections`, `glance.devices_on_network`,
> `glance.malicious_domains_blocked`, `glance.most_severe_detections` and
> `glance.traffic_inspected` are the **same metrics as the stat cards** in §3, and in
> the design the sample numbers deliberately differ. Confirm with product whether both
> sections cover the same period — if they do, render both from one aggregation so the
> card and the table can never disagree in a customer's inbox.

---

## 5. Key Takeaways — 3 cards

Weekly lines 495-588 · monthly 744-837. Same structure in both; only the card
background differs (grey cards on white in weekly, white cards on grey in monthly),
which is design, not data.

Three fixed types. Recommended as **typed slots rather than free HTML** — only the
numbers vary, so nothing HTML-shaped comes out of the database.

| Type | Title (keep static) | Icon (keep static) | Text accent (keep static) | Variables | W line | M line |
|---|---|---|---|---|---|---|
| `action` | Action required | `icon-action.png` | `#ee0004` | `takeaways.action.count` | 518 | 767 |
| `watch` | Watch item | `icon-watch.png` | `#a67d03` | `takeaways.watch.count`, `takeaways.watch.ports` | 548 | 797 |
| `good` | Working well | `icon-working.png` | `#009919` | none — copy is fully static | — | — |

Rendered bodies:

```
action: <span style="font-weight:500; color:#ee0004;">{{takeaways.action.count}}</span> high-severity
        detections. Please review the detection list and confirm each source.

watch:  <span style="font-weight:500; color:#a67d03;">{{takeaways.watch.count}}</span> detections from
        an internal address.<br>Services targeted:
        <span style="font-weight:500; color:#a67d03;">{{takeaways.watch.ports}}</span>

good:   Your appliance is healthy and all security services are running as expected.
```

`takeaways.watch.ports` is a comma-separated list rendered as one string
(`443, 53, 80`). Cap it — the card is 109px tall and a long port list pushes the body
to a third line.

Note the watch accent: the **icon** is `#EBB000` but the **text** accent is `#a67d03`,
a darker amber chosen for contrast on a light card. They are not interchangeable.

If product later needs the sentences themselves to vary (different findings, not just
different numbers), switch to a `takeaways[]` array of
`{ type, title, icon_url, accent_color, body_html }` where `body_html` is composed
server-side from a **whitelist of sentence templates** with escaped scalars
substituted in — never from free text stored in the DB. Copy budget: the body measure
is 264px at 12px/16px, so roughly **55 characters per line, 2 lines comfortably**
(about 120 chars). The card is `height:109px`; a 3rd line grows it to ~125px, which is
tolerable but shifts the 8px rhythm.

---

## 6. Conditional states — how the template handles "it depends"

Every condition here is resolved into data, not markup. **The rule the design
implements is direction-based, not semantics-based:** any increase is red, any
decrease is green. It is *not* "good news vs bad news" — a drop in traffic and a drop
in malicious domains are both green.

### 6.1 Trend direction (the up/down arrow)

Compute `delta = current - previous`, then resolve all fields for that metric together:

| State | Condition | `trend.icon_url` (cards) | `trend.color` (cards) | `change.glyph` (glance) | `change.color` (glance) | `pct` |
|---|---|---|---|---|---|---|
| **Up** | `delta > 0` | `icon-red-up.png` | `#ee0004` | `&#9650;` | `#ee0004` | `abs(pct)%` |
| **Down** | `delta < 0` | `icon-green-down.png` | `#04c023` | `&#9660;` | `#008a17` | `abs(pct)%` |
| **Flat** | `delta == 0` | `icon-flat.png` **(new asset)** | `#8e8e8e` | `&mdash;` | `#0563ff` | `""` (empty) |
| **No baseline** | no previous period, or `previous == 0` | `icon-blank.png` **(new asset)** | `#8e8e8e` | `&mdash;` | `#0563ff` | `""` (empty) |

Percentage: `round(abs(delta) / previous * 100)`, integer, no decimals, no sign.
**Guard `previous == 0`** — that is the "no baseline" row, not a 100% or infinite
change. Cap the display at `>999%` so the string stays inside the 82px trend column.

Two 12x12 PNGs are needed to keep the templates logic-free, and they are the only new
assets this contract requires. Upload both to
`https://d12sb60thikcyd.cloudfront.net/static-assets/email/`:

- **`icon-flat.png`** — a grey (`#8E8E8E`) dash or equals mark, for a genuine 0% change.
- **`icon-blank.png`** — fully transparent, for the first-ever report where there is
  no previous period. Paired with `trend.pct: ""` and `report.comparison_label: ""`,
  the whole trend block renders blank while the 82px column holds its width, so no
  card shifts.

> `icon-green-down.png` is a pre-flipped PNG — Figma exports both arrows on the same
> up-right path and rotates the green one in CSS, which email clients cannot do, so
> the rotation is baked into the file. Never swap the two URLs expecting a flip.

**If you would rather branch in the template** (only with a real engine — Handlebars,
Jinja2, Liquid — never a regex replace), the equivalent of one trend cell is:

```handlebars
{{#if trend.is_up}}
  <img src="{{cdn}}/icon-red-up.png" width="12" height="12" alt="" style="...">
  <span style="...; color:#ee0004;">{{trend.pct}}</span>
{{else if trend.is_down}}
  <img src="{{cdn}}/icon-green-down.png" width="12" height="12" alt="" style="...">
  <span style="...; color:#04c023;">{{trend.pct}}</span>
{{else}}
  <span style="...; color:#8e8e8e;">&mdash;</span>
{{/if}}
```

The pre-resolved version that ships in these files is still preferred: the rule lives
in one place, the Outlook-critical markup is never duplicated across three branches,
and a rule change never needs a template redeploy or another round of client testing.

### 6.2 A whole stat card has no data

Do not remove the `<td>` — the row is a fixed 3-column table and dropping a cell
re-flows the other two in Outlook. Send `stats.<key>.value: "—"`, the no-baseline
trend triple, and `report.comparison_label: ""`. The card keeps its 185px slot and
reads as "not measured".

### 6.3 Fewer than 3 takeaway cards

All three cards render unconditionally today. Making the count variable is the one case
that genuinely needs a loop or three `{{#if}}` blocks, because omitting a card must
also omit its 8px gap: cards 2 and 3 carry `padding-top:8px` on their wrapper `<td>`,
card 1 does not.

```handlebars
{{#each takeaways}}
  <tr><td{{#unless @first}} style="padding-top:8px;"{{/unless}}>
     ... card markup ...
  </td></tr>
{{/each}}
```

Recommended policy so the section is never empty or ragged:

- **0 findings** -> render the `good` card alone ("Your appliance is healthy...").
- **1-3 findings** -> fixed severity order: `action`, then `watch`, then `good`.
- **More than 3** -> render the 3 highest-severity and let "VIEW FULL REPORT" carry
  the rest. Do not add a 4th card; the left column's copy and button are sized to a
  331 x ~343px right column.
- Never hide the section heading on its own — either the whole Key Takeaways row
  renders or none of it does.

### 6.4 The "at a glance" table

All 7 rows always render, in the fixed order in §4. A metric the appliance did not
report gets `value: "—"` and the no-baseline change — not a dropped row. A missing row
silently changes the card height and hides from the customer that the metric exists at
all.

### 6.5 Not conditional today, but worth a decision

- The grey chevron on each takeaway card (`icon-gray-right.png`) **is not a link** in
  either template — it reads as clickable but is inert. Either wrap it (and the card
  title) in an `<a>` pointing at a per-finding deep link, or accept it as decoration.
  Flagging it because it will generate support tickets as-is.
- The monthly hero (`monthly-hero.png`) is a 640px @1x export and renders soft on
  retina; the weekly hero ships at @4x. Re-export at @2x (1280 x 626). Not a variable,
  but it ships with this template.

---

## 7. Number and date formatting (backend-side)

| Value | Rule | Example |
|---|---|---|
| Counts | Comma thousands separator, en-US | `17,059` · `447,054` |
| Large counts | The 32px card value sits in a 135px measure — **abbreviate above 6 digits** or it clips | `1,234,567` -> `1.2M` |
| Data volume | Auto-scale GB/TB/PB, 1 decimal, `&nbsp;` before the unit | `68.3&nbsp;TB` |
| Percentages | Integer, absolute, `%` suffix, no sign | `7%` · `>999%` (capped) |
| Zero | Literal `0`, not `—`. `—` means "not measured" | `0` |
| Date range | `DD Mmm - DD Mmm`, site-local timezone | `01 Sep - 07 Sep` |
| Year | 4 digits, separate variable | `2026` |
| Locale | en-US throughout; no localisation in scope for v1 | |

Any string landing in a `white-space:nowrap` cell must be truncated server-side:
`report.site_name` (~20 chars), the glance `value` column, and the trend `pct`.

---

## 8. Security notes for the send pipeline

Report emails carry customer network telemetry, so the send path is in scope for the
same controls as the app:

- **`recipient.unsubscribe_url` must be a signed, single-purpose, expiring token.**
  Never an email address, customer ID or tenant ID in the query string — a guessable
  or enumerable unsubscribe endpoint lets anyone unsubscribe another customer, and
  confirms a valid address to whoever probes it. Add `List-Unsubscribe` and
  `List-Unsubscribe-Post` headers alongside it.
- **`report.url` and `report.all_reports_url` must land on an authenticated page** that
  re-verifies the viewer's identity, tenant membership and ownership of that report
  server-side. A report ID in a URL is not an authorization decision, and email links
  leak — forwarded threads, mail archives, corporate scanners.
- **Recipient resolution is a tenant-scoped query.** Build the send list from the
  authenticated tenant/site relationship, not from a job payload that names a site. A
  report delivered to the wrong tenant is a cross-tenant data disclosure.
- **Escape every interpolated value** except the two documented in §1.3. Port lists,
  site names and hostnames come from device data and are untrusted input.
- **Log each send** with recipient, tenant, site, template, report period and template
  version — SOC 2 evidence, and the only way to answer "who received which numbers"
  after the fact.
- Confirm with security whether raw counts of high-severity detections are acceptable
  in unencrypted email for every customer tier, or whether some tiers should receive a
  notification-only variant.

---

## 9. Example payload — weekly (`index.html`)

```json
{
  "subject": "Your weekly network report — 01-07 Sep",
  "preheader": "Here's what protected your network this week — detected, blocked and monitored.",
  "report": {
    "type_label": "Weekly Report",
    "site_name": "Tilbury House",
    "period_range": "01 Sep - 07 Sep",
    "period_year": "2026",
    "comparison_label": "VS PREVIOUS WEEK",
    "intro_line": "Here's what your network handled over the past seven days.",
    "url": "https://app.astralinkconnect.com/reports/2026-w36"
  },
  "recipient": {
    "unsubscribe_url": "https://www.astralinkconnect.com/u/eyJhbGciOi..."
  },
  "stats": {
    "threat_detections": {
      "value": "17,059",
      "trend": {
        "pct": "7%",
        "icon_url": "https://d12sb60thikcyd.cloudfront.net/static-assets/email/icon-red-up.png",
        "color": "#ee0004"
      }
    },
    "malicious_domains_blocked": {
      "value": "9,059",
      "trend": {
        "pct": "7%",
        "icon_url": "https://d12sb60thikcyd.cloudfront.net/static-assets/email/icon-green-down.png",
        "color": "#04c023"
      }
    },
    "devices_on_network": {
      "value": "59",
      "trend": {
        "pct": "7%",
        "icon_url": "https://d12sb60thikcyd.cloudfront.net/static-assets/email/icon-red-up.png",
        "color": "#ee0004"
      }
    }
  },
  "takeaways": {
    "action": { "count": "5,616" },
    "watch":  { "count": "3,512", "ports": "443, 53, 80" }
  }
}
```

## 10. Example payload — monthly (`monthly.html`)

Everything in §9 (with the monthly values below) plus two more stat cards and the
`glance` object.

```json
{
  "report": {
    "type_label": "Monthly Report",
    "period_range": "01 Sep - 01 Oct",
    "period_year": "2026",
    "comparison_label": "VS PREVIOUS MONTH",
    "intro_line": "Here's what your network handled over the previous month.",
    "url": "https://app.astralinkconnect.com/reports/2026-09",
    "all_reports_url": "https://app.astralinkconnect.com/reports"
  },
  "stats": {
    "most_severe_detections": {
      "value": "28,991",
      "trend": { "pct": "7%", "icon_url": ".../icon-red-up.png", "color": "#ee0004" }
    },
    "traffic_inspected": {
      "value": "68.3 TB",
      "trend": { "pct": "12%", "icon_url": ".../icon-green-down.png", "color": "#04c023" }
    }
  },
  "glance": {
    "traffic_inspected":         { "value": "68.3&nbsp;TB", "change": { "glyph": "&#9660;", "pct": "12%", "color": "#008a17" } },
    "threat_detections":         { "value": "62,317",       "change": { "glyph": "&#9660;", "pct": "18%", "color": "#008a17" } },
    "most_severe_detections":    { "value": "28,441",       "change": { "glyph": "&#9650;", "pct": "9%",  "color": "#ee0004" } },
    "malicious_domains_blocked": { "value": "9,465",        "change": { "glyph": "&#9650;", "pct": "46%", "color": "#ee0004" } },
    "name_lookups_handled":      { "value": "447,054",      "change": { "glyph": "&#9650;", "pct": "16%", "color": "#ee0004" } },
    "devices_on_network":        { "value": "52",           "change": { "glyph": "&#9650;", "pct": "13%", "color": "#ee0004" } },
    "appliance_restarts":        { "value": "0",            "change": { "glyph": "&mdash;", "pct": "",    "color": "#0563ff" } }
  }
}
```

(`icon_url` values are abbreviated with `...` here for readability — send the full
absolute CloudFront URL.)

---

## 11. Do NOT template these

Everything not listed in §13 is design, not data. In particular, leave alone:

- All inline `style="..."` values other than the resolved colours in §6.1, and every
  `width` / `height` attribute. The pre-blended hex values (`#3782ff`, `#e9eef6`,
  `#7aacff`, `#bcbcbc`, `#acafb1`) are rgba colours flattened by hand for Outlook —
  recomputing them changes the rendering.
- The hero image and the footer band. Both are flat PNGs with the type baked in
  *specifically* because Gmail's forced dark-mode inversion recolours live text but
  cannot touch pixels. Do not replace either with live HTML text.
- The `@font-face` block, the `<!--[if mso]>` conditional comments, the
  `prefers-color-scheme` / `[data-ogsc]` dark-mode rules, and the `.serif-i` class —
  all client-compatibility scaffolding.
- The section headings ("Detected, blocked, monitored", "This month at a glance",
  "Key Takeaways"), the at-a-glance metric labels, the stat-card labels, the takeaway
  titles, the CTA band copy and the sign-off. Static in v1 — raise a change request if
  marketing wants any of them variable.
- The `<title>` tag and the hero `alt` text. Fixed per template.

---

## 12. QA checklist before the first live send

- [ ] Diff your tokenized copy against `index.html` / `monthly.html` before the first
      render. Only the §13 lines should differ — 20 lines in weekly, 42 in monthly.
      Anything else has changed the design.
- [ ] Render both templates with the §9 / §10 payloads and diff against
      `index.html` / `monthly.html` again. The output should come back to the same 20
      and 42 lines, now carrying live values instead of the samples.
- [ ] Grep the rendered output for `{{` and for `href="#"` — both must return zero hits.
- [ ] Render each of the four §6.1 states, including a first-ever report with no
      baseline, and confirm no card shifts.
- [ ] Render the widest realistic values: a 20-char site name, a 6-digit card value, a
      `>999%` trend, a long port list, a 3-line takeaway body.
- [ ] Confirm `icon-flat.png` and `icon-blank.png` are live on CloudFront before any
      send that could hit a flat or no-baseline state.
- [ ] Client test at minimum: Gmail web + iOS + Android (forced dark mode), Outlook
      Windows (Word engine), Apple Mail macOS + iOS, Outlook iOS/Android.
- [ ] Confirm the unsubscribe token is per-recipient, signed and expiring, and that
      `report.url` requires authentication.

---

## 13. Insertion map

Every value to replace, with the line it sits on and the static text currently there.
Nothing outside this list should change. Weekly is 20 lines, monthly 42.

### `index.html` — weekly

| Line | Static text now | Replace with |
|---|---|---|
| 212 | `Here's what protected your network this week — detected, blocked and monitored.` | `{{preheader}}` |
| 254 | `Weekly Report` | `{{report.type_label}}` |
| 266 | `Tilbury House` | `{{report.site_name}}` |
| 283 | `01 Sep - 07 Sep` | `{{report.period_range}}` |
| 294 | `2026` | `{{report.period_year}}` |
| 320 | `Here&rsquo;s what your network handled over the past seven days.` | `{{report.intro_line}}` |
| 347 | `.../icon-red-up.png` · `color:#ee0004` · `7%` | `{{stats.threat_detections.trend.icon_url}}` · `.trend.color` · `.trend.pct` |
| 351 | `VS PREVIOUS WEEK` | `{{report.comparison_label}}` |
| 361 | `17,059` | `{{stats.threat_detections.value}}` |
| 385 | `.../icon-green-down.png` · `color:#04c023` · `7%` | `{{stats.malicious_domains_blocked.trend.icon_url}}` · `.trend.color` · `.trend.pct` |
| 389 | `VS PREVIOUS WEEK` | `{{report.comparison_label}}` |
| 399 | `9,059` | `{{stats.malicious_domains_blocked.value}}` |
| 423 | `.../icon-red-up.png` · `color:#ee0004` · `7%` | `{{stats.devices_on_network.trend.icon_url}}` · `.trend.color` · `.trend.pct` |
| 427 | `VS PREVIOUS WEEK` | `{{report.comparison_label}}` |
| 437 | `59` | `{{stats.devices_on_network.value}}` |
| 477 | `href="#"` | `href="{{report.url}}"` |
| 480 | `href="#"` | `href="{{report.url}}"` |
| 518 | `5,616` | `{{takeaways.action.count}}` |
| 548 | `3,512` · `443, 53, 80` | `{{takeaways.watch.count}}` · `{{takeaways.watch.ports}}` |
| 676 | `href="https://www.astralinkconnect.com/"` | `href="{{recipient.unsubscribe_url}}"` |

### `monthly.html` — monthly

| Line | Static text now | Replace with |
|---|---|---|
| 232 | `Here's what protected your network this month — detected, blocked and monitored.` | `{{preheader}}` |
| 274 | `Monthly Report` | `{{report.type_label}}` |
| 286 | `Tilbury House` | `{{report.site_name}}` |
| 303 | `01 Sep - 01 Oct` | `{{report.period_range}}` |
| 314 | `2026` | `{{report.period_year}}` |
| 344 | `Here&rsquo;s what your network handled over the previous month.` | `{{report.intro_line}}` |
| 371 | `.../icon-red-up.png` · `color:#ee0004` · `7%` | `{{stats.threat_detections.trend.icon_url}}` · `.trend.color` · `.trend.pct` |
| 378 | `VS PREVIOUS MONTH` | `{{report.comparison_label}}` |
| 388 | `17,059` | `{{stats.threat_detections.value}}` |
| 412 | `.../icon-green-down.png` · `color:#04c023` · `7%` | `{{stats.malicious_domains_blocked.trend.*}}` |
| 419 | `VS PREVIOUS MONTH` | `{{report.comparison_label}}` |
| 429 | `9,059` | `{{stats.malicious_domains_blocked.value}}` |
| 453 | `.../icon-red-up.png` · `color:#ee0004` · `7%` | `{{stats.devices_on_network.trend.*}}` |
| 460 | `VS PREVIOUS MONTH` | `{{report.comparison_label}}` |
| 470 | `59` | `{{stats.devices_on_network.value}}` |
| 507 | `.../icon-red-up.png` · `color:#ee0004` · `7%` | `{{stats.most_severe_detections.trend.*}}` |
| 514 | `VS PREVIOUS MONTH` | `{{report.comparison_label}}` |
| 524 | `28,991` | `{{stats.most_severe_detections.value}}` |
| 548 | `.../icon-green-down.png` · `color:#04c023` · `12%` | `{{stats.traffic_inspected.trend.*}}` |
| 555 | `VS PREVIOUS MONTH` | `{{report.comparison_label}}` |
| 565 | `68.3 TB` | `{{stats.traffic_inspected.value}}` |
| 581 | `href="#"` | `href="{{report.all_reports_url}}"` |
| 588 | `href="#"` | `href="{{report.all_reports_url}}"` |
| 655 | `68.3&nbsp;TB` | `{{glance.traffic_inspected.value}}` |
| 656 | `color:#008a17` · `&#9660;&nbsp;12%` | `color:{{glance.traffic_inspected.change.color}}` · `{{...change.glyph}}&nbsp;{{...change.pct}}` |
| 660 | `62,317` | `{{glance.threat_detections.value}}` |
| 661 | `color:#008a17` · `&#9660;&nbsp;18%` | `{{glance.threat_detections.change.*}}` |
| 665 | `28,441` | `{{glance.most_severe_detections.value}}` |
| 666 | `color:#ee0004` · `&#9650;&nbsp;9%` | `{{glance.most_severe_detections.change.*}}` |
| 670 | `9,465` | `{{glance.malicious_domains_blocked.value}}` |
| 671 | `color:#ee0004` · `&#9650;&nbsp;46%` | `{{glance.malicious_domains_blocked.change.*}}` |
| 675 | `447,054` | `{{glance.name_lookups_handled.value}}` |
| 676 | `color:#ee0004` · `&#9650;&nbsp;16%` | `{{glance.name_lookups_handled.change.*}}` |
| 680 | `52` | `{{glance.devices_on_network.value}}` |
| 681 | `color:#ee0004` · `&#9650;&nbsp;13%` | `{{glance.devices_on_network.change.*}}` |
| 685 | `0` | `{{glance.appliance_restarts.value}}` |
| 686 | `color:#0563ff` · `&mdash;` | `{{glance.appliance_restarts.change.*}}` |
| 726 | `href="#"` | `href="{{report.url}}"` |
| 729 | `href="#"` | `href="{{report.url}}"` |
| 767 | `5,616` | `{{takeaways.action.count}}` |
| 797 | `3,512` · `443, 53, 80` | `{{takeaways.watch.count}}` · `{{takeaways.watch.ports}}` |
| 925 | `href="https://www.astralinkconnect.com/"` | `href="{{recipient.unsubscribe_url}}"` |

`.trend.*` and `.change.*` above are shorthand for the three fields on that one line —
`icon_url` / `color` / `pct` for a card, `color` / `glyph` / `pct` for a glance row.
Each glance change cell composes as `{{glyph}}&nbsp;{{pct}}`.

**Note on the two `href="#"` pairs per file.** Both the label and its arrow image are
separate `<a>` elements and both need the same URL. A placeholder on only one of them
leaves a dead `#` link that scrolls the message to the top instead of opening the
report.
