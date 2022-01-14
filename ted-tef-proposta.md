# Indices
- [Introdução](#introdução)
- [Consentimento](#consentimento)
  - [Criar consentimento](#criar-consentimento)
    - [Estruturação do payload de requisição](#estruturação-do-payload-de-requisição)
  - [Ciclo de vida](#ciclo-de-vida-do-consentimento)
  - [Agendamento](#agendamento)
- [Regras de negócio](#regras-de-negcio)


# Introdução
   Esta proposta visa adicionar suporte a iniciação de pagamentos através do arranjo de **TED (Transferência Eletrônica Disponível) e TEF (Transferência Eletrônica de Fundos)**.  
   **TED** é um arranjo de pagamentos que permite a realização de transferências financeiras entre instituições detentoras de contas no Banco Central.  
   **TEF** é um arranjo de pagamentos que permite a realização de transferências financeiras entre contas dentro de uma mesma instituição financeira.  


# Consentimento
   Para realizar uma iniciação de pagamento através de qualquer arranjo de pagamentos pelo Open Banking é necessário a criação de um consentimento que representa o desejo do usuário
   de permitir que sejam utilizados recursos financeiros em uma das suas contas presentes em uma detentora de conta por uma iniciadora de pagamentos.  
   Este conceito já é utilizado no Open Banking em face de outros arranjos de pagamentos e agora será expandido para suporte aos dois novos arranjos apresentados nesta proposta.  
   O consentimento é criado via endpoint [REST](https://pt.wikipedia.org/wiki/REST) e o seu funcionamento para adição do suporte a **TED/TEF** será detalhado a seguir.  

## Criar consentimento
  - **Path** : POST - /payments/v1/consents  
  - **Request Payload** : [CreatePaymentConsent](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_CreatePaymentConsent)
  - **HTTP 200 Response Payload**: [ResponsePaymentConsent](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_ResponsePaymentConsent)
  - **HTTP 422 Response Payload**: [422ResponseErrorCreateConsent](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_422ResponseErrorCreateConsent)  

O payload de requisição deste endpoint deverá ser expandido para suportar os dados necessários para os novos arranjos propostos.  
Antes de ser apresentado as expansões necessárias para cada arranjo será descrita uma reestruturação da composição do objeto requisição deste endpoint com finalidade de tornar mais clara a documentação do mesmo em face das necessidades específicas de cada arranjo suportado atualmente e os vindouros.  

### Estruturação do payload de requisição

Para facilitar a compreensão da nova estrutura proposta de payload e as relações entre os elementos que o compõe será utilizado o diagrama de classes abaixo.  

![Estrutura payload criação de consentimento](estrutura-payload-create-consent.png)  

Toda a mudança proposta apresentada no diagrama se concentra no campo **data.payment** do schema **CreatePaymentConsent**.  
O schema desse campo (**PaymentConsent**) se tornaria uma estrutura [polimórfica](https://pt.wikipedia.org/wiki/Polimorfismo_(ci%C3%AAncia_da_computa%C3%A7%C3%A3o)) abstrata com três schemas concretos: **PixPaymentConsent**, **TedPaymentConsent** e **TefPaymentConsent** respectivamente para os arranjos **PIX**, **TED** e **TEF**.  
O campo determinante ([discriminator](https://swagger.io/docs/specification/data-models/inheritance-and-polymorphism/)) de qual schema apropriado será o **type** .   
Uma conjectura surge a partir desta proposta que seria a não necessidade de existir um campo **details** para Pix visto que tudo poderia estar simplesmente definido no **PixPaymentConsent** diminuindo a profundidade do payload, contudo, neste momento essa alteração acarretaria em quebra de contrato de API o que é muito complexo de ser absorvido pelo ecossistema agora.  

Abaixo são apresentados fragmentos de payloads de consentimentos contendo a utilização dos três novos schemas com toda a informação disponível em cada um deles.  
Também serão descritos os campos novos, ou seja, introduzidos pelos novos arranjos. 

**PIX**  
```
{
   "data":{
      "payment":{
         "type":"PIX",
         "date":"2021-01-01",
         "schedule":{
            "single":{
               "date":"2021-01-01"
            }
         },
         "currency":"BRL",
         "amount":"100000.12",
         "ibgeTownCode":"5300108",
         "details":{
            "localInstrument":"DICT",
            "qrCode":"00020104141234567890123426660014BR.GOV.BCB.PIX014466756C616E6F32303139406578616D706C652E636F6D27300012\nBR.COM.OUTRO011001234567895204000053039865406123.455802BR5915NOMEDORECEBEDOR6008BRASILIA61087007490062\n530515RP12345678-201950300017BR.GOV.BCB.BRCODE01051.0.080450014BR.GOV.BCB.PIX0123PADRAO.URL.PIX/0123AB\nCD81390012BR.COM.OUTRO01190123.ABCD.3456.WXYZ6304EB76\n",
            "proxy":"12345678901",
            "creditorAccount":{
               "ispb":"12345678",
               "issuer":"1774",
               "number":"1234567890",
               "accountType":"CACC"
            }
         }
      }
   }
}
```

**TED**

```
{
   "data":{
      "payment":{
         "type":"TED",
         "date":"2021-01-01",
         "schedule":{
            "single":{
               "date":"2021-01-01"
            }
         },
         "currency":"BRL",
         "amount":"100000.12",
         "creditorAccount":{
            "ispb":"12345678",
            "issuer":"1774",
            "number":"1234567890",
            "accountType":"CACC"
         },
         "purpose":1,
         "purposeAdditionalInfo":"Informações adicionais"
      }
   }
}
```
**Campos novos do payload para TED**

|**Campo**|**Tipo**|**Requerido**|**Descrição**|Regras de negócio|
|----------|------|---------|--------------------------------------------------------|---------|
|**data.payment.purpose**|enum|sim|Define a finalidade da transferência. O domínio deste enumerado está no [anexo](lista-de-finalidades.txt)|N/A|
|**data.payment.purposeAdditionalInfo**|string|condicionalmente|Define o complemento da finalidade da transferência de forma textual.|[RN301](#regras-de-validação)|


**TEF**
```
{
   "data":{
      "payment":{
         "type":"TEF",
         "date":"2021-01-01",
         "schedule":{
            "single":{
               "date":"2021-01-01"
            }
         },
         "currency":"BRL",
         "amount":"100000.12",
         "creditorAccount":{
            "ispb":"12345678",
            "issuer":"1774",
            "number":"1234567890",
            "accountType":"CACC"
         }
      }
   }
}
```

A iniciação de pagamentos para TED não suporta todos os tipos de contas de crédito disponíveis pelo Open Banking, sendo suportado somente conta corrente, poupança e pagamentos ([RN302](#regras-de-validação)).  
O arranjo de TED possui uma grade de horários definida de funcionamento pelo Bacen. Além disso, muitas instituições limitam ainda mais a grade de funcionamento deste arranjo nos seus canais de oferta de TED, portanto esse aspectos precisam ser validados no momento do consentimento ([RN303](#regras-de-validação)).  
Tanto o arranjo TED quanto TEF podem apresentar restrições de limites para transferências estipuladas nas detentoras de conta, devendo isso ser considerado ao criar o consentimento([RN304](#regras-de-validação)).  
As alterações na requisição de criação do consentimento devem refletir também tanto na resposta do endpoint de criação de pagamento quanto na resposta do endpoint de busca do consentimento. 
   
  
## Ciclo de vida do consentimento
  Nenhuma alteração no ciclo de vida do consentimento será mudado para inclusão dos novos arranjos.  

## Agendamento
  Todas regras e fluxos envolvendo o agendamento de pagamentos também se aplicarão aos dois novos arranjos.   


# Pagamento

   O pagamento representa a execução do consentimento dado pelo cliente.  
   Ele pode tanto ter uma liquidação imediata quanto pode ser postergada para um momento no futuro definido pelo cliente.  
   Para dar suporte aos novos arranjos de TED/TEF será necessário a criação de novos endpoints REST que viabilizarão o fluxo de pagamento a partir das iniciadoras de pagamentos se forma análoga ao que hoje já praticado no PIX.  



# Regras de negócio

Nesta sessão serão listadas todas as regras de negócio envolvidas nos endpoints citados nas sessões anteriores.

## Regras funcionais

|**Código**|**Descrição**|**Endpoint**|**Resposta HTTP**|**Código de Erro**|**Título**|**Mensagem**|**Schema**|
|----------|------------------------------------------------------------------------------------------------------------------------------------------|-----------------|-------|----------------|--------------------------|-----------------------------|--------------------------|

## Regras de validação

|**Código**|**Descrição**|**Endpoint**|**Resposta HTTP**|**Código de Erro**|**Título**|**Mensagem**|**Schema**|
|----------|------------------------------------------------------------------------------------------------------------------------------------------|-----------------|-------|----------------|--------------------------|-----------------------------|--------------------------|
|**RN301**|O campo **data.payment.purposeAdditionalInfo** deverá ser preenchido apenas quando a **data.payment.purpose** tiver o valor 99999 - Outros|[Criar consentimento](#criar-consentimento)|422|**DETALHE_PGTO_INVALIDO**|Consentimento inválido|Complemento de finalidade requerida|[422ResponseErrorCreateConsent](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_422ResponseErrorCreateConsent)|
|**RN302**|O campo **data.payment.creditorAccount.accountType**  quando o arranjo alvo for TED só suportará os tipos **CACC** (Conta corrente), **SVGS** (Poupança) e **TRAN** (Conta de Pagamento pré-paga) |[Criar consentimento](#criar-consentimento)|422|**TIPO_CONTA_NAO_SUPORTADO**|Consentimento inválido|Tipo de conta não suportado pelo arranjo alvo|[422ResponseErrorCreateConsent](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_422ResponseErrorCreateConsent)|
|**RN303**|A detentora de conta deve validar se o consentimento para pagamento de TED está dentro de sua grade funcionamento para o arranjo|[Criar consentimento](#criar-consentimento)|422|**JANELA_HORARIO_NAO_PERMITIDA**|Consentimento inválido|Janela de horário do arranjo de pagamento alvo não permitida|[422ResponseErrorCreateConsent](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_422ResponseErrorCreateConsent)|
|**RN304**|A detentora de conta deve validar se o consentimento para pagamento de TED ou TEF está com o valor dentro dos limites de transferência por ela estabelecidos|[Criar consentimento](#criar-consentimento)|422|**VALOR_LIMITE_ATINGINDO**|Consentimento inválido|Valor limite de transferência do usuário no arranjo alvo atingido|[422ResponseErrorCreateConsent](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_422ResponseErrorCreateConsent)|
