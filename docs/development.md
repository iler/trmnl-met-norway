# Development loop

The repo is the source of truth for the Recipe (ADR: ticket #12). Edit the files
in `src/`, preview them locally, and push to TRMNL only when they are right. Do
not edit the markup in the TRMNL browser editor: after publication, an edit to
the Recipe Master is a production change for every installed user.

## Preview locally

`trmnlp` is TRMNL's own dev server. It reads `src/`, polls MET Norway for real,
and renders the screen as TRMNL does.

```sh
docker run --rm --name trmnlp -p 4567:4567 \
  -v "$PWD:/plugin" trmnl/trmnlp serve
```

Open <http://localhost:4567/full>. The page auto-reloads when a file in `src/`
changes, but it keeps the last polled payload. Use the **Poll** button, or
`curl http://localhost:4567/poll`, to fetch MET again.

Routes that are useful without a browser:

| Route | Gives |
|---|---|
| `/render/full.png?width=800&height=480` | the 1-bit PNG, the same as the device |
| `/render/full.html?width=800&height=480` | the rendered HTML, for reading the markup |
| `/data` | the merge variables as JSON |
| `/poll` | a fresh poll of both URLs |

## Preview the small layouts

`trmnlp` cannot render a PNG narrower than about 450px: its headless Firefox
refuses to size the window below that, and `/render/half_vertical.png` fails
with `the browser clamped the viewport to 450x480`. That covers
**half_vertical** (400x480) and **quadrant** (400x240). The full layout
(800x480) and **half_horizontal** (800x240) are wide enough to render normally.

Use the HTML render in a browser sized to the real viewport instead:

```
http://localhost:4567/render/half_vertical.html?width=400&height=480
http://localhost:4567/render/quadrant.html?width=400&height=240
```

The full layout is 800x480 and renders to PNG normally.

## Test a warning when Norway is calm

MET issues warnings for one area at a time and does exact point-in-polygon, so
a point falls in exactly one area. On a calm day no real coordinate produces two
warnings, and the "+N more" and severity-tie paths cannot be reached from live
data.

Override the payload in `.trmnlp.yml`. The `variables:` block replaces any
top-level key, not only `trmnl`:

```yaml
variables:
  trmnl:
    plugin_settings:
      instance_name: Helsinki
  IDX_1:
    lastChange: '2026-08-31T00:00:00+00:00'
    features:
      - properties:
          riskMatrixColor: Orange
          eventAwarenessName: Gale
          area: Vestfjorden
        when:
          interval: ['2026-08-31T18:00:00+00:00', '2026-09-01T06:00:00+00:00']
      - properties:
          riskMatrixColor: Orange
          eventAwarenessName: Extreme rainfall
          area: Nordland
        when:
          interval: ['2026-08-31T03:00:00+00:00', '2026-08-31T15:00:00+00:00']
```

Two warnings of equal severity with different onsets test the tie-break: the
screen must show **Extreme rainfall**, the earlier one, not the first in the
list.

The override **deep-merges**, so it can add or replace a value but cannot empty
one out. `IDX_0: {}` and `timeseries: []` both leave the real payload in place.
To test a failed poll, point the URL in `src/settings.yml` at a path that does
not exist, as described below.

## Preview another place

Edit `custom_fields` and `time_zone` in `.trmnlp.yml`. Do not use
`{{ env.VARIABLE }}` there: `trmnlp` interpolates it into the polling URL but
gives the templates the raw string, so the screen shows the Liquid source
instead of the value.

Places that were useful while building:

| Place | Latitude | Longitude | Shows |
|---|---|---|---|
| Helsinki | 60.1699 | 24.9384 | the calm case, and the demo location of the master |
| Hammerfest | 70.6634 | 23.6821 | a Norwegian point that often carries a warning |
| Greenland Sea | 78.5000 | -15.0000 | a marine warning area, and a point with no precipitation probability |

Warnings move. To find a point that has one now:

```sh
curl -H 'User-Agent: trmnl-met-norway/1.0 github.com/iler/trmnl-met-norway' \
  'https://api.met.no/weatherapi/metalerts/2.0/current.json?lang=en' \
  | jq -r '.features[].properties | "\(.riskMatrixColor) \(.eventAwarenessName) — \(.area)"'
```

To see a failed poll, point one of the two URLs in `src/settings.yml` at a path
that does not exist, then poll again.

## Push to TRMNL

`src/settings.yml` carries `id: 460164`. `trmnlp push` updates that plugin.
**Without the id it creates a new plugin on every run**, so never remove it.

```sh
bin/push            # asks before it overwrites
bin/push --force    # no prompt
```

**Run it from your own terminal.** A coding agent's sandbox cannot read
`~/.config/op`, so `op read` fails there with `operation not permitted`, and no
push happens. In Claude Code, type `! bin/push` so the output lands in the
conversation.

`bin/push` takes the key from the first source that has it:

1. `TRMNL_API_KEY` already in the environment — GitHub Actions, or an export.
2. `op read` of `TRMNL_API_KEY_REF`, which defaults to
   `op://Private/TRMNL/credential`. Override it for a different vault or item:

   ```sh
   export TRMNL_API_KEY_REF="op://Vault/Item/field"
   ```

Either way it hands the key to Docker **by name, not by value**, so it never
reaches your shell history or the process list. Do not write
`-e TRMNL_API_KEY=<the key>` by hand.

Lint first. It needs no key, so it costs nothing:

```sh
docker run --rm -v "$PWD:/plugin" trmnl/trmnlp lint
```

`.github/workflows/trmnl.yml` runs the same lint on every pull request and on
`main`. It has no push job on purpose: lint needs no key, so the gate costs no
exposure, while a push job would put the account key where every action in the
workflow can read it.

### After publication, a push is a production change

Today the plugin is private and a bad push costs nothing. Once the Recipe is
published, the Recipe Master is live for every installed user, and its own
generated screen is the public preview. At that point `bin/push --force` is a
deploy, not a save. Lint, preview locally, and look at the screen before you
run it.

## Brand assets

`docs/brand/` holds the plugin icon: `icon.svg` is the source, `icon-512.png`
is the export TRMNL's settings view takes.

```sh
magick docs/brand/icon.svg -background white -alpha remove -resize 512x512 \
  -colorspace sRGB -type TrueColor -depth 8 -strip docs/brand/icon-512.png
```

**This is a manual sync point.** `bin/push` does not carry the icon: it is not
one of `settings.yml`'s fields, and TRMNL takes it through the settings view
only. The repo holds the source so the mark can be recut or restored; the
served copy lives in TRMNL and has to be uploaded by hand.

The mark is drawn from primitives on purpose. The `wi-*` glyphs the markup uses
are outlines that thin to a scribble below about 32px, and they are SIL OFL 1.1;
keeping them out of the icon leaves it wholly MIT. It carries no Yr, NRK or MET
Norway reference, which the MET terms require.

## The featured image is not a file

Nothing to design or upload. TRMNL captures it from the plugin's **current
generated screen**: "edit your recipe settings, and there will be a button to
set the preview image as the current screens image." So the work is to make the
screen worth capturing, then press the button.

Keep the Recipe Master out of every playlist. A full-view plugin in no playlist
does not have its screen refreshed, which is what freezes the install page
preview.

The screen worth capturing is one with a warning band, because the warnings are
what this recipe adds. That needs a real warning live at a real coordinate at
capture time: point the master at it, let it refresh, capture, then set the
fields back to the demo location. The stored image keeps the warning.

**Never capture a screen built from a `.trmnlp.yml` fixture.** The fixtures
exist to test paths the weather will not produce on demand. A fixture screen
presented as a marketplace preview is a fabricated product shot.

## The API key, and who can hold one

The key is **per account, not per plugin**. It reads and writes every private
plugin on the account.

**It is not in this repo's 1Password Environment, and it should not be added.**
That Environment holds the secrets of one plugin; a key with a wider scope than
the mount would be copied into every plugin repo, which means many places to
rotate and an account-wide key readable by everything in each of them. 1Password
also allows only ten mounted `.env` files per device, so each repo's mount is a
budget worth spending on plugin-scoped secrets.

Instead, one canonical item feeds every plugin repo through
`TRMNL_API_KEY_REF`. A rotation is one edit, and `bin/push` is byte-identical
across repos, so it can be copied to a new plugin unchanged.

There is no narrower key to hand out, so:

- A second person cannot be given a push key scoped to this plugin. Either the
  account owner does the pushes, or that person is given the account key, which
  gives them every plugin on it.
- The way to share the *process* is this file plus `.env.example`, never the
  value. Someone with their own TRMNL account points `TRMNL_API_KEY_REF` at
  their own item, or exports `TRMNL_API_KEY` for one command.
- Agents must not read `./.env`. It is a named pipe that streams the real
  secrets; the repo installs the 1Password validator hook to catch misuse.

## Traps found the hard way

- `{% endtemplate %}` must be written exactly like that. The `trmnl-liquid` tag
  closes only on that literal token, so `{%- endtemplate -%}` swallows the rest
  of the file and the screen renders empty, with no error.
- `{% render %}` takes comma-separated arguments:
  `{% render "weather_icon", code: x, box: "..." %}`.
- `{% render %}` gets an isolated scope. Everything it needs must be an argument.
- **`trmnlp push` rewrites `src/settings.yml`.** It replaces the file with the
  server's canonical form: comments stripped, keys reordered, `description` and
  every unset `oauth_*` field added. Do not put anything in that file you need
  to survive a push. That is why the reason the `id` matters is written here and
  not as a comment beside it.
- **A 1px vertical rule can vanish on a 1-bit screen.** `divider--v` computes to
  0.97px wide, and whether it survives rasterisation depends on where its
  sub-pixel x position falls. In the half horizontal layout two identical rules
  behaved differently: the one at x=533 painted 29 pixels, and the one at x=213
  painted **nothing at all**, in any neighbouring column. Add `w--[2px]` to a
  vertical rule so it always covers a whole device pixel. A horizontal rule is
  safe, because its 1px height lands on a pixel row.
- `probability_of_precipitation` is a Nordic-area value. It was absent at every
  one of the 62 time steps of the Greenland Sea point, where `precipitation_amount`
  was still there. Any value from `/complete` needs a check before it is trusted
  worldwide.
