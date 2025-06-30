# AutoHunter
AutoHunter is a collection of custom Splunk commands and dashboards that serve to quickly ingest and extract IOC data from online articles. The included dashboard also allows you to immediately hunt the collected IOCs in various telemetry sources.

While using the main dashboard and/or the `webreader` command, you will notice the `mode` parameter. This determines how the remote content is retrieved. Normal mode uses Python's basic requests to retrieve the content and BeautifulSoup for text parsing. This is fast and ideal for basic parsing and will run right on the Splunk instance natively.

If an article's content is not being retrieved in normal mode, there may be a Javascript requirement or some kind of webscraping protection. If this is the case you'll need to use advanced mode. Advanced mode passes the responsibility of retrieving the web content off to a Selenium API. You'll need to setup something like Selenium Grid for this. When I say setup I mean pull down and start a Selenium docker container and expose the API port, it takes all of 60 seconds.

> Example Selenium container: docker run -d -p 4444:4444 selenium/standalone-chrome

> API for above container would be http://\<IP\>:4444/wd/hub

While this product was initially meant to be a way to manually ingest OSINT and hunt, I will be adding features that will pair with my RSS Streamer app (https://github.com/ben3636/splunk-rss) to iterate over RSS links in the index and automatically read and hunt the IOCs contained in each. A hunt report will then be delivered via scheduled search for the results.

## Commands // Usage

Normal Mode
> | webreader url="\<URL\>" mode=normal

Advanced Mode
> | webreader url="\<URL\>" mode=advanced selenium_server="http://\<IP\>:4444/wd/hub"

Elements of the main dashboard use the `askllama` command to summarize articles using a local LLM, you can install the command here: https://github.com/ben3636/splunk-llama

## Demo
https://drive.google.com/file/d/12mVLEtFxBs5_Jraz8Xf_aMuzLSFCtbOJ/view?usp=sharing
