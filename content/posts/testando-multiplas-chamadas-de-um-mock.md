---
title: "Testando múltiplas chamadas de uma função com mocks"
publishdate: 2022-11-07
tags: ["python", "testes", "pytest"]
---

Em algumas situações durante o teste e alguma função, queremos confirmar
que uma determinada função foi chamada uma quantidade de vezes esperada
e também com os argumentos corretos.

Isso pode ser obtido utilizando o [mock](https://docs.python.org/3/library/unittest.mock.html) da biblioteca padrão do Python ou, caso estejamos utilizando o [pytest](https://docs.pytest.org/en/7.2.x/), com o auxílio da biblioteca [pytest-mock](https://pypi.org/project/pytest-mock/).

Como exemplo, vamos testar a função `main()` a seguir. Ela obtém uma lista de URLs que neste exemplo é fixa, mas em um caso real, ela pode ser o resultado de uma condição especial do sistema que você está testando.

{{<highlight python>}}
# app.py
import requests

def do_something_with_response(response):
    ...

def get_urls_to_call():
    return [
        "http://www.url1.com",
        "http://www.url2.com",
    ]

def main():
    urls_to_call = get_urls_to_call()
    for url in urls_to_call:
        response = requests.post(url)
        do_something_with_response(response)
{{</highlight>}}

Meu objetivo aqui é verificar se `requests.post()` foi chamado apenas duas vezes e se nessas duas vezes, passamos como argumento o valor das duas URLs retornadas pela função `get_urls_to_call()`.

O primeiro passo é criar um [`patch`](https://docs.python.org/3/library/unittest.mock.html#patch) de `requests.post`. Fazemos isso da seguinte maneira:

{{<highlight python>}}
# test_app.py
def test_assert_called_with_all_urls(mocker):
    requests_post_mock = mocker.patch("app.requests.post")
{{</highlight>}}

A partir desse momento, qualquer chamada de `requests.post` dentro do módulo `app` será uma chamada a um `MagicMock` e não a função original. Atente-se que o [`patch`](https://docs.python.org/3/library/unittest.mock.html#patch) **não** é realizado com `requests.post` e sim com `app.requests.post`.

> Para mais detalhes sobre como usar corretamente a função `patch`, assista [esta apresentação](https://www.youtube.com/watch?v=ww1UsGZV8fQ) de Lisa Roach na PyCon 2018.

Em seguida podemos fazer a chamada na função que estamos testando:

{{<highlight python>}}
# test_app.py
from app import main

def test_assert_called_with_all_urls(mocker):
    requests_post_mock = mocker.patch("app.requests.post")

    main()
{{</highlight>}}

> [`mocker`](https://pytest-mock.readthedocs.io/en/latest/usage.html#usage) é apenas uma [`fixture`](https://docs.pytest.org/en/7.2.x/explanation/fixtures.html#about-fixtures) do [`pytest-mock`](https://pytest-mock.readthedocs.io/) auxiliar de [`unittest.mock`](https://docs.python.org/dev/library/unittest.mock.html).

E agora iremos verificar as chamadas que foram realizadas e se or argumentos passados foram os corretos:

{{<highlight python>}}
from app import main

def test_assert_called_with_all_urls(mocker):
    requests_post_mock = mocker.patch("app.requests.post")

    main()

    # Garante que 'requests.post' foi chamado duas vezes
    # e cada uma delas a URL correta foi passada como argumento
    requests_post_mock.assert_has_calls(
        [
            mocker.call("http://www.url1.com"),
            mocker.call("http://www.url2.com"),
        ]
    )
{{</highlight>}}

Existem outras validações possível, como por exemplo verificarmos se `requests.post` foi chamado uma única vez ([`assert_called_once`](https://docs.python.org/3/library/unittest.mock.html#the-mock-class)) ou mesmo se a função não foi chamada nenhuma vez ([`assert_not_called`](https://docs.python.org/3/library/unittest.mock.html#unittest.mock.Mock.assert_not_called)) o que pode ser útil para testar condições de erro.

Só é preciso tomar cuidado para não abusar desse tipo de teste, já que ele é altamente acoplado a implementação do seu código e dependendo de como ele é definido e colocado no seu projeto, eventualmente ele pode não testar adequadamente o que você quer testar.
