---
title: "Validando dados extraídos com Scrapy"
publishdate: 2019-03-08
tags: ["scrapy", "scraping", "monitoramento"]
slug: validando-dados-extraidos-com-scrapy
---

Como você garante a qualidade e a confiabilidade dos dados que você está
extraindo em seu crawler?

Se o seu crawler foi desenvolvido com [Scrapy](https://scrapy.org), você tem a
disposição a extensão [Spidermon](https://spidermon.readthedocs.io/), que junto
com a biblioteca [Schematics](https://schematics.readthedocs.io/) permite que
você inclua validações de dados de maneira simples e altamente customizável.

Definindo modelos de dados, é possível comparar os items retornados com uma
estrutura pré-determinada, garantido que todos os campos contém dados no
formato esperado.

## Nosso exemplo

Para explicar o uso do **[Spidermon](https://spidermon.readthedocs.io/)**
para realizar a validação dos dados, criamos um projeto de exemplo simples,
que extrai citações do site [http://quotes.toscrape.com/](http://quotes.toscrape.com/).

Começamos instalando as bibliotecas necessárias:

{{<highlight bash>}}
$ python3 -m venv .venv
$ source .venv/bin/activate
(.venv) $ pip install scrapy spidermon schematics
{{</highlight>}}

Em seguida criamos um novo projeto **[Scrapy](https://scrapy.org)**:

{{<highlight bash>}}
$ scrapy startproject quotes_crawler
$ cd myproject & scrapy genspider quotes quotes.com
{{</highlight>}}

Definimos um [item](https://docs.scrapy.org/en/latest/topics/items.html) que
será retornado pelo nosso
[spider](https://docs.scrapy.org/en/latest/topics/spiders.html):

{{<highlight python>}}
# quotes_crawler/spiders/items.py
import scrapy

class QuoteItem(scrapy.Item):
    quote = scrapy.Field()
    author = scrapy.Field()
    author_url = scrapy.Field()
    tags = scrapy.Field()
{{</highlight>}}

E finalmente criamos o nosso spider:

{{<highlight python>}}
# quotes_crawler/spiders/quotes.py
import scrapy
from myproject.items import QuoteItem

class QuotesSpider(scrapy.Spider):
    name = "quotes"
    allowed_domains = ["quotes.toscrape.com"]
    start_urls = ["http://quotes.toscrape.com/"]

    def parse(self, response):
        for quote in response.css(".quote"):
            item = QuoteItem(
                quote=quote.css(".text::text").get(),
                author=quote.css(".author::text").get(),
                author_url=response.urljoin(
                    quote.css(".author a::attr(href)").get()),
                tags=quote.css(".tag *::text").getall(),
            )
            yield item
        yield scrapy.Request(
            response.urljoin(response.css(
                ".next a::attr(href)").get())
        )
{{</highlight>}}

Com isso já é possível executar o nosso crawler e obter os dados desejados:

{{<highlight bash>}}
$ scrapy crawl quotes -o quotes.csv
{{</highlight>}}

## Definindo nosso modelo de validação

**[Schematics](https://schematics.readthedocs.io/)** é uma biblioteca de validação de dados baseada em modelos (muito
parecidos com modelos do Django). Esses modelos incluem alguns tipos de dados
comuns e validadores, mas também é possível extendê-los e definir regras de
validação customizadas.

Baseado no nosso exemplo anterior, vamos criar um modelo para o nosso item
contendo algumas regras básicas de validação:

{{<highlight python>}}
# quotes_crawler/spiders/validators.py
from schematics.models import Model
from schematics.types import URLType, StringType, ListType

class QuoteItem(Model):
    # String de preenchimento obrigatório
    quote = StringType(required=True)

    # String de preenchimento obrigatório
    author = StringType(required=True)

    # Uma string que represente uma URL de preenchimento obrigatório
    author_url = URLType(required=True)

    # Lista de Strings de preenchimento não obrigatório
    tags = ListType(StringType)
{{</highlight>}}

Precisamos agora habilitar o **Spidermon** e configurá-lo para que os campos
retornados pelo nosso spider sejam validados contra esse modelo:

{{<highlight python>}}
# quotes_crawler/settings.py
SPIDERMON_ENABLED = True

EXTENSIONS = {
    "spidermon.contrib.scrapy.extensions.Spidermon": 500,
}

ITEM_PIPELINES = {
    "spidermon.contrib.scrapy.pipelines.ItemValidationPipeline": 800
}

SPIDERMON_VALIDATION_MODELS = (
    "quotes_crawler.validators.QuoteItem",
)

# Inclui um campo '_validation' nos items que contenham erros
# com detalhes do problema encontrado
SPIDERMON_VALIDATION_ADD_ERRORS_TO_ITEMS = True
{{</highlight>}}

Conforme os dados forem extraidos, eles serão validados de acordo com as regras
definidas no modelo e, caso algum erro seja encontrado, um novo campo
`_validation` será incluído no item com detalhes do erro.

Caso um item receba uma URL inválida no campo `author_url` por exemplo, o
resultado será:

{{<highlight js>}}
{
    '_validation': defaultdict(
        <class 'list'>, {'author_url': ['Invalid URL']}),
    'author': 'C.S. Lewis',
    'author_url': 'invalid_url',
    'quote': 'Some day you will be old enough to start reading fairy tales '
        'again.',
    'tags': ['age', 'fairytales', 'growing-up']
}
{{</highlight>}}
