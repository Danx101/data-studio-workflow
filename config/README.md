# Dashboard config layer

Per-client config describing the conversion taxonomy, filters, calculated
fields, blends and metrics behind a Looker Studio lead-gen report. This is
the piece that's actually slow to redo by hand for every new customer; the
report *layout* is handled separately by duplicating a Looker Studio
template report and pointing the copy at the new client's data source.

## Files

- `schema/dashboard-config.schema.json` — JSON Schema for the config format.
- `clients/peneder.example.yaml` — worked example, reconstructed from a live
  walkthrough of the existing Peneder GADs report. A few entries are
  inferred rather than confirmed (flagged inline) — verify before reuse.
- `clients/_new-client.template.yaml` — blank starter for onboarding.

## Where conversion names come from

Pull the raw conversion action list from **Google Ads → Tools & Settings →
Conversions** for the client's account, not from Looker Studio's
"Conversion" dimension picker.

Reasons:
- Looker Studio's dimension only lists values that had ≥1 conversion in the
  currently selected date range — new or low-volume conversion actions can
  be silently missing.
- It mixes in GA4-linked/auto-imported noise (`Local actions - Website
  visits`, per-URL GA4 conversions) that were never meant to count as leads.
- The Google Ads Conversions page carries metadata Looker Studio's dimension
  doesn't: **category** (Submit lead form, Phone call leads, Contact, ...),
  **status** (enabled/paused/removed), and the **"Include in Conversions"**
  flag — all useful signal for `lead_categories` grouping.
- It's the same resource (`conversion_action`) the Google Ads API exposes,
  so building `conversion_actions` around this source now means the later
  automation agent can pull it programmatically with no rework.

For now: open that settings page, copy the table into
`conversion_actions` by hand. Flag anything that isn't a real lead signal
with `include_in_conversions: false` and a `source` other than the default.

## Onboarding a new client

1. Copy `clients/_new-client.template.yaml` to `clients/<slug>.yaml`.
2. Fill `conversion_actions` from the client's Google Ads Conversions page.
3. Group them into `lead_categories` (always include a `catch_all` category
   named `other` — this surfaces anything unclassified instead of silently
   dropping it, same role "Sonstige" plays in the Peneder report).
4. Derive `filters` and `calculated_fields` from the categories — see
   `peneder.example.yaml` for the generation pattern: one master
   include-regex across all non-`other` categories, one include filter per
   category, and exclude filters for any `Sonstige`/noise buckets.
5. Fill `blends` for whatever grains the target `report_template` needs
   (per-campaign, per-date, per-region, totals, ...) — each blend is the
   same source joined to itself: one leg unfiltered, one leg filtered by
   the lead filter, joined on the shared dimension.
6. Fill `metrics` — ratios computed across a blend's two legs (cost per
   lead, conversion rate on leads).
7. Validate the file against `schema/dashboard-config.schema.json`.

The automation agent (later) will take this config plus a freshly
duplicated template report and mechanically create the matching
filters/calculated fields/blends through Looker Studio's dialogs.

## Data hygiene

The live Peneder report had leftover, unused data sources (4 of 5 Google
Ads connectors wired up but 0 charts using them) and two orphaned blends
(0 charts using them either) — remnants of earlier iterations that were
never cleaned up. Don't reproduce that pattern: a config file (and the
report it drives) should only reference sources/blends actually used by a
chart. `peneder.example.yaml` deliberately leaves the two orphaned blends
out.
