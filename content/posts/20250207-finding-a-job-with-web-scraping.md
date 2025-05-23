---
title: "Finding a job with web scraping"
date: 2025-02-07
tags: ["web scraping", "scrapy", "playwright"]
slug: finding-a-job-with-web-scraping
params:
  mastodon_id: 114557079957825421
---

A few months ago I was looking for a new job. This means spending hours browsing LinkedIn and/or job boards and drowning into outdated ads or positions that weren't looking for someone with my skills.

After applying for some of these jobs, I realized that many companies around the world use recruitment platforms such as _greenhouse.io_ and _lever.co_ to receive applications. However, it is not possible through these platforms to search directly for the current openings. I was still relying on Linkedin ads or being lucky to see some of them being announced by a recruiter in some social network.

Most search engines allow us to search for keywords by limiting the results only within a specific domain. Usually adding `site:<domain.com>` together with our search terms will return only results in this domain. So if I search for the keywords related to the positions I want limiting my results to the recruitment platforms that I know would give me a list of job postings that probably I wouldn't find easily and more likely to be still accepting applicants.

If I also gather information inside each job posting, I can filter them so I can apply only in the ones that are looking for someone with my skills, experience and location.

I didn't want to do it manually, and given that I have a good experience with web scraping I decided to implement something that would help me to collect all this data.

## Tools and Libraries

### Scrapy

