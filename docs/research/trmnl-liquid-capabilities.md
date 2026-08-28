# TRMNL Liquid capabilities for the MET Norway forecast

Research for ticket #5. Date: 2026-08-28.

## Short answer

**Yes. A Recipe with no server of our own can do the work.**

The Liquid template alone can group the timeseries entries (87 in my test fetch) into
local days, find the minimum and maximum temperature of each day, and choose one
`symbol_code` for each day. You do this with one `for` loop and `assign` statements.
It is not elegant, but it works. I tested the pattern with Liquid 5.12.0 and the live
MET Norway payload. The full loop took about 2 ms.

Two facts make the loop necessary:

- The array filters (`map`, `where`, `sort`, `sum`, `find`) read only **one
  top-level key** of each item. They cannot read `data.instant.details.air_temperature`.
- The TRMNL `group_by` filter groups by one top-level key only. The only top-level
  key is `time`, and every `time` value is different. So `group_by` gives one group
  per entry.

The TRMNL `where_exp` filter is different. It **does** read nested keys, and it
supports `and`, `or`, `contains`, `>=` and `<`. It is a good second option.

TRMNL also has a "Serverless" sandbox that runs JavaScript, Ruby, PHP or Python on
the payload before Liquid sees it. It needs no server of ours. But TRMNL says
downstream installations of a published Recipe may not have the runtime. Do not
build the public Recipe on it yet. See the "Open questions" section.

Payload size is safe: the limit is 100 KB and the `compact` response is 38 KB.

---

## 1. Which Liquid does TRMNL run?

TRMNL runs **stock Shopify Liquid** plus a small extension gem.

- The extension gem is open source: `usetrmnl/trmnl-liquid`.
  Its gemspec depends on `liquid ~> 5.12`.
  Source: <https://github.com/usetrmnl/trmnl-liquid/blob/main/trmnl-liquid.gemspec>
- The gem registers its filters and one tag on a Liquid `Environment`.
  Source: <https://github.com/usetrmnl/trmnl-liquid/blob/main/lib/trmnl/liquid.rb>
- The TRMNL docs also name Shopify Liquid.
  Source: <https://docs.trmnl.com/go/private-plugins/templates>

### Standard tags (Liquid 5.12)

`assign`, `break`, `capture`, `case`, `comment`, `continue`, `cycle`, `decrement`,
`doc`, `echo`, `for`, `if`, `ifchanged`, `include`, `increment`, `raw`, `render`,
`tablerow`, `unless`.
Source: <https://github.com/Shopify/liquid/tree/v5.12.0/lib/liquid/tags>

The inline form `{% liquid ... %}` also works. TRMNL uses it in its own guide.
Source: <https://help.trmnl.com/en/articles/10693981-advanced-liquid>

`{% render %}` runs in an **isolated** context. It cannot see the variables of the
parent template. You must pass each value as an argument.
Source: <https://github.com/Shopify/liquid/blob/v5.12.0/lib/liquid/tags/render.rb>
(line `inner_context = context.new_isolated_subcontext`)

### Standard filters (Liquid 5.12)

`abs`, `append`, `at_least`, `at_most`, `base64_*`, `capitalize`, `ceil`, `compact`,
`concat`, `date`, `default`, `divided_by`, `downcase`, `escape`, `escape_once`,
`find`, `find_index`, `first`, `floor`, `has`, `join`, `last`, `lstrip`, `map`,
`minus`, `modulo`, `newline_to_br`, `plus`, `prepend`, `reject`, `remove`,
`remove_first`, `remove_last`, `replace`, `replace_first`, `replace_last`,
`reverse`, `round`, `rstrip`, `size`, `slice`, `sort`, `sort_natural`, `split`,
`squish`, `strip`, `strip_html`, `strip_newlines`, `sum`, `times`, `truncate`,
`truncatewords`, `uniq`, `upcase`, `url_decode`, `url_encode`, `where`.
Source: <https://github.com/Shopify/liquid/blob/v5.12.0/lib/liquid/standardfilters.rb>
Reference: <https://shopify.github.io/liquid/>

### The TRMNL tag

`{% template <name> %} ... {% endtemplate %}` defines a partial. You then call it
with the standard `{% render "<name>", key: value %}`. Put the definition in the
"Shared" markup, because shared markup is added before every layout.
Source: <https://docs.trmnl.com/go/private-plugins/reusing-markup>

