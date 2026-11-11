# Máquina de Clientes — prospecção fria por WhatsApp (n8n)

Fluxos do n8n da Agência OnlineAgora para prospectar negócios locais (farmácias de manipulação, clínicas, studios etc.) pelo WhatsApp, com CRM no Supabase e envio pelo WAHA (número 2144).

Exportado em 11/11/2026, direto do n8n em produção. O copiloto usa o modelo Claude Opus 5.5 (`claude-opus-5-5`) com saída estruturada em JSON.

## Como os fluxos se encaixam

```
WF00 (captação)  →  WF01 (mensagem 1)  →  WF01b (robô respondeu?)  →  WF01c link (Mensagem B)
                                                                     →  conversa humana → WF02 copiloto → WF02b link
WF01c Resumo Diário: todo dia às 18h30, números do dia no WhatsApp do Hamilton
```

| Arquivo | O que faz | Quando roda |
|---|---|---|
| `WF00_Captacao_Google_Maps.json` | Formulário (nicho + cidade) → Google Places → telefone, avaliações e Instagram → salva contato e lead em "Captado" no CRM. Não repete nicho+cidade já prospectado (a menos que "forçar busca"). | Sob demanda (formulário) |
| `WF01_Disparo.json` | Envia a mensagem 1 (pergunta de abertura) para os leads da fila. Tem travas anti-duplicidade, respeita o horário de funcionamento do Google Meu Negócio e só manda dentro das janelas de envio. | Agendado, dias úteis |
| `WF01b_Detectar_Robo.json` | De 1 a 30 min depois do envio, lê as respostas. Se for robô/menu/assistente virtual (lista de palavras ou resposta em menos de 8 s), bloqueia o lead e prepara a **Mensagem B**. Também marca possível **IA da Meta** (resposta em até 90 s com 2+ sinais). | A cada 5 min, 9h–19h, seg–sex |
| `WF01c_Enviar_Mensagem_B_por_Link.json` | Link de uso único (vale 72 h) que o Hamilton recebe no WhatsApp: abre a Mensagem B, revisa e envia pelo 2144. Registra no CRM. | Webhook (GET mostra, POST envia) |
| `WF01c_Resumo_Diario.json` | Resumo do dia: enviados, respostas, robôs, sem WhatsApp, erros, fila restante e status do canal. | 18h30, seg–sex |
| `WF02_Copiloto_Respostas.json` | Copiloto de conversa. Para leads em "Em Qualificação" e "Lead Quente": quando o lead responde (espera 90 s) ou some (follow-up em 24 h e mais 48 h, no máximo 2), o Claude lê a conversa e escreve a próxima mensagem. O Hamilton recebe no WhatsApp a sugestão + link. Objetivo: conversa de 15 min pelo link da agenda; nunca fala preço. Em contrato, dúvida técnica ou reclamação, só avisa o Hamilton. | A cada 2 min, 8h–19h, seg–sáb |
| `WF02b_Enviar_Sugestao_por_Link.json` | Página do link do copiloto: mostra a sugestão num campo editável, avisa se já saiu mensagem ou se o lead escreveu de novo, e envia pelo 2144 registrando no CRM. Token de uso único, vale 48 h. | Webhook (GET mostra, POST envia) |

### Copiloto: como pausar um lead

Grave `custom_fields.wf02.pausado = true` no lead. Leads cuja conversa segue por e-mail (`wf01.etapa` contém "email") são pulados automaticamente. O histórico do copiloto fica em `custom_fields.wf02.log`.

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
| `<SUPABASE_PROJECT_REF_NICHOS>` | ref do projeto Supabase onde o WF00 registra os nichos prospectados |

As credenciais do n8n (service role do Supabase, API key do WAHA, Google Places, API key da Anthropic) **não vão no arquivo**: os nós só citam o nome da credencial. Recrie as credenciais no n8n de destino e selecione-as nos nós.