[Scrapy](https://scrapy.org/) is my default choice when performing web scraping projects. It is a simple and extensible framework that allows
me to start to gather data from websites very quickly but powerful
enough if I want to expand and build a more robust project.

### Playwright

Although it is possible to scrape JavaScript-heavy websites without requiring a real browser to render the content, I decided to use [Playwright](https://playwright.dev/python/), a tool for testing web applications that automate browser interactions but it also can be used in web scraping tasks. It also helps me to avoid beingh easily identified as a bot and being blocked to scrape the data.

### scrapy-playwright

[scrapy-playwright](https://github.com/scrapy-plugins/scrapy-playwright) is a plugin that makes easier to integrate Playwright and make it to adhere to the regular Scrapy workflow.

## Development

### Preparing our environment

Project is a common Python project developed inside a virtualenv.

```bash
mkdir job_search
cd job_search
python -m venv .venv
source .venv/bin/activate
pip install scrapy scrapy-playwright
```

Then create a Scrapy project and configure `scrapy-playwright` following the [installation](https://github.com/scrapy-plugins/scrapy-playwright?tab=readme-ov-file#installation) and [activation](https://github.com/scrapy-plugins/scrapy-playwright?tab=readme-ov-file#activation) instructions available in the extension documentation.

```bash
scrapy startproject postings
cd postings
```

```python
# postings/settings.py
# (...)

# Add the following to the existing file
DOWNLOAD_HANDLERS = {
    "http": "scrapy_playwright.handler.ScrapyPlaywrightDownloadHandler",
    "https": "scrapy_playwright.handler.ScrapyPlaywrightDownloadHandler",
}
```

### Gathering URL of job postings

In Scrapy terminology, a [Spider](https://docs.scrapy.org/en/latest/topics/spiders.html#topics-spiders) is a class which we define how to scrape and crawl information from a certain site. So let's create one **Spider** to perform a search in DuckDuckGo passing as search parameters: (1) the domain we want the results and (2) keywords we want to be inside the results (so we can filter only the job postings of the technology/position we are interested).

```python
# postings/spiders/duckduckgo.py
import itertools
from urllib.parse import urlparse
import scrapy
import w3lib.url

class DuckDuckGoSpider(scrapy.Spider):
    name = "duckduckgo"
    allowed_domains = ["duckduckgo.com"]

    def start_requests(self):
        ...

    def parse(self, response, keyword):
        ...
```

Inside `start_request` method, we schedule the initial requests sending search queries to DuckDuckGo passing domains and keywords we are interested.

Adding `meta={"playwright": True}` to the request ensures that a real browser (managed by playwright and scrapy-playwright) will be used. This will help us not to be easily identified as a bot (and blocked). Using a real browser (instead of sending plain requests only) will make our spider to be slower. Given that this spider will not be running frequently, we can accept the slowness of opening a real browser and perform the operations on it.

We also add `cb_kwargs` as a way to send [some metadata](https://docs.scrapy.org/en/latest/topics/request-response.html#scrapy.http.Request.cb_kwargs) to the request that will be used later to add extra information in the data returned.

```python
    def start_requests(self):
        keywords = ["python", "django", "flask", "fastapi"]
        domains = ["jobs.lever.co", "boards.greenhouse.io"]

        for domain, keyword in itertools.product(domains, keywords):
            yield scrapy.Request(
                f"https://duckduckgo.com/?q=site%3A{domain}+{keyword}",
                meta={"playwright": True},
                cb_kwargs={"keyword": keyword},
            )
```

Next step is to implement `parse` method that will get the response from DuckDuckGo and gather the URLs of the job postings.

```python
    def parse(self, response, keyword):
        for url in response.css("a::attr(href)").getall():
            parsed_url = urlparse(url)
            if parsed_url.netloc not in ["jobs.lever.co", "boards.greenhouse.io"]:
                # Ignore results not in the domains we are interested
                continue

            company_name = parsed_url.path.split("/")[1]

            yield {
                "company": company_name,
                "job_posting_url": url,
                "keyword": keyword,
            }
```

Running the Spider and exporting the results into a CSV file.

```bash
scrapy crawl duckduckgo -o job_postings.csv
```

And now we have an initial version of our spider and we are able to collect links of job postings that matches our search keywords.

### Handling page pagination

This initial run of the spider probably will return just a few results (~10). Performing this search manually, we will notice that what we got until now is just the _first page_ of the results. Looking at DuckDuckGo results page, we can notice in the bottom the following button that when clicked will load more results.

![DuckDuckGo - More Results button](ddg-more-results.png)

`scrapy-playwright` (and `playwright`) allows us to [perform actions on pages](https://github.com/scrapy-plugins/scrapy-playwright?tab=readme-ov-file#executing-actions-on-pages) such filling forms or clicking elements, so we can change our spider that when we perform the search, we will click in `More Results` button until we can't find it in the page anymore (indicating that we don't have more hidden results to be shown).

First we create our function that will get the page, and click the button as many times as needed. When the button is not visible anymore (we reach our maximum number of results), `PlaywrightTimeoutError` is raised stopping the interactions with the page and releasing it to be parsed.

```python
# postings/spiders/duckduckgo.py
from playwright.async_api import TimeoutError as PlaywrightTimeoutError


async def more_results(page):
    while True:
        try:
            await page.locator(selector="#more-results").click()
        except PlaywrightTimeoutError:
            break
    return page.url
```

Then we add `playwright_page_methods` with the list of [methods](https://github.com/scrapy-plugins/scrapy-playwright?tab=readme-ov-file#pagemethod-class) that we want to be called on the page.

```python
# postings/spiders/duckduckgo.py

class DuckDuckGoSpider(scrapy.Spider):
    # (...)

    def start_requests(self):
        keywords = ["python", "django", "flask", "fastapi"]
        domains = ["jobs.lever.co", "boards.greenhouse.io"]

        for domain, keyword in itertools.product(domains, keywords):
            yield scrapy.Request(
                f"https://duckduckgo.com/?q=site%3A{domain}+{keyword}",
                meta={
                    "playwright": True,
                    "playwright_page_methods": [
                        PageMethod(more_results),
                    ],
                },
                cb_kwargs={"keyword": keyword},
            )
```

Running the Spider again, we will get all the results.

### Removing duplicated results

After running the Spider, we will notice that some URLs are duplicated. One of the reasons is that more than one search keyword can return the same job posting result (we would expect a _Django_ job posting also having _Python_ keyword inside it).

To drop the duplicated values, we can create an [Item Pipeline](https://docs.scrapy.org/en/latest/topics/item-pipeline.html) that will check, for each job posting returned whether it was returned before or not.

An item pipeline is a simple class that implements `process_item` method that receives items returned by the Spider, perform some processing on the item and then return it or drop it. When enabled *all* items returned by `DuckDuckGoSpider` will pass through it.

A `seen_urls` set is defined, so we can check if that particular URL was already returned (and then we can skip it) or if it is a new URL.

```python
# postings/pipelines.py
from scrapy.exceptions import DropItem


class JobPostingDuplicatesPipeline:
    seen_urls = set()

    def process_item(self, item, spider):
        if item["job_posting_url"] in self.seen_urls:
            # We drop items which the URL was already returned before
            raise DropItem("Already returned")

        # Add the URL to the set when it was processed for the first time
        self.seen_urls.add(item["job_posting_url"])

        yield item
```

We need to enable this pipeline in our project.

```python
# postings/settings.py
# (...)

ITEM_PIPELINES = {
   "postings.pipelines.JobPostingDuplicatesPipeline": 300,
}
```

We can run again the Spider, exporting the results into a CSV file. This time, we will have only unique job posting URLs.

```bash
scrapy crawl duckduckgo -o job_postings.csv
```

### Possible improvements

This is the starting point of our job search. It was useful to me because it returned some companies that I have never heard about it, looking for professionals with the skills that I have. Certainly this narrowed more my options and helped me to apply to positions that made more sense to me.

A possible improvement would be create a specific spider for each recruitment platform to parse the content of the job postings. This would allow us to filter even more our data. For example, we can check for specific benefits or other keywords that would be useful for us to decide to apply or not to the job.

### Summary

Here we have the complete code:

```python
# postings/pipelines.py
import itertools
from urllib.parse import urlparse

import scrapy
import w3lib.url
from playwright.async_api import TimeoutError as PlaywrightTimeoutError
from scrapy_playwright.page import PageMethod


async def more_results(page):
    while True:
        try:
            await page.locator(selector="#more-results").click()
        except PlaywrightTimeoutError:
            break
    return page.url


class DuckDuckGoSpider(scrapy.Spider):

    def start_requests(self):
        keywords = ["python", "django", "flask", "fastapi"]
        domains = ["jobs.lever.co", "boards.greenhouse.io"]

        for domain, keyword in itertools.product(domains, keywords):
            yield scrapy.Request(
                f"https://duckduckgo.com/?q=site%3A{domain}+{keyword}",
                meta={
                    "playwright": True,
                    "playwright_page_methods": [
                        PageMethod(more_results),
                    ],
                },
                cb_kwargs={"keyword": keyword},
            )

    def parse(self, response, keyword):
        for url in response.css("a::attr(href)").getall():
            parsed_url = urlparse(url)
            if parsed_url.netloc not in ["jobs.lever.co", "boards.greenhouse.io"]:
                # Ignore results not in the domains we are interested
                continue

            company_name = parsed_url.path.split("/")[1]

            yield {
                "company": company_name,
                "job_posting_url": url,
                "keyword": keyword,
            }
```

```python
# postings/settings.py
# (...)

# Add the following to the existing file
DOWNLOAD_HANDLERS = {
    "http": "scrapy_playwright.handler.ScrapyPlaywrightDownloadHandler",
    "https": "scrapy_playwright.handler.ScrapyPlaywrightDownloadHandler",
}

ITEM_PIPELINES = {
   "postings.pipelines.JobPostingDuplicatesPipeline": 300,
}
```

```python
# postings/pipelines.py
from scrapy.exceptions import DropItem


class JobPostingDuplicatesPipeline:
    seen_urls = set()

    def process_item(self, item, spider):
        if item["job_posting_url"] in self.seen_urls:
            # We drop items which the URL was already returned before
            raise DropItem("Already returned")

        # Add the URL to the set when it was processed for the first time
        self.seen_urls.add(item["job_posting_url"])

        yield item
```

Good luck with your job hunt!