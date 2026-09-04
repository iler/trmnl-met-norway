# TRMNL recipe polling: limits, intervals and header syntax

Research for ticket #2. All facts come from TRMNL primary sources. Each claim has a
source URL. Where a source does not exist, the answer is "UNKNOWN — not confirmed".

A Recipe is a Private Plugin that the TRMNL team approved for public listing. It has
the same settings and the same limits as a Private Plugin.
[Source](https://help.trmnl.com/en/articles/10546870-compare-custom-plugin-types)

## Short answer

- Set `refresh_interval: 60`. The permitted values are 15, 60, 360, 720 and 1440
  minutes. 15 minutes is the shortest. 15 is faster than the 30-minute `Expires`
  header from MET. So 60 is the first safe value.
- The header syntax `key=value`, joined with `&`, is correct. A `user-agent=...`
  pair overrides the default agent that TRMNL would send. Encode `=` as `%3D` and
  spaces as `%20` in the value.
- The polling response must be 100 kB or smaller. Our MET response is 38 kB. This is
  safe. The 2 kB limit applies to webhooks only, not to polling.
- Interpolation uses plain Liquid: `{{ keyname }}`. The `##` prefix is an artifact of
  the help centre pages. It is not real syntax. Interpolation works in the polling
  URL, the polling headers and the polling body. TRMNL does not escape values
  automatically. You must add the `url_encode` filter yourself.
- One recipe can poll many URLs. Each response goes into an `IDX_0`, `IDX_1` node.
- Whether TRMNL sends `Accept-Encoding: gzip`, sends `If-Modified-Since`, or obeys
  `Expires` for the polling strategy is UNKNOWN — not confirmed.

Effect on the plan: use `refresh_interval: 60`, add an explicit `user-agent` polling
header, and treat the 38 kB uncompressed response as the size we must live with.

---

## 1. Refresh intervals

The plugin ZIP format documents the field and its permitted values:

```yaml
refresh_interval: 60      # minutes; 15 | 60 | 360 | 720 | 1440
```

[Source](https://help.trmnl.com/en/articles/10542599-importing-and-exporting-private-plugins)

So 15 minutes is the shortest interval. It is faster than MET's 30-minute `Expires`
header. Therefore we must use 60 minutes.

Since 22 May 2026, TRMNL uses "on-demand" refresh. TRMNL now refreshes a plugin just
before the device shows it. The `refresh_interval` value is now a **minimum**:

> Now that setting has been repurposed to have a new meaning: it's the minimum plugin
> refresh rate. The plugin will refresh as often as needed to display fresh data your
> device, but never faster than your preference.

[Source](https://help.trmnl.com/en/articles/15123293-on-demand-plugin-refresh)

The same page warns plugin authors that this change may alter how often their API gets
requests. This is good for us: with `refresh_interval: 60`, one install polls MET one
time per hour at most. It may poll less often if the device is asleep or if the
playlist is long.

Other refresh settings sit above the plugin setting and can only make refresh slower,
not faster: the device refresh rate (minimum 5 minutes), the playlist item duration,
the mashup minimum, and the account refresh rate (15 minutes by default, 5 minutes
with TRMNL+). TRMNL also skips screen generation when the merged data did not change.
[Source](https://help.trmnl.com/en/articles/10113695-how-refresh-rates-work)

Device-level rate limits (429) apply to the device-to-TRMNL connection, not to
TRMNL-to-MET polling. [Source](https://help.trmnl.com/en/articles/11652861-429-rate-limit)

## 2. Polling header syntax

The unverified reading is **correct**. The help centre says:

> Assign header key/values with `=` and separate them with `&`. So
> `authorization=bearer xxx&content-type=application/json` becomes
> `{"authorization":"bearer xxx", "content-type":"application/json"}`

And, directly on the point that decides 200 or 403:

> You can even use this field to override default headers such as the `user-agent`
> header by including something like `user-agent=your_agent`.

[Source](https://help.trmnl.com/en/articles/9510536-private-plugins)

The same page gives the escaping rule:

> If a key/value requires an `=` sign in the value, for example `bearer jwt==`, simply
> encode it as `%3D`, so `bearer%20jwt%3D%3D`.

Note that `%20` in that example becomes a space. So the value is percent-decoded.

Two first-party code sources confirm the format is a URI query string.

TRMNL's own local dev server splits on `&`, then on the first `=`, then percent-decodes
the value:

```ruby
def string_to_hash(str, delimiter: '=')
  str.split('&').map do |k_v|
    key, value = k_v.split(delimiter)
    next if value.nil?
    { key => CGI.unescape_uri_component(value) }
  end.compact.reduce({}, :merge)
end
```

[Source](https://github.com/usetrmnl/trmnlp/blob/main/lib/trmnlp/config/plugin.rb)

TRMNL's own BYOS server imports a plugin from trmnl.com and parses the stored
`polling_headers` value with `Rack::Utils.parse_query`, a URI query parser.
[Schema source](https://github.com/usetrmnl/terminus/blob/main/app/aspects/extensions/importers/remote/schema.rb) ·
[Coercer source](https://github.com/usetrmnl/terminus/blob/main/app/schemas/coercers/uri_query_to_hash.rb)

Practical rules for our `user-agent` value:

- Do not use a raw `=`. Use `%3D`.
- Do not use a raw space. Use `%20`.
- Do not use `+` for a space. Some query parsers turn `+` into a space and some do
  not. Use `%20` and avoid `+` completely.
- Do not use `&` or `;` in the value.

An example that is safe with both parsers:

```
user-agent=trmnl-met-norway%2F1.0%20https%3A%2F%2Fgithub.com%2Filer%2Ftrmnl-met-norway
```

Confirmed: a `user-agent` pair replaces the agent TRMNL would otherwise send. TRMNL
calls it an override of a "default header".
[Source](https://help.trmnl.com/en/articles/9510536-private-plugins)

UNKNOWN — not confirmed: the exact default user agent string that TRMNL sends when no
`user-agent` header is given. TRMNL does not publish it. The `http.rb/6.0.0 (TRMNL
API)` string found in `usetrmnl/trmnl-api` belongs to a client library that calls the
TRMNL API. It is not the polling fetcher. Do not rely on it. Because MET returns 403
for a browser-like agent, always set the header.

Corrected on 2026-09-04 by measurement against
`locationforecast/2.0/complete`. MET does not reject a *missing* agent, it
rejects one that looks like a browser:

| Request | Response |
|---|---|
| Default `curl/8.x` agent | 200 |
| `Mozilla/5.0` | **403** |
| No `User-Agent` header at all | 200 |
| `trmnl-met-norway/1.0 github.com/iler/trmnl-met-norway` | 200 |

This does not change the decision. The MET terms require an identifying agent
with contact information whether or not MET enforces it, and enforcement can
tighten without notice. It does change the diagnosis: a plugin that loses the
header will not necessarily fail, so a missing header cannot be ruled in or out
from a 200 alone.

TRMNL polling comes from fixed server IPs on Hetzner Cloud, published at
`https://trmnl.com/api/ips`.
[Source](https://help.trmnl.com/en/articles/12386583-plugin-not-receiving-data-from-polling-url)

## 3. Maximum polling response size

Yes. Polling has its own limit, and it is not the webhook limit.

> Create a private plugin that sometimes yields a large JSON object. As is, this plugin
> will soon go into a degraded state when the response is too large (> 100 kb).

[Source](https://help.trmnl.com/en/articles/12996946-parsing-plugins-with-the-sandbox-runtime)

The 2 kB limit is the **webhook** limit, not the polling limit:

> If you choose Webhook, you'll need to set up your own web service that sends a POST
> request to TRMNL with a payload of content (max 2kb size).

[Source](https://help.trmnl.com/en/articles/9510536-private-plugins)

Our MET `/compact` response is 38 kB uncompressed. It is inside the 100 kB limit.

If we ever need to shrink the payload, TRMNL has a "Serverless" step. It runs
JavaScript (Node v22) on the polled payload and returns a smaller object before Liquid
runs.
[Source](https://help.trmnl.com/en/articles/12996946-parsing-plugins-with-the-sandbox-runtime)

The Liquid template file itself has a 1 MB limit.
[Source](https://help.trmnl.com/en/articles/10542599-importing-and-exporting-private-plugins)

## 4. `Accept-Encoding`, conditional requests and `Expires`

**UNKNOWN — not confirmed** for all three, for the polling strategy.

What was searched: the whole docs site as one file
(`https://docs.trmnl.com/go/llms-full.txt`, 62 kB, no match for "poll",
"Accept-Encoding", "If-Modified-Since" or "gzip"), the docs sitemap, and the help
centre articles on private plugins, dynamic polling URLs, refresh rates, on-demand
refresh, polling troubleshooting and the sandbox runtime. None of them states what
request headers TRMNL adds, and none of them says TRMNL reads `Expires`,
`Cache-Control` or `ETag` on a polling response.

There is one nearby statement, but it is about the **Image Display** plugin, not about
the polling strategy:

> But with images, we use the headers `etag` with `If-None-Match` and `last-modified`
> with `If-Modified-Since` to determine if we need to process that image. Adjust your
> image endpoint to support these headers.

[Source](https://help.trmnl.com/en/articles/11479051-image-display)

Do not read that as polling behaviour. It is written only for image endpoints.

Indirect, first-party but non-production evidence. Two TRMNL open-source fetchers make
a plain request. Neither sends `If-Modified-Since`, and neither reads `Expires`:

- `trmnlp`, the official local dev server, builds a Faraday connection with only the
  user's own polling headers.
  [Source](https://github.com/usetrmnl/trmnlp/blob/main/lib/trmnlp/poller.rb)
- `terminus`, the flagship BYOS application, calls `http.headers(input.headers)` and
  then the verb, with no cache headers.
  [Source](https://github.com/usetrmnl/terminus/blob/main/app/aspects/extensions/fetcher/client.rb)

Neither of these is the trmnl.com production fetcher. Treat them as a hint only.

Effect on the plan. We cannot depend on TRMNL to obey `Expires` or to send
`If-Modified-Since`. We must obey the MET terms with the one control we do have: the
`refresh_interval` setting. 60 minutes is more than the 30-minute `Expires`, so we stay
compliant even if TRMNL sends an unconditional request every time.

We also cannot depend on gzip. Plan for 38 kB per request per install.

## 5. How a polling URL interpolates form field values

The correct syntax is plain Liquid output: `{{ keyname }}`.

The `##` prefix is an artifact of the help centre pages. It is not syntax. Proof:

1. The help centre itself writes the same Liquid without `##` on other pages. The form
   builder page gives this polling URL:

   ```
   https://yoururl.com/?lat={{ lat_lon | split: ',' | first }}&lon={{ lat_lon | split: ',' | last }}
   ```

   [Source](https://help.trmnl.com/en/articles/10513740-custom-plugin-form-builder)

   And the multiple-URL page writes `{{ IDX_0 }}`, `{{ IDX_1 }}`.
   [Source](https://help.trmnl.com/en/articles/12385769-missing-data-in-multiple-polling-urls)

2. On the pages that do show `##`, only the `{{ ... }}` output tags carry it. The
   `{% ... %}` logic tags never do. This is what a broken escape of `{{` looks like.
   [Source](https://help.trmnl.com/en/articles/12689499-dynamic-polling-urls)

3. TRMNL's own test suite uses plain `{{ }}` in both the URL and the headers:

   ```ruby
   plugin.instance_variable_set(:@config, { 'polling_url' => 'https://api.test/?t={{ oauth_access_token }}' })
   plugin.instance_variable_set(:@config, { 'polling_headers' => 'Authorization=Bearer {{ oauth_access_token }}' })
   ```

   [Source](https://github.com/usetrmnl/trmnlp/blob/main/spec/lib/trmnlp/config/plugin_spec.rb)

4. TRMNL's own agent prompt writes `{{ variable }}` with no prefix.
   [Source](https://github.com/usetrmnl/trmnl-agent-skills/blob/main/skills/trmnl/references/agent_prompt.md)

Where it works. In the polling URL, the polling headers and the polling body. The help
centre states this for all three fields.
[Source](https://help.trmnl.com/en/articles/9510536-private-plugins)

The full Liquid language is available, not only variable output:

> In addition to user-provided values, you have access to the entire Liquid templating
> library.

[Source](https://help.trmnl.com/en/articles/12689499-dynamic-polling-urls)

TRMNL renders the whole field through Liquid first, then splits the URL field on line
breaks and splits the headers field on `&` and `=`.
[URL split source](https://help.trmnl.com/en/articles/12689499-dynamic-polling-urls) ·
[Header split source](https://github.com/usetrmnl/trmnlp/blob/main/lib/trmnlp/config/plugin.rb)

**Escaping.** TRMNL does not escape a form field value automatically. The author must
add the filter:

> You can also use `apikey={{ api_key | url_encode }}` to programmatically encode the
> key as required.

[Source](https://help.trmnl.com/en/articles/9510536-private-plugins)

This matters for us. A user types the latitude and longitude. Untrusted text goes into
the URL. Always apply `url_encode` to any user value that goes into the URL.

There is a `lat_lon` form field type. It gives the user an autocomplete search for a
city, address or postal code, and stores a `lat,lon` string. The user may also type
coordinates directly, for example `33.7490,-84.3880`.

```yaml
- keyname: lat_lon
  field_type: lat_lon
  name: Location
  description: Search for a city or postal code, or provide a comma-separated lat/long.
```

[Source](https://help.trmnl.com/en/articles/10513740-custom-plugin-form-builder)

Remember that MET returns 403 for coordinates with 5 or more decimals. The `lat_lon`
field can produce more decimals than that. Round the values in the polling URL, for
example with the Liquid `round: 4` filter, or use two `number` fields with `step` set
to `0.0001`. The `number` field type supports `min`, `max` and `step`.
[Source](https://help.trmnl.com/en/articles/10513740-custom-plugin-form-builder)

## 6. Multiple polling URLs

Yes. One recipe can poll more than one URL.

> Provide 1 or more URLs, line-break separated, in the text box.

> Endpoints will be pinged sequentially, with results going inside index-based JSON
> nodes like so:
> `{ "IDX_0": {...}, "IDX_1": {...}, "IDX_2": {...} }`

[Source](https://help.trmnl.com/en/articles/12689499-dynamic-polling-urls) ·
[Source](https://help.trmnl.com/en/articles/9510536-private-plugins)

With one URL only, the data is available at the root, with no `IDX_` prefix.
[Source](https://help.trmnl.com/en/articles/9510536-private-plugins)

Liquid can build the URL list at run time, for example with a `{% for %}` loop over a
`multi_string` form field.
[Source](https://help.trmnl.com/en/articles/12689499-dynamic-polling-urls)

Constraints to know:

- All URLs share one HTTP verb, one header set and one body. These are single fields on
  the plugin, not per URL.
  [Source](https://help.trmnl.com/en/articles/12689499-dynamic-polling-urls)
- Each URL may return a different content type. JSON, RSS, XML, plaintext and CSV are
  accepted. [Source](https://help.trmnl.com/en/articles/9510536-private-plugins)
- The 100 kB limit and the `refresh_interval` are per plugin. Two URLs means two
  requests to MET per refresh, not one.

## Open questions

1. Does the trmnl.com production fetcher send `Accept-Encoding: gzip`? UNKNOWN — not
   confirmed. Not in the docs, the help centre or the blog.
2. Does it send `If-Modified-Since` or `If-None-Match` for the polling strategy?
   UNKNOWN — not confirmed. The documented conditional-request behaviour is for the
   Image Display plugin only.
3. Does it obey the `Expires` or `Cache-Control` response header? UNKNOWN — not
   confirmed. No source found. Assume it does not.
4. What is the default user agent for polling? UNKNOWN — not confirmed. We must set it,
   so this does not block us.
5. Can a user who installs our Recipe set the refresh interval faster than the value we
   ship? UNKNOWN — not confirmed. The refresh-rates page says the plugin settings page
   "usually" offers a Minimum Plugin Refresh Rate control, but it does not say whether
   an installed Recipe shows that control. If it does, a user could pick 15 minutes and
   break the MET terms. This needs a live check on a real install.
6. Is there a maximum number of polling URLs, or a per-request timeout? UNKNOWN — not
   confirmed. The docs say "as many URLs as you'd like" and give no timeout.
7. Does the 100 kB limit apply to each URL or to the combined `IDX_*` payload? UNKNOWN
   — not confirmed.

The way to close 1, 2 and 3 is a live test: point a test private plugin at a small
endpoint we control that logs the request headers, then read the log.

## Sources

TRMNL help centre:

- Private Plugins — https://help.trmnl.com/en/articles/9510536-private-plugins
- Importing and exporting private plugins —
  https://help.trmnl.com/en/articles/10542599-importing-and-exporting-private-plugins
- How refresh rates work — https://help.trmnl.com/en/articles/10113695-how-refresh-rates-work
- On-demand plugin refresh — https://help.trmnl.com/en/articles/15123293-on-demand-plugin-refresh
- Dynamic Polling URLs — https://help.trmnl.com/en/articles/12689499-dynamic-polling-urls
- Parsing plugins with the Sandbox Runtime —
  https://help.trmnl.com/en/articles/12996946-parsing-plugins-with-the-sandbox-runtime
- Plugin Not Receiving Data from Polling URL —
  https://help.trmnl.com/en/articles/12386583-plugin-not-receiving-data-from-polling-url
- Missing Data in Multiple Polling URLs —
  https://help.trmnl.com/en/articles/12385769-missing-data-in-multiple-polling-urls
- Custom plugin form builder — https://help.trmnl.com/en/articles/10513740-custom-plugin-form-builder
- Image Display — https://help.trmnl.com/en/articles/11479051-image-display
- Compare custom plugin types — https://help.trmnl.com/en/articles/10546870-compare-custom-plugin-types
- Plugin recipes — https://help.trmnl.com/en/articles/10122094-plugin-recipes
- 429 Rate Limit — https://help.trmnl.com/en/articles/11652861-429-rate-limit

TRMNL developer docs:

- Sitemap — https://docs.trmnl.com/go/sitemap.md
- Whole docs as one file — https://docs.trmnl.com/go/llms-full.txt
- Webhooks — https://docs.trmnl.com/go/private-plugins/webhooks.md

TRMNL open source (`usetrmnl` GitHub org):

- trmnlp plugin config (header parsing) —
  https://github.com/usetrmnl/trmnlp/blob/main/lib/trmnlp/config/plugin.rb
- trmnlp poller (request building) —
  https://github.com/usetrmnl/trmnlp/blob/main/lib/trmnlp/poller.rb
- trmnlp plugin config spec (exact Liquid syntax) —
  https://github.com/usetrmnl/trmnlp/blob/main/spec/lib/trmnlp/config/plugin_spec.rb
- trmnlp new-plugin template settings.yml —
  https://github.com/usetrmnl/trmnlp/blob/main/templates/init/src/settings.yml
- trmnlp Hacker News example settings.yml —
  https://github.com/usetrmnl/trmnlp/blob/main/examples/hn-stories/src/settings.yml
- terminus remote import schema —
  https://github.com/usetrmnl/terminus/blob/main/app/aspects/extensions/importers/remote/schema.rb
- terminus URI query coercer —
  https://github.com/usetrmnl/terminus/blob/main/app/schemas/coercers/uri_query_to_hash.rb
- terminus fetcher client —
  https://github.com/usetrmnl/terminus/blob/main/app/aspects/extensions/fetcher/client.rb
- terminus request builder —
  https://github.com/usetrmnl/terminus/blob/main/app/aspects/extensions/exchanges/request_builder.rb
- TRMNL agent skills prompt —
  https://github.com/usetrmnl/trmnl-agent-skills/blob/main/skills/trmnl/references/agent_prompt.md

Other:

- TRMNL polling server IP list — https://trmnl.com/api/ips
