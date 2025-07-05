# AutoHunter
![Alt text](Logo.png)

AutoHunter is a collection of custom Splunk commands and dashboards that serve to quickly ingest and extract IOC data from online articles. Once extracted, the IOCs are then instantly hunted in various telemetry sources.

> NOTE: Elements of the main dashboard use the `askllama` command to summarize articles using a local LLM, you can install the command here if you'd like to use this functionality: https://github.com/ben3636/splunk-llama

## Inital Setup & Basic Usage
For basic autohunting by pasting a URL into the main dashboard (`AutoHunter`), there is no setup required. This should be ready to go out of the box. Notice there is a `mode` option that defaults to `normal`. This should be fine in most cases but if you find that your URL isn't returning results you will need to switch to `advanced` mode which requires a Selenium API to be specified. The difference between `normal` and `advanced` mode is covered in detail in the `Webreader Command Usage` section below.

The second dashboard in the app (`AutoHunts`) is the hub for `Autonomous Hunting`. Feel free to reread that sentence again, it gives me chills every time. This functionality is made possible by AutoHunter's sister app, `RSS Streamer`. RSS Streamer (https://github.com/ben3636/splunk-rss) allows you to specify RSS feeds in a lookup and Splunk will automatically ingest the feed data to a stash index. AutoHunter can then collect all the links to the articles from this data, start reading the articles, extracting the IOCs, and performing hunts autonomously (the crowd gasps). Neat trick right? AutoHunter is a beast on her own but when she teams up with her sister they're a force to be reckoned with. 

AutoHunter has scheduled searches running out of the box looking for RSS data so all you need to do is install RSS Streamer and manually run its "Pull RSS Feeds" scheduled search after you've reviewed the feeds lookup and made any desired changes. That "Pull RSS Feeds" scheduled search is disabled by default but if the feeds look good feel free to kick that on so it pulls fresh OSINT in on its own. Once RSS data is in pop back over to AutoHunter's `AutoHunts` dashboards and within an hour you should see it acknowledging the RSS articles and queuing them up for parsing. 

AutoHunter has a few scheduled searches that make the magic happen including:

1. Checking the RSS stash index for new articles
2. Performing a `basic` pass at the source attempting to read and extract any IOCs
3. Performing a secondary fallback pass with `advanced` mode (Selenium) for any sources that `basic` mode was not able to handle
4. Performing housekeeping on the IOC lookup to remove duplicate entries
5. Hunting the IOCs stored in the lookup in various datasources (network, web, dns, etc)
6. Notifying you via webhook when a new article is processed and if any IOCs where found in it (optional, I send this to a muted Discord channel I can pop into from time to time)
7. Alerting you of any actual IOC findings within your environment. This will send an alert to the webhook or other alert action you configure with basic information about the IOC and impacted asset.

## Demo
https://drive.google.com/file/d/12mVLEtFxBs5_Jraz8Xf_aMuzLSFCtbOJ/view?usp=sharing

## Technical Stuff - What Happens Behind the Curtain
The app's core ability to read web content is driven by the `webreader` command. This takes a few arguments such as URL and mode and grabs the contents of the specified article so the IOC extraction regex monster can sniff out those sweet sweet IOCs.

While using the main dashboard and/or the `webreader` command directly, you will notice the `mode` parameter. This determines how the remote content is retrieved. Normal mode uses Python's basic requests to retrieve the content and BeautifulSoup for text parsing. This is fast and ideal for basic parsing and will run right on the Splunk instance natively.

If an article's content is not being retrieved in normal mode, there may be a Javascript requirement or some kind of webscraping protection. If this is the case you'll need to use advanced mode. Advanced mode passes the responsibility of retrieving the web content off to a Selenium API. You'll need to setup something like Selenium Grid for this. When I say setup I mean pull down and start a Selenium docker container and expose the API port, it takes all of 60 seconds.

> Example Selenium container: docker run -d -p 4444:4444 --shm-size=2g selenium/standalone-chrome

> API for above container would be http://\<IP\>:4444/wd/hub

## Webreader Command Usage
The `webreader` command is the heart of the product. It takes a `url` argument and retrieves the contents of the article. Specifically how this is done depends on whether you specify `mode=normal` or `mode=advanced`. Remember, if you're using advanced mode you'll need to specify `selenium_server` so `webreader` knows where to contact Selenium.

Below are some examples of how you can use `webreader` to ingest the content of online articles directly into Splunk:

Normal Mode
> | webreader url="\<URL\>" mode=normal

Advanced Mode
> | webreader url="\<URL\>" mode=advanced selenium_server="http://\<IP\>:4444/wd/hub"

`webreader` in its basic form is a generating command because it will be the first command in your search. Sometimes you may encounter instances where you already have a search with results containing a link in each event. If this is the case, an alternate version of `webreader` is available called `streamingwebreader`. `streamingwebreader` takes the same arguments as `webreader` but it is a streaming command and can be used during an existing search to use links present in event fields. Instead of passing an actual URL in the `url` argument, simply use the field name that contains the links in your events:

> | makeresults | eval link="https://google[.]com" | streamingwebreader url=link mode=normal

This functionality is key for the integration with AutoHunter's sister app, RSS Streamer so the app can autonomously ingest fresh OSINT and action it without any human interaction.

## Autonomous Operation - OSINT's Journey from Start to Finish
AutoHunter can automatically read, extract IOCs from, and hunt articles from RSS data from RSS Streamer (https://github.com/ben3636/splunk-rss). This process leverages the `streamingwebreader` command and the scheduled searches included in the app. I've listed out the general flow by which this happens below:

1. RSS data is ingested from RSS Streamer, landing in a stash index
2. AutoHunter parses this index and builds a lookup of article titles and links (`autohunter_article_log.csv`)
3. AutoHunter performs a first pass analysis on the links using `normal` mode and attempts to extract IOCs (regex is stored in the `extract_iocs` macro)
4. If the article was unable to be parsed with `normal` mode the article will be flagged as `TORETRY` in `autohunter_ioc_log.csv`
5. A second scheduled search passes over `autohunter_ioc_log.csv` for articles that were unable to be parsed with `normal` mode (marked as `TORETRY`)
6. These remaining articles are parsed using `advanced` mode and IOCs are extracted. If AutoHunter was able to extract IOCs from the article at this point, the article is marked as `DONE` in `autohunter_ioc_log.csv`. If there are still no yield from extraction, it is marked with `NOIOCS`.
7. Extracted IOCs (stored in `autohunter_ioc_log.csv` are hunted by data type with the remaining scheduled searches and any findings are sent to you via webhook or whichever custom alert action you've setup.
