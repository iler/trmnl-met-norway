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

Warnings move. To find a point that has one now:

```sh
curl -H 'User-Agent: trmnl-met-norway/1.0 github.com/iler/trmnl-met-norway' \
  'https://api.met.no/weatherapi/metalerts/2.0/current.json?lang=en' \
  | jq -r '.features[].properties | "\(.riskMatrixColor) \(.eventAwarenessName) — \(.area)"'
```

To see a failed poll, point one of the two URLs in `src/settings.yml` at a path
that does not exist, then poll again.

## Traps found the hard way

- `{% endtemplate %}` must be written exactly like that. The `trmnl-liquid` tag
  closes only on that literal token, so `{%- endtemplate -%}` swallows the rest
  of the file and the screen renders empty, with no error.
- `{% render %}` takes comma-separated arguments:
  `{% render "weather_icon", code: x, box: "..." %}`.
- `{% render %}` gets an isolated scope. Everything it needs must be an argument.
