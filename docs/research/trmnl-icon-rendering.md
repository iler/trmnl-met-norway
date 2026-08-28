# How a TRMNL recipe shows icons on a 1-bit screen

Research for ticket #3. Date: 2026-08-28.
All facts have a source URL. Facts that we could not confirm are marked
`UNKNOWN — not confirmed`.

## Short answer

Recipe markup can load an image from an external host. TRMNL renders the markup
as a web page in a cloud worker, then makes a screenshot. But TRMNL tells you
not to do it: if a worker cannot get the URL, the full screen render fails.
TRMNL tells you to use inline SVG or a `data:` URI instead.

**Do not use the MET colour icons as they are.** We measured three MET PNG
icons. Every opaque pixel of `clearsky_day` and `partlycloudy_day` has a
luminance between 163 and 222 out of 255. Not one pixel is darker than 50% grey.
The art has no dark outline. On a 1-bit screen these icons become white, or a
faint speckle after dithering. They are not readable.

Use the icon set that TRMNL hosts instead:
`https://trmnl.com/images/plugins/weather/wi-*.svg`. These are black
silhouettes from Weather Icons by Erik Flowers, under the SIL Open Font
License 1.1. TRMNL's own MET Norway example uses them. The most installed
weather recipes use the same kind of black line art. Map each MET
`symbol_code` to a `wi-*` name.

---

## 1. External images, inline assets, and size limits

### Markup can load external files

TRMNL renders your template "as a normal webpage, then screenshot[s]" it
([template guide, §1][tg]). The documented quickstart markup loads an external
stylesheet, an external script and an external image:

```erb
<link rel="stylesheet" href="https://trmnl.com/css/latest/plugins.css">
<script src="https://trmnl.com/js/latest/plugins.js"></script>
...
<img class="image" src="https://trmnl.com/images/plugins/trmnl--render.svg" />
```

Source: [Screen Templating][tmpl]. The graphics page also loads Highcharts from
`code.highcharts.com` and Chartkick from `cdn.jsdelivr.net`
([Screen Templating (Graphics)][tmplg]). So external hosts are not blocked.

### But TRMNL says not to depend on them

TRMNL's own error article says:

> "Our screen rendering workers are unable to access a URL specified within
> your plugin markup, most commonly an image `src`."

Source: [Plugins::Base.process! -> StandardError: private_plugin][err]. This is
an error page. A URL that the worker cannot get makes the **whole screen
render fail**, not only the image.

The TRMNL agent prompt makes this a hard rule for icons:

> "to customize the title_bar icon, use **inline SVG or base64-encoded PNG** —
> never a URL. network requests can fail on the device."

Source: [`agent_prompt.md`, line 307][ap]. The template guide repeats it:
"inline images eliminate network requests — more reliable on the device" and
"never use a URL-based icon when inline SVG is possible" ([template guide,
§4][tg]).

The help centre gives the same advice and the tools to do it: capture the SVG
in `shared.liquid`, then use the Liquid `base64_encode` filter:

```html
<img class="image image-stroke" src="data:image/svg+xml;base64,##{{ svg_logo | base64_encode }}" />
```

Source: [Creating Inline Images for Plugins][inline].

### Where the workers run

The render workers run on Hetzner Cloud. Some servers block their IP ranges,
and Cloudflare bot rules can block them
([Plugin Not Receiving Data from Polling URL][poll]).

### Size limits

| Item | Limit | Source |
| --- | --- | --- |
| One Liquid template file | 1 MB | [Importing and exporting private plugins][imp] |
| Webhook payload | 2 kB, or 5 kB for TRMNL+ | [Webhooks][wh] |
| Polling response | UNKNOWN — not confirmed | We read [Private Plugins][pp] and [Plugin Not Receiving Data][poll]. Neither states a limit. |
| Size of an image asset | UNKNOWN — not confirmed | No limit found in the docs, the help centre, or the framework source. |

### Caching and proxying