### The TRMNL filters

From the source file
<https://github.com/usetrmnl/trmnl-liquid/blob/main/lib/trmnl/liquid/filters.rb>:

`append_random`, `days_ago`, `group_by`, `find_by`, `markdown_to_html`,
`number_with_delimiter`, `number_to_currency`, `l_word`, `l_date`, `map_to_i`,
`pluralize`, `json`, `parse_json`, `random_number`, `sample`, `where_exp`,
`ordinalize`, `qr_code`.

The help centre documents all of these except `map_to_i`. `map_to_i` converts each
item of an array to an integer.
Source: <https://help.trmnl.com/en/articles/10347358-custom-plugin-filters>

---

## 2. Can the template group the timeseries into days?

Yes, with a loop. Here is the evidence for each part.

### 2.1 What does not work

**`map`, `where`, `sort`, `sum`, `find`, `has` read one key only.** The Liquid source
uses `item[property]`. A dotted string is used as a literal key name, so it finds
nothing.
Source: <https://github.com/Shopify/liquid/blob/v5.12.0/lib/liquid/standardfilters.rb>
(see `map`, `filter_array`, `sort`)

I confirmed this with the live payload:

```liquid
{{ properties.timeseries | map: "data.instant.details.air_temperature" | join: "," }}
```

Output: `,,,,,,,,,` (87 empty values). The same test with `sum:` gave `0`.
`sort: "time"` works, because `time` is a top-level key.

**`group_by` is not useful here.** Its whole body is
`collection.group_by { it[key] }` — one top-level key.
Source: <https://github.com/usetrmnl/trmnl-liquid/blob/main/lib/trmnl/liquid/filters.rb>
The only top-level key on each entry is `time`, and each value is unique.
`{{ properties.timeseries | group_by: "time" | size }}` gave `87` groups.

**There is no sort by a computed key.** Liquid has no lambda and no block.
Source: <https://shopify.github.io/liquid/filters/sort/>

**`increment` and `decrement` do not help.** They keep their own counters and print
the value. They cannot hold a minimum or a maximum.
Source: <https://shopify.github.io/liquid/tags/variable/>

### 2.2 What does work

**`assign` inside a `for` loop survives the loop.** The Liquid `assign` tag writes to
the outermost scope: `context.scopes.last[@to] = val`.
Source: <https://github.com/Shopify/liquid/blob/v5.12.0/lib/liquid/tags/assign.rb>
So you can keep a running minimum, a running maximum and a running day key.

**Dot paths work everywhere else.** `{{ e.data.instant.details.air_temperature }}`
and `{% if e.data.next_6_hours.summary.symbol_code %}` are normal variable lookups.

**`where_exp` understands dot paths.** Its expression is parsed as a Liquid condition
and evaluated with the item in the context.
Source: <https://github.com/usetrmnl/trmnl-liquid/blob/main/lib/trmnl/liquid/filters.rb>
The spec file shows a dotted example: `"town.label == 'Boulder' or town.id < 2"`.
Source: <https://github.com/usetrmnl/trmnl-liquid/blob/main/spec/trmnl/liquid/filters_spec.rb>
You **cannot** use filters inside the expression. Only operands, comparison
operators, `and` and `or`.

### 2.3 Worked pattern — one pass, per-day minimum, maximum and symbol

This is the pattern I recommend. It uses only stock Liquid. It reads the array once.
The day key comes from the UTC time plus the user offset, so the days are the user's
local days.

