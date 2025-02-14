---
title: "Dynamic rules for following links declaratively with Scrapy"
date: 2025-02-14
tags: ["web scraping", "scrapy"]
slug: dynamic-rules-for-following-links-declaratively-with-scrapy
---

When using `CrawlSpider`, we have a fixed set of rules that declares how we should follow and process links extracted in the website.

But sometimes we don't want the rules to be static. We need a certain level of dynamism, where the rules vary according to parameters provided as input in our spiders.

Consider that we are scraping product URLs from an ecommerce website, and we have the following patterns for category URLs:

- `https://store.example.com/` - main page of our store
- `https://store.example.com/electronics` - list of products of `electronics` category
- `https://store.example.com/food` - list of products of `food` category

We can notice the pattern `https://store.example.com/<CATEGORY_SLUG>` in our URLs. Using `CrawlSpider` as [explained in my last post]({{< ref "20250212-following-links-declaratively-with-scrapy" >}}) this set of `rules` can be defined as:

```python
import scrapy
from scrapy.spiders import CrawlSpider, Rule
from scrapy.linkextractors import LinkExtractor

class StoreSpider(CrawlSpider):
    name = "store"
    start_urls = ["https://store.example.com/"]

    rules = (
        Rule(
            LinkExtractor(allow=r"\/electronics$"),
            callback=self.parse_category,
        ),
        Rule(
            LinkExtractor(allow=r"\/electronics/ID\d+$"),
            callback=self.parse_product,
        ),
        Rule(
            LinkExtractor(allow=r"\/food$"),
            callback=self.parse_category,
        ),
        Rule(
            LinkExtractor(allow=r"\/food/ID\d+$"),
            callback=self.parse_product,
        ),
    )

    def parse_category(self, response):
        ...  # Code to parse a category

    def parse_product(self, response):
        ...  # Code to parse a product
```

There are a few potential problems with this approach:

- We need to create a rule for each category we want to extract data from;
- We need to change the code to add any new categories that we want to start processing;
- If processing a particular category takes too long, we might want to run the spider in parallel so that each process extracts data from just one category.

What if we could send the name of the category that we want to process as an argument to our spider? We can do it using `-a argument=value` when calling `scrapy crawl` such as:

```bash
scrapy crawl store -a category=food
```

If we run the spider passing this argument, we now have access on the spider instance the attribute `self.category` with the value `food` which can be used to limit the links we want to be extracted.

Then we can filter our requests, preventing any page that is not in the desired category from being processed.

```python
    def parse_category(self, response):
        if self.parse_category not in response.url:
            return

        ...  # Code to parse a category
```

The problem with this approach is that we send a real request to all links (even for the categories we don't want to), no matter if the response will be discarded or not.

A better solution would be making the rules collection to be dynamically:

```python
    rules = (
        Rule(
            LinkExtractor(allow=rf"\/{self.category}$"),
            callback=self.parse_category,
        ),
        Rule(
            LinkExtractor(allow=rf"\/{self.category}/ID\d+$"),
            callback=self.parse_product,
        ),
    )
```

Unfortunately this will not work, as the `rules` is a class attribute, so we don't have an instance of the spider to get the content of our input.

Investigating Scrapy's code, we found that the defined rules are [processed in a call](https://github.com/scrapy/scrapy/blob/f041f26a6ff636b764d2bf584ddbc9b9e4334d1b/scrapy/spiders/crawl.py#L97) to the [`_compile_rule`](https://github.com/scrapy/scrapy/blob/f041f26a6ff636b764d2bf584ddbc9b9e4334d1b/scrapy/spiders/crawl.py#L181) method.

Inside this method, each rule in `self.rules` is evaluated and then appended to `self._rules` attribute, which is where the spider [decides which link to follow](https://github.com/scrapy/scrapy/blob/f041f26a6ff636b764d2bf584ddbc9b9e4334d1b/scrapy/spiders/crawl.py#L127).

In this way, we can define our custom `_compile_rules` method, which will take the value passed as an argument to our spider and define the rules to only extract links that are of the desired category.

Our spider can look like this:

```python
import scrapy
from scrapy.spiders import CrawlSpider, Rule
from scrapy.linkextractors import LinkExtractor

class StoreSpider(CrawlSpider):
    name = "store"
    start_urls = ["https://store.example.com/"]

    def _compile_rules(self):
        self.rules = (
            Rule(
                LinkExtractor(allow=rf"\/{self.category}$"),
                callback=self.parse_category,
            ),
            Rule(
                LinkExtractor(allow=rf"\/{self.category}/ID\d+$"),
                callback=self.parse_product,
            ),
        )

        # After setting our rules, just use the existing _compile_rules() method
        super()._compile_rules()

    def parse_category(self, response):
        ...  # Code to parse a category

    def parse_product(self, response):
        ...  # Code to parse a product
```

We can now run a separate spider job for each category and extract product data from each category individually:

```bash
# Extracts data only of 'electronics' category
scrapy crawl store -a category=electronics
```

```bash
# Extracts data only of 'food' category
scrapy crawl store -a category=food
```

If we have new categories, we just pass them as a new value for the argument:

```bash
# Extracts data only of 'cars' category
scrapy crawl store -a category=cars
```