# Indices
- [Alterações no consentimento](#alterações-no-consentimento)
- [Alterações no pagamento](#alterações-no-pagamento)


# Alterações no consentimento

Nesta sessão são mostradas todas as mudanças na api de consentimento de pagamentos 

## Paylod do endpoint de consentimento

```
{
   "data":{
      "consentId":"urn:bancoex:C1DD33123",
      "creationDateTime":"2021-05-21T08:30:00Z",
      "expirationDateTime":"2021-05-21T08:30:00Z",
      "ibgeTownCode":"5300108",
      "statusUpdateDateTime":"2021-05-21T08:30:00Z",
      "status":"REVOKED",
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
         "currency":"BRL",
         "amount":"100000.12",
         "ibgeTownCode":"5300108",
         "schedule":{
            "single":{
               "date":"2035-01-01"
            }
         }
         "details":{
            "localInstrument":"DICT",
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
   "links":{
      "self":"https://api.banco.com.br/open-banking/api/payments/v1/consents/urn:bancoex:C1DD33123",
      "revocation":"https://api.banco.com.br/open-banking/api/payments/v1/consents/urn:bancoex:C1DD33123/revocations/06b3ec6220ac413fb4e1f0b12b0ff926"
   }
}
```

O campo **schedule** deverá ser adicionado ao payload de consentimento no objeto payment que hoje já compõe o mesmo.
Ele será um objeto onde a sua estrutura será descrita posteriormente neste documento.
Este campo será mutuamente exclusivo contra o campo **date** hoje já presente no objeto payment.
O campo **date** só será usado para pagamentos normais, ou seja, não agendados/recorrentes.
Caso os dois campos estejam presentes simultaneamente no payload da requisição, a detentora deve devolver uma **resposta HTTP 400**.

**Campos**

1. **schedule**: **tipo**: Objeto, **obrigatório em caso de agendamento**, **descrição**: Descreve a política de agendamento de pagamento.
   1. **single**: **tipo**: Objeto, **obrigatório em caso de agendamento único**, descrição: Descreve a política de agendamento de pagamento único.
      1. **date** : **tipo**: string(date), **obrigatório**, **descrição**: Data do agendamento do pagamento. Essa data deve ser futura em relação ao dia do consentimento.


**Links**  
 Adição do link **revocation** apontando para o recurso de revogação **caso o consentimento esteja no status REVOKED**.

## Respostas do endpoint de criação do consentimento
**HTTP 422 (Schema: https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_422ResponseErrorCreateConsent)**
1. Introdução do valor: **INVALID_SCHEDULE** no enumerado [422ResponseErrorCreateConsent](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_422ResponseErrorCreateConsent) usado no campo **"code"** já presente no payload de resposta para quando qualquer validação de negócio falhar.
2. Introdução da mensagem: **"Agendamento inválido."** no campo **"title"** caso o campo **"code"** tenha o valor definido no **item 1 desta lista**.
3. Introdução da mensagem: **"Agendamento inválido."** no campo **"details"** caso o campo **"code"** tenha o valor definido no **item 1 desta lista**.

## Revogação do consentimento

**POST - /payments/v1/consents/{consentId}/revocations**

**Payload de requisição**

```
{
   "data":{
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
      "revoked_by":"USER",
      "reason":{
         "code":"OTHER",
         "additionalInformation":"Não quero mais o serviço"
      }
   }
}
```

**Descrição dos campos da requisição**

1. **data** : **Tipo**: objeto, **obrigatório**, **Descrição**: Campo que contém os dados da requisição de cancelamento
    1. **revoked_by** : **Tipo** : enumerado, **obrigatório**, **Descrição**: Descreve quem solicitou a revogação do consentimento. Valores: **USER** (Revogado pelo usuário), **ACCOUNT_HOLDER** (Dententora de conta), **INITIATOR** (iniciadora de pagamentos)
    2. **loggedUser** : **Tipo** : objeto, **obrigatório quando o campo revoked_by tiver o valor USER e se tratar de uma pessoa física** , **Descrição**: Indica o usuário pessoa física logado na instituição escolhida por ele (Detentora ou iniciadora) para realizar a revogação do consentimento.
        1. **document**: **Tipo**: objeto, **obrigatório**, **Descrição**: Indica o documento de identificação do usuário.
            1. **identification**: **Tipo** : string, **obrigatório**, **Max length**: 11, **Pattern**: ^\d{11}$ , **Descrição**: Número do documento de identificação oficial do usuário.
            2. **rel**: **Tipo** : string, **obrigatório**, **Max length**: 3, **Pattern**: ^[A-Z]{3}$ , **Descrição** : Tipo do documento de identificação oficial do usuário
    3. **businessEntity**: **Tipo** : objeto, **obrigatório quando o campo revoked_by tiver o valor USER e se tratar de uma pessoa jurídica** , **Descrição**: Indica o usuário pessoa jurídica logado na instituição escolhida por ele (Detentora ou iniciadora) para realizar a revogação do consentimento. 
       1. **document**: **Tipo**: objeto, **obrigatório**, **Descrição**: Indica o documento de identificação do usuário.
          1. **identification**: **Tipo** : string, **obrigatório**, **Max length**: 14, **Pattern**: ^\d{11}$ , **Descrição**: Número do documento de identificação oficial do usuário.
          2. **rel**: **Tipo** : string, **obrigatório**, **Max length**: 4, **Pattern**: ^[A-Z]{4}$ , **Descrição** : Tipo do documento de identificação oficial do usuário
    4. **reason** : **Tipo** : objeto, **obrigatório**, **Descrição**: Razão da revogação do consentimento
        1. **code**: **Tipo** : enumerado, **obrigatório**, **Descrição**: Código do motivo da revogação do consentimento.   
           Valores:
            1. **FRAUD** - Indica suspeita de fraude. Só deve ser usado quando o requisitante for a detentora ou iniciadora
            2. **ACCOUNT_CLOSURE** - Indica que a conta do usuário foi encerrada. Só deve ser usado quando o requisitante for a detentora ou iniciadora
            3. **OTHER** - Indica que motivo do cancelamento está fora dos motivos pré-estabelecidos. O campo additionalInformation deverá descrever o motivo do cancelamento.
        2. additionalInformation: **Tipo**: String, **obrigatório apenas quando o campo code for igual a OTHER**, **Max length**: 255, **Descrição** : Contém informações adicionais definidas pelo requisitante da revogação.

**Parâmetros**

1. **consentId** : **Origem**: path, **tipo**: string, **obrigatório**, **descrição**: O consentId é o identificador único do consentimento a ser revogado.
2. **Authorization** : **Origem**: header, **tipo**: string, **obrigatório**, **descrição**: Cabeçalho HTTP padrão. Permite que as credenciais sejam fornecidas dependendo do tipo de recurso solicitado
3. **x-fapi-auth-date** : **Origem**: header, **tipo**: string, **opcional**, **descrição**: Data em que o usuário logou pela última vez com o receptor. Representada de acordo com a RFC7231.Exemplo: Sun, 10 Sep 2017 19:43:31 UTC
4. **x-fapi-customer-ip-address**: **Origem**: header, **tipo**: string, **opcional**, **descrição**: O endereço IP do usuário se estiver atualmente logado com o receptor.
5. **x-fapi-interaction-id** : **Origem**: header, **tipo**: string, **opcional**, **descrição**: Um UID RFC4122 usado como um ID de correlação. Se fornecido, o transmissor deve "reproduzir" esse valor no cabeçalho de resposta.
6. **x-idempotency-key** : **Origem**: header, **tipo**: string, **obrigatório**, **Cabeçalho HTTP personalizado. Identificador de solicitação exclusivo para suportar a idempotência.**
7. **x-customer-user-agent** : **Origem**: header, **tipo**: string, **opcional**, **descrição**: Indica o user-agent que o usuário utiliza.

**Respostas**

**HTTP 201** : Ordem de cancelamento criada com êxito  
**payload de resposta**
```
{
   "data":{
      "revocationId": "d490493fff6e4fbd8a3eeb0cc247537c",
      "consentId":"urn:bancoex:C1DD33123",
      "creationDateTime":"2021-05-21T08:30:00Z",
      "statusUpdateDateTime":"2021-05-21T08:30:00Z",
      "loggedUser":{
         "document":{
            "identification":"11111111111",
            "rel":"CPF"
         }
      },
      "revoked_by":"USER",
      "reason":{
         "code":"OTHER",
         "additionalInformation":"Não quero mais o serviço"
      }
   }
}
```

**HTTP 422** : A solicitação foi bem formada, mas não pôde ser processada devido à lógica de negócios específica da solicitação.
**Payload**:

```
{
  "errors": [
    {
      "code": "OPERATION_NOT_ALLOWED_BY_STATUS",
      "title": "Operação não permitida.",
      "detail": "Operação não permitida devido ao status atual do consentimento."
    }
  ],
  "meta": {
    "totalRecords": 1,
    "totalPages": 1,
    "requestDateTime": "2021-05-21T08:30:00Z"
  }
}
```
O payload deste erro segue a mesma estrutura descrita em [422ResponseErrorCreateConsent](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_422ResponseErrorCreateConsent),
com exceção do campo code que neste caso terá um tipo enumerado novo cujos valores possíveis serão: **OPERATION_NOT_SUPPORTED_BY_CONSENT_TYPE**,
**REVOCATION_TIME_LIMIT_EXCEEDED**, **OPERATION_NOT_ALLOWED_BY_STATUS**.

**Mensagens de erro**
1. code: **OPERATION_NOT_SUPPORTED_BY_CONSENT_TYPE**, title: Operação não suportada, details: Operação não suportada pelo tipo de consentimento
2. code: **REVOCATION_TIME_LIMIT_EXCEEDED**, title: Prazo limite para revogação excedido, details: Prazo limite para revogação excedido
3. code: **OPERATION_NOT_ALLOWED_BY_STATUS**, title: Operação não suportada, details: Operação não suportada para o status atual do consentimento

**GET - /payments/v1/consents/{consentId}/revocations/{revocationId}**

**Parâmetros**

1. **consentId** : **Origem**: path, **tipo**: string, **obrigatório**, **descrição**: O consentId é o identificador único do consentimento que foi revogado.
2. **revocationId**: **Origem**: path, **tipo**: string, **obrigatório**, **descrição**: O revocationId é o identificador único da revogação do consentimento. o valor do campo deverá ser preenchido com o UUID definido pela instituição de acordo com a RFC 4122 usando o versão 4.
3. **Authorization** : **Origem**: header, **tipo**: string, **obrigatório**, **descrição**: Cabeçalho HTTP padrão. Permite que as credenciais sejam fornecidas dependendo do tipo de recurso solicitado
4. **x-fapi-auth-date** : **Origem**: header, **tipo**: string, **opcional**, **descrição**: Data em que o usuário logou pela última vez com o receptor. Representada de acordo com a RFC7231.Exemplo: Sun, 10 Sep 2017 19:43:31 UTC
5. **x-fapi-customer-ip-address**: **Origem**: header, **tipo**: string, **opcional**, **descrição**: O endereço IP do usuário se estiver atualmente logado com o receptor.
6. **x-fapi-interaction-id** : **Origem**: header, **tipo**: string, **opcional**, **descrição**: Um UID RFC4122 usado como um ID de correlação. Se fornecido, o transmissor deve "reproduzir" esse valor no cabeçalho de resposta.
7. **x-customer-user-agent** : **Origem**: header, **tipo**: string, **opcional**, **descrição**: Indica o user-agent que o usuário utiliza.

**Respostas**  
**HTTP 200** : Revogação do consentimento obtida com êxito  
**payload de resposta**
```
{
   "data":{
      "revocationId": "d490493fff6e4fbd8a3eeb0cc247537c",
      "consentId":"urn:bancoex:C1DD33123",
      "creationDateTime":"2021-05-21T08:30:00Z",
      "statusUpdateDateTime":"2021-05-21T08:30:00Z",
      "loggedUser":{
         "document":{
            "identification":"11111111111",
            "rel":"CPF"
         }
      },
      "revoked_by":"USER",
      "reason":{
         "code":"OTHER",
         "additionalInformation":"Não quero mais o serviço"
      }
   }
```

## Ciclo de vida do consentimento

1. **SCHEDULED** : Indicará que o processo de agendamento ocorreu com sucesso.
2. **REVOKED** : Indicará que o consentimento foi revogado a pedido de alguma das partes (Usuário, iniciadora ou detentora) e com isso, 
o agendamento do pagamento será cancelado.

o consentimento deve ser marcado como [CONSUMED](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_EnumAuthorisationStatusType) .  
Enquanto o consentimento estiver no status **SCHEDULED** será possível obter renovar tokens de acesso para os endpoints relacionado ao fluxo da funcionalidade.  
O processo de **revogação do consentimento** será detalhado numa sessão adiante neste documento.

## Controle de andamento de modificações no consentimento

Para a iniciadora acompanhar o andamento dos status do consentimnto ela deverá utilizar o mecanismo que hoje ela já usa para fazê-lo.
O mecanismo consiste em realizar um pooling no endpoint de busca do consentimento até o consentimento alcançar algum status final (CONSUMED, REJECTED, REVOKED).

# Alterações no pagamento

Nesta sessão serão apresentadas as alterações no recurso de pagamentos para suportar o agendamento.  

## Ciclo de vida do pagamento

O pagamento ganhará dois novos status como descrito a seguir para suportar a funcionalidade de agendamento.
Com isso dois novos status listados abaixo deverão ser adicionados ao pagamento.
1. **SASP (SCHEDULE_ACCEPTED_SETTLEMENT_IN_PROCESS)** : Processo de agendamento de iniciação de pagamento aceita e em processamento
2. **SASC (SCHEDULE_ACCEPTED_SETTLEMENT_COMPLETED)** : Processo de agendamento de iniciação de pagamento concluído

O status **SASP** abri a possibilidade de o processo de agendamento ser feito assincronamente na detentora.
Isso será especialmente útil para o caso da detentora tiver diferentes sistemas para se comunicar de modo a realizar o agendamento.

O **consentimento** deverá ser marcado como **SCHEDULED** e ao fim do processo de agendamento bem-sucedido ou **CONSUMED** caso contrário.
Quando o pagamento for liquidado no dia do agendamento sendo bem-sucedido ou não, o consentimento deverá ser marcado como **consumido**, ou seja, atribuir o status **CONSUMED**. 


## Payload do pagamento

A máquina de estados do pagamento está intimamente ligada a máquina de estados do consentimento, pois algumas translações de estados
de cada um dos recursos citados dispara mudanças no outro. Um desses casos é a revogação do consentimento que terá o seu impacto no pagamento descrito a seguir.  
Quando o consentimento for revogado o pagamento agendado vinculado ao mesmo deve ir para o status **RJCT** (Rejeitado) e o motivo da sua rejeição descrito no campo **rejectionReason**  
deve assumir o valor **CONSENT_REVOKED** que deverá ser adicionado ao schema [EnumRejectionReasonType](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_EnumRejectionReasonType).

**Exemplo**

```
{
   "data":{
      "paymentId":"TXpRMU9UQTROMWhZV2xSU1FUazJSMDl",
      "consentId":"urn:bancoex:C1DD33123",
      "creationDateTime":"2020-07-21T08:30:00Z",
      "statusUpdateDateTime":"2020-07-21T08:30:00Z",
      "proxy":"12345678901",
      "status":"RJCT",
      "rejectionReason":"CONSENT_REVOKED",
      "localInstrument":"DICT",
      "cnpjInitiator":"50685362000135",
      "payment":{
         "amount":"100000.12",
         "currency":"BRL"
      },
      "remittanceInformation":"Pagamento da nota RSTO035-002.",
      "creditorAccount":{
         "ispb":"12345678",
         "issuer":"1774",
         "number":"1234567890",
         "accountType":"CACC"
      }
   }
}
```