```liquid
{%- assign off = trmnl.user.utc_offset -%}
{%- assign cur = "" -%}
{%- assign tmin = 999 -%}
{%- assign tmax = -999 -%}
{%- assign sym = "" -%}
{%- for e in properties.timeseries -%}
  {%- assign local = e.time | date: "%s" | plus: off -%}
  {%- assign day = local | date: "%Y-%m-%d" -%}
  {%- assign hour = local | date: "%H" | plus: 0 -%}

  {%- comment -%} the day changed, so print the day that just ended {%- endcomment -%}
  {%- if day != cur -%}
    {%- unless cur == "" -%}
      <span>{{ cur }} {{ tmin }} / {{ tmax }} {{ sym }}</span>
    {%- endunless -%}
    {%- assign cur = day -%}
    {%- assign tmin = 999 -%}
    {%- assign tmax = -999 -%}
    {%- assign sym = "" -%}
  {%- endif -%}

  {%- assign t = e.data.instant.details.air_temperature -%}
  {%- if t and t < tmin %}{% assign tmin = t %}{% endif -%}
  {%- if t and t > tmax %}{% assign tmax = t %}{% endif -%}

  {%- comment -%} first symbol at or after 12:00 local time {%- endcomment -%}
  {%- if hour >= 12 and sym == "" and e.data.next_6_hours.summary.symbol_code -%}
    {%- assign sym = e.data.next_6_hours.summary.symbol_code -%}
  {%- endif -%}
{%- endfor -%}
<span>{{ cur }} {{ tmin }} / {{ tmax }} {{ sym }}</span>
```

Tested output, with `utc_offset = 10800` (Helsinki, summer) and the live payload of
2026-08-28. The test printed `min=` and `max=` in place of the `<span>` above:

```
2026-08-28 min=17.4 max=19.0 sym=cloudy
2026-08-29 min=16.7 max=20.1 sym=cloudy
2026-08-30 min=16.3 max=18.8 sym=partlycloudy_day
...
2026-09-07 min=12.7 max=16.4 sym=
```

Notes:

- Always test the temperature with `{% if t %}` first. A comparison with `nil` raises
  a Liquid error.
- The last day of the payload has few entries. It has no `symbol_code` after 12:00.
  Show only the first 4 to 6 days, or use a fallback.
- The 6-hourly part of the array does not change the pattern.
- To print the days in a grid, print each day inside the loop as above, or use the
  two-pass form below.

### 2.4 Alternative pattern — two passes with `where_exp`

Pass 1 builds the list of local day keys. Pass 2 selects the entries of one day with
a time window. ISO 8601 strings compare correctly with `<` and `>=`, because
alphabetical order is the same as time order.

```liquid
{%- assign off = trmnl.user.utc_offset -%}
{%- assign now = "now" | date: "%s" | plus: off -%}
{%- assign midnight = now | divided_by: 86400 | times: 86400 | minus: off -%}
{%- assign d1 = midnight | plus: 86400 -%}
{%- assign from_s = midnight | date: "%Y-%m-%dT%H:%M:%SZ" -%}
{%- assign to_s = d1 | date: "%Y-%m-%dT%H:%M:%SZ" -%}
{%- assign today = properties.timeseries
      | where_exp: "e", "e.time >= from_s and e.time < to_s" -%}
{{ today.size }} entries from {{ today.first.time }} to {{ today.last.time }}
```

Tested output: `window 2026-08-27T21:00:00Z .. 2026-08-28T21:00:00Z`, `entries=5`,
`first=2026-08-28T16:00:00Z`, `last=2026-08-28T20:00:00Z`. This is correct: the
payload started at 16:00Z.

`e.time contains "2026-08-29"` also works, but it matches the **UTC** day. Use it
only when the offset is zero.

You still need a small inner loop for the minimum and the maximum, because no filter
can read the nested temperature.

`find_by` picks one entry by a top-level key, for example the 12:00 UTC entry:

```liquid
{% assign noon = properties.timeseries | find_by: "time", "2026-08-29T12:00:00Z" %}
```

### 2.5 Cost

An empty double loop of 7 x 87 = 609 iterations took 1.6 ms. The full grouping loop
over 87 entries took 2.4 ms. Measured locally with Liquid 5.12.0 on Ruby 3.4.
Iteration count is not a problem for this data.

---

## 3. Dates and the user's time zone

TRMNL injects a `trmnl` variable into every plugin. The useful fields are:

| Variable | Meaning | Example |
|---|---|---|
| `trmnl.user.utc_offset` | offset in **seconds** | `-14400` |
| `trmnl.user.time_zone_iana` | IANA name | `America/New_York` |
| `trmnl.user.time_zone` | Rails name | `Eastern Time (US & Canada)` |
| `trmnl.user.locale` | locale | `en` |

Sources:
<https://help.trmnl.com/en/articles/10693981-advanced-liquid>,
<https://docs.trmnl.com/go/plugin-marketplace/plugin-screen-generation-flow>,
<https://help.trmnl.com/en/articles/9510536-private-plugins>

