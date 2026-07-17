# 🤖 Serverless AI Delivery Assistant with AWS Step Functions & Amazon Bedrock

Este repositório contém o projeto prático desenvolvido durante o Bootcamp **AWS - Agentes de IA em Campo** da **DIO (Digital Innovation One)**. O objetivo principal deste projeto é criar um assistente de delivery inteligente e totalmente *serverless* (sem servidor) utilizando o **AWS Step Functions** para orquestrar o fluxo de pedidos e o **Amazon Bedrock** para injetar Inteligência Artificial Generativa no processo de atendimento ao cliente.

---

## 📌 Sobre o Projeto

O projeto simula um fluxo de delivery automatizado (semelhante ao funcionamento de aplicativos como iFood ou Uber Eats), controlando desde o recebimento do pedido, validação, simulação de pagamento, até a preparação e a entrega física. 

O grande diferencial é a utilização de **IA Generativa** integrada diretamente ao fluxo de trabalho (workflow) para gerar mensagens personalizadas e dinâmicas de atualização do status de entrega para o cliente, baseadas nos itens que ele comprou.

---

## 📐 Arquitetura do Workflow (State Machine)

A orquestração do fluxo foi desenvolvida utilizando a **Amazon States Language (ASL)**, uma linguagem baseada em JSON que define os estados do **AWS Step Functions**.

O fluxo é dividido nas seguintes etapas lógicas:
1. **Recebimento do Pedido (`IniciarPedido`):** Estado do tipo `Pass` que recebe os detalhes do pedido (cliente, itens, endereço e forma de pagamento).
2. **Validação do Pedido (`ValidarItens`):** Valida se as informações do pedido são consistentes.
3. **Fluxo de Decisão de Pagamento (`ProcessarPagamento`):** Um estado `Choice` que avalia se a transação financeira foi aprovada ou recusada.
    * *Se aprovado:* O pedido segue para a cozinha/preparação.
    * *Se recusado:* O fluxo é direcionado para um estado de cancelamento amigável.
4. **Acionamento da IA (`InvocarAmazonBedrock`):** Uma integração direta (`Task`) com a API `InvokeModel` do **Amazon Bedrock**. Aqui, o modelo de fundação (como o *Claude 3.5 Sonnet* ou o *Amazon Nova*) recebe os dados do pedido e cria uma notificação customizada, empática e com sugestões de consumo personalizadas para o cliente.
5. **Finalização do Pedido (`NotificarCliente`):** O sistema conclui o processo exibindo ou enviando o retorno gerado pela IA.

---

## 💻 Definição da State Machine (ASL JSON)

Abaixo está o modelo da estrutura JSON utilizada para configurar o fluxo de trabalho no AWS Step Functions:

```json
{
  "Comment": "Um fluxo serverless de entrega inteligente integrado ao Amazon Bedrock",
  "StartAt": "IniciarPedido",
  "States": {
    "IniciarPedido": {
      "Type": "Pass",
      "Result": {
        "pedidoId": "12345",
        "cliente": "Alexandre",
        "itens": ["1x Pizza de Calabresa Grande", "1x Refrigerante de Guaraná 2L"],
        "endereco": "Av. Paulista, 1000 - São Paulo, SP",
        "pagamentoAprovado": true
      },
      "ResultPath": "$.pedidoInfo",
      "Next": "ValidarItens"
    },
    "ValidarItens": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.pedidoInfo.itens[0]",
          "IsPresent": true,
          "Next": "ProcessarPagamento"
        }
      ],
      "Default": "PedidoInvalido"
    },
    "ProcessarPagamento": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.pedidoInfo.pagamentoAprovado",
          "BooleanEquals": true,
          "Next": "InvocarAmazonBedrock"
        }
      ],
      "Default": "PagamentoRecusado"
    },
    "InvocarAmazonBedrock": {
      "Type": "Task",
      "Resource": "arn:aws:states:::bedrock:invokeModel",
      "Parameters": {
        "ModelId": "anthropic.claude-3-5-sonnet-20240620-v1:0",
        "Body": {
          "anthropic_version": "bedrock-2023-05-31",
          "max_tokens": 300,
          "messages": [
            {
              "role": "user",
              "content": [
                {
                  "type": "text",
                  "text": "Você é um assistente de delivery super simpático e divertido. Escreva uma mensagem de status enviada pelo WhatsApp avisando que o pedido de $.pedidoInfo.cliente que contém $.pedidoInfo.itens está saindo para o endereço $.pedidoInfo.endereco. Adicione um toque bem-humorado e faça uma sugestão de sobremesa que combine com o pedido."
                }
              ]
            }
          ]
        }
      },
      "ResultPath": "$.resultadoIA",
      "Next": "NotificarCliente"
    },
    "NotificarCliente": {
      "Type": "Pass",
      "End": true
    },
    "PedidoInvalido": {
      "Type": "Fail",
      "Error": "PedidoInvalidoError",
      "Cause": "O pedido não possui itens válidos para processamento."
    },
    "PagamentoRecusado": {
      "Type": "Fail",
      "Error": "PagamentoRecusadoError",
      "Cause": "Infelizmente, a transação do seu pedido foi recusada pelo banco emissor."
    }
  }
}
