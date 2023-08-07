---
title: "Revisão de Código: Extraindo dados do site 'Aos Fatos'"
publishdate: 2022-04-08
tags: ["revisão de código", "scrapy", "raspagem de dados", "python", "live"]
slug: revisao-de-codigo-extraindo-dados-do-site-aos-fatos
---

Algumas semanas atrás, fiz a revisão de um código para extrair informações de
[campeonatos da CBF](https://peertube.lhc.net.br/w/g3zhbDB7b81Sx8LWLpdhAk).

A experiência de fazer isso em uma *live* foi muito boa, pois consegui
ajudar alguém compartilhando um pouco da minha experiência,
mas também foi uma maneira de eu aprender mais, já que para comentar sobre algum
assunto eu precisei ler e estudar (e relembrar) algumas coisas que já fazia
algum tempo que eu não olhava.

Depois de algum tempo, encontrei outra pessoa que me autorizou a fazer essa revisão
em uma *live*. Dessa vez, fiz a revisão do código que extraia informações de
notícias do [Aos Fatos](https://www.aosfatos.org/).

O [código](https://github.com/diegofan-code/scrapy-aosfatos) é um projeto
feito com o [Scrapy](https://scrapy.org), um framework Python para o
desenvolvimento de raspadores de dados.

{{< peertube "https://peertube.lhc.net.br/videos/embed/874fa418-026c-4f9d-8285-12fb796575a0" >}}

Se você quiser que eu faça uma revisão do seu código em um vídeo, é só
entrar em contato comigo. Se eu achar que consigo ajudar de alguma
maneira, combinamos uma nova transmissão.

## Links de Referência

- [black](https://pypi.org/project/black/) - formatador de código automático

- [CrawlSpider](https://docs.scrapy.org/en/latest/topics/spiders.html?highlight=CrawlSpider#crawlspider) - tipo de Spider que ajuda a escrever códigos mais organizados

- [dateparser](https://dateparser.readthedocs.io/en/latest/) - biblioteca para conversão de texto em data

- [LinkExtractor](https://docs.scrapy.org/en/latest/topics/link-extractors.html) - class que auxilia a extração de links dentro de um HTML

- [Laboratório Hacker de Campinas](https://lhc.net.br) - hackerspace localizado em Campinas

- [owncast](https://owncast.online/) - plataforma *self-hosted* por onde fiz a transmissão

- [PeerTube](https://joinpeertube.org/) - plataforma de vídeos livre, decentralizada e federada

- [Meu PeerTube](https://peertube.lhc.net.br/a/rocha/video-channels) - instância do PeerTube onde armazeno todos os meus vídeos