### The documented conversion

```liquid
{{ "2025-02-28T13:35:00Z" | date: "%s" | plus: trmnl.user.utc_offset | date: "%H:%M" }}
```

The first `date: "%s"` gives the Unix epoch. `plus` adds the offset. The second
`date` formats it. With an offset of -3 hours the result is `10:35`.
Source: <https://help.trmnl.com/en/articles/10693981-advanced-liquid>

This pattern works only if the render process runs in UTC. The TRMNL example implies
that it does. My test container also ran in UTC.

How the `date` filter behaves:

- A string goes through `Time.parse`. A `Z` suffix is read as UTC.
- An integer, or a string of digits, goes through `Time.at`.
- `"now"` and `"today"` give the current time.
- The format string is `strftime`.
  Source: <https://github.com/Shopify/liquid/blob/v5.12.0/lib/liquid/utils.rb>

### Local day names

`l_date` formats a date in the user's language:

```liquid
{{ '2025-01-11' | l_date: '%a %e.%m.', trmnl.user.locale }}
```

Source: <https://help.trmnl.com/en/articles/10347358-custom-plugin-filters>

Note: `l_date` parses the value with `Time.parse` and does **not** apply the user
offset. Convert to local time first (add the offset, format with `date:
"%Y-%m-%dT%H:%M:%SZ"`), then pass that string to `l_date`.
Source: <https://github.com/usetrmnl/trmnl-liquid/blob/main/lib/trmnl/liquid/filters.rb>

`l_word` translates common words, for example "today" and "tomorrow".

### Warning about date-only strings

TRMNL warns that a date-only string such as `2025-02-28` is read as `00:00:00` UTC.
A user behind UTC then sees the previous day. Always work from the full timestamp.
Source: <https://help.trmnl.com/en/articles/10693981-advanced-liquid>

---

## 4. Practical limits

| Limit | Value | Source |
|---|---|---|
| Polling response size | **100 KB**. A larger response is rejected and the plugin goes to a degraded state. | <https://help.trmnl.com/en/articles/12996946-parsing-plugins-with-the-sandbox-runtime> |
| Webhook payload size | 2 KB, 5 KB for TRMNL+ (not our strategy) | <https://docs.trmnl.com/go/private-plugins/webhooks> |
| Minimum refresh rate | 15 minutes, 5 minutes with TRMNL+ | <https://help.trmnl.com/en/articles/10113695-how-refresh-rates-work> |
| Serverless sandbox | 128 MB, 5 seconds | <https://help.trmnl.com/en/articles/14130649-serverless> |
| Older "transformer" sandbox | 1 second, no network | <https://help.trmnl.com/en/articles/12996946-parsing-plugins-with-the-sandbox-runtime> |
| Screen size | 800 x 480 for the standard device | <https://docs.trmnl.com/go/plugin-marketplace/plugin-screen-generation-flow> |
| Loop iteration cap | UNKNOWN — not confirmed | see below |
| Template size cap | UNKNOWN — not confirmed | see below |
| Render timeout | UNKNOWN — not confirmed | see below |

Our payload is 38 KB, so we are inside the 100 KB limit. But we have little space.
Do not add a second large polling URL.

I searched docs.trmnl.com (`llms-full.txt`), the help centre "Plugin Guides" and
"Advanced" collections, and the `usetrmnl` GitHub organisation for a render timeout,
a template size cap and a loop cap. I found none.

Liquid itself has optional resource limits (`render_length_limit`,
`render_score_limit`, `assign_score_limit`). They are off by default.
Source: <https://github.com/Shopify/liquid/blob/v5.12.0/lib/liquid/resource_limits.rb>
Whether trmnl.com sets them is UNKNOWN. The open source BYOS server, Terminus, does
not set them.
Source: <https://github.com/usetrmnl/terminus/blob/main/config/providers/liquid.rb>

What happens when something fails:

- A failed or invalid polling URL puts the plugin in a "degraded state". After many
  failures TRMNL stops refreshing it, and you must reset the plugin health.
  Source: <https://help.trmnl.com/en/articles/12384091-plugin-in-a-degraded-state-reset>