UNKNOWN — not confirmed. We found no document that says how the render worker
caches or proxies an image that markup loads. A related but different feature,
the Image Display plugin, uses `etag`/`If-None-Match` and
`last-modified`/`If-Modified-Since` to skip a re-render when the image did not
change ([Image Display][imgdisp]). That article is about that plugin, not about
`<img>` tags in recipe markup.

### Measured: the MET host does send CORS

We measured `raw.githubusercontent.com` on 2026-08-28:

```
HTTP/2 200
content-type: image/png
cache-control: max-age=300
access-control-allow-origin: *
cross-origin-resource-policy: cross-origin
```

So a CSS mask can read a MET icon. See section 4 for why that still does not
help.

---

## 2. How TRMNL turns colour into 1 bit

### The pipeline

> "your template is rendered as a normal webpage, then screenshotted.
> ImageMagick converts it to the device's target bit depth — on 1-bit, every
> pixel becomes pure black or pure white."

Source: [template guide, §1][tg]. The template guide and the agent prompt ship
in `usetrmnl/trmnl-agent-skills`, which states that it embeds the production
prompts of TRMNL's MCP server.

### Dithering is opt-in per image

Add the class `image-dither` to an `<img>`:

> "Add `image-dither` to a raster image to have it dithered to the screen's
> palette. The dithering itself is a platform behavior: TRMNL applies it when
> it renders the screen."

Source: [framework image docs source][fwimg] and the published page
[trmnl.com/framework/docs/3.3/image][fwimgweb].

The algorithm is named as Floyd-Steinberg:

> "`image-dither` triggers Floyd-Steinberg dithering in the rendering pipeline"

Source: [template guide, §11][tg]. The agent prompt agrees: "e-ink displays
need Floyd-Steinberg dithering to render photos and complex images properly.
without it, images look washed out" ([`agent_prompt.md`][ap]).

The threshold value is UNKNOWN — not confirmed. We searched
`docs.trmnl.com/go/llms-full.txt`, the framework docs, the framework source
(`usetrmnl/trmnl-framework`), `plugins.css`, `plugins.js` and the help centre.
No page gives a threshold number for the markup render path.

### The DIY ImageMagick guide is a different path

`-dither FloydSteinberg -posterize 16` in the [ImageMagick Guide][im] is for
images you build yourself and then serve to an Alias or Redirect plugin, or to
DIY hardware. The guide says so: "Below are some tips to generate TRMNL
compatible images for DIY devices or Alias/Redirect plugin applications." It
does not describe what happens to plugin markup. The same command with
`-posterize 4` (2-bit) is marked "experimental".

### Greys in the framework are patterns, not colour

On 1-bit, framework grey tokens paint as tiled black-and-white patterns, not as
grey. We confirmed this in `https://trmnl.com/css/latest/plugins.css`: the
`.screen` rule defines `--tile-gray-10-1bit` … as `data:image/svg+xml` tile
images, sized by `--dither-bg-size: calc(16px / var(--dither-ratio))`. The
template guide says the same and adds a warning:

> "never use raw `background-color: gray` — it'll be crushed to pure black or
> white on 1-bit. never use CSS `opacity`, `box-shadow`, or
> `background: linear-gradient(...)` — they all posterize unpredictably."

Source: [template guide, §14][tg].

The help centre states that 1-bit "uses dithering to approximate 14 shades of
gray", and that the only non-dithered classes on 1-bit are `*--black` and
`*--white` ([Grayscale: 1-bit, 2-bit, 4-bit in Framework][gray]).

### Do not dither an icon

> "dithering is for photos. icons and text need hard crisp edges"

Source: [template guide, §16][tg]. And: "the only images that DON'T need
`image-dither` are small icons in the title_bar" ([`agent_prompt.md`][ap]).
So `image-dither` will not save a colour icon. It is the wrong tool.

---

## 3. Are the MET colour icons readable at 48-64 px?

No.

### What we measured

We fetched three PNGs from
`raw.githubusercontent.com/metno/weathericons/main/weather/png/` on 2026-08-28
and decoded them with a script (no ImageMagick on this machine). All three are
200x200 RGBA. Luminance uses Rec. 709 (`0.2126R + 0.7152G + 0.0722B`), over the
opaque pixels only.

