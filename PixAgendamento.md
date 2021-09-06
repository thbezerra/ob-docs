# Pix Agendamento

Para possibilitar o agendamento único ou recorrente
de pagamentos é necessário ter o conceito de "agenda de pagamentos".  

Mais adiante são mostradas e discutidas algumas formas de materialização dessa **agenda**.

## Premissas de negócio

1. Todo o consentimento de pagamento tem o prazo máximo de validade de um ano.
2. A agenda de pagamento de um dado consentimento deve ser validada pela dententora de conta se está de acordo com o aprovado pelo usuário final em todo evento de pagamento para o consentimento referido.

## 1 - Modelo com datas explícitas

Neste modelo a agenda é definida com todas as datas quando os pagamentos serão feitos.  

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

Nesse modelo é introduzido o campo **"schedule"** sendo um array de datas sem repetição.
Esse campo seria mutuamente exclusivo contra o campo **date** já presente na estrutura **"payment"** mostrada no exemplo.
Para facilitar o entendimento é recomendado que as datas sejam definidas de forma ordenada da menor para a maior.  
A **menor** data presente neste array deve ser uma data futura.  
A **maior** data presente neste array deve ser no máximo um ano no futuro.

Esse modelo tem as seguintes vantagens:

1. Flexibilidade total na definição nos períodos de pagamento
2. Toda a lógica de referente a dias úteis e corridos sobre os períodos de pagamentos definidos fica a cargo das iniciadoras, portanto, concentradas em único player.
3. Clareza para a detentora das datas a serem validadas nos pagamentos
4. Clareza para os usuários finais de quando serão executados seus pagamentos

Esse modelo tem as seguintes desvantagens:

1. Aumento da complexidade em UX para prover formas paginadas de apresentação das datas para o usuário final
2. Dependendo da estratégia de definição dos períodos de pagamentos, podem acarretar em payloads com quantidade relativamente alta de informação (Ex: 365 datas períodos diários, 48 datas para períodos semanais) 

## 2 - Modelo com datas implícitas

Nesse modelo datas de pagamentos a serem realizadas são derivadas de políticas de período.

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

Nesse modelo é introduzido o campo **"schedule"** sendo um objeto que contém a definição da política de agendamento dos pagamentos.  
Esse campo seria mutuamente exclusivo contra o campo **date** já presente na estrutura **"payment"** mostrada no exemplo.  

O campo **"first_date"** representa o início do período de pagamentos e deve ser uma data futura de no máximo um ano de diferença da data atual.  

O campo **"last_date** representa o fim do período de pagamentos devendo ser maior ou igual ao definido no campo **"first_date"** e no máximo com um ano de diferença da data atual.     

O campo **recurring_type** representa a recorrência dos pagamentos que serão realizados no período especificado em **"first_date"** e **"last_date"**.
Esse campo seria um enumerado com os seguintes valores possíveis:

1. **SINGLE** - Define um pagamento agendado sem recorrência. Para esse valor os campos **"first_date"** e **"last_date"** devem ser iguais.
2. **DAILY** - Define que os pagamentos serão feitos diariamente a partir do **"first_date"** até **"last_date"**.
3. **WEEKLY** - Define que os pagamentos serão feitos a cada 7 dias a partir do **"first_date"** até **"last_date"**.
4. **FORTNIGHT** - Define que os pagamentos serão feitos a cada 15 dias a partir do **"first_date"** até **"last_date"**.
5. **MONTHLY** - Define que os pagamentos serão feitos a cada 1 mês a partir do **"first_date"** até **"last_date"**.
6. **HALF_YEAR** - Define que os pagamentos serão feitos a cada 6 meses a partir do **"first_date"** até **"last_date"**.

Esse modelo possui as seguintes vantagens:

1. Não há aumento de complexidade significativo em UX
2. O payloads dos consentimentos não teriam aumento de tamanho significativo
3. Para o usuário final teria menos "overhead" no momento da aprovação do consentimento

Esse modelo possui as seguintes desvantagens:

1. Como a detentora tem que validar se o pagamento recebido esta em sinergia com agendamento ele teria que concluir os períodos de pagamento da mesma forma que a iniciadora idealizou. Isso acarreta em definições sobre dias úteis vs dia corridos que pode se tornar um tópico bem grande para configuração.