- A render failure appears in the server-side Debug Logs, for example
  `Plugins::Base.process! -> StandardError: private_plugin`.
  Source: <https://help.trmnl.com/en/articles/12814634-plugins-base-process-standarderror-private>
- If the merge variables did not change since the last request, TRMNL does not make a
  new screen. Use "Force Refresh" during development.
  Source: <https://help.trmnl.com/en/articles/9510536-private-plugins>

---

## 5. How the polled JSON reaches the template

With **one** polling URL, the response is merged at the **root**. There is no wrapper
variable. So our payload is read as:

```liquid
{{ properties.timeseries }}
{{ properties.meta.units.air_temperature }}
{{ geometry.coordinates[0] }}
```

With **more than one** URL, each response gets its own node: `IDX_0`, `IDX_1`, and so
on.
Source: <https://help.trmnl.com/en/articles/9510536-private-plugins>

Access rules:

- Nested keys use dot notation: `properties.timeseries`. TRMNL suggests flat root
  keys, but a full path works.
  Source: <https://help.trmnl.com/en/articles/9510536-private-plugins> ("Payload data
  isn't merging into my markup template")
- Arrays use an index (`data[0]`), the `first` and `last` filters, or a `for` loop.
- The `trmnl` namespace exists next to the payload.

Polling also gives us what MET Norway needs:

- **Custom headers.** You can set a `user-agent` header. MET Norway requires one.
  Format: `key=value`, joined with `&`.
  Source: <https://help.trmnl.com/en/articles/9510536-private-plugins>
- **Form fields** can be interpolated into the URL and the headers with
  `{{ keyname }}`. This gives the user a latitude and longitude field.
  Source: <https://help.trmnl.com/en/articles/9510536-private-plugins>

---

## 6. Seeing the parsed data while you develop

Yes. There are five ways.

1. **"Your Variables" dropdown** in the Markup Editor. It shows the parsed payload
   plus the `trmnl` node. Click "Force Refresh" on the plugin settings page to fetch
   fresh data.
   Source: <https://help.trmnl.com/en/articles/9510536-private-plugins>
2. **Live preview** in the Markup Editor, and the newer **Visual Editor (IDE)** with a
   preview pane, a code editor, and Shared / Serverless tabs.
   Source: <https://help.trmnl.com/en/articles/14726819-visual-editor-ide>
3. **Debug Logs**, enabled for 24 hours from the plugin settings page. They show API
   failures, image generation exceptions and Liquid filter syntax errors. A separate
   **JS Logs** dropdown shows client-side logs.
   Source: <https://help.trmnl.com/en/articles/11586187-debugging-private-plugins>
4. **`trmnlp`**, the first-party local dev server. It is the best option for this
   project. It keeps the plugin in git (`src/*.liquid`, `src/settings.yml`), watches
   the files, previews as HTML or PNG, lints, and pushes to TRMNL. It renders with the
   same `trmnl-liquid` gem, so the filters match. `.trmnlp.yml` can set the
   `time_zone` injected into `trmnl.user`, so we can test other time zones.
   Source: <https://github.com/usetrmnl/trmnlp>
5. **Plugin Data API** ("data mode") returns the parsed JSON of a plugin. It needs the
   Developer edition.
   Source: <https://docs.trmnl.com/go/private-api/plugin-data>

In the template you can also print any object as JSON with the TRMNL `json` filter:
`{{ properties.timeseries[0] | json }}`.
Source: <https://help.trmnl.com/en/articles/10347358-custom-plugin-filters>

---

## Open questions

1. **Is the Serverless sandbox safe for a public Recipe?** The help centre says: "Can
   I refactor my existing (published) plugins to use this feature? Please don't, at
   least not until April 2026. Because this feature is in private beta, downstream
   installations won't have access to the runtime." The date has passed, but the
   article does not say the beta ended. Ask TRMNL support or the Discord before we
   depend on it.
   Source: <https://help.trmnl.com/en/articles/14130649-serverless>
2. **Render timeout, template size cap, loop cap.** UNKNOWN — not confirmed. Not in
   the docs, the help centre or the public source.
3. **Does `trmnl.user.utc_offset` follow daylight saving time at render time?** The
   docs show a fixed number of seconds. Whether TRMNL recomputes it for the current
   date is UNKNOWN — not confirmed. This matters at the March and October changes.
   `trmnl.user.time_zone_iana` is safe, but Liquid cannot use an IANA name.
