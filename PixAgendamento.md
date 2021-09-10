# Indices
- [Agendamento de iniciação de pagamentos](#agendamento-de-iniciação-de-pagamentos)
    - [Regras gerais de negócio](#regras-gerais-de-negócio)
    - [Políticas de agendamento](#políticas-de-agendamento)
    - [Política de agendamento de pagamento único](#política-de-agendamento-de-pagamento-único)
    - [Política de agendamento de pagamento recorrente com frequência fixa](#política-de-agendamento-de-pagamento-recorrente-com-frequência-fixa)
    - [Política de agendamento de pagamento recorrente por repetição](#política-de-agendamento-de-pagamento-recorrente-por-repetição)
    - [Política de agendamento recorrente por configuração customizada](#política-de-agendamento-recorrente-por-configuração-customizada)
    - [Alterações no endpoint de criação de pagamentos](#alterações-no-endpoint-de-criação-de-pagamentos)
    - [Controle de andamento de modificações no consentimento](#controle-de-andamento-de-modificações-no-consentimento)


# Agendamento de iniciação de pagamentos

Para possibilitar o agendamento único ou recorrente
de pagamentos iniciados pelo Open Banking (OB) seria necessário a inclusão do conceito de "política de agendamento" no consentimento.
A proposta aqui apresentada tem o canal OB sendo agnóstico ao arranjo do produto final (Pix, TED, TEF, débito em conta) sendo chamado, suas eventuais interseções com a funcionalidade proposta, de modo a ter menos impacto possível no OB em possíveis expansões de funcionalidades destes arranjos futuramente ou regras de negócio deste contexto dos mesmos.
Isso acarreta que para produto final o agendamento único ou sucessivo seria totalmente desconhecido se comportando apenas como um pagamento normal.

Mais adiante são mostradas e discutidas algumas formas de materialização dessa **"política de agendamento"**.

## Regras gerais de negócio

1. Todo o consentimento de pagamentos agendados ou recorrentes tem o prazo máximo de validade decorrente da política de agendamento associada.
2. A execução de pagamento conforme a agenda fica de responsabilidade da iniciadora como hoje já é feito com pagamentos normais.
3. Todo pagamento para um consentimento vinculado a uma agenda de pagamento deve ser validado contra a mesma pela detentora de modo a aferir se o momento do pagamento está em conformidade com o aprovado pelo usuário final no momento do consentimento.
4. Pagamentos mal sucedidos por qualquer motivo não invalidam o consentimento ou impactam os próximos pagamentos.
5. Todo o pagamento mal sucedido pode ser refeito até a data em que o pagamento foi agendado terminar. (Sugestão de periodicidade: A cada 2 horas)
6. Uma rentativa de pagamento deve ser negada caso haja outro pagamento qualquer para o consentimento alvo no mesmo dia com status diferente de **REJECTED**.
7. O último pagamento agendado, se bem-sucedido, deve marcar o consentimento como **consumed**.
8. Todo consentimento de pagamentos agendados/recorrentes deverá ser marcado como **consumed** caso a sua data de expiração seja ultrapassada.

## Políticas de agendamento

Refletindo as discussões no grupo de trabalho sobre o tema de agendamento e recorrência de pagamentos de valor fixo,  
fora entendido que um modelo de dados que permitisse a conclusão de datas de pagamento a partir de parâmetros e não de datas explícitas seria o  
mais adequado para maioria dos casos de uso. Além disso, verificou-se a necessidade que esses parâmetros deveriam expor de forma inequívoca, tanto para as iniciadoras quanto para as detentoras,  
como a recorrência de pagamentos deveria interpretada.
Com isso os modelos propostos foram revistos para estarem aderentes ao exposto além de permitir um maior nível de escolha pelos membros do grupo de trabalho  
quais *features* poderão ser adicionadas.

### Especificações

O campo **schedule** deverá ser adicionado ao payload de consentimento no objeto payment que hoje já compõe o mesmo.
Ele será um objeto onde a sua estrutura será descrita posteriormente neste documento.
Este campo será mutuamente exclusivo contra o campo **date** hoje já presente no objeto payment.
O campo **date** só será usado para pagamentos normais, ou seja, não agendados/recorrentes.
Caso os dois campos estejam presentes simultaneamente no payload da requisição, a detentora deve devolver uma **resposta HTTP 400**.

Observe o exemplo abaixo

### Exemplo de fragmento do payload de consentimento para pagamentos atual

```
{
  "data": {
    "payment": {
      "type": "PIX",
      "date": "2021-01-01",
      "currency": "BRL",
      "amount": "100000.12"
    }
  }
}

```
### Exemplo de fragmento do payload de consentimento para pagamentos agendados/recorrentes

```{
   "data":{
      "payment":{
         "type":"PIX",
         "currency":"BRL",
         "amount":"100000.12",
         "schedule":{
            "single":{
               "date":"2035-01-01"
            }
         }
      }
   }
}
```

## Tipos de políticas de agendamento 

Quatro tipos de políticas de agendamento foram definidas com base nas discussões do grupo de trabalho:

1. Pagamento único
2. Pagamento recorrente com frequência fixa
3. Pagamento recorrente por repetição
4. Pagamento recorrente por configuração *customizada*

Todas as políticas de agendamento são mutuamente exclusivas entre si, ou seja, não podem coabitar a definição de um mesmo payload de consentimento para pagamentos simultaneamente.  
Caso isso ocorra a detentora deve retornar a **resposta HTTP 400** para a iniciadora.

### Política de agendamento de pagamento único ###

Essa política determina o agendamento de um pagamento único numa data futura.
O consentimento neste caso deve ter o campo **expirationDateTime** definido para a última hora, minuto e segundo do dia agendado.   
Ex: dia do agendamento = "2021-01-01" então o **expirationDateTime** seria "2021-01-01T23:59:59Z".   

#### Especificação ####

```
{
   "data":{
      "payment":{
         "type":"PIX",
         "currency":"BRL",
         "amount":"100000.12",
         "schedule":{
            "single":{
               "date":"2035-01-01"
            }
         }
      }
   }
}
```

A política de agendamento de pagamento único é introduzida pelo campo de nome **single** no objeto do campo **schedule** conforme mostrado no exemplo acima.  
Ela define a seguinte estrutura:

**Campos**  
  
1. **date** : **tipo**: string(date), **obrigatório**, **descrição**: Data do agendamento do pagamento. Essa data deve ser futura em relação ao dia do consentimento. 

Para o contexto de "UX", a intenção dessa política pode ser expressa para usuário final da seguinte forma: "Pagamento agendado para o dia {date}"

### Política de agendamento de pagamento recorrente com frequência fixa ###

```
{
   "data":{
      "payment":{
         "type":"PIX",
         "currency":"BRL",
         "amount":"100000.12",
         "schedule":{
            "fixed":{
               "delay":"2022-08-10",
               "frequency":2,
               "count":3
            }
         }
      }
   }
}
```

Essa política determina **uma quantidade de pagamentos a ser realizada distribuída em uma frequência de datas futuras determinadas em dias corridos a partir de uma data inicial**.  
Esta política pode conter um campo de *delay* para determinar a partir de quando no futuro a recorrência iniciará.  
Caso não definido, **o dia atual do consentimento será assumido**.    
O consentimento neste caso deve ter o campo **expirationDateTime** definido para última hora, minuto e segundo da última recorrência  
pela fórmula: **delay + dias_corridos(count * frequency)** .    
Com exemplo de payload mostrado acima seria: 2022-08-10 + 2 * 3 = "2022-08-16T23:59:59Z"  
No exemplo teremos pagamentos em: 2022-08-12, 2022-08-14 e 2022-08-16.   

A política de agendamento de pagamento recorrente com frequência fixa é introduzida pelo campo de nome **fixed** no objeto do campo **schedule** conforme mostrado no exemplo acima.  
Ela define a seguinte estrutura:

**Campos**
  1. **delay** : **tipo**: string(date), **opcional**, **valor padrão**: data atual do consentimento, **descrição**: Data de onde irá começar a contar a recorrência de pagamentos. Essa data deve ser futura ou igual ao dia do consentimento em questão.
  2. **frequency**: **tipo**: inteiro, **obrigatório**, **valor mínimo: 2**, **valor máximo**: 365, **descrição**: Frequência em dias corridos da recorrência de pagamentos.
  3. **count**: **tipo**: inteiro, **obrigatório**, **valor mínimo: 2**, **valor máximo**: 200, **descrição**: Quantidade de pagamentos a ser realizada a partir da data inicial de recorrência.

Para o contexto de "UX", a intenção dessa política pode ser expressa para usuário final da seguinte forma: "Pagamento recorrente agendado a cada {frequency} dias a partir da data {delay}"


### Política de agendamento de pagamento recorrente por repetição ###


```
{
   "data":{
      "payment":{
         "type":"PIX",
         "currency":"BRL",
         "amount":"100000.12",
         "schedule":{
            "iteration":{
               "delay":"2022-08-19",
               "day_of_month":18,
               "count":3
            }
         }
      }
   }
}
```

Essa política define **uma quantidade específica de pagamentos a serem feitos distribuídos no tempo sempre no mesmo dia a cada mês a partir de uma data inicial.**  
Esta política pode conter um campo de *delay* para determinar a partir de quando no futuro a recorrência iniciará.  
Caso não definido, **o dia posterior ao dia atual do consentimento será assumido** com intuito de não acontecer pagamento no mesmo dia do consentimento.  
O consentimento neste caso deve ter o campo **expirationDateTime** definido para última hora, minuto e segundo da última recorrência  
pela fórmula: **proximo_dia_do_mes_a_partir(delay, day_of_month) + meses(count)** .       
Com exemplo de payload mostrado acima seria: 
data := proximo_dia_do_mes_a_partir(delay, day_of_month) = 2022-09-18  
data + 3meses = "2022-11-18T23:59:59Z"  
No exemplo teremos pagamentos em: 2022-09-18, 2022-10-18 e 2022-11-18.  
**Em caso da combinação de day_of_month de valor 29 com períodos passando no mês de fevereiro o dia 28 deve ser assumido como data de pagamento em anos não bissextos.**

A política de agendamento de pagamento recorrente por repetição é introduzida pelo campo de nome **iteration** no objeto do campo **schedule** conforme mostrado no exemplo acima.  
Ela define a seguinte estrutura:

**Campos**
1. **delay** : **tipo**: string(date), **opcional**, **valor padrão**: data posterior corrida a data atual do consentimento, **descrição**: Data de onde irá começar a contar a recorrência de pagamentos. Essa data deve ser futura ao dia do consentimento em questão.
2. **day_of_month**: **tipo**: inteiro, **obrigatório**, **valor mínimo: 1**, **valor máximo**: 31, **descrição**: Dia do mês em que cada pagamento irá ocorrer. 
3. **count**: **tipo**: inteiro, **obrigatório**, **valor mínimo: 2**, **valor máximo**: 200, **descrição**: Quantidade de pagamentos a ser realizada a partir da data inicial de recorrência.


Para o contexto de "UX", a intenção dessa política pode ser expressa para usuário final da seguinte forma: "Pagamento recorrente agendado para todo dia {day_of_month} a partir da data {delay}"

### Política de agendamento recorrente por configuração customizada ###

```
{
   "data":{
      "payment":{
         "type":"PIX",
         "currency":"BRL",
         "amount":"100000.12",
         "schedule":{
            "custom":{
               "dates":[
                  "2022-08-10",
                  "2022-09-18",
                  "2022-10-26"
               ]
            }
         }
      }
   }
}
```

Essa política **define que seja descrita de forma explicita cada momento no futuro quando os pagamentos serão feitos.** 
Todas as datas informadas devem únicas e futuras. A quantidade máxima de datas deve ser 50 e a minima 2.
A detentora deve ordenar essas datas da menor para a maior antes de armazenar e devolver a iniciadora o consentimento criado.
O consentimento neste caso deve ter o campo **expirationDateTime** definido para última hora, minuto e segundo da última recorrência  
definido pela maior data informada.  
No exemplo a maior data informada seria 2022-10-26 então o **expirationDateTime** deste consentimento seria "2022-10-26T23:59:59Z"  

A política de agendamento de pagamento recorrente por configuração *customizada* é introduzida pelo campo de nome **custom** no objeto do campo **schedule** conforme mostrado no exemplo acima.  
Ela define a seguinte estrutura:

**Campos**
1. **dates** : **tipo**: array de string(date), **obrigatório**, **quantidade minina de elementos**: 2, **quantidade máxima de elementos**: 50, **descrição**: Lista de datas de recorrência de pagamentos. Todas as datas presentes no array devem únicas e futuras.

Para o contexto de "UX", a intenção dessa política pode ser expressa para usuário final da seguinte forma: "Pagamento recorrente agendado para os dias {dates}"

### Erros de resposta

**HTTP 422 (Schema: https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_422ResponseErrorCreateConsent)**
1. Introdução do valor: **INVALID_SCHEDULE** no enumerado [422ResponseErrorCreateConsent](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_422ResponseErrorCreateConsent) usado no campo **"code"** já presente no payload de resposta para quando qualquer validação de negócio falhar.
2. Introdução da mensagem: **"Agendamento inválido."** no campo **"title"** caso o campo **"code"** tenha o valor definido no **item 1 desta lista**.
3. Introdução da mensagem: **"Agendamento inválido."** no campo **"details"** caso o campo **"code"** tenha o valor definido no **item 1 desta lista**.

## Alterações no endpoint de criação de pagamentos

### Respostas ###
**HTTP 422 (Schema: https://openbanking-brasil.github.io/areadesenvolvedor/#schema422responseerrorcreatepixpayment)**
1. Introdução do valor: **"PAYMENT_RETRY_NOT_ALLOWED"** no enumerado [EnumErrorsCreatePayment](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_EnumErrorsCreatePayment) usado no campo **"code"** já presente no payload de resposta para este tipo de erro.
2. Introdução da mensagem: **"Retentativa de pagamento não permitida."** no campo **"title"** caso o campo **"code"** tenha o valor definido no **item 1 desta lista**.
3. Introdução da mensagem: **"Retentativa de pagamento não permitida, pois já existe um pagamento liquidado ou em processo de liquidação no mesmo dia."** no campo **"details"** caso o campo **"code"** tenha o valor definido no **item 1 desta lista**.


## Revogação de consentimento para pagamentos agendados

Em consequência do aumento do tempo de vida do consentimento por conta do agendamento em até um ano, será requerida uma forma de revogação do mesmo pelos usuários finais.
Para alcançar este objetivo um novo **"status"** no consentimento será criado de valor: **REVOKED** .
Este **status não pode ser revertido** e a partir da sua definição em consentimento nenhum outro pagamento poderá ser feito para o mesmo.
Além disso, o token de acesso vinculado ao consentimento alvo também deve ser revogado.

### Regras de negócio ###

1. O consentimento só pode ser revogado se o mesmo estiver no status **AUTHORISED**.
2. O consentimento só pode ser revogado 1 dia corrido antes de um pagamento agendado para o mesmo.

### Especificação ###

A intenção do usuário final de revogar um consentimento deverá ser expressa através de um novo endpoint na api de pagamentos no formato descrito abaixo:

**PATCH /payments/v1/consents/{consentId}**

**Payload**

Exemplo

```
{
   "data":{
      "status":"REVOKED"
   }
}
```

Para realizar a alteração deverá ser enviado um objeto como descrito no exemplo acima.

**Campos**

1. **data** : **Tipo**: objeto, **obrigatório**, **descrição**: Objeto contendo as informações de alteração do consentimento.
    1. **status**: **Tipo**: [EnumAuthorisationStatusType](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_EnumAuthorisationStatusType) , **obrigatório**, **Descrição**: Indica para qual status o consentimento deve progredir. Este campo só deve permitir o valor **"REVOKED"**.

#### Parâmetros ####

1. **consentId** : **Origem**: path, **tipo**: string, **obrigatório**, **descrição**: O consentId é o identificador único do consentimento e deverá ser um URN - Uniform Resource Name.
2. **Authorization** : **Origem**: header, **tipo**: string, **obrigatório**, **descrição**: Cabeçalho HTTP padrão. Permite que as credenciais sejam fornecidas dependendo do tipo de recurso solicitado
3. **x-fapi-auth-date** : **Origem**: header, **tipo**: string, **opcional**, **descrição**: Data em que o usuário logou pela última vez com o receptor. Representada de acordo com a RFC7231.Exemplo: Sun, 10 Sep 2017 19:43:31 UTC
4. **x-fapi-customer-ip-address**: **Origem**: header, **tipo**: string, **opcional**, **descrição**: O endereço IP do usuário se estiver atualmente logado com o receptor.
5. **x-fapi-interaction-id** : **Origem**: header, **tipo**: string, **opcional**, **descrição**: Um UID RFC4122 usado como um ID de correlação. Se fornecido, o transmissor deve "reproduzir" esse valor no cabeçalho de resposta.
6. **x-customer-user-agent** : **Origem**: header, **tipo**: string, **opcional**, **descrição**: Indica o user-agent que o usuário utiliza.

#### Respostas ####

1. **HTTP 200** : Indica que a revogação do consentimento alvo foi realizada com sucesso.  
   **Response** : https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_ResponsePaymentConsent .
2. **HTTP 422** : A solicitação foi bem formada, mas não pôde ser processada devido à lógica de negócios específica da solicitação.  
   **Response** : https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_422ResponseErrorCreateConsent  
   2.1. Deve ser incluído o valor **OPERATION_NOT_ALLOWED_BY_STATUS** no enum [EnumErrorsCreateConsent](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_EnumErrorsCreateConsent) para representar que a ação atual não é permitida para o status atual do consentimento. Neste caso os campos **"title"** e **"details"** deverão ser preenchidos com a mensagem: **"Operação atual não permitida para o status atual do consentimento alvo."**   
   2.2  Deve ser incluído o valor **NEXT_SCHEDULED_PAYMENT_TOO_CLOSE** no enum [EnumErrorsCreateConsent](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_EnumErrorsCreateConsent) para representar que a revogação não foi atendida por que há um pagamento agendado a menos de um dia do momento do pedido de revogação. Neste caso os campos **"title"** e **"details"** deverão ser preenchidos com a mensagem: **"Não foi possível realizar a revogação do consentimento por que há um pagamento agendado para o próximo dia."**

## Controle de andamento de modificações no consentimento

Por conta da funcionalidade de revogação do consentimento pelo usuário final direto na detentora de conta será necessária
uma forma da iniciadora se manter a iniciadora ciente de modificações no consentimento.  
Para atender essa necessidade duas possibilidades técnicas são possíveis: Pooling ou Webhooks para um modelo de push.  
Dado que a infraestrutura, requisitos de segurança, mecanismo de retentativas e outras questões necessárias para viabilizar webhooks são complexas, esta proposta sugere a solução por pooling agora visando o prazo.  
Hoje a única forma de obter as informações atuais do consentimento é através de um endpoint usando id do mesmo.
Desta forma as iniciadoras ou são serão muito cerceadas na quantidade de requisições possíveis, ou darão um overhead computacional grande nas detentoras devido ao nível de granularidade da busca atual.   
Essa proposta vem sugerir um novo endpoint para ser usado de pooling que tanto servirá para atender o contexto da revogação como os casos atuais de
acompanhamento de mudança de estado do consentimento. Do ponto de vista de autenticação ele usará o mesmo mecanismo de *client credentials* hoje presente no endpoint de busca do pagamento.  
Os consentimentos retornados por esta api tem que estar filtrados pelo **clientId** conseguido na camada de segurança.  
A ideia do endpoint em /consents com query parameters não tem o objetivo neste momento de estabelecer uma listagem de propósito geral de consentimentos, pois teríamos que entender quais critérios de filtro deveriam ser adicionados contra as necessidades computacionais de busca e armazenamento de dados das detentoras e prevenção de abusos.
Neste momento, os parâmetros escolhidos visam a atender apenas a conseguir monitorar as mudanças de status dos consentimentos ao longo do tempo com uma granularidade maior do que o praticado hoje.



### Especificação

**GET /payments/v1/consents?status={status}&status-update-datetime-from={dateFrom}&status-update-datetime-to{dateTo}&page=1&page-size=25**

#### Parâmetros ####

1. **dateFrom**: **Origem**: query, **tipo**: string(date-time), **obrigatório**, **descrição**: Filtra consentimentos com `statusUpdateDateTime` maior ou igual ao informado. Uma string com data e hora conforme especificação RFC-3339, sempre com a utilização de timezone UTC(UTC time format). O intervalo de tempo entre este campo e o campo **modifiedBefore** deve ser no máximo de 30 minutos
2. **dateTo**: **Origem**: query, **tipo**: string(date-time), **obrigatório**, **descrição**: Filtra consentimentos com `statusUpdateDateTime` menor ou igual ao informado. Uma string com data e hora conforme especificação RFC-3339, sempre com a utilização de timezone UTC(UTC time format). O intervalo de tempo entre este campo e o campo **modifiedAfter** deve ser no máximo de 30 minutos
5. **status**: **Origem**: query, **tipo**: [EnumAuthorisationStatusType](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_EnumAuthorisationStatusType), **obrigatório**, **descrição**: Filtra consentimentos com status igual informado.
6. **page**: **Origem**: query, **tipo**: inteiro, **opcional**, **valor padrão**: 1, **descrição**: Número da página de dados do universo resposta retornado. [Paginação](https://openbanking-brasil.github.io/areadesenvolvedor/#paginacao)
7. **page-size**: **Origem**: query, **tipo**: inteiro, **opcional**, **valor padrão**: 25, **descrição**: Número máximo de elementos em cada página de dados recebida. [Paginação](https://openbanking-brasil.github.io/areadesenvolvedor/#paginacao)
8. **Authorization** : **Origem**: header, **tipo**: string, **obrigatório**, **descrição**: Cabeçalho HTTP padrão. Permite que as credenciais sejam fornecidas dependendo do tipo de recurso solicitado
9. **x-fapi-auth-date** : **Origem**: header, **tipo**: string, **opcional**, **descrição**: Data em que o usuário logou pela última vez com o receptor. Representada de acordo com a RFC7231.Exemplo: Sun, 10 Sep 2017 19:43:31 UTC
10. **x-fapi-customer-ip-address**: **Origem**: header, **tipo**: string, **opcional**, **descrição**: O endereço IP do usuário se estiver atualmente logado com o receptor.
11. **x-fapi-interaction-id** : **Origem**: header, **tipo**: string, **opcional**, **descrição**: Um UID RFC4122 usado como um ID de correlação. Se fornecido, o transmissor deve "reproduzir" esse valor no cabeçalho de resposta.
12. **x-customer-user-agent** : **Origem**: header, **tipo**: string, **opcional**, **descrição**: Indica o user-agent que o usuário utiliza.

## Respostas ##

**HTTP 200** : Busca realizada com sucesso.  
Campos:
1. **consents**: array de elementos do tipo: [ResponsePaymentConsent](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_ResponsePixPayment), obrigatório, mínimo de 0 elementos .
2. **links**: objeto do tipo: [Links](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_LoansBalloonPayment), obrigatório .
3. **meta**: objeto do tipo: [Meta](https://openbanking-brasil.github.io/areadesenvolvedor/#schemameta), obrigatório.  
   Exemplo:
```
{
   "data":[
        {
          "consentId":"urn:bancoex:C1DD33123",
          "creationDateTime":"2021-05-21T08:30:00Z",
          "expirationDateTime":"2021-05-21T08:30:00Z",
          "statusUpdateDateTime":"2021-05-21T08:30:00Z",
          "status":"AWAITING_AUTHORISATION",
          "loggedUser":{
             "document":{
                "identification":"11111111111",
                "rel":"CPF"
             }
          },
          "businessEntity":{
             "document":{
                "identification":"11111111111111",
                "rel":"CNPJ"
             }
          },
          "creditor":{
             "personType":"PESSOA_NATURAL",
             "cpfCnpj":"58764789000137",
             "name":"Marco Antonio de Brito"
          },
          "payment":{
             "type":"PIX",
             "date":"2021-01-01",
             "currency":"BRL",
             "amount":"100000.12",
             "details":{
                "localInstrument":"DICT",
                "qrCode":"00020104141234567890123426660014BR.GOV.BCB.PIX014466756C616E6F32303139406578616D706C652E636F6D27300012  \nBR.COM.OUTRO011001234567895204000053039865406123.455802BR5915NOMEDORECEBEDOR6008BRASILIA61087007490062  \n530515RP12345678-201950300017BR.GOV.BCB.BRCODE01051.0.080450014BR.GOV.BCB.PIX0123PADRAO.URL.PIX/0123AB  \nCD81390012BR.COM.OUTRO01190123.ABCD.3456.WXYZ6304EB76\n",
                "proxy":"12345678901",
                "creditorAccount":{
                   "ispb":"12345678",
                   "issuer":"1774",
                   "number":"1234567890",
                   "accountType":"CACC"
                }
             }
          },
          "debtorAccount":{
             "ispb":"12345678",
             "issuer":"1774",
             "number":"1234567890",
             "accountType":"CACC"
          }
      },
      {
          "consentId":"urn:bancoex:C1DD33908",
          "creationDateTime":"2021-05-21T08:30:00Z",
          "expirationDateTime":"2021-05-21T08:30:00Z",
          "statusUpdateDateTime":"2021-05-21T08:30:00Z",
          "status":"AWAITING_AUTHORISATION",
          "loggedUser":{
             "document":{
                "identification":"11111111111",
                "rel":"CPF"
             }
          },
          "businessEntity":{
             "document":{
                "identification":"11111111111111",
                "rel":"CNPJ"
             }
          },
          "creditor":{
             "personType":"PESSOA_NATURAL",
             "cpfCnpj":"58764789000137",
             "name":"Marco Antonio de Brito"
          },
          "payment":{
             "type":"PIX",
             "date":"2021-01-01",
             "currency":"BRL",
             "amount":"100000.12",
             "details":{
                "localInstrument":"DICT",
                "qrCode":"00020104141234567890123426660014BR.GOV.BCB.PIX014466756C616E6F32303139406578616D706C652E636F6D27300012  \nBR.COM.OUTRO011001234567895204000053039865406123.455802BR5915NOMEDORECEBEDOR6008BRASILIA61087007490062  \n530515RP12345678-201950300017BR.GOV.BCB.BRCODE01051.0.080450014BR.GOV.BCB.PIX0123PADRAO.URL.PIX/0123AB  \nCD81390012BR.COM.OUTRO01190123.ABCD.3456.WXYZ6304EB76\n",
                "proxy":"12345678901",
                "creditorAccount":{
                   "ispb":"12345678",
                   "issuer":"1774",
                   "number":"1234567890",
                   "accountType":"CACC"
                }
             }
          },
          "debtorAccount":{
             "ispb":"12345678",
             "issuer":"1774",
             "number":"1234567890",
             "accountType":"CACC"
          }
       }
   ],
   "links":{
      "self":"https://api.banco.com.br/open-banking/payments/v1/consents?status=AWAITING_AUTHORISATION&status-update-datetime-from=2021-05-21T08:30:00Z&status-update-datetime-to=2021-05-21T08:35:00Z&page=1&page-size=25",
      "next":"https://api.banco.com.br/open-banking/payments/v1/consents?status=AWAITING_AUTHORISATION&status-update-datetime-from=2021-05-21T08:30:00Z&status-update-datetime-to=2021-05-21T08:35:00Z&page=2&page-size=25",
      "last":"https://api.banco.com.br/open-banking/payments/v1/consents?status=AWAITING_AUTHORISATION&status-update-datetime-from=2021-05-21T08:30:00Z&status-update-datetime-to=2021-05-21T08:35:00Z&page=200&page-size=25"   },
   "meta":{
      "totalRecords":2,
      "totalPages":200,
      "requestDateTime":"2021-05-21T08:30:00Z"
   }
}
```
