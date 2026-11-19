# Assistente de Atendimento e Consulta de Estoque via WhatsApp — TechStore

Automação de atendimento comercial via WhatsApp utilizando **n8n**, **WAHA (WhatsApp HTTP API)**, **Google Gemini** e **Google Sheets**, operando em contêineres **Docker**. O fluxo é ativado pela tag `#loja`, consulta o estoque da loja em tempo real e responde ao cliente de forma imediata, formal e contextualizada.

---

## 1. Descrição do Problema
O atendimento comercial via WhatsApp em lojas de tecnologia enfrenta gargalos recorrentes: demora na resposta sobre disponibilidade de produtos, consultas manuais constantes a planilhas de estoque e perda de oportunidades de venda fora do horário comercial.

Este projeto resolve esse problema criando um **assistente virtual automatizado integrado ao WhatsApp**, capaz de:
- Filtrar mensagens direcionadas à loja por meio da palavra-chave `#loja`.
- Evitar loops infinitos ignorando mensagens enviadas pelo próprio bot (`fromMe: false`).
- Consultar dados atualizados de produtos, preços e quantidades em tempo real antes de responder.
- Manter o histórico recente da conversa individualizado por cliente.

---

## 2. Arquitetura da Solução
A infraestrutura roda em ambiente local containerizado (**Docker**), orquestrando a comunicação entre os serviços via rede interna (`host.docker.internal`):

1. **Gatilho (`Webhook1` - Método POST):** Escuta os eventos de mensagens recebidas disparados pelo contêiner do **WAHA** (`http://host.docker.internal:5678/webhook/webhook`).
2. **Filtro de Segurança e Intenção (`If`):** Valida duas condições simultâneas (`AND`) antes de acionar a IA:
   - Verifica se o conteúdo contém a tag de ativação `#loja` (`{{ $json.body.payload.body }}`).
   - Garante que a mensagem não foi enviada pelo próprio número do bot (`{{ $json.body.payload.fromMe }}` é `false`).
3. **Orquestração Cognitiva (`AI Agent`):** Recebe a mensagem validada do cliente e gerencia o raciocínio utilizando três sub-nós conectados:
   - **Modelo de Linguagem (`Google Gemini Chat Model`):** Utiliza o modelo `gemini-1.5-flash` para interpretar a linguagem natural do cliente e formular respostas estruturadas.
   - **Memória Conversacional (`Simple Memory`):** Armazena o histórico recente da conversa (janela de contexto de 5 interações) indexado pelo identificador único do remetente (`{{ $('Webhook1').item.json.body.payload.from }}`).
   - **Ferramenta de Consulta (`Consultar_Estoque` - HTTP Request GET):** Permite que o agente acesse em tempo real a base oficial de produtos, preços e quantidades da TechStore hospedada no Google Sheets.
4. **Envio da Resposta (`HTTP Request` - Método POST):** Recebe o texto final gerado pelo `AI Agent` e realiza uma requisição autenticada (`X-Api-Key`) para o endpoint `/api/sendText` do WAHA (`http://host.docker.internal:3000/api/sendText`), entregando a mensagem de volta no WhatsApp do cliente.

---

## 3. Engenharia de Prompts

### Prompt do Sistema (System Message)
Define a persona, o tom de voz e as restrições operacionais do agente:
> Você é o assistente virtual oficial da TechStore, uma loja especializada em produtos de tecnologia. Seu atendimento deve ser estritamente formal, cordial, claro e objetivo.
>
> **REGRAS DE ATENDIMENTO:**
> 1. Sempre que o cliente perguntar sobre produtos, valores ou disponibilidade, utilize obrigatoriamente a ferramenta `Consultar_Estoque` antes de responder.
> 2. Informe apenas produtos que constam na base de dados, especificando nome, valor e quantidade disponível.
> 3. Caso o produto não esteja disponível ou não exista no estoque, informe educadamente e sugira alternativas semelhantes presentes na planilha.
> 4. Nunca invente preços, especificações técnicas ou promoções que não estejam registradas na consulta.

### Prompt do Usuário (User Message / Expressão Dinâmica)
Captura dinamicamente o texto ou legenda enviada pelo cliente no WhatsApp:
```javascript
{{ $('Webhook1').item.json.body.payload.body || $('Webhook1').item.json.body.payload.caption }}# IA-atendimento