4. **Is the render process always in UTC?** The documented conversion pattern needs
   it. UNKNOWN — not confirmed in writing.
5. **Is `error_mode` strict on trmnl.com?** Terminus (the open source BYOS server)
   uses `:strict`. The hosted server is UNKNOWN — not confirmed. In strict mode a
   typo in a tag raises instead of printing nothing.
6. **Does the 100 KB limit apply before or after decompression?** UNKNOWN — not
   confirmed. Our 38 KB leaves room either way.

---

## Sources

Primary TRMNL documentation:

- <https://docs.trmnl.com/go/sitemap.md>
- <https://docs.trmnl.com/go/llms-full.txt>
- <https://docs.trmnl.com/go/private-plugins/templates>
- <https://docs.trmnl.com/go/private-plugins/templates-advanced>
- <https://docs.trmnl.com/go/private-plugins/reusing-markup>
- <https://docs.trmnl.com/go/private-plugins/webhooks>
- <https://docs.trmnl.com/go/private-api/plugin-data>
- <https://docs.trmnl.com/go/plugin-marketplace/plugin-screen-generation-flow>

TRMNL help centre:

- <https://help.trmnl.com/en/articles/9510536-private-plugins>
- <https://help.trmnl.com/en/articles/10671186-liquid-101>
- <https://help.trmnl.com/en/articles/10693981-advanced-liquid>
- <https://help.trmnl.com/en/articles/10347358-custom-plugin-filters>
- <https://help.trmnl.com/en/articles/10113695-how-refresh-rates-work>
- <https://help.trmnl.com/en/articles/10122094-plugin-recipes>
- <https://help.trmnl.com/en/articles/11586187-debugging-private-plugins>
- <https://help.trmnl.com/en/articles/11395668-recipe-best-practices>
- <https://help.trmnl.com/en/articles/12384091-plugin-in-a-degraded-state-reset>
- <https://help.trmnl.com/en/articles/12386583-plugin-not-receiving-data-from-polling-url>
- <https://help.trmnl.com/en/articles/12814634-plugins-base-process-standarderror-private>
- <https://help.trmnl.com/en/articles/12996946-parsing-plugins-with-the-sandbox-runtime>
- <https://help.trmnl.com/en/articles/14130649-serverless>
- <https://help.trmnl.com/en/articles/14726819-visual-editor-ide>

First-party source code:

- <https://github.com/usetrmnl/trmnl-liquid>
- <https://github.com/usetrmnl/trmnl-liquid/blob/main/lib/trmnl/liquid/filters.rb>
- <https://github.com/usetrmnl/trmnl-liquid/blob/main/spec/trmnl/liquid/filters_spec.rb>
- <https://github.com/usetrmnl/trmnl-liquid/blob/main/lib/trmnl/liquid.rb>
- <https://github.com/usetrmnl/trmnl-liquid/blob/main/trmnl-liquid.gemspec>
- <https://github.com/usetrmnl/trmnlp>
- <https://github.com/usetrmnl/terminus/blob/main/config/providers/liquid.rb>

Stock Liquid:

- <https://shopify.github.io/liquid/>
- <https://github.com/Shopify/liquid/blob/v5.12.0/lib/liquid/standardfilters.rb>
- <https://github.com/Shopify/liquid/blob/v5.12.0/lib/liquid/tags/assign.rb>
- <https://github.com/Shopify/liquid/blob/v5.12.0/lib/liquid/tags/render.rb>
- <https://github.com/Shopify/liquid/blob/v5.12.0/lib/liquid/utils.rb>
- <https://github.com/Shopify/liquid/blob/v5.12.0/lib/liquid/resource_limits.rb>
- <https://github.com/Shopify/liquid/tree/v5.12.0/lib/liquid/tags>

Test method: I rendered the templates above with the `liquid` gem 5.12.0 on Ruby 3.4
in a Docker container. I copied the `group_by`, `find_by`, `map_to_i` and `where_exp`
filters from the `trmnl-liquid` source. The data was a live response of
`https://api.met.no/weatherapi/locationforecast/2.0/compact?lat=60.1699&lon=24.9384&altitude=10`
from 2026-08-28 (87 entries, 37.9 KB). This test is my own. It is not a TRMNL source.
