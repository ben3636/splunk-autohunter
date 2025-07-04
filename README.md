# AutoHunter - Intelligence, Actioned
AutoHunter is a collection of custom Splunk commands and dashboards that serve to quickly ingest and extract IOC data from online articles. Once extracted, the IOCs are then instantly hunted in various telemetry sources.

> NOTE: Elements of the main dashboard use the `askllama` command to summarize articles using a local LLM, you can install the command here if you'd like to use this functionality: https://github.com/ben3636/splunk-llama

## Demo
https://drive.google.com/file/d/12mVLEtFxBs5_Jraz8Xf_aMuzLSFCtbOJ/view?usp=sharing

## Modes of Operation
While using the main dashboard and/or the `webreader` command, you will notice the `mode` parameter. This determines how the remote content is retrieved. Normal mode uses Python's basic requests to retrieve the content and BeautifulSoup for text parsing. This is fast and ideal for basic parsing and will run right on the Splunk instance natively.

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

This functionality is key for the integration with AutoHunter's sister app, RSS Streamer. RSS Streamer (https://github.com/ben3636/splunk-rss) allows you to specify RSS feeds in a lookup and Splunk will automatically ingest the feed data to a stash index. AutoHunter can then collect all the links to the articles from this data, start reading the articles, extracting the IOCs, and performing hunts autonomously (the crowd gasps). Neat trick right? AutoHunter is a beast on her own but when she teams up with her sister they're a force to be reckoned with. These autonomous hunt reports go out as part of the scheduled searches, more on this in the next section.

## Autonomous Operation
AutoHunter can automatically read, extract IOCs from, and hunt articles from RSS data from RSS Streamer (https://github.com/ben3636/splunk-rss). This process leverages the `streamingwebreader` command and the scheduled searches included in the app:

1. RSS data is ingested from RSS Streamer, landing in a stash index
2. AutoHunter parses this index and build a lookup of article titles and links
3. AutoHunter performs a first pass analysis on the links using normal mode and attempts to extract IOCs
4. If the article was unable to be parsed with normal mode the article will be flagged as `TORETRY` in `autohunter_ioc_log.csv`
5. A second scheduled search passes over `autohunter_ioc_log.csv` for articles that were unable to be parsed with normal mode (marked as `TORETRY`)
6. These remaining articles are parsed using advanced mode and IOCs are extracted
7. Extracted IOCs (stored in `autohunter_ioc_log.csv` are hunted by data type with the remaining scheduled searches with results notifying you by your chosen alert action.
