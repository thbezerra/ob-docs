#Indices
- [Agendamento de iniciação de pagamentos](#Agendamento de iniciação de pagamentos)
  - [Regras gerais de negócio](#Regras gerais de negócio)
  - [Modelo com datas explícitas](#Modelo com datas explícitas)
  - [Modelo com datas implícitas](#Modelo com datas implícitas)
  - [Revogação de consentimento para pagamentos agendados](#Revogação de consentimento para pagamentos agendados)
  - [Discussões sobre alterações de premissas de iniciação de pagamentos por conta de multi autorizadores](#Discussões sobre alterações de premissas de iniciação de pagamentos por conta de multi autorizadores)

# Agendamento de iniciação de pagamentos

Para possibilitar o agendamento único ou recorrente
de pagamentos iniciados pelo Open Banking (OB) seria necessário a inclusão do conceito de "agenda de pagamentos" no consentimento.
A proposta aqui apresentada tem o canal OB sendo agnóstico ao arranjo do produto final (Pix, TED, TEF, débito em conta) sendo chamado, as suas eventuais interseções com a funcionalidade proposta,
de modo a ter menos impacto possível no OB em possíveis expansões de funcionalidades destes arranjos futuramente ou regras de negócio deste contexto dos mesmos.
Isso acarreta que para produto final o agendamento único ou sucessivo seria totalmente desconhecido se comportando apenas como um pagamento normal.

Mais adiante são mostradas e discutidas algumas formas de materialização dessa **"agenda de pagamentos"**.

## Regras gerais de negócio

1. Todo o consentimento de pagamento tem o prazo máximo de validade de um ano, pois seria limite máximo de datas na agenda de pagamentos.
2. A execução de pagamento conforme a agenda fica de responsabilidade da iniciadora como hoje já é feito com pagamentos normais. 
3. Todo o pagamento para um consentimento vinculado a uma agenda de pagamento deve ser validado contra a mesma pela detentora de modo a aferir se o momento do pagamento está em conformidade com o aprovado pelo usuário final no momento do consentimento.
4. Pagamentos mal sucedidos por qualquer motivo não invalidam o consentimento ou impactam os próximos pagamentos.

## Modelo com datas explícitas

Neste modelo a agenda é definida com todas as datas em que serão realizados os pagamentos.  

### Fragmento do payload de consentimento para pagamentos atual

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
### Fragmento do payload de consentimento para pagamentos agendados com o modelo de datas explícitas

```{
   "data":{
      "payment":{
         "type":"PIX",
         "currency":"BRL",
         "amount":"100000.12",
         "schedule":[
            "2021-01-01",
            "2021-02-19",
            "2021-06-25",
            "2021-12-31"
         ]
      }
   }
}
```

Nesse modelo é introduzido o campo **"schedule"** sendo um array de datas sem repetição com no máximo 48 elementos (O número 48 de elementos vem da possibilidade de pagamentos semanais durante um ano).
Esse campo seria **mutuamente exclusivo** contra o campo **"date"** já presente na estrutura **"payment"** como mostrado no exemplo.  
Para facilitar o entendimento dos usuários finais, a **detentora** deverá armazenar e devolver na resposta essas datas de forma ordenada da menor para a maior.  
A **menor** data presente no array **"schedule"** deve ser uma data futura.  
A **maior** data presente no array **"schedule"** deve ser no máximo um ano no futuro a partir do dia do consentimento.  
O novo campo será incluído tanto no payload de requisição quanto no payload de resposta bem sucedida do endpoint de criação de consentimento para pagamentos (Respectivamente: https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_CreatePaymentConsent e https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_ResponsePaymentConsent) 


### Erros de resposta 

**HTTP 422 (Schema: https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_422ResponseErrorCreateConsent)** 
1. Introdução do valor: **INVALID_SCHEDULE** no enumerado **"code"** já presente no payload de resposta para este tipo de erro.
2. Introdução da mensagem: **"Agendamento inválido."** no campo **"title"** caso o campo **"code"** tenha o valor definido no item 1. 
3. Introdução da mensagem: **"Agendamento inválido."** no campo **"details"** caso o campo **"code"** tenha o valor definido no item 1.

### Vantagens do modelo:

1. Flexibilidade total na definição nos períodos de pagamentos pelas iniciadoras
2. Toda a lógica de referente a dias úteis e corridos sobre os períodos de pagamentos definidos ficam a cargo das iniciadoras, portanto, concentradas em único player.
3. Clareza para as detentoras das datas a serem validadas nos pagamentos
4. Clareza para os usuários finais de quando serão executados seus pagamentos

### Desvantagens do modelo:

1. Aumento da complexidade em UX para prover formas paginadas de apresentação das datas para o usuário final
2. Dependendo da estratégia de definição dos períodos de pagamentos, podem acarretar em payloads com quantidade relativamente alta de informação (Ex: 48 datas em períodos semanais) 

## Modelo com datas implícitas

Nesse modelo as datas de pagamentos a serem realizadas são derivadas de uma política de agendamento.

### Fragmento do payload de consentimento para pagamentos atual
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
### Fragmento do payload de consentimento para pagamentos agendados com o modelo de datas implícitas


```{
   "data":{
      "payment":{
         "type":"PIX",
         "currency":"BRL",
         "amount":"100000.12",
         "schedule":{
            "first_date":"2021-01-01",
            "last_date":"2021-12-31",
            "recurring_type":"MONTHLY"
         }
      }
   }
}
```

Nesse modelo é introduzido o campo **"schedule"** sendo um objeto que contém a definição de uma **política de agendamento** dos pagamentos.  
Esse campo seria mutuamente exclusivo contra o campo **"date"** já presente na estrutura **"payment"** como mostrado no exemplo.  

O campo obrigatório **"first_date"** representa o início do período de pagamentos e deve ser uma data futura de no máximo um ano de diferença da data atual.  

O campo obrigatório **"last_date** representa o fim do período de pagamentos devendo ser maior ou igual ao definido no campo **"first_date"** e no máximo com um ano de diferença da data atual.     

O campo obrigatório **"recurring_type"** representa a recorrência dos pagamentos que serão realizados no período especificado entre o **"first_date"** e o **"last_date"**.
Esse campo seria um enumerado com os seguintes valores possíveis:

1. **SINGLE** - Define um pagamento agendado sem recorrência. Para esse valor os campos **"first_date"** e **"last_date"** devem ser iguais.
3. **WEEK** - Define que os pagamentos serão feitos a cada 7 dias a partir do **"first_date"** até **"last_date"**.
4. **FORTNIGHT** - Define que os pagamentos serão feitos a cada 15 dias a partir do **"first_date"** até **"last_date"**.
5. **MONTH** - Define que os pagamentos serão feitos a cada 30 dias a partir do **"first_date"** até **"last_date"**.
6. **QUARTER** - Define que os pagamentos serão feitos a cada 90 dias a partir do **"first_date"** até **"last_date"**.
7. **HALF_YEAR** - Define que os pagamentos serão feitos a cada 180 dias a partir do **"first_date"** até **"last_date"**.

### Erros de resposta

**HTTP 422 (Schema: https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_422ResponseErrorCreateConsent)**
1. Introdução do valor: **INVALID_SCHEDULE** no enumerado **"code"** já presente no payload de resposta para este tipo de erro.
2. Introdução da mensagem: **"Agendamento inválido."** no campo **"title"** caso o campo **"code"** tenha o valor definido no item 1.
3. Introdução da mensagem: **"Agendamento inválido."** no campo **"details"** caso o campo **"code"** tenha o valor definido no item 1.

### Vantagens do modelo:

1. Não há aumento de complexidade significativo em UX
2. O payloads dos consentimentos não teriam aumento de tamanho significativo
3. Para o usuário final teria menos *"overhead"* cognitivo no momento da aprovação do consentimento

### Desvantagens do modelo:

1. Como a detentora tem que validar se o pagamento recebido está em sinergia com agendamento consentido seria necessário a detentora concluir os períodos de pagamento da mesma forma que a iniciadora idealizou. 
Isso acarretaria definições sobre dias úteis vs dias corridos, tratamento de anos bissextos, regionalidade de feriados e etc que pode se tornar um tópico bem grande para configuração.

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

**POST /payments/v1/consents/{consentId}/revoke**

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

## Discussões sobre alterações de premissas de iniciação de pagamentos por conta de multi autorizadores

Atualmente há três propostas para lidar com a questão de multi-autorizadores de pagamentos oriundas da discussão
sobre o agendamento de pagamentos.

1. **Na proposta um nada será alterado e todo fluxo de pagamento que hoje já suportado pelas detentoras de conta que possuem contas com multi-autorizadores será usado.**
2. **A segunda proposta tem como premissa fazer com que a detentora seja responsável pela execução agendada dos pagamentos.**  
   **Análise:** Essa proposta oferece um aumento de risco para a detentora que pode, por efeito de algoritmo defeituoso, concluir equivocadamente a realização ou não de pagamentos
   ficando com toda a responsabilização disso. Fora isso todas as detentoras teriam No modelo onde a iniciadora realiza a execução dos pagamentos a responsabilização e compartilhada. Além disso,
   em nada isso ajuda na questão de multi-autorizadores que teriam ainda que realizar a autorização de cada pagamento pelos controladores da conta de onde o recurso financeiro será retirado.
3. **A terceira proposta altera o fluxo de consentimento para suportar os multi-autorizadores antes de mesmo da tentativa de pagamento.**
   **Análise:** É legitima a questão de que se há diversos controladores de uma conta todas as decisões deveriam ser tomadas em conjunto, contudo, isso cria uma série de fricções de experiência e técnicas.
   Fricção de experiência estaria em mesmo que o consentimento suportasse um consenso de muitos aprovadores, estes ainda teriam que passar pela mesma rodada de aprovações nos pagamentos, pois o produto final do arranjo suportado
   por esses tipos de contas ainda será o mesmo nas detentoras.
   Fricção técnica estaria em como suportar um modelo de autorizações baseado em redirect para um único usuário como é hoje com multiplos aprovadores. Coisas pooling de consentimento ou push notification teriam que ser feitas.
   Tudo isso para um caso de uso onde nem todas as detentoras possuem.
   Como mencionado, a proposta de alteração realmente agrega valor, contudo a sua extensão de discussão e execução é bem maior do que o prazo atual que temos para o assunto.     
