---
title: "Monitorando seus projetos de raspagem de dados"
publishdate: 2022-01-13
tags: ["monitoramento", "scrapy", "raspagem de dados", "python", "spidermon"]
slug: monitorando-seus-projetos-de-raspagem-de-dados
---

Quando passamos a extrair dados de uma página com regularidade, com
uma quantidade grande de dados, é muito importante monitorar a saúde
dos seus raspadores e a qualidade das informações extraídas.

As páginas podem mudar (as vezes sutilmente), medidas anti-bot podem
ser adicionadas e outros problemas temporários e inesperados podem
ocorrer e, caso estejamos tratando de uma quantidade grande de
*scripts* retornando milhares de itens, é praticamente impossível
monitorar manualmente todo o nosso projeto.

Deste modo, devemos adicionar em nosso projeto, ferramentas que auxiliem
no monitoramento.

## Spidermon

O **[Scrapy](https://scrapy.org)** é um dos principais *frameworks* para
raspagem de dados em **[Python](https://www.python.org/)** possuindo
diversas bibliotecas e ferramentas que facilitam a criação de projetos
de raspagem de dados.

O **[Spidermon](https://spidermon.readthedocs.io/)**
é uma extensão do **Scrapy** que podemos adicionar ao nosso projeto
para facilitar o monitoramento da execução dos seus *spiders* (como são
chamados os *scripts* que realizam a extração de dados).

Com ela, é possível definir validar dados retornados, monitorar as
**estatísticas** da execução (*jobs*) de seu *spider* e, a partir do
resultadodessas validações, tomar **ações** como enviar notificações,
gerar relatórios, abrir chamados em sistemas externos quando algo não
esteja funcionando corretamente de uma maneira extensível.

O **[Spidermon](https://spidermon.readthedocs.io/)** pode ser instalado
com o **[pip](https://pypi.org/project/pip/)**:

{{<highlight bash>}}
pip install spidermon
{{</highlight>}}

Uma vez instalado, precisamos habilitá-lo em nosso projeto
**[Scrapy](https://scrapy.org)**:

{{<highlight python>}}
# myproject/settings.py
SPIDERMON_ENABLED = True
EXTENSIONS = {
    "spidermon.contrib.scrapy.extensions.Spidermon": 500,
}
{{</highlight>}}

Feito isso, podemos começar a pensar no que será monitorado.

## Monitores

É no **[monitor](https://spidermon.readthedocs.io/en/latest/monitors.html#monitors)**
onde a maioria das validações são definidas. Nele temos acesso as
estatísticas e informações das execuções de nossos *spiders* e podemos
definir as nossas regras de monitoramento.

Esse **monitor** precisa estar dentro de uma
**[suíte de monitores](https://spidermon.readthedocs.io/en/latest/monitors.html#monitor-suites)**, além de definir um conjunto de **[ações](https://spidermon.readthedocs.io/en/latest/actions/index.html)** que devem ser executadas após a execução deles.

Como exemplo, queremos monitorar o número de mensagens de erro emitidas durante
os nossos *jobs*. Isso pode ser feito verificando se o valor dentro da estatística
`log_count/ERROR` é menor a um determinado valor que podemos, por exemplo, definir
no arquivo de configuração do nosso projeto.

Um monitor que realiza essa tarefa pode ser definido como:

{{<highlight python>}}
# myproject/monitors.py
from spidermon import Monitor

class JobErrorsMonitor(Monitor):
    def test_error_count(self):
        num_errors = self.data.stats.get("log_count/ERROR")
        num_errors_threshold = self.data.crawler.settings.getint(
            "MAX_NUM_ERRORS", 0
        )
        if num_errors:
            self.assertTrue(
                num_errors <= num_errors_threshold,
                msg=f"No more than {num_errors_threshold} allowed."
            )
{{</highlight>}}

Porém, só definir o **monitor** não é o suficiente para que esse
teste seja rodado ao final da execução do seu *spider*. Precisamos
adicioná-lo a uma **suíte de monitores**:

{{<highlight python>}}
# myproject/monitors.py
from spidermon import Monitor, MonitorSuite

class JobErrorsMonitor(Monitor):
    # (...) O código do nosso monitor

class SpiderCloseMonitorSuite(MonitorSuite):
    monitors = [
        JobErrorsMonitor,
    ]
{{</highlight>}}

E definir o momento em que momento a suíte será executada, além
do nosso valor de referência com o limite da quantidade de erros
que aceitamos. Nesse caso, quando o *spider* terminar sua execução
as verificações será feita.

{{<highlight python>}}
# myproject/settings.py
SPIDERMON_ENABLED = True
EXTENSIONS = {
    "spidermon.contrib.scrapy.extensions.Spidermon": 500,
}
SPIDERMON_SPIDER_CLOSE_MONITORS = (
    "myproject.monitors.SpiderCloseMonitorSuite",
)
MAX_NUM_ERRORS = 10
{{</highlight>}}

Ao executar *spider* podemos ver os resultados desse monitor nos nossos
*logs*:

{{<highlight python>}}
[scrapy.core.engine] INFO: Closing spider (finished)
[myproject] INFO: [Spidermon] ---------------------- MONITORS ----------------------
[myproject] INFO: [Spidermon] JobErrorsMonitor/test_error_count... OK
[myproject] INFO: [Spidermon] ------------------------------------------------------
[myproject] INFO: [Spidermon] 1 monitor in 0.000s
[myproject] INFO: [Spidermon] OK
[myproject] INFO: [Spidermon] ------------------ FINISHED ACTIONS ------------------
[myproject] INFO: [Spidermon] ------------------------------------------------------
[myproject] INFO: [Spidermon] 0 actions in 0.000s
[myproject] INFO: [Spidermon] OK
[myproject] INFO: [Spidermon] ------------------- PASSED ACTIONS -------------------
[myproject] INFO: [Spidermon] ------------------------------------------------------
[myproject] INFO: [Spidermon] 0 actions in 0.000s
[myproject] INFO: [Spidermon] OK
[myproject] INFO: [Spidermon] ------------------- FAILED ACTIONS -------------------
[myproject] INFO: [Spidermon] ------------------------------------------------------
[myproject] INFO: [Spidermon] 1 action in 0.000s
[myproject] INFO: [Spidermon] OK (skipped=1)
{{</highlight>}}

Ou caso aconteça um erro:

{{<highlight python>}}
[scrapy.core.engine] INFO: Closing spider (finished)
[myproject] INFO: [Spidermon] ---------------------- MONITORS ----------------------
[myproject] INFO: [Spidermon] JobErrorsMonitor/test_error_count... FAIL
[myproject] INFO: [Spidermon] ------------------------------------------------------
[myproject] ERROR: [Spidermon]
======================================================================
FAIL: JobErrorsMonitor/test_error_count
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/home/renne/projects/rennerocha/spidermon/examples/tutorial/tutorial/monitors.py", line 44, in test_error_count
    self.assertTrue(
AssertionError: False is not true : No more than 10 errors allowed.

[myproject] INFO: [Spidermon] 1 monitor in 0.001s
[myproject] INFO: [Spidermon] FAILED (failures=1)
[myproject] INFO: [Spidermon] ------------------ FINISHED ACTIONS ------------------
[myproject] INFO: [Spidermon] ------------------------------------------------------
[myproject] INFO: [Spidermon] 0 actions in 0.000s
[myproject] INFO: [Spidermon] OK
[myproject] INFO: [Spidermon] ------------------- PASSED ACTIONS -------------------
[myproject] INFO: [Spidermon] ------------------------------------------------------
[myproject] INFO: [Spidermon] 0 actions in 0.000s
[myproject] INFO: [Spidermon] OK
[myproject] INFO: [Spidermon] ------------------- FAILED ACTIONS -------------------
{{</highlight>}}

## Simplificando o nosso monitor

Grande parte dos monitores que precisamos, verifica
o valor de uma estatística do seu *job* com um valor
pré-determinado. Por isso, podemos usar a classe
*BaseStatMonitor* com um conjunto definido de atributos
para simplificar o nosso código.

O mesmo monitor anterior, verificando se o valor
de `log_count/ERROR` é menor ou igual ao valor definido
em `MAX_NUM_ERRORS` nas configurações do projeto
pode ser definido como:

{{<highlight python>}}
# myproject/monitors.py
from spidermon.contrib.scrapy.monitors import BaseStatMonitor

class JobErrorsMonitor(BaseStatMonitor):
    stat_name = "log_count/ERROR"
    threshold_setting = "MAX_NUM_ERRORS"
    assert_type = "<="
{{</highlight>}}

O resultado da execução desse monitor é o mesmo do anterior, porém
com um código muito mais simples.

## Tornando o monitor mais dinâmico

Não precisamos nos restringir a testes de um valor nas estatísticas
em relação a um valor estático definido nas configurações do projeto.

Para isso, ao invés de definir um valor para o atributo
`threshold_setting`, podemos definir o método `get_threshold` que
deverá retornar o valor que será usado como referência para o nosso
monitor.

Isso é muito útil por exemplo, quando queremos verificar mais de
uma estatística para definir o nosso valor de referência. No
exemplo anterior, definimos uma valor fixo aceitável de mensagens
de erro durante a execução do nosso spider. Porém termos 10 erros
em uma execução que retorna 100k itens (0.001%) pode ser mais aceitável
por exemplo do que 10 erros em uma execução que retornou 1k itens (1%).

Então podemos melhoramos o nosso monitor de erros, considerando além
da quantidade de erros, a quantidade de itens retornados.
Desse modo só haverá falha se o número de erros for menor do que 0.001%
da quantidade de itens retornados.

{{<highlight python>}}
# myproject/monitors.py
from spidermon.contrib.scrapy.monitors import BaseStatMonitor

class JobErrorsMonitor(BaseStatMonitor):
    stat_name = "log_count/ERROR"
    assert_type = "<="

    def get_threshold(self):
        item_scraped_count = self.data.stats.get(
            "item_scraped_count"
        )
        return item_scraped_count * 0.0001
{{</highlight>}}

Podemos também buscar dados de fontes externas, comparar com dados de execuções
passadas que estejam armazenados em um banco de dados ou fazer cálculos mais
complexos para definir o nosso valor de referência.

O importante é que sempre o retorno do método `get_threshold` seja um valor que
possa ser usado na comparação com a estatística que queremos.

Na [documentação](https://spidermon.readthedocs.io/en/latest/monitors.html#base-stat-monitor)
é possível encontrar mais exemplo e opções para os nossos monitores, além de um conjunto
de monitores pré-definidos para as validações mais comuns nos projetos diminuindo a quantidade
de código que precisamos escrever para começar a monitorar nossos projetos.

## Próximos passos

Ver o resultado dos nossos monitores apenas nos *logs* não é muito prático. O
que precisamos fazer em seguida é configurar nosso projeto para que alguma **ação**
seja tomada em caso de falha (ou de sucesso).

Para isso, precisamos definir *actions*. Na extensão temos integrações
prontas para receber notificações por e-mail, Slack, Telegram ou Sentry,
além de ser possível criar suas próprias ações de acordo com as
necessidades do projeto.

Na [documentação](https://spidermon.readthedocs.io/en/latest/actions/index.html)
é possível encontrar mais informações sobre como defini-las e configurá-las.