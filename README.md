# AutoHunter
AutoHunter is a collection of custom Splunk commands and dashboards that serve to quickly ingest and extract IOC data from online articles. Once extracted, the IOCs are then instantly hunted in various telemetry sources.

> NOTE: Elements of the main dashboard use the `askllama` command to summarize articles using a local LLM, you can install the command here if you'd like to use this functionality: https://github.com/ben3636/splunk-llama

## Modes of Operation
While using the main dashboard and/or the `webreader` command, you will notice the `mode` parameter. This determines how the remote content is retrieved. Normal mode uses Python's basic requests to retrieve the content and BeautifulSoup for text parsing. This is fast and ideal for basic parsing and will run right on the Splunk instance natively.

If an article's content is not being retrieved in normal mode, there may be a Javascript requirement or some kind of webscraping protection. If this is the case you'll need to use advanced mode. Advanced mode passes the responsibility of retrieving the web content off to a Selenium API. You'll need to setup something like Selenium Grid for this. When I say setup I mean pull down and start a Selenium docker container and expose the API port, it takes all of 60 seconds.

> Example Selenium container: docker run -d -p 4444:4444 selenium/standalone-chrome

> API for above container would be http://\<IP\>:4444/wd/hub

## Webreader Command Usage
The `webreader` command is the heart of the product. It takes a url argument and retrieves the contents of the article. Specifically how this is done depends on whether you specific `mode=normal` or `mode=advanced`. Remember, if you're using advanced mode you'll need to specify `selenium_server` so `webreader` knows where to contact Selenium.

Below are some examples of how you can use `webreader` to ingest the content of online articles directly into Splunk:

Normal Mode
> | webreader url="\<URL\>" mode=normal

Advanced Mode
> | webreader url="\<URL\>" mode=advanced selenium_server="http://\<IP\>:4444/wd/hub"

## Demo
https://drive.google.com/file/d/12mVLEtFxBs5_Jraz8Xf_aMuzLSFCtbOJ/view?usp=sharing
