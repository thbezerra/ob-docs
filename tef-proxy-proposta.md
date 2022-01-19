#Indices

- [Introdução](#introdução)
- [Criar consentimento](#criar-consentimento)
- [Regras de negócio](#regras-de-negcio)

#Introdução

A iniciação de pagamento para TEF permite que a iniciadora informe a conta de destino diretamente ou o proxy (identificador da conta na instituição ) no momento da iniciação.
Caso a detentora de conta identifique a conta informada pelo Alias e a jornada de consentimento se concretiza, a detentora deverá retonar no payload de response o objeto creditorAccount com a informação da conta associada ao proxy informado.
Importante atentar que a iniciadora deverá utilizar o creditorAccount retornado para prosseguir com a transação.

## Criar consentimento
- **Path** : POST - /payments/v1/consents
- **Request Payload** : [CreatePaymentConsent](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_CreatePaymentConsent)
- **HTTP 200 Response Payload**: [ResponsePaymentConsent](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_ResponsePaymentConsent)
- **HTTP 422 Response Payload**: [422ResponseErrorCreateConsent](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_422ResponseErrorCreateConsent)

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
         "proxy":"xpot@xpot.com.br",
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
**Campos novos ou alterados do payload para TEF**

|**Campo**|**Tipo**|**Requerido**|**Descrição**|Regras de negócio|
|----------|------|---------|--------------------------------------------------------|---------|
|**data.payment.proxy**|string|condicionalmente|Alias utilizado pela instituição para identificar a conta de destino do transferência. Mutuamente exclusivo com o campo data.payment.creditorAccount|[RN307](#regras-de-validação), [RN308](#regras-de-validação), [RN309](#regras-de-validação)|
|**data.payment.creditorAccount**|Object|condicionalmente|Conta do destinatário do TEF. Mutuamente exclusivo com o campo data.payment.proxy|[RN307](#regras-de-validação)|

## Regras de negocio

|**Código**|**Descrição**|**Endpoint**|**Resposta HTTP**|**Código de Erro**|**Título**|**Mensagem**|**Schema**|
|----------|------------------------------------------------------------------------------------------------------------------------------------------|-----------------|-------|----------------|--------------------------|-----------------------------|--------------------------|
|**RN307**|O campo **data.payment.proxy** e **data.payment.creditorAccount** para o contexto de TEF são mutuamente exclusivos entre si.|[Criar consentimento](#criar-consentimento)|422|**DETALHE_PGTO_INVALIDO**|Consentimento inválido|O campo proxy ou creditorAccount devem ser informados|[422ResponseErrorCreateConsent](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_422ResponseErrorCreateConsent)|
|**RN308**|Caso o campo **data.payment.proxy** para o contexto de TEF não seja suportado pela detentora a mesma deve retornar uma mensagem de erro.|[Criar consentimento](#criar-consentimento)|422|**IDENTIFICADOR_DE_CONTA_NAO_SUPORTADO**|Consentimento inválido|Alias de conta não suportado|[422ResponseErrorCreateConsent](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_422ResponseErrorCreateConsent)|
|**RN309**|Caso uma conta não for encontrada quando o valor do campo **data.payment.proxy** for fornecido a detentora deve comunicar um erro.|[Criar consentimento](#criar-consentimento)|422|**CONTA_NAO_ENCONTRADA**|Consentimento inválido|Conta não encontrada com o alias informado|[422ResponseErrorCreateConsent](https://openbanking-brasil.github.io/areadesenvolvedor/#tocS_422ResponseErrorCreateConsent)|