| Icon | Colours | Luminance min / median / max | Pixels darker than 50% grey |
| --- | --- | --- | --- |
| `clearsky_day` | 414 | 163 / 203 / 220 | 0 % |
| `partlycloudy_day` | 277 | 169 / 217 / 222 | 0 % |
| `heavyrainandthunder` | 30 | 83 / 153 / 215 | 9.4 % |

The main colours are the yellow sun `#ffd348` (luminance 210), the grey cloud
`#dddddd` (221) and `#999999` (153), and the blue rain `#0062bf` (84).

The important point: **the art has no dark outline.** The darkest pixel in
`clearsky_day` is luminance 163. The shapes are flat fills that rely on hue,
not on lightness, to separate from each other and from the white page.

### What that means on the screen

We rendered the same three icons to 1-bit at 56 px, once by hard threshold at
128 and once with Floyd-Steinberg:

* Hard threshold: `clearsky_day` and `partlycloudy_day` become **fully white**.
  Nothing is left. `heavyrainandthunder` keeps only the blue raindrops, as a
  few small marks. The cloud and the lightning vanish.
* Floyd-Steinberg: the sun and the cloud become a light, even speckle with no
  edge. At 56 px the sun rays and the cloud outline are gone. Only
  `heavyrainandthunder` stays a dark blob, because its grey cloud is at
  luminance 153.

A viewer cannot tell "clear sky" from "partly cloudy" from these results.

### What real TRMNL weather recipes do

Every published example uses black line art, not colour.

* **Met.no Forecast** (recipe 132448, in the public gallery) is a real MET
  Norway recipe. We downloaded its published screenshot: a 800x480 1-bit PNG.
  It shows sun, cloud and rain as crisp black glyphs of about 32-40 px, each
  one clear. Get the URL with
  `https://trmnl.com/recipes/132448.json` → `data.screenshot_url`.
* **Weather Glance** (recipe 181200, 1909 installs) uses the same black
  outline glyphs on dithered grey bars.
* TRMNL's own LaraPaper reference server has a MET Norway recipe. It calls
  `properties.timeseries.0.data...` from api.met.no, and for the icon it loads
  `.../images/plugins/weather/wi-thermometer.svg` from TRMNL. It does **not**
  load a MET icon. Source:
  [`resources/views/recipes/weather.blade.php`][larapaper].
* The framework's own weather example fixture uses
  `<img class="image--adaptive ..." src="/images/plugins/weather/wi-day-sunny.svg">`.
  Source: [`public/framework/example_fixtures/weather/full.html`][fixture].

Use the Recipes API to repeat this check:
`https://trmnl.com/recipes.json?search=weather&sort_by=popularity`
([Recipes API][rapi]).

---

## 4. What the framework already gives you

### A hosted weather icon set

TRMNL hosts an icon pack for plugin authors at
`https://trmnl.com/images/plugins/weather/`. The help centre calls it "our
hosted icon pack for your own plugins" ([Weather icons][wicons]).

The framework's licence file names the set:

> "Weather Icons by Erik Flowers, <https://erikflowers.github.io/weather-icons/>.
> The icon designs are originally by Lukas Bischoff. Licensed under the SIL
> Open Font License, Version 1.1"

Source: [`THIRD_PARTY_NOTICES.md`][tpn]. (That file lists the 15 glyphs the
framework repo itself ships. The hosted pack on trmnl.com is larger.)

We tested 18 names on 2026-08-28. All returned HTTP 200:

```
wi-day-sunny        wi-night-clear      wi-day-cloudy       wi-cloudy
wi-day-rain         wi-rain             wi-snow             wi-thunderstorm
wi-fog              wi-day-sleet        wi-day-snow         wi-showers
wi-sleet            wi-day-fog          wi-night-alt-cloudy wi-day-thunderstorm
wi-night-alt-rain   wi-day-sunny-overcast
```

