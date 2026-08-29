# TRMNL MET Norway forecast

A weather plugin for the TRMNL e-ink display. It shows a local forecast from
MET Norway, and it is published for anyone to install.

## Language

### TRMNL

**Recipe**:
A plugin that lives inside TRMNL, made of a Liquid template plus HTTP polling,
with no server of the author. This is the shape of our plugin.
_Avoid_: integration, app

**Private Plugin**:
A Recipe before publication, visible only in the account that made it.
_Avoid_: draft recipe, local plugin

**Recipe Master**:
The source instance of a published Recipe. An edit to it reaches every
installed copy, and its own generated screen becomes the public preview.
_Avoid_: original, template, parent

**Third Party plugin**:
The other publication shape, backed by the author's own server and OAuth. We do
not use it.
_Avoid_: public plugin, marketplace plugin

**Chef**:
TRMNL's automated linter. It checks a Recipe before a person reviews it.

**Polling**:
The strategy where TRMNL's servers fetch a URL for us at a fixed interval. The
other strategy is a webhook, which we do not use.

**Layout**:
One of the four screen shapes a Recipe can fill: full, half-vertical,
half-horizontal, quadrant.
_Avoid_: view, size, format

**Mashup**:
One screen shared by more than one plugin. A Recipe needs a half or quadrant
layout to take part.

### MET Norway

**Locationforecast**:
The MET Norway product that gives a forecast for one point. It has a `/compact`
and a `/complete` variant.
_Avoid_: the API, met.no, Yr

**MetAlerts**:
The MET Norway product that gives severe-weather warnings. It covers Norway
only, so most installs of the plugin never receive one.
_Avoid_: alerts API, warnings API

**Warning**:
One severe-weather notice from MetAlerts, for one Norwegian area, carrying a
severity of red, orange or yellow.
_Avoid_: alert, alarm, notice

**symbol_code**:
MET's identifier for one weather symbol, for example `partlycloudy_day`. It
carries its own `_day`, `_night` and `_polartwilight` variant, so the plugin
never calculates sunrise.
_Avoid_: icon name, weather code, condition

**Time step**:
One entry of the forecast series. The series is hourly at first, then
six-hourly.
_Avoid_: timeseries entry, data point, forecast hour
