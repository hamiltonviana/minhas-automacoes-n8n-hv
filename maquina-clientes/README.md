# Máquina de Clientes — prospecção fria por WhatsApp (n8n)

Fluxos do n8n da Agência OnlineAgora para prospectar negócios locais (farmácias de manipulação, clínicas, studios etc.) pelo WhatsApp, com CRM no Supabase e envio pelo WAHA (número 2144).

Exportado em 11/11/2026, direto do n8n em produção.

## Como os fluxos se encaixam

```
WF00 (captação)  →  WF01 (mensagem 1)  →  WF01b (robô respondeu?)  →  WF01c link (Mensagem B)
                                                                     →  conversa humana (manual / copiloto)
WF01c Resumo Diário: todo dia às 18h30, números do dia no WhatsApp do Hamilton
```

| Arquivo | O que faz | Quando roda |
|---|---|---|
| `WF01_Disparo.json` | Envia a mensagem 1 (pergunta de abertura) para os leads da fila. Tem travas anti-duplicidade, respeita o horário de funcionamento do Google Meu Negócio e só manda dentro das janelas de envio. | Agendado, dias úteis |
| `WF01b_Detectar_Robo.json` | De 1 a 30 min depois do envio, lê as respostas. Se for robô/menu/assistente virtual (lista de palavras ou resposta em menos de 8 s), bloqueia o lead e prepara a **Mensagem B**. Também marca possível **IA da Meta** (resposta em até 90 s com 2+ sinais). | A cada 5 min, 9h–19h, seg–sex |
| `WF01c_Enviar_Mensagem_B_por_Link.json` | Link de uso único (vale 72 h) que o Hamilton recebe no WhatsApp: abre a Mensagem B, revisa e envia pelo 2144. Registra no CRM. | Webhook (GET mostra, POST envia) |
| `WF01c_Resumo_Diario.json` | Resumo do dia: enviados, respostas, robôs, sem WhatsApp, erros, fila restante e status do canal. | 18h30, seg–sex |

Ainda não estão aqui:
- **WF00 (captação de leads):** não está liberado para exportação via MCP no n8n.
- **WF02 Copiloto:** ainda em construção. Quando um lead em conversa responder, a IA vai escrever a sugestão de resposta e mandar para o Hamilton aprovar por link, com follow-up de agendamento (link da agenda, conversa de 15 min).

## Regras de negócio importantes

- Depois do robô e da Mensagem B, o lead fica **bloqueado para automação**. Daí em diante a conversa é humana: o Hamilton responde, e o copiloto só sugere.
- Mensagens curtas, humanas, sem criticar o robô do lead.
- Lojas da mesma rede: se uma unidade está em conversa, as outras ficam pausadas (`wf01.bloqueado = true`).
- Pizzarias e hamburguerias fecham de dia: o WF01 usa o horário do Google Meu Negócio para não mandar com a loja fechada.
- Não aumentar o corte de 8 s do detector de robô: a IA da Meta responde em uns 12 s e seria confundida com robô.

## Para importar em outro n8n

Os segredos e dados pessoais foram trocados por marcadores. Antes de usar, troque:

| Marcador | O que colocar |
|---|---|
| `<SUPABASE_PROJECT_REF>` | ref do projeto Supabase (o pedaço antes de `.supabase.co`) |
| `<SUPABASE_ANON_KEY>` | anon key do Supabase (header `apikey`) |
| `<ORGANIZATION_ID>` | id da organização no CRM |
| `<WAHA_SESSION>` | nome da sessão do WAHA |
| `<TELEFONE_HAMILTON>` / `<TELEFONE_HAMILTON_FORMATADO>` | número que recebe os avisos |
| `<EMAIL_HAMILTON>` | e-mail de contato usado nas mensagens |

As credenciais do n8n (service role do Supabase, API key do WAHA, Google Places) **não vão no arquivo**: os nós só citam o nome da credencial. Recrie as credenciais no n8n de destino e selecione-as nos nós.
