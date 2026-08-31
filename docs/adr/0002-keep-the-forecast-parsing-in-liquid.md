# Keep the forecast parsing in Liquid, and keep every payload read in one file

TRMNL can run a **transform** on a polled response before Liquid sees it. This
plugin does not use one. The parsing stays in Liquid, and `src/shared.liquid`
owns every read of the MET payload.

## What opened the question, and what was wrong with it

The question was opened on a measurement that said the forecast payload was
89,238 bytes at 08:33 and 93,211 bytes at 13:34 on 2026-08-30, against TRMNL's
100 kB cap. Two samples were read as a trend. They were not one.

Measured properly on 2026-08-30, across twelve points:

| Points | Payload | Time steps |
|---|---|---|
| Oslo, Helsinki, Tromsø | 88,521 – 89,551 bytes | 84 |
| London, New York, Sydney, Singapore, Ushuaia, Everest, Reykjavík, Moscow, Greenland Sea | 62,500 – 63,665 bytes | 91 |

There are two populations, not a trend. MET's high-resolution Nordic model
returns about 89 kB with extra fields per step; the rest of the world gets about
63 kB with more steps and fewer fields. Three consecutive fetches of Helsinki
returned a byte-identical 89,118 with the same `Last-Modified`. The 93,211 was a
different model run of the same point.

**The two populations are real. "Stable" was too strong**, and this document
said it on one day of data — the same mistake, in the other direction, as the
growth claim it was written to correct.

Measured again on 2026-08-31, at four Nordic points:

| Point | 2026-08-30 | 2026-08-31 |
|---|---|---|
| Oslo | 89,551 | 92,273 |
| Helsinki | 89,118 | 91,966 |
| Tromsø | 88,521 | 90,947 |
| Trondheim | — | 91,937 |

This is still not evidence of growth: two rounds is not a trend, and Helsinki
alone ranged 89,118 to 93,211 inside one day. What it does show is that the
payload **moves by about 4 kB between model runs**, so no single sample
describes it.

**The number that matters is the observed peak: 93,211 bytes**, Helsinki,
2026-08-30. That is 93.2 per cent of the cap, so the margin is **under 7 per
cent**, not the 10 this document first claimed.

## Decision

**No transform.** The day grouping in `src/shared.liquid` is a single-pass loop
with a string accumulator, because Liquid cannot append to an array. It is about
45 lines, it is commented, `trmnlp lint` passes, and its output matches a `jq`
derivation of the same payload exactly. TRMNL's advice to prefer a transform
over complex Liquid is written for somebody about to write that loop, not for
somebody who has already written and verified it.

**Every read of the MET payload lives in `src/shared.liquid`.** A layout names
`now_*`, `warn_*`, `hour_list` and `day_list`. No layout names a MET field. The
derived lists are packed strings, `"a|b|c"` joined with `;`, because that is the
only array-of-records Liquid offers.

## Considered options

- **Add a transform now.** Rejected. It buys three lines in place of forty-five,
  and costs a second language in the repo, a second thing for Chef and the human
  reviewer, and a 5-second serverless timeout whose failure is a blank screen.
- **Drop to `/compact`.** Rejected. ADR-0001's endpoint choice (#14) bought
  feels-like, gusts and the 6-hourly `air_temperature_max`/`min` that gives the
  daily band its grouping. Nothing currently justifies giving those up.

## Consequences

- The 100 kB cap becomes a **watch item, not a driver**. Watch the **peak, not a
  sample**: the highest yet seen is **93,211 bytes**, MET Norway `/complete`,
  Helsinki, 2026-08-30, leaving under 7 per cent. Take several Nordic points on
  several days before publication and after any MET model change, and compare
  the highest reading, because one reading understates the payload by up to
  4 kB.
  The Nordic area is where the margin is thinnest and also where the warnings
  are, so a MET field addition would hit our own users first.
- Because the layouts read only derived values, adopting a transform later is a
  change to one file, not to four. This is the reason the seam matters more than
  the decision it defers.
- The units are written where the payload is read, not in a layout:
  `probability_of_precipitation` exists in the Nordic area only, so
  `shared.liquid` chooses `"%"` or `" mm"` and carries the spacing. That line is
  where the language table of #16 will plug in.

## Open question, deliberately not closed

**Where the 100 kB cap is measured is unconfirmed.** The only source is one
sentence — "Private plugins enforce a maximum 100 kilobyte blob of data from
external resources" — which reads as the *incoming* response. But it appears in
the article announcing the transform runtime as the answer to large payloads,
which only makes sense if the cap is measured *after* the transform. The
developer docs and the private-plugins page say nothing. It is also unknown
whether the cap counts each polling URL or the combined `IDX_*` payload, and
whether it counts raw or compressed bytes; MET gzips 89,551 bytes to 7,032, a
ratio of nearly 13 to 1. If the cap counts what arrives on the wire, none of
this matters at all.

If the cap is measured on the incoming response, **a transform cannot help with
it at all**, and the only lever is `/compact`. This decision does not depend on
the answer, which is why it was not worth a detour to find. Ask TRMNL at
publication review:

> For a polling private plugin, is the 100 kB limit measured on the response we
> fetch, or on the merge data after a transform runs? Is it per polling URL or
> for the combined IDX_* payload, and does it count raw or compressed bytes?
