---
title: "Following links declaratively with Scrapy"
date: 2025-02-12
tags: ["web scraping", "scrapy"]
slug: following-links-declaratively-with-scrapy
---

> This post assumes that you have a basic understanding of how [Scrapy](https://scrapy.org/), a web scraping framework, works, discussing some lesser-known features. You can have an [introduction](https://docs.scrapy.org/en/latest/intro/overview.html) to it in your documentation.

Extract links from a response, filtering by a specific pattern and following them is a common task that you will face when scraping a website.

As an example, suppose that you are scraping a store website and you want to process two different URLs:

1. Product URLs to gather product details
1. Category URLs to collect information about a category (e.g. number of items, subcategories, etc.) and/or more product URLs

We can implement our spider like the following:

```python
class StoreSpider(scrapy.Spider):
    name = "store"
    start_urls = ["https://store.example.com/"]

    def parse(self, response):
        products = response.css('a.product::attr(href)')
        for link in links:
            yield scrapy.Request(link, callback=self.parse_product)

        categories = response.css('a.category::attr(href)')
        for link in links:
            yield scrapy.Request(link, callback=self.parse_category)

    def parse_product(self, response):
        ...  # Code to parse a product

    def parse_category(self, response):
        ...  # Code to parse a category
```

Only [anchor elements](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/a) with CSS classes `product` or `category` are of our interest. We have a different `parse` method based on the type of element being followed.

As soon you have more types of links and more complex rules to find them in the website response, your parsing methods will become more complicated and prone to have some code duplication.

For example, if inside your `parse_category` you also want to find more products, more categories and also subcategories, you will need to duplicate some code such as:

```python
class StoreSpider(scrapy.Spider):
    name = "store"
    start_urls = ["https://store.example.com/"]

    # (...)

    def parse_category(self, response):
        # (...) Code to parse a category

        products = response.css('a.product::attr(href)')
        for link in links:
            yield scrapy.Request(link, callback=self.parse_product)

        categories = response.css('a.category::attr(href)')
        for link in links:
            yield scrapy.Request(link, callback=self.parse_category)

        sub_categories = response.css('a.sub_category::attr(href)')
        for link in links:
            yield scrapy.Request(link, callback=self.parse_subcategory)
```

To avoid all this repeated code, Scrapy comes with [generic spiders](https://docs.scrapy.org/en/latest/topics/spiders.html#generic-spiders) providing special functionality for common scraping cases.

A [`CrawlSpider`](https://docs.scrapy.org/en/latest/topics/spiders.html#crawlspider) is a generic spider (inherited from the regular `scrapy.Spider` class) that provides a mechanism for following links by defining a set of rules (just like we did in our previous example).

Instead of actively looking for each link in its response, iterating over the results, and sending a request for each one, we can use a declarative pattern by providing a list of [`Rule`](https://docs.scrapy.org/en/latest/topics/spiders.html#scrapy.spiders.Rule) objects with arguments that state which links we want to follow and what to do with the response.

Our previous spider can be rewritten as:

```python
import scrapy
from scrapy.spiders import CrawlSpider, Rule
from scrapy.linkextractors import LinkExtractor

class StoreSpider(CrawlSpider):
    name = "store"
    start_urls = ["https://store.example.com/"]

    rules = (
        Rule(
            LinkExtractor(restrict_css="a.product"),
            callback=self.parse_product
        ),
        Rule(
            LinkExtractor(restrict_css="a.category"),
            callback=self.parse_category
        ),
        Rule(
            LinkExtractor(
                restrict_css=".pagination",
                restrict_text="Next Page",
            ),
        ),
    )

    def parse_product(self, response):
        ...  # Code to parse a product

    def parse_category(self, response):
        ...  # Code to parse a category
```

You may notice that we now have a `rules` tuple. Each [`Rule`](https://docs.scrapy.org/en/latest/topics/spiders.html#scrapy.spiders.Rule) is assigned a [`LinkExtractor`](https://docs.scrapy.org/en/latest/topics/link-extractors.html) object that defines how (and which) links will be extracted from each page.

In addition, we have a `callback`, which tells us which parse method should be used to process the return of the request from each of these links.

Given that rule as example:

```python
Rule(
    LinkExtractor(restrict_css="a.product"),
    callback=self.parse_product
)
```

It extracts all the links with the CSS class `product`, requests them and processes the response in the spider's method `parse_product`.

The following [`Rule`](https://docs.scrapy.org/en/latest/topics/spiders.html#scrapy.spiders.Rule) extracts only links with the CSS class equal to `next`, but that contains the text `Next Page` on it. So `<a class="next" href='https://store.example.com/page/2'>Next Page</a>` will be extracted, but `<a class="next" href='https://store.example.com/events/'>Future Events</a>` will not.

```python
Rule(
    LinkExtractor(
        restrict_css="a.next",
        restrict_text="Next Page",
    ),
),
```

Notice that we don't have a `callback` provided for this [`Rule`](https://docs.scrapy.org/en/latest/topics/spiders.html#scrapy.spiders.Rule). If you don't provide one, the link will still be requested and the response will be parsed using all the `rules` we have to find more links to follow.

You can see more ways to filter the links you want to extract in [ Link Extractor](https://docs.scrapy.org/en/latest/topics/link-extractors.html#link-extractors) documentation.

Using [`CrawlSpider`](https://docs.scrapy.org/en/latest/topics/spiders.html#crawlspider) avoids some complexities in our parsing  methods, keeping them focused only on how to scrape data into a page response, leaving the task of crawling the website (i.e., finding the following links) to a more declarative, easier-to-read and understand pattern.