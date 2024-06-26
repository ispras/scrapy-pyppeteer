# Scrapy-puppeteer-client
This package aims to manage Chrome browser with [Puppeteer](https://github.com/GoogleChrome/puppeteer) from [Scrapy](https://github.com/scrapy/scrapy/) spiders. 
This allows to scrape sites that require JS to function properly and to make the scraper more similar to humans.
It is a client library for [scrapy-puppeteer-service](https://github.com/ispras/scrapy-puppeteer-service).

## ⚠️ This repository is under development.

This project is under development. Use it at your own risk.

## Installation

Using pip (master branch):
```shell script
$ pip install scrapy-puppeteer-client
```

## Configuration

You should have [scrapy-puppeteer-service](https://github.com/ispras/scrapy-puppeteer-service) started.
Then add its URL to `settings.py` and enable puppeteer downloader middleware:
```python
DOWNLOADER_MIDDLEWARES = {
    'scrapypuppeteer.middleware.PuppeteerServiceDownloaderMiddleware': 1042
}

PUPPETEER_SERVICE_URL = 'http://localhost:3000'
``` 

## Basic usage

Use `scrapypuppeteer.PuppeteerRequest` instead of `scrapy.Request` to render URLs with Puppeteer:
```python
import scrapy
from scrapypuppeteer import PuppeteerRequest

class MySpider(scrapy.Spider):
    ...
    def start_requests(self):
        yield PuppeteerRequest('https://exapmle.com', callback=self.parse)
    
    def parse(self, response):
        links = response.css(...)
        ...
```

## Advanced usage

`PuppeteerRequest`'s first argument is a browser action.
Available actions are defined in `scrapypuppeteer.actions` module as subclasses of `PuppeteerServiceAction`.
Passing a URL into request is a shortcut for `GoTo(url)` action. 

Here is the list of available actions:
- `GoTo(url, options)` - navigate to URL 
- `GoForward(options)` - navigate forward in history
- `GoBack(options)` - navigate back in history
- `Click(selector, click_options, wait_options)` - click on element on page
- `Scroll(selector, wait_options)` - scroll page
- `Screenshot(options)` - take screenshot
- `RecaptchaSolver(solve_recaptcha)` - find or solve recaptcha on page
- `CustomJsAction(js_function)` - evaluate JS function on page
- `CloseContext(contexts)` - close active contexts in the service

Available options essentially mirror [service](https://github.com/ispras/scrapy-puppeteer-service) method parameters, which in turn mirror puppeteer API functions to some extent.
See `scrapypuppeteer.actions` module for details.

You may pass `close_page=False` option to a request to retain browser tab and its state after request's completion.
Then use `response.follow` to continue interacting with the same tab:

```python
import scrapy
from scrapypuppeteer import PuppeteerRequest
from scrapypuppeteer.actions import Click

class MySpider(scrapy.Spider):
    ...
    def start_requests(self):
        yield PuppeteerRequest('https://exapmle.com', close_page=False, callback=self.parse)
    
    def parse(self, response):
        ...
        # parse and yield some items
        ...
        next_page_selector = 'button.next-page-or-smth'
        if response.css(next_page_selector ):
            yield response.follow(Click(next_page_selector ,
                                        wait_options={'selectorOrTimeout': 3000}), # wait 3 seconds
                                  close_page=False,
                                  callback=self.parse)
```

On your first request service will create new incognito browser context and new page in it.
Their ids will be in returned in response object as `context_id` and `page_id` attributes.
Following such response means passing context and page ids to next request.
You also may specify requests context and page ids directly.

Once your spider is closed, middleware will take care of closing all used browser contexts.

One may customize which `PuppeteerRequest`'s headers will be sent to remote website by the service 
via `include_headers` attribute in request or globally with `PUPPETEER_INCLUDE_HEADERS` setting. 
Available values are True (all headers), False (no headers) or list of header names.
By default, only cookies are sent.

You would also like to send meta with your request. By default, you are not allowed to do this
in order to sustain backward compatibility. You can change this behaviour by setting `PUPPETEER_INCLUDE_META` to True.

## Automatic recaptcha solving

Enable PuppeteerRecaptchaDownloaderMiddleware to automatically solve recaptcha during scraping. We do not recommend
to use RecaptchaSolver action when the middleware works.

```Python
DOWNLOADER_MIDDLEWARES = {
    'scrapypuppeteer.middleware.PuppeteerRecaptchaDownloaderMiddleware': 1041,
    'scrapypuppeteer.middleware.PuppeteerServiceDownloaderMiddleware': 1042
}
```
Note that the number of RecaptchaMiddleware has to be lower than ServiceMiddleware's.
You must provide some settings to use the middleware:
```Python
PUPPETEER_INCLUDE_META = True  # Essential to send meta

RECAPTCHA_ACTIVATION = True  # Enables the middleware
RECAPTCHA_SOLVING = False  # Automatic recaptcha solving
RECAPTCHA_SUBMIT_SELECTORS = {  # Selectors for "submit recaptcha" button
    'www.google.com/recaptcha/api2/demo': '',  # No selectors needed
}
```
If you set RECAPTCHA_SOLVING to False the middleware will try to find captcha
and will notify you about number of found captchas on the page.

If you don't want the middleware to work on specific request you may provide special meta key: `'dont_recaptcha': True`.
In this case RecaptchaMiddleware will just skip the request.

## TODO

- [x] skeleton that could handle goto, click, scroll, and actions
- [ ] headers and cookies management
- [ ] proxy support for puppeteer
- [ ] error handling for requests
- [ ] har support
