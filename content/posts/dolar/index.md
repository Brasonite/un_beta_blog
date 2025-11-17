+++
date = '2025-11-17T17:28:52-03:00'
draft = false
title = 'Cotação do Dólar'
description = "Estudo de APIs."
+++

A primeira atividade realizada para este projeto foi um estudo sobre o uso de APIs. Mais especificamente, como é possível utilizar a API [Olinda](https://olinda.bcb.gov.br/olinda/servico/PTAX/versao/v1/swagger-ui3#/), disponibilizada pelo Banco Central, para consultar a cotação do dólar em certos períodos.

Ao longo desse post, vou explicar o processo e a lógica por traz do código que escrevi.

# O desafio

O desafio proposto pelo professor foi relativamente simples: Dados um mês e um ano, construir um gráfico das cotações do dólar naquele período. Isso pode ser feito usando o endpoint `/CotacaoDolarPeriodo` da API, que retorna uma lista de dicionários, cada um com as informações da cotação naquele dia.

No entanto, havia um requerimento extra: Nos dias não úteis, que não possuem cotação, é necessário utilizar a última cotação disponível. Ou seja, se o dia for um domingo, se usaria a cotação de sexta, etc.

Pode não parecer que isso impactaria muito a estrutura do código, mas veremos depois o que tive que fazer para contornar esse problema.

# O código

A primeira etapa para construir esse programa foi determinar o período que seria consultado. Sim, o professor já havia disponibilizado o mês que usaria, Maio de 2014, mas a API requere uma data inicial e uma data final. Por isso, eu construí essa função conveniente para fazer justamente isso:

```Python
def datas(mes: str) -> tuple[datetime, datetime]:
    data_inicial = datetime.strptime(mes, "%m%Y")
    data_final = data_inicial.replace(
        day=calendar.monthrange(data_inicial.year, data_inicial.month)[1]
    )

    return (data_inicial, data_final)
```

Por padrão, construir um `datetime` com apenas o mês e o ano usará o primeiro dia do mês, o que é conveniente. Mesmo assim, eu precisei usar o módulo `calendar` para achar qual é o último dia do mês.

Após isso, eu construí a função que consulta todas as cotações do dólar dentro de um período:

```Python
def consultar_periodo(data_inicial: datetime, data_final: datetime) -> list[dict]:
    url = URL_PERIODO + construir_query_periodo(data_inicial, data_final)
    res = requests.get(url)
    res = res.json()
    return res["value"]
```

Note que resposta do servidor segue essa estrutura:

```JSON
{
  "@odata.context": "string",
  "value": [
    {
      "cotacaoCompra": 0,
      "cotacaoVenda": 0,
      "dataHoraCotacao": "string"
    }
  ]
}
```

Por isso, eu retorno apenas o item `value` da resposta, que é uma lista de dicionários, cada um com as informações da cotação de um dia.

Mas o que é a função `construir_query_periodo()`? Bem, ela simplesmente constrói a *query string* que eu usarei para acessar a API:

```Python
def construir_query_periodo(data_inicial: datetime, data_final: datetime) -> str:
    return f"?%40dataInicial='{data_inicial.strftime('%m%d%Y')}'&%40dataFinalCotacao='{data_final.strftime('%m%d%Y')}'&%24format=json"
```

Agora era a hora de "plottar" o gráfico. No entanto, é aí que entra o desafio adicional que mencionei no começo do artigo.

Veja, o mês de Maio de 2014 possui vários fins de semana, como tende a ser o caso para a maioria dos meses. Assim, eu terei que preencher os dias não úteis com a cotação anterior mais recente.

Basicamente, eu começo com uma lista com um valor para cada dia do mês. Então, para cada cotação que nós possuímos, eu extraio o dia de sua data e o valor do dólar e, então, defino o valor para aquele dia na lista com o valor que nós temos.

```Python
def processar_cotacoes(cotacoes_uteis: list[dict], data_inicial: datetime, data_final: datetime) -> list[float]:
    cotacoes = [0.0 for _ in range(data_final.day)]

    for cotacao in cotacoes_uteis:
        dia = datetime.strptime(cotacao["dataHoraCotacao"], "%Y-%m-%d %H:%M:%S.%f").day
        valor = cotacao["cotacaoCompra"]

        cotacoes[dia - 1] = valor
```

Após isso, nós passamos por cada item da lista e, se ele for `0` (ou seja, nenhuma cotação), nós substituímos o valor dele pelo valor do item anterior. Faça isso sequencialmente e todos os valores estarão definidos!

```Python
def processar_cotacoes(cotacoes_uteis: list[dict], data_inicial: datetime, data_final: datetime) -> list[float]:
    cotacoes = [0.0 for _ in range(data_final.day)]

    for cotacao in cotacoes_uteis:
        dia = datetime.strptime(cotacao["dataHoraCotacao"], "%Y-%m-%d %H:%M:%S.%f").day
        valor = cotacao["cotacaoCompra"]

        cotacoes[dia - 1] = valor
    
    for i in range(data_final.day):
        if cotacoes[i] == 0.0:
            cotacoes[i] = cotacoes[i - 1]

        cotacoes[dia - 1] = valor

    return cotacoes
```

Eu também aproveitei para simplificar a estrutura dos dados, já que não precisamos de todas aquelas informações extras. Assim, podemos usar o índice na lista como o dia e o item como o valor.

Contudo, essa implementação tem um grave problema. Você enxergou ele? Se não, aqui está a linha problemática:

```Python
cotacoes[i] = cotacoes[i - 1]
```

Fazendo isso, nós obtemos a cotação anterior. Mas e se `i`, o dia, for `0`? Bem, nesse caso, `i - 1` seria `0` e nós usariamos a última cotação do mês!

É claro que isso não é o que queremos, então eu substituí essa linha com o seguinte bloco:

```Python
if i == 0:
    dia_fallback = data_inicial
    cotacao_fallback = None

    while cotacao_fallback == None:
        dia_fallback -= timedelta(days=1)
        cotacao_fallback = consultar_dia(dia_fallback)
    
    cotacoes[i] = cotacao_fallback["cotacaoCompra"]
else:
    cotacoes[i] = cotacoes[i - 1]
```

Se `i` for maior que zero, nós prosseguimos como sempre. Se não, nós procuramos por cotações anteriores pela API até acharmos um dia válido e, então, usamos aquele valor.

É claro, se quisermos fazer isso, precisaremos de uma função para consultar um dia específico. Eu a nomeei `consultar_dia()` e escrevi uma função `construir_query_dia()` para acompanhá-la.

```Python
def construir_query_dia(data: datetime) -> str:
    return f"?%40dataCotacao='{data.strftime('%m%d%Y')}'&%24format=json"


def consultar_dia(data: datetime) -> dict | None:
    url = URL_DIA + construir_query_dia(data)
    res = requests.get(url)

    try:
        res = res.json()
        return res["value"][0]
    except:
        return None
```

É importante notar que, caso o dia que consultamos não seja útil, a API não retornará a estrutura normal. Por isso, usamos um `try`/`except` para prevenir que o programa entre em combustão espontânea quando esse for o caso.

OK, agora podemos finalmente construir o gráfico. Felizmente, isso é fácil com a biblioteca `plotly`.

Primeiramente, temos que processar nossas cotações, garantindo que teremos um valor para cada dia do mês.

```Python
def construir_grafico(mes: str, cotacoes: list[dict], data_inicial: datetime, data_final: datetime) -> None:
    valores = processar_cotacoes(cotacoes, data_inicial, data_final)
```

Então, temos que dividir as informações em um eixo `x` e um eixo `y`. Note que, como usamos índices (que começam no `0`), cada valor de `x` será `i + 1`, não apenas `i`.

```Python
def construir_grafico(mes: str, cotacoes: list[dict], data_inicial: datetime, data_final: datetime) -> None:
    valores = processar_cotacoes(cotacoes, data_inicial, data_final)

    x: list[float] = []
    y: list[float] = []
    for i in range(len(valores)):
        x.append(i + 1)
        y.append(valores[i])
```

Fazemos isso porque precisamos de um `DataFrame` da biblioteca `pandas` para construir o gráfico. Criá-lo é bem fácil, então isso não nos traz grandes problemas.

```Python
def construir_grafico(mes: str, cotacoes: list[dict], data_inicial: datetime, data_final: datetime) -> None:
    valores = processar_cotacoes(cotacoes, data_inicial, data_final)

    x: list[float] = []
    y: list[float] = []
    for i in range(len(valores)):
        x.append(i + 1)
        y.append(valores[i])

    df = pandas.DataFrame({"Data": x, "Valor": y})
```

Finalmente, nós podemos construir o gráfico e mostrá-lo no navegador.

```Python
def construir_grafico(mes: str, cotacoes: list[dict], data_inicial: datetime, data_final: datetime) -> None:
    valores = processar_cotacoes(cotacoes, data_inicial, data_final)

    x: list[float] = []
    y: list[float] = []
    for i in range(len(valores)):
        x.append(i + 1)
        y.append(valores[i])

    df = pandas.DataFrame({"Data": x, "Valor": y})

    fig = px.line(df, x="Data", y="Valor", title=f"Cotações no mês {mes}")
    fig.show()
```

Agora, tudo o que resta é criar a função principal do programa, que irá realizar todas as operações que precisamos, e pronto!

```Python
def main():
    mes = "052014"
    data_inicial, data_final = datas(mes)
    cotacoes = consultar_periodo(data_inicial, data_final)
    construir_grafico(mes, cotacoes, data_inicial, data_final)
```

# Os resultados

Executar esse código nos levará a uma página com as seguintes informações:

![O dito gráfico. Se você está vendo isso, tem algo de errado. Ou sua conexão não foi capaz de carregar essa imagem, ou você é cego e usa um leitor de tela. Se este último for o caso, por que você está lendo um artigo sobre construção de gráficos visuais? Não que isso não seja permitido. Na verdade, eu te encorajo a aprender o máximo possível. Mesmo assim, não deixa de ser meio engraçado.](./plot.png)

Como podemos ver, além de o valor do dólar estar muito mais baixo do que hoje (que era para se viver, não?), o nosso gráfico está ali!

Aqui está o código completo, se te interessar:

```Python
import calendar
from datetime import datetime, timedelta
import pandas
import plotly.express as px
import requests

URL_PERIODO = "https://olinda.bcb.gov.br/olinda/servico/PTAX/versao/v1/odata/CotacaoDolarPeriodo(dataInicial=@dataInicial,dataFinalCotacao=@dataFinalCotacao)"
URL_DIA = "https://olinda.bcb.gov.br/olinda/servico/PTAX/versao/v1/odata/CotacaoDolarDia(dataCotacao=@dataCotacao)"


def datas(mes: str) -> tuple[datetime, datetime]:
    data_inicial = datetime.strptime(mes, "%m%Y")
    data_final = data_inicial.replace(
        day=calendar.monthrange(data_inicial.year, data_inicial.month)[1]
    )

    return (data_inicial, data_final)


def construir_query_periodo(data_inicial: datetime, data_final: datetime) -> str:
    return f"?%40dataInicial='{data_inicial.strftime('%m%d%Y')}'&%40dataFinalCotacao='{data_final.strftime('%m%d%Y')}'&%24format=json"

def construir_query_dia(data: datetime) -> str:
    return f"?%40dataCotacao='{data.strftime('%m%d%Y')}'&%24format=json"

def consultar_periodo(data_inicial: datetime, data_final: datetime) -> list[dict]:
    url = URL_PERIODO + construir_query_periodo(data_inicial, data_final)
    res = requests.get(url)
    res = res.json()
    return res["value"]

def consultar_dia(data: datetime) -> dict | None:
    url = URL_DIA + construir_query_dia(data)
    res = requests.get(url)

    try:
        res = res.json()
        return res["value"][0]
    except:
        return None

def processar_cotacoes(cotacoes_uteis: list[dict], data_inicial: datetime, data_final: datetime) -> list[float]:
    cotacoes = [0.0 for _ in range(data_final.day)]

    for cotacao in cotacoes_uteis:
        dia = datetime.strptime(cotacao["dataHoraCotacao"], "%Y-%m-%d %H:%M:%S.%f").day
        valor = cotacao["cotacaoCompra"]

        cotacoes[dia - 1] = valor

    for i in range(data_final.day):
        if cotacoes[i] == 0.0:
            if i == 0:
                dia_fallback = data_inicial
                cotacao_fallback = None

                while cotacao_fallback == None:
                    dia_fallback -= timedelta(days=1)
                    cotacao_fallback = consultar_dia(dia_fallback)
                
                cotacoes[i] = cotacao_fallback["cotacaoCompra"]
            else:
                cotacoes[i] = cotacoes[i - 1]

    return cotacoes


def construir_grafico(mes: str, cotacoes: list[dict], data_inicial: datetime, data_final: datetime) -> None:
    valores = processar_cotacoes(cotacoes, data_inicial, data_final)

    x: list[float] = []
    y: list[float] = []
    for i in range(len(valores)):
        x.append(i + 1)
        y.append(valores[i])

    df = pandas.DataFrame({"Data": x, "Valor": y})

    fig = px.line(df, x="Data", y="Valor", title=f"Cotações no mês {mes}")
    fig.show()


def main():
    mes = "052014"
    data_inicial, data_final = datas(mes)
    cotacoes = consultar_periodo(data_inicial, data_final)
    construir_grafico(mes, cotacoes, data_inicial, data_final)


if __name__ == "__main__":
    main()
```