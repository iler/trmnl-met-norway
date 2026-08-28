# TRMNL: requirements to publish a public Recipe

Research for ticket #4. All facts come from TRMNL primary sources (docs.trmnl.com,
help.trmnl.com, trmnl.com blog) and from MET Norway's own Terms of Service.
Sources are listed at the end. Date of research: 2026-08-28.

## Short answer

The path is simple. Build a Private Plugin, then click **Publish as a Recipe** on
the plugin settings page. A linter called **Chef** runs first. Then a human on the
TRMNL team reviews the plugin. Review "usually takes just a day or two"
([Private Plugins](https://help.trmnl.com/en/articles/9510536-private-plugins)).

There is no email application and no video for a Recipe. The e-mail flow on the
"Going Live" docs page belongs to Third Party (OAuth) plugins, not Recipes
([Going Live](https://docs.trmnl.com/go/plugin-marketplace/going-live.md)).

Nothing blocks our plan:

- No licence condition and no exclusivity condition was found for publication or
  for the Creator Fund. MIT stays possible.
- TRMNL has no branding rule that forces us to show a data-source logo. The one
  branding tip is a suggestion, not a requirement. So MET Norway's ban on "Yr",
  "NRK" and "Norwegian Meteorological Institute" branding does not conflict.
- We must supply an icon, a featured image, an `author_bio` field with contact
  details and a category, content in every layout, and demo data that contains no
  personal information.

Two facts change how we work:

1. **Our published plugin instance is live.** Changes to the Recipe Master go to
   all installed users automatically. So we must test on a copy, not on the master.
2. **We must never put our own home location into the Recipe Master.** The
   generated screen becomes the public preview image.

---

## 1. The path from a private plugin to a published Recipe

Steps, in order:

1. **Get the Developer add-on.** "TRMNL customers with the Developer add-on or a
   BYOD license may use our API to quickly and easily create a plugin."
   ([Private Plugins](https://help.trmnl.com/en/articles/9510536-private-plugins))
2. **Build a Private Plugin.** Plugins tab > "Private Plugin". Choose a strategy
   (Polling, Webhook or Static). ([Private Plugins](https://help.trmnl.com/en/articles/9510536-private-plugins))
3. **Click "Publish as a Recipe"** on the right side of the plugin settings page.
   ([Private Plugins](https://help.trmnl.com/en/articles/9510536-private-plugins))
   The Recipes article calls the same control the icon beside "Publish plugin?".
   ([Plugin recipes](https://help.trmnl.com/en/articles/10122094-plugin-recipes))
4. **Chef runs.** "Our linter, named Chef, will run a few helpful checks to make
   sure your work is in good shape."
   ([Private Plugins](https://help.trmnl.com/en/articles/9510536-private-plugins))
5. **A human reviews.** "Then our team will be notified to review + publish your
   plugin, which usually takes just a day or two."
   ([Private Plugins](https://help.trmnl.com/en/articles/9510536-private-plugins))
   The blog names the two steps: "Automated feedback from CHEF" and "Manual
   feedback from a human".
   ([Publishing checklist blog](https://trmnl.com/blog/plugin-recipe-publishing-tips))
6. **The plugin is listed** at <https://trmnl.com/recipes>.
   ([Plugin recipes](https://help.trmnl.com/en/articles/10122094-plugin-recipes))

**Public or Unlisted.** "Publicly published Recipes can be found by anyone from
/recipes or /plugins." If you do not want to satisfy Chef or the Moderation
Guidelines, you may publish as **Unlisted**. "Unlisted plugins skip the automated +
manual moderation steps and generate a shareable link immediately."
([Plugin recipes](https://help.trmnl.com/en/articles/10122094-plugin-recipes))

**Recipe versus Third Party plugin.** A Recipe "lives inside TRMNL" and needs no
server of our own. This is TRMNL's recommended path.
([Marketplace introduction](https://docs.trmnl.com/go/plugin-marketplace/introduction.md))
"Recipes are simply Private Plugins that have been approved by the TRMNL team to be
listed publicly."
([Compare plugin types](https://help.trmnl.com/en/articles/10546870-compare-custom-plugin-types))
Our project is a Recipe.

**Warning about the "Going Live" docs page.** That page tells you to e-mail
`team@trmnl.com` with a Plugin ID, a video, and test credentials. It sits in the
Third Party / OAuth part of the docs. It does not describe the Recipe flow.
([Going Live](https://docs.trmnl.com/go/plugin-marketplace/going-live.md))

---

## 2. What Chef checks, and what makes it fail

Chef is "a Recipe linting utility".
([Recipe best practices](https://help.trmnl.com/en/articles/11395668-recipe-best-practices))
TRMNL publishes the Chef source code in a public gist linked from that article.
([Chef source gist](https://gist.github.com/ryanckulp/fbe5f68c51db1ae214a97da24be4d62b))

### Blocking checks (a suggestion is raised if the check fails)

| Check | What it does |
|---|---|
| `async_functions_are_not_present` | Markup must not contain the text `async function`. Screenshot generation times out. |
| `author_bio_is_present` | A custom field with `field_type: author_bio` is required. |
| `category_is_present` | The `author_bio` field must carry a `category`. |
| `custom_fields_values_are_used` | Every custom field keyname must appear in the polling URL / headers / body, in the markup, or in the transform JS. Unused fields fail. |
| `featured_image_is_present` | A featured image must be attached. Generate it in the settings view. |
| `highcharts_animations_are_disabled` | If markup contains "highcharts", it must match `animation: false`. |
| `highcharts_elements_are_unique` | If markup contains "highcharts", it must use the `append_random` filter. |
| `icon_is_present` | An icon must exist. |
| `image_links_respond_ok` | Every static `<img src>` must be an absolute `http(s)` URL that answers a GET with a success code. Dynamic (`{{ }}`) and `data:` URIs are skipped. |
| `inline_styles_are_not_present` | Counts the strings `justify-content`, `padding`, `margin`, `background-color`, `border-radius`, `text-align`, `object-fit`, `font-size` in all markup. **More than 6 total fails** (`MAX_INLINE_STYLES = 6`). |
| `markup_size_elements_are_excluded` | Do not add `view--full`, `view--half_horizontal`, `view--half_vertical` or `view--quadrant` classes. TRMNL adds them. |
| `markups_have_content` | Every non-shared layout must hold at least 10 characters (or the Shared view must). Empty layouts fail. |
| `not_a_fork` | The plugin must not be a Fork of another Recipe. Export and re-import a zip to make a fresh submission. |
| `opacity_is_not_present` | Markup must not match `opacity: <number>`. Use `--gray--##` Framework classes. |
| `title_casing` | The title must start with a capital letter. |
| `title_length` | The title must be 50 characters or less (`MAX_TITLE_LENGTH = 50`). |
| `waits_for_dom_load` | JavaScript must not use `window.onload` or `window.addEventListener("load")`. Use `DOMContentLoaded`. |
| `webhook_strategy_has_copyable_url` | Webhook plugins need a `copyable_webhook_url` field. Not relevant to us (we poll). |

### Advisory warnings (they never block)

The code comment says: "Advisory warnings must never block submission".

- `responsive_classes_are_present`: no `lg:` or `portrait:` classes found. Check
  your layouts on TRMNL X in landscape and portrait.
- `form_field_links_use_html`: a `description` or `help_text` holds a plain-text
  URL. Use an HTML `<a href>` link. Tip from the help centre: remove `https://`
  from the front of a string if the link is only an example.
  ([Recipe best practices](https://help.trmnl.com/en/articles/11395668-recipe-best-practices))
- Liquid markup validation warnings, per layout.

### What the human reviewer adds

From the TRMNL blog checklist
([Publishing checklist blog](https://trmnl.com/blog/plugin-recipe-publishing-tips)):

- **Collaboration over competition.** A new Recipe "should not be seen by a user as
  indistinguishable from an existing recipe". A similar plugin "should add easily
  identifiable value". "Similar topics, but different data sources, are great
  examples of where competition is valuable and completely acceptable."
  (A MET Norway source is a different data source. This helps us.)
- **Depth, not numbers.** One Recipe with form fields beats several small Recipes.
- **Family friendly by default.**
- **Form fields.** `author_bio` must give at least one way to contact the author.
  Correct use of `default` and `placeholder`. Do not write `optional: false`; only
  add `optional` when it is `true`. Plain-text links become `<a href>` links, with
  `<br/>` where a line break was intended. Test each field works.
  "Validate no personal information will leak and that, if necessary, demo data is
  used."
- **Polling.** The reviewer parses the URL. If the endpoint needs authorisation, the
  form fields must carry links and instructions.
- **Markup.** Framework and Liquid first. CSS and JavaScript only when those reach
  their limit.
- **Red flags.** async API calls (5 second renderer limit); charts without a unique
  class identifier; JS waiting for page load; inline `opacity`; custom fonts;
  `label label--small label--gray` without a `1bit:text--black` override.
- **Layouts.** "We preview every view (e.g. Full, Quadrant) across both TRMNL OG and
  TRMNL X in landscape mode, as well as TRMNL X in portrait mode, to verify the
  screen is mostly content, not whitespace." They also check for cut-off data.
- **Liquid templates.** Pass native variables in:
  `{% render "my_template", trmnl: {{ trmnl }} %}`.
- **Nice to have.** If you use a `title_bar`, do not use the default TRMNL logo from
  the example docs. Use "an image appropriate for the data source". Inline static
  images instead of calling an external URL. Keep markup DRY with the Shared view.

**Moderation Guidelines.** Recipes with NSFW or suggestive imagery, egregious
profanity, or depictions of physical pain or suffering "will most likely be denied
from publication".
([Plugin recipes](https://help.trmnl.com/en/articles/10122094-plugin-recipes))
A weather Recipe is not affected.

---

## 3. What we must supply

| Item | Required? | Source |
|---|---|---|
| **Icon** | Yes. Chef check `icon_is_present`. Add it in the settings view. | [Chef source](https://gist.github.com/ryanckulp/fbe5f68c51db1ae214a97da24be4d62b) |
| **Featured image** | Yes. Chef check `featured_image_is_present`. "Featured image should exist for the marketplace preview." It is set from the current generated screen: "edit your recipe settings, and there will be a button to set the preview image as the current screens image." | [Chef source](https://gist.github.com/ryanckulp/fbe5f68c51db1ae214a97da24be4d62b), [Demo Data](https://help.trmnl.com/en/articles/12772238-demo-data-for-publishing-plugins) |
| **Separate screenshots** | Not found. The Recipes API returns one `screenshot_url` per Recipe, generated by TRMNL. UNKNOWN — no upload of extra screenshots is documented. | [Recipes API](https://docs.trmnl.com/go/public-api/recipes-api.md) |
| **Title / name** | Yes. Capital first letter, 50 characters or less. | [Chef source](https://gist.github.com/ryanckulp/fbe5f68c51db1ae214a97da24be4d62b) |
| **Description** | The `author_bio` field "operates like a README. It appears below your plugin's preview image on logged out + logged in pages. Here you can provide a description, verbose installation instructions, and more." | [Form builder](https://help.trmnl.com/en/articles/10513740-custom-plugin-form-builder) |
| **Author details** | Yes, through `author_bio`. Optional special properties: `email_address`, `category` (up to 2, comma separated), `github_url`, `learn_more_url`, `youtube_url`. The reviewer needs "at least one way that a user could get in-touch with the author". | [Form builder](https://help.trmnl.com/en/articles/10513740-custom-plugin-form-builder), [Blog](https://trmnl.com/blog/plugin-recipe-publishing-tips) |
| **Category** | Yes. Chef fails without one. Up to 2. | [Chef source](https://gist.github.com/ryanckulp/fbe5f68c51db1ae214a97da24be4d62b) |
| **All layouts** | Yes. Chef check `markups_have_content`. Also: "in order for your plugin to be published in the TRMNL public marketplace, you must provide HTML for all available markup layouts." | [Chef source](https://gist.github.com/ryanckulp/fbe5f68c51db1ae214a97da24be4d62b), [llms-full.txt](https://docs.trmnl.com/go/llms-full.txt) |
| **Demo data** | Yes, in practice. The reviewer validates that no personal information leaks. | [Blog](https://trmnl.com/blog/plugin-recipe-publishing-tips) |

### Example `author_bio` (from TRMNL's own docs)

```yaml
- keyname: doesnt_matter
  name: About This Plugin
  category: life,news
  field_type: author_bio
  description: Dad Jokes Daily™ was created by Abraham, father to many.
  github_url: https://github.com/father-abraham
  learn_more_url: https://www.bible.com/
  email_address: abraham@example.com
  youtube_url: https://www.youtube.com/@useTRMNL
```

### Categories

The list is served live at <https://trmnl.com/api/categories>
([Categories API](https://docs.trmnl.com/go/public-api/categories-api.md)).
On 2026-08-28 the API returned:

```
album, analytics, art, calendar, comics, crm, custom, discovery, ecommerce,
education, email, entertainment, environment, finance, games, humor, images,
kpi, life, marketing, morbid, nature, news, personal, productivity,
programming, sales, sports, travel
```

There is no "weather" category. The best fits for us are **environment** and
**nature**. Use both (the limit is 2).

Note: "Recipe `author_bio` values are cached for ~2 hours. If you update a plugin's
category, it will take up to 2 hours for it to appear in search results."
([Form builder](https://help.trmnl.com/en/articles/10513740-custom-plugin-form-builder))

### Demo data methods (help centre)

From [Demo Data for Publishing Plugins](https://help.trmnl.com/en/articles/12772238-demo-data-for-publishing-plugins):

1. **Webhook**: push fake data by cURL, then force refresh. (Not our strategy.)
2. **Demo account**: make a free account with sample data.
3. **Different URL**: "Create a sample response, for instance JSON, with fake data,
   then host that file on a personal server or as part of a GitHub project."
4. **API key trick**: when a form field value matches a sentinel value, swap in
   fake data in the Shared view:

   ```liquid
   {% if trmnl.plugin_settings.custom_fields_values.apikey == "0" %}
     {% assign data = '[{"id":"01ABC123XYZ456"}]' | parse_json %}
   {% endif %}
   ```

5. **Featured image**: pick which screen becomes the preview.

**Freeze tip**: "You can freeze your plugin's screen by removing it from any
playlists. A plugin (in full view) **not** in a playlist will not have its (full
view) screen refreshed, which is the view used for the install page of a recipe."

Best practice: "When publishing a plugin, it will have a new badge, 'Recipe Master'.
This should stay in a 'demo' state, and you should Install the recipe again in your
account, just as other users would do, for personal use."

---

## 4. Review time, and changes after publication

**Review time.** "usually takes just a day or two."
([Private Plugins](https://help.trmnl.com/en/articles/9510536-private-plugins))

**Do changes reach existing users?** Yes, automatically.

- "Please be careful when modifying your origin plugin instance, as updates will be
  pushed to all other users automatically. We suggest using the copy feature (clone
  icon) to test changes first, then update your 'Recipe Master' instance when you
  feel comfortable."
  ([Plugin recipes](https://help.trmnl.com/en/articles/10122094-plugin-recipes))
- "As recipe master, any changes you make will automatically propagate to others who
  have installed your recipe."
  ([Compare plugin types](https://help.trmnl.com/en/articles/10546870-compare-custom-plugin-types))
- Installed Recipes "receive automatic updates (bug fixes, markup improvements)".
  Forked Recipes "do not receive any updates from the original creator".
  ([Plugin recipes](https://help.trmnl.com/en/articles/10122094-plugin-recipes))

**Do changes need a new review?** UNKNOWN — not confirmed. No TRMNL page says that
an edit to a published Recipe re-runs Chef or a human review. We searched
docs.trmnl.com (sitemap and llms-full.txt), the Plugin recipes article, Recipe best
practices, Private Plugins, Compare custom plugin types, and the publishing blog
post. All wording points to changes going live at once, with no gate.

**What can be changed after publication?**

- Markup, form fields and settings: yes, they propagate (see above).
- Status: "By design, published plugins may only be Unlisted by contacting our team.
  They also may not be deleted without coordination by our team and a notification
  to affected users (external installs)."
  ([Plugin recipes](https://help.trmnl.com/en/articles/10122094-plugin-recipes))
- Before publication: "If your plugin is Unlisted or In Review, it may be edited."
  Unlisted plugins may go back to Private or be submitted to the public marketplace.
  In Review plugins may go back to Private.
  ([Plugin recipes](https://help.trmnl.com/en/articles/10122094-plugin-recipes))

**Disclaimer TRMNL shows users.** "The TRMNL team manually approves every Recipe.
This does not mean they are immune from errors."
([Plugin recipes](https://help.trmnl.com/en/articles/10122094-plugin-recipes))

---

## 5. The Creator Fund

Source: [Get paid to build on TRMNL](https://trmnl.com/blog/creator-fund).
The post is dated 22 July 2025 and was updated with the first payout, made
mid-November 2025.

**What it requires of a published Recipe:**

- The contribution must be "a published plugin or a merged commit to our open source
  repositories".
- "there is a minimum threshold of 50 total connections (installs + forks) to
  qualify for monthly payouts."
- "as of April 1, plugins in the 'comics' category are not monetizable." Our
  category (environment / nature) is fine.
- Earnings use "a plugin's age (on live playlists), presence (by % dominance on
  playlists), and impressions (times displayed on device), relative to all other
  community plugins."
- Money comes from 80% of TRMNL+ subscription revenue plus 10-15% of other revenue
  streams. Payouts are monthly, from early December 2025.
- "over time the above weights, revenue share percentages, and minimum thresholds
  may change."

**Licence or exclusivity condition:** none found. The blog post has no licence
clause, no assignment clause and no exclusivity clause. The TRMNL Terms of Service
also have no clause about plugin or Recipe intellectual property
([Terms of Service](https://trmnl.com/terms)). TRMNL in fact encourages publishing
source code: the Moderation Guidelines suggest you "Upload the source code or zip
export to your GitHub or personal profile"
([Plugin recipes](https://help.trmnl.com/en/articles/10122094-plugin-recipes)), and
TRMNL runs its own open-source plugin repository at
<https://github.com/usetrmnl/plugins>.

**Conclusion:** an MIT licence on our repository does not conflict with publication
or with the Creator Fund, from the sources we can read. Caveat: there may be terms
inside the TRMNL account UI (a Creator Fund sign-up form or payout agreement) that
we cannot read without an account. See Open questions.

---

## 6. Personal data: what we must strip

The risk is the **Recipe Master** plugin instance. It is our own live plugin.

**What is safe.** "Note: if your Recipe has custom fields, their values will not be
copied over. So your personal information will not be shared."
([Plugin recipes](https://help.trmnl.com/en/articles/10122094-plugin-recipes))
The same point: "no form field data is visible to anyone installing a published
recipe".
([Demo Data](https://help.trmnl.com/en/articles/12772238-demo-data-for-publishing-plugins))

**What leaks.** The rendered screen. "we use the latest screen generation as the
Recipe preview image, so just be careful about which data you plug into the official
'master' Recipe."
([Plugin recipes](https://help.trmnl.com/en/articles/10122094-plugin-recipes))
"the number one risk is with the screen generated to show what the plugin does."
([Demo Data](https://help.trmnl.com/en/articles/12772238-demo-data-for-publishing-plugins))

**For our weather Recipe, this means:**

- Do not put our real home coordinates or our home place name in the Recipe Master.
  A latitude/longitude is personal data: it shows where we live.
- Use a well-known public place instead. Example: Oslo, or Bergen, or Trondheim.
- Do not put a real personal e-mail into the `polling_headers` User-Agent if we do
  not want it public. The polling headers are part of the Recipe and are copied to
  every install. (MET Norway requires an identifying User-Agent with a contact
  address — see section 7. Use a project address, not a private one.)
- Do not put any API key, token or account identifier into the master. MET Norway
  needs none.
- Keep the Recipe Master out of all playlists to freeze the preview screen.
- Install the Recipe a second time in our own account for real personal use.

---

## 7. Naming and branding: TRMNL rules versus MET Norway rules

### What TRMNL requires

- Title starts with a capital letter, 50 characters or less (Chef).
- Recipe titles have no other documented rule. UNKNOWN — not confirmed: we found no
  TRMNL trademark or naming policy for plugin names. We checked
  <https://trmnl.com/brand> (assets and colours only), the Terms of Service, the
  Moderation Guidelines and all plugin docs pages.
- The one branding tip, and it is only a nice-to-have: "If you use a `title_bar`,
  don't use the default TRMNL logo that is in our example docs, but instead use an
  image appropriate for the data source."
  ([Blog](https://trmnl.com/blog/plugin-recipe-publishing-tips)) The blog labels this
  section "The Little Things ... mostly they are nice-to-haves and not required for
  publishing."
- TRMNL does supply its own brand assets for partners at <https://trmnl.com/brand>,
  but nothing forces a plugin to carry a data-source logo.

### What MET Norway forbids

From [MET Norway Terms of Service](https://api.met.no/doc/TermsOfService), section
"Trademarks and naming restrictions":

> "You are not allowed to make services/sites that appear to be made by Yr, NRK or
> The Norwegian Meteorological Institute. In other words, you may not use the word
> 'Yr' as a part of your service name, or attach the Yr logo to your service."

MET Norway also requires:

- **Attribution.** "All open data require attribution as specified in the CC BY 4.0
  license: You must give appropriate credit, provide a link to the license, and
  indicate if changes were made ... but not in any way that suggests the licensor
  endorses you or your use."
- **Identification.** "All requests must (if possible) include an identifying User
  Agent-string (UA) in the request with the application/domain name, optionally
  version number. You should also include a company email address or a link to the
  company website ... If we cannot contact you in case of problems, you risk being
  blocked without warning." Valid example: `AcmeWeatherApp/0.9 github.com/acmeweatherapp`.

### Is there a conflict?

**No.** TRMNL never demands a data-source logo or a data-source name. Its only
suggestion is to use "an image appropriate for the data source", and a neutral
weather glyph satisfies that.

Our rules, then:

- Do **not** name the Recipe "Yr", "Yr Weather", "NRK ..." or "Norwegian
  Meteorological Institute ...".
- Do **not** use the Yr logo or the MET Norway logo as the plugin icon, the
  `title_bar` icon or anywhere in the markup.
- Do use a neutral name, for example "Norway Weather Forecast" or "Nordic Forecast".
  UNKNOWN — not confirmed whether TRMNL objects to a country name in a title; nothing
  suggests it does.
- Do give the CC BY 4.0 credit and a link to the licence in the `author_bio`
  description, worded so it does not suggest MET Norway endorses us.
- Do set a MET-compliant `User-Agent` polling header with the project name and the
  GitHub URL.

---

## Publishing checklist

Do before we click "Publish as a Recipe".

**Account**

- [ ] Developer add-on (or BYOD licence) active on the TRMNL account.

**Naming and identity**

- [ ] Title starts with a capital letter and is 50 characters or less.
- [ ] Title has no "Yr", "NRK" or "Norwegian Meteorological Institute".
- [ ] Icon uploaded in the settings view. Not a Yr or MET logo.
- [ ] Featured image set from a clean demo screen.

**Form fields**

- [ ] An `author_bio` field exists.
- [ ] `author_bio` has `category: environment,nature` (verify against
      <https://trmnl.com/api/categories> on the day).
- [ ] `author_bio` has at least one contact route: `email_address` and/or
      `github_url`.
- [ ] `author_bio` description says what the plugin does, gives the CC BY 4.0
      attribution to MET Norway with a link to the licence, and says the plugin is
      not made by Yr / NRK / MET Norway.
- [ ] Every custom field keyname is used in the polling URL, the markup or the
      transform JS. No orphan fields.
- [ ] Use `default` and `placeholder` correctly. Never write `optional: false`.
- [ ] All links in `description` / `help_text` are HTML `<a href>` links, with
      `<br/>` where a line break is wanted.
- [ ] Each field tested: change it, save, check "edit markup".

**Markup**

- [ ] No `async function` anywhere.
- [ ] No `window.onload` or `window.addEventListener('load')`. Use
      `DOMContentLoaded`.
- [ ] No inline `opacity:`. Use `--gray--##` / `text--black 2bit:text--gray-30`
      classes.
- [ ] 6 or fewer occurrences, in total across all layouts, of `justify-content`,
      `padding`, `margin`, `background-color`, `border-radius`, `text-align`,
      `object-fit`, `font-size`.
- [ ] No `view--full`, `view--half_horizontal`, `view--half_vertical`,
      `view--quadrant` classes written by us.
- [ ] Every layout has content (10+ characters), or the Shared view does.
- [ ] Responsive classes (`lg:`, `portrait:`) used. Check TRMNL OG landscape,
      TRMNL X landscape and TRMNL X portrait.
- [ ] Static `<img src>` URLs answer HTTP 200, or are inlined as `data:` URIs.
- [ ] If we use Highcharts: `animation: false` and `append_random` on element ids.
- [ ] Liquid templates get the native variables:
      `{% render "x", trmnl: {{ trmnl }} %}`.
- [ ] `title_bar` icon is not the TRMNL example logo, and is not a MET or Yr logo.
- [ ] The plugin is not a Fork. If it started as a fork, export a zip and import it
      as a fresh plugin.

**Data and privacy**

- [ ] Recipe Master uses a public demo location, not our home.
- [ ] `User-Agent` polling header carries the project name and a public contact URL
      or address, not a private e-mail we want kept private.
- [ ] No API key, token or account id in the master.
- [ ] Recipe Master removed from all playlists, to freeze the preview screen.
- [ ] After publication: install the Recipe a second time for our own real use.

**After we publish**

- [ ] Test every future change on a clone (the copy / clone icon), not on the
      Recipe Master. Changes on the master go to all users at once.
- [ ] Remember that unlisting or deleting a published Recipe needs the TRMNL team.

---

## Open questions

1. **Does an edit to a published Recipe need a new review?** UNKNOWN — not
   confirmed. No source says Chef or a human re-runs. All sources say changes
   propagate automatically. We should ask TRMNL support, or watch what happens on
   the first edit.
2. **Is there a Creator Fund agreement with terms we cannot read?** The blog post
   has no licence or exclusivity clause and the public Terms of Service have none.
   But payout probably needs a form inside the account UI. UNKNOWN — not confirmed.
   Check when we have an account.
3. **Can we upload extra marketing screenshots?** UNKNOWN — not confirmed. Only one
   icon and one featured image are documented.
4. **Exact "Publish as a Recipe" dialogue fields.** UNKNOWN — not confirmed. The
   help centre shows the button but not a field-by-field list of the submit form.
   We will see it in the UI.
5. **Does TRMNL restrict country or brand words in Recipe titles?** UNKNOWN — not
   confirmed. No naming policy was found.
6. **Which category do reviewers prefer for weather?** There is no "weather"
   category. `environment` and `nature` are our best guess. Existing weather Recipes
   on <https://trmnl.com/recipes> could be checked for the convention.

---

## Sources

TRMNL docs:

- Sitemap: <https://docs.trmnl.com/go/sitemap.md>
- Whole docs as one file: <https://docs.trmnl.com/go/llms-full.txt>
- Plugin marketplace introduction: <https://docs.trmnl.com/go/plugin-marketplace/introduction.md>
- Going Live (Third Party plugins): <https://docs.trmnl.com/go/plugin-marketplace/going-live.md>
- Recipes API: <https://docs.trmnl.com/go/public-api/recipes-api.md>
- Categories API: <https://docs.trmnl.com/go/public-api/categories-api.md>
- Live category list: <https://trmnl.com/api/categories>

TRMNL help centre:

- Private Plugins: <https://help.trmnl.com/en/articles/9510536-private-plugins>
- Plugin recipes: <https://help.trmnl.com/en/articles/10122094-plugin-recipes>
- Recipe best practices: <https://help.trmnl.com/en/articles/11395668-recipe-best-practices>
- Demo Data for Publishing Plugins: <https://help.trmnl.com/en/articles/12772238-demo-data-for-publishing-plugins>
- Custom plugin form builder: <https://help.trmnl.com/en/articles/10513740-custom-plugin-form-builder>
- Compare custom plugin types: <https://help.trmnl.com/en/articles/10546870-compare-custom-plugin-types>
- Importing and exporting private plugins: <https://help.trmnl.com/en/articles/10542599-importing-and-exporting-private-plugins>

TRMNL blog and site:

- A Checklist for Turning a Private Plugin into a Recipe for Others: <https://trmnl.com/blog/plugin-recipe-publishing-tips>
- Get paid to build on TRMNL (Creator Fund): <https://trmnl.com/blog/creator-fund>
- Terms of Service: <https://trmnl.com/terms>
- Brand assets: <https://trmnl.com/brand>
- Recipes directory: <https://trmnl.com/recipes>
- Chef linter source (gist linked from Recipe best practices):
  <https://gist.github.com/ryanckulp/fbe5f68c51db1ae214a97da24be4d62b>

MET Norway:

- Terms of Service: <https://api.met.no/doc/TermsOfService>
