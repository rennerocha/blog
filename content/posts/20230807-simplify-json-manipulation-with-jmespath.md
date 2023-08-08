---
title: "Simplify JSON Manipulation with JMESPath"
publishdate: 2023-08-07
tags: ["json", "jmespath", "api"]
slug: simplify-json-manipulation-with-jmespath
---

A common method to retrieve data from dynamic websites involves
emulating the internal API calls made by the page, foregoing
the need to render the entire page through a web browser.
This entails processing the response, typically in the form of JSON
content. Given the lack of access to API documentation, we cannot
anticipate all potential response variations.

Similarly, when integrating with third-party APIs, whether or not
in a web scraping context, the responses are seldom completely
reliable. This calls for a defensive coding approach to avoid
potential breakdowns due to invalid responses.

So let's see how we can do that approach using pure Python and also
explore an alternative using JMESPath to achieve more concise and comprehensible code.

## Sample JSON payload

This is a simplified version of the JSON payload of a payment
event sent by [Paypal](https://developer.paypal.com/api/rest/webhooks/#link-samplemessagepayload) webhooks already converted to a Python
[dictionary](https://docs.python.org/3/tutorial/datastructures.html#dictionaries).

As an example, let's extract all the URLs that exists inside the `resource`
and `links` keys.

```python
response = {
  "id": "8PT597110X687430LKGECATA",
  "event_type": "PAYMENT.AUTHORIZATION.CREATED",
  "resource": {
    "id": "2DC87612EK520411B",
    "state": "authorized",
    "amount": {
      "total": "7.47",
      "currency": "USD",
    },
    "parent_payment": "PAY-36246664YD343335CKHFA4AY",
    "links": [
      {
        "href": "https://sandbox.paypal.com/2DC87612EK520411B",
        "method": "GET"
      },
      {
        "href": "https://sandbox.paypal.com/2DC87612EK520411B",
        "method": "POST"
      },
      {
        "href": "https://sandbox.paypal.com/2DC87612EK520411B",
        "method": "POST"
      },
      {
        "href": "https://sandbox.paypal.com/PAY-36246664YD343335CKHFA4AY",
        "method": "GET"
      }
    ]
  }
}
```

## Extracting the information

Let's consider the ideal structure of our dictionary. To extract the links, a simple function like the following seems to fit:

```python
def get_links(response):
    return [
        link["href"] for link in response["resource"]["links"]
    ]
```

However, reality often introduces complexity, particularly when dealing with third-party responses. There's no guarantee that we will consistently receive all required key-value pairs or data types as anticipated. Consequently, a need arises to gracefully manage potential errors and exceptions.

What if our JSON lacks a `response` key? Alternatively, what if `links` appears as `None` instead of a `list`? Furthermore, some instances might deliver a list of links, while others may not include it at all.

As we integrate additional validations and error-handling measures, our code inevitably becomes convoluted, making it progressively more challenging to discern its underlying functionality.

```python
def get_links(response):
    try:
        resource = response.get("resource", {})

        if not isinstance(resource, dict):
            raise ValueError("Invalid 'resource' type")

        links = resource.get("links", [])

        if links is None:
            raise ValueError("'links' is None")

        if not isinstance(links, list):
            raise ValueError("Invalid 'links' type")

        return [
            link["href"] for link in links
            if isinstance(link, dict) and "href" in link
        ]
    except (KeyError, ValueError):
        return []
```

[Readability counts.](https://peps.python.org/pep-0020/#the-zen-of-python)

## Introducing JMESPath

[JMESPath](https://jmespath.org/) serves as a query language designed for JSON, enabling the extraction and transformation of elements from a JSON document. This specification encompasses implementations in numerous popular programming languages. One such implementation is the Python library [jmespath](https://pypi.org/project/jmespath/).

By employing JMESPath, we can streamline the code we have previously crafted, resulting in the elimination of numerous lines of code, all while upholding its robustness. This approach also mitigates the occurrence of unforeseen errors and exceptions.

```python
import jmespath

def get_links(response):
    return jmespath.search("resource.links[].href", response) or []
```

This code adeptly manages various scenarios, including the absence of keys or encountering a data type other than a `list`, such as `None`. In instances where the expression fails to evaluate correctly, the `search()` function returns `None`. This concise and well-structured code not only trims unnecessary elements but also enhances readability.

It is possible to perform much more complex queries in a JSON using
this language, so it is worthwhile to look at
[JMESPath](https://jmespath.org/tutorial.html) tutorial
and add this as a new tool that you can use in your projects.