The files are plain SVG. `wi-day-sunny.svg` is a 30x30 viewBox with `<path>`
elements and no `fill` attribute, so it paints solid black. The host sends:

```
content-type: image/svg+xml
access-control-allow-origin: *
cache-control: max-age=14400
```

This set has day and night names, which fits the MET `_day` / `_night` /
`_polartwilight` suffixes.

### An image component

The framework `image` component ships these classes
([trmnl.com/framework/docs/3.3/image][fwimgweb], source [`image.html.erb`][fwimg]):

| Class | Effect |
| --- | --- |
| `image` | Base class. |
| `image-dither` | Marks the image for platform dithering. For photos only. |
| `image--small` | Caps width at 80 px. |
| `image--xsmall` | Caps width at 40 px. |
| `image--fill` / `--contain` / `--cover` | Object fit. |
| `invert` | Swaps black and white. |
| `image--adaptive` | Recolours a monochrome silhouette to the screen icon colour. |
| `image-stroke` (+ size and colour modifiers) | Draws an outline around an image so it stays readable on a shaded background ([image_stroke][fwstroke]). |

`image--adaptive` is the icon tool. The docs say:

> "Use `image--adaptive` to recolor a monochrome silhouette icon in the
> screen's icon color. Only the icon's shape is used: its own pixel colors are
> ignored."

and:

> "Silhouettes only. The image is flattened to its alpha shape, so never use it
> on photos or multi-color logos … Not meaningful with `image-dither` or
> `invert`."

It works by a CSS mask, so:

> "Recoloring uses a CSS mask, which the browser only permits for same-origin
> icons or hosts that send `Access-Control-Allow-Origin`. An icon on an
> arbitrary third-party host stays a plain image in its own colors. Serve
> recolorable icons from your own origin, or inline the SVG with
> `fill="currentColor"`, which recolors with no classes and no hosting
> constraint."

We confirmed the mechanism in `plugins.css`:

```css
.trmnl img.image--adaptive[data-adaptive]{
  background-color:var(--framework-semantic-icon-color, …);
  background-size:var(--dither-bg-size, auto),auto;
  -webkit-mask:var(--framework-icon-src) center/contain no-repeat;
  mask:var(--framework-icon-src) center/contain no-repeat;
}
```

A comment in `plugins.js` explains the CORS fetch:

> "A CSS mask source is CORS-restricted, so we can only recolor an icon we are
> actually allowed to read: we fetch it and, on success, arm the element with a
> same-origin data: URI".

**Note for the MET icons:** `raw.githubusercontent.com` does send
`Access-Control-Allow-Origin: *`, so the mask would work. But the mask reads
only the **alpha shape**. A MET icon's alpha shape is the whole filled sun or
cloud, with no interior lines. Masking `clearsky_day` gives a solid black blob,
not a sun. So `image--adaptive` does not rescue them either.

### Documented "use high contrast" rules

Yes, several:

* "always use framework classes (`bg--gray-30`, `label--gray`) instead of raw
  CSS colors" ([template guide, §1][tg]).
* "never use raw `background-color: gray` … never use CSS `opacity`,
  `box-shadow`, or `background: linear-gradient(...)`"
  ([template guide, §14][tg]).
* "**never use emoji characters in markup.** e-ink displays have no emoji font
  support — they render as missing glyphs (empty boxes)."
  ([`agent_prompt.md`][ap]).
* "For crisp dithering, supply images at or above the slot's native pixel
  dimensions" ([Image Display][imgdisp]).
* The design system blog post: "Vibrant color palette? Not possible."
  ([TRMNL design system][blog]).

---

## 5. Grayscale on newer firmware

Grayscale exists, but it does not change the answer.

### What is supported

The framework has three grayscale tiers, set by a class on `.screen`
([rendering_modes][fwrm]):

* `screen--1bit`: "every gray token paints as a dither pattern of black and
  white pixels".
* `screen--2bit`: "tokens still dither, now between four gray tones instead of
  two".
* `screen--4bit`: "every gray token paints as a solid".

