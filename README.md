<p align="center"><a href="https://apify.com/DevilScrapes/meetup-events-scraper"><img src="assets/banner.svg" alt="Meetup Events Scraper — Python example" width="100%"></a></p>

# Meetup Scraper in Python: keyword and city searches into pandas

<p align="center">
<a href="https://apify.com/DevilScrapes/meetup-events-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="Python" src="https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/meetup-scraper-python/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/meetup-scraper-python/actions/workflows/ci.yml/badge.svg"></a>
</p>

Meetup.com hosts a large number of recurring community groups: language user groups, data science circles, cloud and AI study groups. This repository is a short, tested Python client for the **[Meetup Events Scraper](https://apify.com/DevilScrapes/meetup-events-scraper)** on Apify. You give it search terms and, if you like, a city; it returns a DataFrame-ready list where each record is one event with its start time and offset, whether it is in person or online, how many members said they are going, the capacity limit, any fee, the venue and the group running it. The searching happens in Apify's cloud, and the script saves `results.json` and `results.csv` locally for whatever analysis comes next.

> **Actor:** [Meetup Events Scraper](https://apify.com/DevilScrapes/meetup-events-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in JavaScript:** [meetup-scraper-javascript](https://github.com/devilscrapes/meetup-scraper-javascript)

## 🔥 What you get

- **One search per keyword**: every term in `keywords` runs its own Meetup find query, scoped to `location` when you give one and global when you leave it empty.
- **In-person or online on request**: `eventType` keeps only `physical` or only `online` events, and every row carries both `event_type` and a boolean `is_online`.
- **Turnout and capacity side by side**: `going_count` is the public RSVP total and `max_tickets` the host's cap when one exists, which together give a fill rate.
- **Fees made explicit**: `is_free` is true when the event has no fee settings; paid events add `fee_amount` as a number and `fee_currency`.
- **The organizer behind each event**: `group_name` plus `group_url`, a direct link to the hosting group's Meetup page.
- **Leaner rows on demand**: switch `includeDescription` off and the long event text is returned as `null`.
- **Resilient per keyword**: throttled or failing requests are retried up to five times with growing pauses, and a keyword that still fails is logged and skipped without stopping the others.

## 🎯 Use cases

### Find the groups that actually draw a crowd

Search a topic in your city, group the rows by `group_url` and sum `going_count`. The handful of groups that keep showing up with large totals are the ones worth partnering with, speaking at or sponsoring, and the numbers make the case to your manager for you.

### Measure how full events get

Where `max_tickets` is set, `going_count / max_tickets` is a fill rate. Average it per group or per topic in a notebook and you can tell a niche that sells out from one where half the seats stay empty.

### Track the online versus in-person mix

Run the same keywords weekly from cron and store each CSV with the date. Plotting the share of `is_online` rows over time shows whether a community has moved back to venues or settled into video calls.

### Compare cities before you expand

Repeat one keyword list across several `location` values and compare event counts, free versus paid share and typical turnout. It is a quick, quantified read on which city already has an audience for your topic.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/meetup-scraper-python.git
cd meetup-scraper-python
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export APIFY_TOKEN=your_token_here
python main.py                      # uses input.json
python main.py my-input.json --out data --max-charge 2
```

The script starts the [Meetup Events Scraper](https://apify.com/DevilScrapes/meetup-events-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

Stripped of the command-line handling, this is everything the script does: create the client, run the Actor to completion and iterate over the rows it stored.

```python
from apify_client import ApifyClient

client = ApifyClient("YOUR_APIFY_TOKEN")
run = client.actor("DevilScrapes/meetup-events-scraper").call(run_input={'keywords': ['ai'], 'location': 'New York, NY', 'eventType': 'any', 'maxEventsPerKeyword': 10, 'includeDescription': True, 'proxyConfiguration': {'useApifyProxy': True}})
for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item)
```

## 📥 Input

`main.py` hands `input.json` to the Actor without edits. The included input searches `ai` around New York and keeps ten events, enough to inspect real rows for a few cents. Meetup returns about one page of matches per search, so breadth comes from more keywords (`python`, `data science`, `startup`) or more cities, not from a higher `maxEventsPerKeyword`.

```json
{
  "keywords": [
    "ai"
  ],
  "location": "New York, NY",
  "eventType": "any",
  "maxEventsPerKeyword": 10,
  "includeDescription": true,
  "proxyConfiguration": {
    "useApifyProxy": true
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `keywords` | array | yes | `["ai"]` | — |
| `location` | string | no | `"New York, NY"` | — |
| `eventType` | string | no | `"any"` | `any`, `physical`, `online` |
| `maxEventsPerKeyword` | integer | no | `50` | — |
| `includeDescription` | boolean | no | `true` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": true}` | — |

## 📤 Output

Each record describes one event. `date_time` keeps Meetup's local offset (for example `-04:00`), so parse it with `utc=True` when comparing cities. Coordinates are often `null` because the search results rarely include them; the venue name, street, city and state are the reliable location fields.

| Field | Type | Always present |
|---|---|---|
| `search_keyword` | string | yes |
| `event_id` | string | yes |
| `title` | string | yes |
| `event_url` | string | yes |
| `date_time` | string | yes |
| `event_type` | string | yes |
| `is_online` | boolean | no |
| `going_count` | integer | no |
| `max_tickets` | integer or null | no |
| `is_free` | boolean | no |
| `fee_amount` | number or null | no |
| `fee_currency` | string or null | no |
| `venue_name` | string or null | no |
| `venue_address` | string or null | no |
| `venue_city` | string or null | no |
| `venue_state` | string or null | no |
| `venue_country` | string or null | no |
| `latitude` | number or null | no |
| `longitude` | number or null | no |
| `group_name` | string or null | no |
| `group_url` | string or null | no |
| `description` | string or null | no |
| `scraped_at` | string | no |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| Result emitted | $0.004 |

Prices as configured on 2026-08-20. The [Actor page](https://apify.com/DevilScrapes/meetup-events-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in Python

The pandas snippet below (`pip install pandas`) removes events that matched more than one keyword, computes a fill rate, and prints a leaderboard of hosting groups by total RSVPs.

```python
import json

import pandas as pd

events = pd.DataFrame(json.load(open("results.json", encoding="utf-8")))
events = events.drop_duplicates(subset="event_id")
events["fill_rate"] = (events["going_count"] / events["max_tickets"]).where(events["max_tickets"] > 0)

groups = (
    events.groupby(["group_name", "group_url"], dropna=False)
    .agg(
        events=("event_id", "count"),
        total_going=("going_count", "sum"),
        avg_fill=("fill_rate", "mean"),
        online_share=("is_online", "mean"),
        paid_events=("is_free", lambda s: int((~s).sum())),
    )
    .sort_values("total_going", ascending=False)
)
print(groups.head(15).round(2).to_string())
groups.to_csv("groups_leaderboard.csv")
```

## ❓ FAQ

### Do I need a Meetup Pro account or an OAuth app?

No. The Actor reads the same public search results a logged-out visitor sees, so the only credential in this repository is your Apify token.

### Why do I get about 27 events even with a higher limit?

Each keyword fetches one page of Meetup search results, and that page holds roughly 27 events. The limit can only trim that page. For a bigger sample, add related keywords or extra cities and dedupe on `event_id`.

### Can I get the list of attendees?

No. Only the aggregate `going_count` is collected; member names and profiles are never read. That keeps the dataset free of personal data, which matters if you store it next to a CRM.

### Why are some rows filtered out when I pick `physical`?

The type filter runs after the search page is read, so a search where most matches are online yields fewer in-person rows. Rows that are filtered out are not written and not charged.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/meetup-events-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [Meetup Events Scraper on Apify Store](https://apify.com/DevilScrapes/meetup-events-scraper)
- [JavaScript version of this example](https://github.com/devilscrapes/meetup-scraper-javascript)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [Meetup Group Scraper](https://apify.com/DevilScrapes/meetup-group-scraper) — start from specific Meetup groups and pull their upcoming and past events.
- [Lu.ma Event Discovery Scraper](https://apify.com/DevilScrapes/luma-event-discovery) — cover the Luma side of the tech scene with host and ticket fields.
- [Eventbrite Events Scraper](https://apify.com/DevilScrapes/eventbrite-events-scraper) — add ticketed conferences and workshops listed on Eventbrite.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/meetup-events-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with Meetup (Meetup.com).
