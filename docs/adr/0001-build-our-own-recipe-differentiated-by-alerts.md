# Build our own recipe, differentiated by severe-weather warnings

A published TRMNL recipe for MET Norway already exists: "Met.no Forecast" by
Argo Roots (MIT, all four layouts, five languages, 35 installs, source at
`github.com/argoroots/trmnl`). A plain forecast recipe from us would be a second
copy of a maintained plugin. We build our own anyway, and we make MET's
severe-weather warnings (MetAlerts) the thing it does that no existing recipe
does.

## Considered options

- **Contribute upstream.** The existing plugin is MIT and has open faults, for
  example a `optional: false` field that Chef flags. Rejected: it would publish
  nothing of our own.
- **Publish a plain forecast anyway.** Rejected: the obvious differentiators
  (all four layouts, latitude and longitude fields, feels-like temperature) are
  already taken.
- **Keep it private.** Rejected: the goal is a plugin anyone can install.

## Consequences

- **MetAlerts covers Norway only.** The forecast part serves the world; the
  warning part serves Norway. The plugin must behave well for a location with
  no warnings, and for a location outside Norway.
- The plugin polls **two** MET products, not one. Each needs its own interval
  and its own identifying User-Agent.
- Warning severity is a colour in MET's source icons (red, orange, yellow). The
  screen is 1 bit, so severity must be carried by something that is not colour.