`plugins.css` gives each device a `--color-depth`. `screen--og` and
`screen--og_png` have `--color-depth: 1`. `screen--v2` and the Kindle profiles
have `--color-depth: 4`. The framework device list has both a "TRMNL OG
(1-bit)" profile (`og_png`) and a "TRMNL OG (2-bit)" profile (`og_plus`).

The help centre says 4-bit is "Supported by our TRMNL X product, no dithering
is necessary". On 2-bit the only non-dithered classes are `*--black`,
`*--white`, `*--gray-30` and `*--gray-55`
([Grayscale: 1-bit, 2-bit, 4-bit in Framework][gray]).

For the OG device, 2-bit PNG needs firmware 1.6.0 or newer, and the docs mark
the feature "experimental":

> "**This feature is experimental** and designed for OG model devices running
> FW 1.6.0+ with grayscale + fast refresh support."

Source: [ImageMagick Guide][im].

### Why it does not change the answer

On 2-bit there are four tones. The MET sun sits at luminance 210 and the MET
grey cloud at 221. Both land on the lightest tone, next to white. The rain
cloud at 153 lands near `*--gray-55`. The icons stay pale and still have no
outline. Only a true 4-bit device (TRMNL X) would show them as light grey
shapes, and a light grey flat shape on a white page is still weak.

We target the original 800x480 OG device, which is 1-bit.

---

## Open questions

1. Maximum size of an image asset that markup can load. Not documented.
2. Maximum size of a polling response. The 2 kB / 5 kB figures apply to
   webhooks only. We could not confirm a polling limit.
3. Whether the render worker caches or proxies images that markup loads, and
   for how long. Not documented for the markup path.
4. The exact threshold, gamma and matrix that the render worker's ImageMagick
   step uses for markup. Only the algorithm name (Floyd-Steinberg) is public.
5. The full file list of the hosted `wi-*` icon pack. There is no index page.
   We only verified names one by one. Somebody must build and test the full
   `symbol_code` → `wi-*` map.
6. Whether a Recipe may set the `screen--2bit` / `screen--4bit` class itself,
   or whether the platform always sets it from the device. Not documented.

---

## Sources

Primary documentation:

* [Screen Templating][tmpl] — `https://docs.trmnl.com/go/private-plugins/templates.md`
* [Screen Templating (Graphics)][tmplg] — `https://docs.trmnl.com/go/private-plugins/templates-advanced.md`
* [Webhooks][wh] — `https://docs.trmnl.com/go/private-plugins/webhooks.md`
* [ImageMagick Guide][im] — `https://docs.trmnl.com/go/diy/imagemagick-guide.md`
* [Recipes API][rapi] — `https://docs.trmnl.com/go/public-api/recipes-api.md`
* Full docs text — `https://docs.trmnl.com/go/llms-full.txt`
* Sitemap — `https://docs.trmnl.com/go/sitemap.md`

Framework:

* [Image component][fwimgweb] — `https://trmnl.com/framework/docs/3.3/image`
* [Image Stroke][fwstroke] — `https://trmnl.com/framework/docs/3.3/image_stroke`
* [Rendering modes][fwrm] — `https://trmnl.com/framework/docs/3.3/rendering_modes`
* Framework stylesheet — `https://trmnl.com/css/latest/plugins.css`
* Framework runtime — `https://trmnl.com/js/latest/plugins.js`

Help centre:

* [Private Plugins][pp] — `https://help.trmnl.com/en/articles/9510536-private-plugins`
* [Creating Inline Images for Plugins][inline] — `https://help.trmnl.com/en/articles/12391781-creating-inline-images-for-plugins`
* [Plugins::Base.process! -> StandardError: private_plugin][err] — `https://help.trmnl.com/en/articles/12814634-plugins-base-process-standarderror-private_plugin`
* [Weather icons][wicons] — `https://help.trmnl.com/en/articles/11823386-weather-icons`
* [Image Display][imgdisp] — `https://help.trmnl.com/en/articles/11479051-image-display`
* [Grayscale: 1-bit, 2-bit, 4-bit in Framework][gray] — `https://help.trmnl.com/en/articles/12386214-grayscale-1-bit-2-bit-4-bit-in-framework`
* [Plugin Not Receiving Data from Polling URL][poll] — `https://help.trmnl.com/en/articles/12386583-plugin-not-receiving-data-from-polling-url`
* [Importing and exporting private plugins][imp] — `https://help.trmnl.com/en/articles/10542599-importing-and-exporting-private-plugins`

