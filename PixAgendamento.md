# Indices
- [Agendamento de iniciação de pagamentos](#agendamento-de-iniciação-de-pagamentos)
    - [Fluxo de agendamento de pagamentos](#fluxo-de-agendamento-de-pagamentos)
- [Ciclo de vida das entidades da iniciação de pagamentos](#ciclo-de-vida-das-entidades-da-iniciação-de-pagamentos)
- [Cancelamento de pagamento agendado](#cancelamento-de-pagamento-agendado)
- [Alterações no endpoint de criação de pagamentos](#alterações-no-endpoint-de-criação-de-pagamentos)
- [Controle de andamento de modificações no pagamento](#controle-de-andamento-de-modificações-no-pagamento)
- [Políticas de agendamento](#políticas-de-agendamento)
  - [Pagamento único](#política-de-agendamento-de-pagamento-único)
  - [Pagamento recorrente com frequência fixa](#política-de-agendamento-de-pagamento-recorrente-com-frequência-fixa)
  - [Pagamento recorrente por repetição](#política-de-agendamento-de-pagamento-recorrente-por-repetição)
  - [Pagamento recorrente por configuração *customizada*](#política-de-agendamento-recorrente-por-configuração-customizada)


# Agendamento de iniciação de pagamentos

Para possibilitar o agendamento único de pagamentos iniciados pelo Open Banking (OB) seria necessário a inclusão do conceito de "política de agendamento" no consentimento.  
A proposta aqui apresentada tem o canal OB sendo dependente do arranjo do produto final (Pix, TED, TEF, débito em conta) sendo chamado, pois, as regras/funcionalidade deles determinam como o OB é definido.  
Toda a especificação aqui proposta tem como diretriz base que a detentora seja responsável por zelar pelo agendamento conforme deliberado por votação do grupo de trabalho.  
Diante de todo o contexto mencionado é possível utilizar o próprio mecanismo de agendamento do Pix já definido para as detentoras de conta que são PSPs diretos/indiretos.  
Desse modo o cliente final participaria de toda a experiência que lhe é familiar para essa modalidade de pagamento como multiplas alçadas, extrato com lançamentos futuros, cancelamento de pagamento, etc.  

## Fluxo de agendamento de pagamentos

O fluxo de agendamento se dará a partir de um consentimento é criado normalmente com todas as regras do modelo atual e em seguida será solicitado a criação do pagamento agendado requisitado pela iniciadora.
O diagrama abaixo ilustra como se daria o fluxo de agendamento no contexto proposto.  
Vale lembrar que os fluxos no diagrama não são exaustivos, ou seja, não mostram todas as possibilidades de interação e arquitetura de solução.     

![Fluxo de agendamento!](agendamento-de-pagamentos.png)

**Descrição do fluxo:**  
1. O usuário requisita o agendamento de pagamento junto a iniciadora derivado de algum fluxo de negócio entre eles
2. A iniciadora cria o consentimento do agendamento do pagamento junto a detentora
3. O usuário aprova o consentimento
4. A iniciadora solicita a detentora a criação do pagamento agendado
5. A detentora realiza as validações de negócio do consentimento contra o pagamento
6. A detentora cria o PIX agendado nos seus sistemas atuais que realizam este trabalho
7. A detentora marca o consentimento como consumido
8. A iniciadora inicia uma iteração (pooling) junto a detentora para acompanhar o ciclo de vida do pagamento
9. No dia agendado a detentora realiza a liquidação do PIX junto ao Bacen
10. A detentora atualiza o status do pagamento conforme o que aconteceu na execução do pagamento
11. A iniciadora termina a iteração de acompanhamento do pagamento

O fluxo proposto tem como vantagens a não mudança da forma interação entre iniciadora e detentora hoje definido
além de já estar preparado para futuras adições de informação no pagamento de cunho único como, por exemplo, o **endToEndId**.
Com este fluxo serão necessárias adaptações futuras para suportar recorrência de pagamentos de valor fixo/variável também
incluído a questão de dados únicos mencionado anteriormente. Seria necessária ou a adaptação do endpoint atual para receber uma lista de pagamentos
ou implementar algum controle para multiplas requisições de pagamentos relacionadas a um mesmo consentimento.

# Ciclo de vida das entidades da iniciação de pagamentos

Para dar suporte a perspectiva de pagamentos mencionada anteriormente se torna necessária a expansão do ciclo de vida do pagamento tendo a noção clara de agendamento.
Com isso dois novos status listados abaixo deverão ser adicionados ao pagamento.
1. **SASP (SCHEDULE_ACCEPTED_SETTLEMENT_IN_PROCESS)** : Processo de agendamento de iniciação de pagamento aceita e em processamento
2. **SASC (SCHEDULE_ACCEPTED_SETTLEMENT_COMPLETED)** : Processo de agendamento de iniciação de pagamento concluído

Para suportar o cancelamento de um pagamento agendado deverá ser adicionado o novo valor: **CANCELED_BY_USER** para o enumerado [EnumRejectionReasonType](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_EnumPaymentType).  
O **consentimento** deverá ser marcado como **CONSUMED** ao fim do processo de agendamento seja ele bem-sucedido ou não.

![Ciclo de vida do pagamento](ciclo-vida-agendamento-de-pagamentos.png)

# Cancelamento de pagamento agendado

O OB deve permitir que os seus usuários cancelem agendamentos de pagamentos ainda não efetivados.  
As regras sobre limite de tempo para isso acontecer ou a quantidade de retentativas em casos de erro na efetivação dos pagamentos não são definidas pelo arranjo do Pix
ficando a cargo de cada instituição definir a sua política. 
De qualquer forma o usuário final já possui o conhecimento dessas políticas, pois já interage com elas nos fluxos diretos de PIX.  
Há duas formas de cancelamento de agendamento de pagamentos possíveis: a primeira na detentora e a outra na iniciadora.

## Cancelamento de pagamento na detentora
Nesta modalidade o usuário irá cancelar direto na detentora o pix agendado como já pode fazer atualmente.
Neste caso, com o cancelamento bem-sucedido, a detentora apenas deve ser capaz de atualizar o status do pagamento para **RJCT(REJECTED)** como previsto no ciclo de vida proposto no agendamento com 
o rejectionReason **CANCELED_BY_USER** .  

## Cancelamento de pagamento na iniciadora
Nesta modalidade o usuário irá solicitar o cancelamento do pagamento agendado através da iniciadora.  
A detentora deverá só permitir essa operação caso o pagamento esteja no status: **SASC** . Em caso de violação dessa regra a resposta 422 deverá ser devolvida com 
o campo **code** com o valor **OPERATION_NOT_ALLOWED_BY_STATUS**.  
Como não há uma regra uniforme definida no arranjo do PIX de limite de tempo para o cancelamento fica a cargo
de cada instituição verificar as suas políticas. Caso esse limite de tempo seja violado a detentora deverá negar o pedido de cancelamento
com a resposta 422 com o campo code **code** com o valor **CANCELLATION_TIME_LIMIT_EXCEEDED**.    
Caso o cancelamento seja feito com o sucesso a detentora deverá atualizar o status do pagamento para **RJCT(REJECTED)** como previsto no ciclo de vida proposto no agendamento com
o rejectionReason **CANCELED_BY_USER** . 
Para isso ser possível será necessário a criação do endpoint sugerido abaixo:

**PATCH - /payments/v1/pix/payments/{paymentId}**

**Payload**

Exemplo  

```
{
   "data":{
      "status":"RJCT"
   }
}
```

Para realizar a alteração deverá ser enviado um objeto como descrito no exemplo acima.

**Campos**

1. **data** : **Tipo**: objeto, **obrigatório**, **descrição**: Objeto contendo as informações de alteração do pagamento.
    1. **status**: **Tipo**: [EnumPaymentStatusType](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_EnumPaymentStatusType) , **obrigatório**, **Descrição**: Indica para qual status o pagamento deve progredir. Este campo só deve permitir o valor **"RJCT"**.

#### Parâmetros ####

1. **paymentId** : **Origem**: path, **tipo**: string, **obrigatório**, **descrição**: O paymentId é o identificador único do pagamento.
2. **Authorization** : **Origem**: header, **tipo**: string, **obrigatório**, **descrição**: Cabeçalho HTTP padrão. Permite que as credenciais sejam fornecidas dependendo do tipo de recurso solicitado
3. **x-fapi-auth-date** : **Origem**: header, **tipo**: string, **opcional**, **descrição**: Data em que o usuário logou pela última vez com o receptor. Representada de acordo com a RFC7231.Exemplo: Sun, 10 Sep 2017 19:43:31 UTC
4. **x-fapi-customer-ip-address**: **Origem**: header, **tipo**: string, **opcional**, **descrição**: O endereço IP do usuário se estiver atualmente logado com o receptor.
5. **x-fapi-interaction-id** : **Origem**: header, **tipo**: string, **opcional**, **descrição**: Um UID RFC4122 usado como um ID de correlação. Se fornecido, o transmissor deve "reproduzir" esse valor no cabeçalho de resposta.
6. **x-customer-user-agent** : **Origem**: header, **tipo**: string, **opcional**, **descrição**: Indica o user-agent que o usuário utiliza.

#### Respostas ####

1. **HTTP 200** : Indica que o cancelamento do consentimento alvo foi realizada com sucesso.  
   **Response** : https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_ResponsePixPayment .
2. **HTTP 422** : A solicitação foi bem formada, mas não pôde ser processada devido à lógica de negócios específica da solicitação.  
   **Response** : https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_422ResponseErrorCreatePixPayment  
   2.1. Deve ser incluído o valor **OPERATION_NOT_ALLOWED_BY_STATUS** no enum [EnumErrorsCreateConsent](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_EnumErrorsCreatePayment) para representar que a ação atual não é permitida para o status atual do pagamento. Neste caso os campos **"title"** e **"details"** deverão ser preenchidos com a mensagem: **"Operação atual não permitida para o status atual do pagamento alvo."**   
   2.2  Deve ser incluído o valor **CANCELLATION_TIME_LIMIT_EXCEEDED** no enum [EnumErrorsCreateConsent](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_EnumErrorsCreatePayment) para representar que o cancelamento não pode ser feito por conta do limite de tempo para isso ter sido ultrapassado . Neste caso os campos **"title"** e **"details"** deverão ser preenchidos com a mensagem: **"Limite de tempo para cancelamento do pagamento ultrapassado."**

## Alterações no endpoint de criação de pagamentos

Para suportar o agendamento de pagamentos deverá ser necessário saber quando o pagamento será executado.  
Com isso será necessário a inclusão de um campo de nome **date** no pagamento do tipo: **string(date)** sendo obrigatório.  
Esse novo campo deverá constar tanto na requisição: [CreatePixPaymentData](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_CreatePixPaymentData) 
quanto na resposta: [ResponsePixPaymentData](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_ResponsePixPaymentData) do endpoint de criação de pagamento.  
Para pagamentos agendados o valor deverá ser a data futura do pagamento. Para pagamentos normais deverá ser a própria data corrente.  
Este campo também será útil em futuras pesquisas de pagamentos em intervalos de datas sem considerar horário como expresso no campo **creationDateTime** hoje retornado na resposta da criação do pagamento.
No momento da criação do pagamento agendado a detentora deverá verificar a data informada no pagamento está de acordo com consentimento. Caso a regra seja violada a detentora deverá responder com o código HTTP 422
com o campo **code** com o valor **INCONSISTENT_PAYMENT_DATE** .  

### Exemplo de payload modificado ###

```
{
   "localInstrument":"DICT",
   "date":"2022-12-10",
   "payment":{
      "amount":"100000.12",
      "currency":"BRL"
   },
   "creditorAccount":{
      "ispb":"12345678",
      "issuer":"1774",
      "number":"1234567890",
      "accountType":"CACC"
   },
   "remittanceInformation":"Pagamento da nota XPTO035-002.",
   "qrCode":"00020104141234567890123426660014BR.GOV.BCB.PIX014466756C616E6F32303139406578616D706C652E636F6D27300012\nBR.COM.OUTRO011001234567895204000053039865406123.455802BR5915NOMEDORECEBEDOR6008BRASILIA61087007490062\n530515RP12345678-201950300017BR.GOV.BCB.BRCODE01051.0.080450014BR.GOV.BCB.PIX0123PADRAO.URL.PIX/0123AB\nCD81390012BR.COM.OUTRO01190123.ABCD.3456.WXYZ6304EB76\n",
   "proxy":"12345678901",
   "cnpjInitiator":"50685362000135",
   "transactionIdentification":"E00038166201907261559y6j6"
}
```

### Respostas ###
**HTTP 422 (Schema: https://openbanking-brasil.github.io/areadesenvolvedor/#schema422responseerrorcreatepixpayment)**
1. Introdução do valor: **"INCONSISTENT_PAYMENT_DATE"** no enumerado [EnumErrorsCreatePayment](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_EnumErrorsCreatePayment) usado no campo **"code"** já presente no payload de resposta para este tipo de erro.
2. Introdução da mensagem: **"Data de pagamento inconsistente."** no campo **"title"** caso o campo **"code"** tenha o valor definido no **item 1 desta lista**.
3. Introdução da mensagem: **"Data de pagamento inconsistente com relação ao definido no consentimento."** no campo **"details"** caso o campo **"code"** tenha o valor definido no **item 1 desta lista**.

# Controle de andamento de modificações no pagamento

Para a iniciadora acompanhar o andamento dos status do pagamento ela deverá utilizar o mecanismo que hoje ela já usa para fazê-lo.
O mecanismo consiste em realizar um pooling no endpoint de busca do pagamento até o pagamento alcançar algum status não final.

# Políticas de agendamento

Refletindo as discussões no grupo de trabalho sobre o tema de agendamento e recorrência de pagamentos de valor fixo,  
fora entendido que um modelo de dados que permitisse a conclusão de datas de pagamento a partir de parâmetros e não de datas explícitas seria o  
mais adequado para maioria dos casos de uso. Além disso, verificou-se a necessidade que esses parâmetros deveriam expor de forma inequívoca, tanto para as iniciadoras quanto para as detentoras,  
como a recorrência de pagamentos deveria interpretada.
Com isso os modelos propostos foram revistos para estarem aderentes ao exposto além de permitir um maior nível de escolha pelos membros do grupo de trabalho  
quais *features* poderão ser adicionadas.

## Especificações

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

### Tipos de políticas de agendamento 

Quatro tipos de políticas de agendamento foram definidas com base nas discussões do grupo de trabalho:

1. Pagamento único
2. Pagamento recorrente com frequência fixa
3. Pagamento recorrente por repetição
4. Pagamento recorrente por configuração *customizada*

Todas as políticas de agendamento são mutuamente exclusivas entre si, ou seja, não podem coabitar a definição de um mesmo payload de consentimento para pagamentos simultaneamente.  
Caso isso ocorra a detentora deve retornar a **resposta HTTP 400** para a iniciadora.

## Política de agendamento de pagamento único ##

Essa política determina o agendamento de um pagamento único numa data futura.

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

## Política de agendamento de pagamento recorrente com frequência fixa ##

```
{
   "data":{
      "payment":{
         "type":"PIX",
         "currency":"BRL",
         "amount":"100000.12",
         "schedule":{
            "fixed":{
               "startAt":"2022-08-10",
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
No exemplo teremos pagamentos em: 2022-08-12, 2022-08-14 e 2022-08-16.   

A política de agendamento de pagamento recorrente com frequência fixa é introduzida pelo campo de nome **fixed** no objeto do campo **schedule** conforme mostrado no exemplo acima.  
Ela define a seguinte estrutura:

**Campos**
  1. **startAt** : **tipo**: string(date), **opcional**, **valor padrão**: data atual do consentimento, **descrição**: Data de onde irá começar a contar a recorrência de pagamentos. Essa data deve ser futura ou igual ao dia do consentimento em questão.
  2. **frequency**: **tipo**: inteiro, **obrigatório**, **valor mínimo: 2**, **valor máximo**: 365, **descrição**: Frequência em dias corridos da recorrência de pagamentos.
  3. **count**: **tipo**: inteiro, **obrigatório**, **valor mínimo: 2**, **valor máximo**: 200, **descrição**: Quantidade de pagamentos a ser realizada a partir da data inicial de recorrência.

Para o contexto de "UX", a intenção dessa política pode ser expressa para usuário final da seguinte forma: "Pagamento recorrente agendado a cada {frequency} dias a partir da data {delay}"

Quatro tipos de políticas de agendamento foram definidas com base nas discussões do grupo de trabalho:

## Política de agendamento de pagamento recorrente por repetição ##

Para o contexto de "UX", a intenção dessa política pode ser expressa para usuário final da seguinte forma: "Pagamento recorrente agendado a cada {frequency} dias a partir da data {delay}"

```
{
   "data":{
      "payment":{
         "type":"PIX",
         "currency":"BRL",
         "amount":"100000.12",
         "schedule":{
            "iteration":{
               "startAt":"2022-08-19",
               "frequency":"MONTHLY",
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
pela fórmula: **startAt + meses(count)** .       
No exemplo teremos pagamentos em: 2022-09-18, 2022-10-18 e 2022-11-18.  
**Em caso da combinação com o dia 29 com períodos passando no mês de fevereiro o dia 28 deve ser assumido como data de pagamento em anos não bissextos.**

A política de agendamento de pagamento recorrente por repetição é introduzida pelo campo de nome **iteration** no objeto do campo **schedule** conforme mostrado no exemplo acima.  
Ela define a seguinte estrutura:

**Campos**
1. **startAt** : **tipo**: string(date), **opcional**, **valor padrão**: data posterior corrida a data atual do consentimento, **descrição**: Data de onde irá começar a contar a recorrência de pagamentos. Essa data deve ser futura ao dia do consentimento em questão.
2. **frequency**: **tipo**: inteiro, **obrigatório**, **valor mínimo: 1**, **valor máximo**: 31, **descrição**: Dia do mês em que cada pagamento irá ocorrer. 
3. **count**: **tipo**: inteiro, **obrigatório**, **valor mínimo: 2**, **valor máximo**: 200, **descrição**: Quantidade de pagamentos a ser realizada a partir da data inicial de recorrência.

Para o contexto de "UX", a intenção dessa política pode ser expressa para usuário final da seguinte forma: "Pagamento recorrente agendado para todo dia {day_of_month} a partir da data {delay}"

## Política de agendamento recorrente por configuração customizada ##

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

A política de agendamento de pagamento recorrente por configuração *customizada* é introduzida pelo campo de nome **custom** no objeto do campo **schedule** conforme mostrado no exemplo acima.  
Ela define a seguinte estrutura:

**Campos**
1. **dates** : **tipo**: array de string(date), **obrigatório**, **quantidade minina de elementos**: 2, **quantidade máxima de elementos**: 50, **descrição**: Lista de datas de recorrência de pagamentos. Todas as datas presentes no array devem únicas e futuras.

Para o contexto de "UX", a intenção dessa política pode ser expressa para usuário final da seguinte forma: "Pagamento recorrente agendado para os dias {dates}"

### Respostas 
**HTTP 422 (Schema: https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_422ResponseErrorCreateConsent)**
1. Introdução do valor: **INVALID_SCHEDULE** no enumerado [422ResponseErrorCreateConsent](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_422ResponseErrorCreateConsent) usado no campo **"code"** já presente no payload de resposta para quando qualquer validação de negócio falhar.
2. Introdução da mensagem: **"Agendamento inválido."** no campo **"title"** caso o campo **"code"** tenha o valor definido no **item 1 desta lista**.
3. Introdução da mensagem: **"Agendamento inválido."** no campo **"details"** caso o campo **"code"** tenha o valor definido no **item 1 desta lista**.
