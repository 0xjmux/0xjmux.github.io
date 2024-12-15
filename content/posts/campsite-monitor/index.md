---
layout: post
tags: ["Blog", "Python", "self-hosted", "touching-grass"]
title: "Building a Campsite Availability notification service"
date: 2024-12-14T17:50:26-08:00
summary: "Making leaps in Touching Grass™ technology"
math: false
draft: false
---


I love going camping, but getting campsite reservations is becoming increasingly difficult. Nowadays, getting weekend slots is impossible for most campgrounds if you don't book right as the sites are released. However, sometimes you just want to go on a whim - and checking the recreation.gov site for availability manually can [be kind of annoying](./files/recreation_gov_ram_usage_firefox.png).

So, I built a system to send me notifications when new campsites become available.

{{< figure width="600" src="files/good_example_noti_output_with_pfp.png" >}}

I thought it would be a fun, quick little project - and it was fun to write, but did take longer than I was anticipating.

I found a couple projects online that scraped recreation.gov for availability, but they didn't have the features I wanted or a data model flexible enough for what I wanted to do, so I wrote my own from scratch.

## TL;DR: How does it work?

The app uses a `.toml` file to configure application behavior and runs as a background service on my homelab. Every x minutes it reaches out to recreation.gov and checks for availability based on the configuration specified in the toml file.

It then compares the current availability to the last request, and when it detects new availabilities it sends me a notification over discord.


## Application Details

```toml
title = "Example Campsites configuration"

[config]
# number of nights to stay for
num_nights = 2
weekends_only = true
# how often to check recreation.gov
check_frequency_minutes = 20

notify = 'discord'           # ['false', 'discord']
# webhook URI for discord server goes here
discord_webhook_url = ""

[campsites]

[campsites.pinnacles]       # human readable name
# campsite ID
id = 234015
# by recreation.gov unique campsite_id
# preferred_sites = [92930]
season = [['03-01', '05-30'], ['08-15', '10-15']]
avoid_sites = [92707, 92929]
```


Campgrounds are specified via a `[campsites.name]` toml heading and `id`; all other fields are optional. When `preferred_sites` is provided, we only look for those sites. Otherwise, we look at all sites, minus any avoided ones.

We fetch the availability data from recreation.gov for those campgrounds and date ranges, compare the current availability state to the last time we checked, and filter for differences.


{{< figure width="600" src="files/availability_output_example_may_2025.png" >}}

### Requirements and Featureset

I had a few main requirements when starting this project:
* Application behavior configured via `.toml` file
    * Easy to modify config, and can have separate config files for different use cases
* Minimize unnecessary traffic to the recreation.gov endpoint (be nice to their servers)
* Send alerts only on new availability
* Ability to specify both preferred campsites and ones to avoid.
    * This alerts for when my favorite campsites become available, and prevents certain sites from being included in whole-campground results (for example, sites where you have to pitch your tent on asphalt).
* Can specify number of consecutive nights to search for across a given interval, and only show availabilities on weekends

The final featureset satisfies all those requirements, plus a few additional features:
* Can specify 'seasons' for which you'd want to visit that campground
    * for example, avoid checking desert campgrounds for availability in the summer
* Dual operating modes:
  * CLI - show me all sites matching the specified criteria at the current moment
  * Daemonized - check for availabilities at a specified interval, and send alerts when things change
* Modular notification system
    * Currently using discord webhooks but easily extensible to other platforms
    * Output is in markdown format and includes clickable links for each campground/site.
* Automatic deployment onto my homelab using custom CI/CD pipeline and a self-hosted Github actions runner


### Implementation and lessons learned

I wrote this in Python since I was expecting it to be a really quick project. [As usual](https://x.com/Pinboard/status/761656824202276864), it was a bit more effort than anticipated.

Building the data model, ingesting data from recreation.gov and presenting it in a pretty format was a single, very enjoyable evening. Saving the availability, implementing filtering, reducing API calls, and making it reliable took **5x** that.

In a couple cases, this was self-inflicted by language choice: Python lets you do some really cool things, but the lack of a formal type system can be a real pain sometimes.

Take this `DateRange` class, which represents consecutive dates a site is available over:

```python
# represents single availability (start - end dates)
class DateRange:
    def __init__(self, start: date, end: date):
        # handle python being stupid with types
        if isinstance(start, datetime):
            start = start.date()
        if isinstance(end, datetime):
            end = end.date()

        self.start: date = start
        self.end: date = end

    def __eq__(self, other):
        return self.start == other.start and self.end == other.end
    #abriged
```
I lost 2 hours searching for a bug where the availability was fetched from recreation.gov correctly, and was stored correctly - but somewhere in the data being processed `date` objects ended up turning back into `datetime`. I was using the `in` operator to evaluate which date ranges were new, but `__eq__` fails when comparing `date` objects to `datetime` - and therefore, the filter would always return completely empty.

This was annoying and difficult to debug. The logic worked fine in the test harness, but when run from main the application seemed to work right up until the data disappeared, seemingly without cause.

Typehints are nice, but because they're not enforced at compile time (Type***hints***) you can end up with some really cursed logic to compensate. There's probably a better way of handling this, but this (albeit rather ugly) check is the quick fix I used.

### CI/CD
One of the things I tried with this project was the Continuous Deployment part of CI/CD. I've set up CI pipelines before, but since I usually work on embedded systems I hadn't written anything that would really benefit from CD in a while.

I wanted the app to run in a VM on my homelab. Docker is too much for something this simple, so I hacked together a way to auto-deploy updates to the daemonized app via a Github Actions self-hosted runner.

I couldn't find any information online on setting up CD for a self-hosted app when I was trying to do it (especially when using python virtual environments), so I *might* release another blog post on how I did that soon. It's definitely not something I'd run in prod, but works well enough for this use case.


{{< figure width="600" src="files/ci_cd_pipeline.png" >}}


### Conclusion

The app has been deployed on my homelab for about a week now, and has been working well so far. Worst case I have some telemetry set up in case things break.

I enjoyed building it, and it's already been helpful when searching for campsites this summer. Overall, I think it turned out pretty well.

---

This is my first post in a long while, not because I haven't been doing projects but rather that I haven't been writing them up. I have a backlog of finished projects, but since doing projects is more fun than writing about them I haven't gotten around to it. I'm working on the backlog now, so more posts hopefully coming soon.
<!--Since this project is something I can't really release the source for (it would neglect the reason I built the thing in the first place)-->