Blog:

* [TRMNL design system][blog] — `https://trmnl.com/blog/design-system`

Public source in the `usetrmnl` GitHub org:

* [Framework image docs source][fwimg] — `https://github.com/usetrmnl/trmnl-framework/blob/main/app/views/framework/image.html.erb`
* [Third party notices][tpn] — `https://github.com/usetrmnl/trmnl-framework/blob/main/THIRD_PARTY_NOTICES.md`
* [Weather example fixture][fixture] — `https://github.com/usetrmnl/trmnl-framework/blob/main/public/framework/example_fixtures/weather/full.html`
* [Template guide][tg] — `https://github.com/usetrmnl/trmnl-agent-skills/blob/main/skills/trmnl/references/template_guide.md`
* [Agent prompt][ap] — `https://github.com/usetrmnl/trmnl-agent-skills/blob/main/skills/trmnl/references/agent_prompt.md`
* [LaraPaper MET Norway recipe][larapaper] — `https://github.com/usetrmnl/larapaper/blob/main/resources/views/recipes/weather.blade.php`

MET Norway:

* Icon set — `https://github.com/metno/weathericons`
* Icons measured — `https://raw.githubusercontent.com/metno/weathericons/main/weather/png/{clearsky_day,partlycloudy_day,heavyrainandthunder}.png`

[tmpl]: https://docs.trmnl.com/go/private-plugins/templates.md
[tmplg]: https://docs.trmnl.com/go/private-plugins/templates-advanced.md
[wh]: https://docs.trmnl.com/go/private-plugins/webhooks.md
[im]: https://docs.trmnl.com/go/diy/imagemagick-guide.md
[rapi]: https://docs.trmnl.com/go/public-api/recipes-api.md
[fwimgweb]: https://trmnl.com/framework/docs/3.3/image
[fwstroke]: https://trmnl.com/framework/docs/3.3/image_stroke
[fwrm]: https://trmnl.com/framework/docs/3.3/rendering_modes
[pp]: https://help.trmnl.com/en/articles/9510536-private-plugins
[inline]: https://help.trmnl.com/en/articles/12391781-creating-inline-images-for-plugins
[err]: https://help.trmnl.com/en/articles/12814634-plugins-base-process-standarderror-private_plugin
[wicons]: https://help.trmnl.com/en/articles/11823386-weather-icons
[imgdisp]: https://help.trmnl.com/en/articles/11479051-image-display
[gray]: https://help.trmnl.com/en/articles/12386214-grayscale-1-bit-2-bit-4-bit-in-framework
[poll]: https://help.trmnl.com/en/articles/12386583-plugin-not-receiving-data-from-polling-url
[imp]: https://help.trmnl.com/en/articles/10542599-importing-and-exporting-private-plugins
[blog]: https://trmnl.com/blog/design-system
[fwimg]: https://github.com/usetrmnl/trmnl-framework/blob/main/app/views/framework/image.html.erb
[tpn]: https://github.com/usetrmnl/trmnl-framework/blob/main/THIRD_PARTY_NOTICES.md
[fixture]: https://github.com/usetrmnl/trmnl-framework/blob/main/public/framework/example_fixtures/weather/full.html
[tg]: https://github.com/usetrmnl/trmnl-agent-skills/blob/main/skills/trmnl/references/template_guide.md
[ap]: https://github.com/usetrmnl/trmnl-agent-skills/blob/main/skills/trmnl/references/agent_prompt.md
[larapaper]: https://github.com/usetrmnl/larapaper/blob/main/resources/views/recipes/weather.blade.php
