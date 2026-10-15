# Resumo Diário de Vagas (n8n + Gemini)

Automação que roda sozinha todo dia em um horário agendado: busca vagas remotas de tecnologia na API pública do Remotive, usa IA (Google Gemini) para selecionar, classificar o nível e resumir as vagas, e envia o resultado no Telegram.

## Fluxo

```
⏰ Schedule Trigger (todo dia, horário agendado)
        │
        ▼
🌐 HTTP Request ── API do Remotive (category=software-dev, limit=10)
        │
        ▼
✂️ Split Out ── separa a lista em 1 item por vaga
        │
        ▼
📝 Edit Fields ── mantém só título, empresa, categoria, tipo, local e link
        │
        ▼
📦 Aggregate ── junta tudo em 1 item (1 chamada à IA, 1 mensagem)
        │
        ▼
🤖 Google Gemini ── filtra vagas de tecnologia, classifica o nível e formata
        │
        ▼
📲 Telegram ── envia o resumo no chat
```

## Tecnologias

- **n8n** (self-hosted, local): orquestração do fluxo
- **Schedule Trigger**: execução automática por horário
- **API REST do Remotive** (pública e gratuita): fonte das vagas
- **Google Gemini API** (plano gratuito): seleção, classificação e resumo das vagas
- **Telegram Bot API**: envio do resumo

## Decisões de projeto

- **Limpeza dos dados antes da IA:** o nó Edit Fields descarta a descrição completa das vagas (HTML extenso) e envia ao modelo apenas os campos necessários. Isso reduz o consumo de tokens e deixa a resposta mais rápida e precisa.
- **Agregação em uma única lista:** sem o nó Aggregate, o fluxo chamaria a IA e enviaria uma mensagem para cada vaga. Juntando tudo, são feitas apenas 1 chamada ao modelo e 1 mensagem no Telegram, otimizando custo e evitando spam.
- **Engenharia de prompt:** o prompt define regras claras (somente vagas de tecnologia, no máximo 5, classificação de nível, ordem das mais acessíveis para as mais exigentes, sem Markdown e com citação da fonte).
- **Resiliência:** os nós do Gemini e do Telegram têm *Retry On Fail* ativado para lidar com falhas temporárias.

## Como rodar

### Pré-requisitos

- Node.js e n8n instalados (`npm install -g n8n`)
- Bot criado no Telegram pelo **@BotFather** (gera o token)
- Chave da API do Gemini gerada no **Google AI Studio**

### Passo a passo

1. Inicie o n8n:
   ```powershell
   n8n
   ```
2. Acesse **http://localhost:5678** e importe o arquivo `workflow/workflow.json` (menu **⋯ → Import**).
3. Em **⋯ → Settings → Timezone**, selecione **America/Sao_Paulo** (ou o seu fuso).
4. Crie as credenciais no n8n:
   - **Telegram API**: token do bot
   - **Google Gemini (PaLM) API**: chave do AI Studio
5. No nó **Send a text message**, troque `SEU_CHAT_ID` pelo seu Chat ID do Telegram.
6. Ajuste o horário no **Schedule Trigger** (no repositório está configurado para 20h) e clique em **Publish**.

> As credenciais ficam armazenadas criptografadas no n8n. O arquivo `workflow.json` contém apenas referências a elas, sem tokens ou chaves.

## Prints

### Workflow no n8n
![Workflow no n8n](prints/canvas.png)

### Resumo recebido no Telegram (execução automática)
![Mensagem automática](prints/mensagem-automatica.png)

![Início da mensagem](prints/mensagem-inicio.png)

## Relato de problemas enfrentados

### Fuso horário do agendamento
**Problema:** o agendamento não disparou no horário configurado. O n8n usa por padrão o fuso de Nova York, então o horário marcado não correspondia ao horário de Brasília.
**Solução:** configurei o fuso do workflow como **America/Sao_Paulo** em *Settings → Timezone*. Para validar, testei com intervalo de 2 minutos e confirmei que a mensagem chegou sozinha, sem execução manual.

### Falta de vagas júnior na fonte
**Problema:** a proposta inicial era enviar apenas vagas júnior. O feed RSS do Remotive trazia só 8 vagas recentes, todas sênior. A busca por "junior" na API retornava vagas de outras áreas (redação, vendas, atendimento), porque o termo era encontrado em qualquer parte do texto, e nenhuma era de fato júnior de tecnologia.
**Solução:** troquei o RSS pela **API do Remotive**, filtrando pela categoria de desenvolvimento, e mudei o papel da IA: em vez de apenas filtrar júnior, ela **classifica o nível** de cada vaga, ordena das mais acessíveis para as mais exigentes e marca com ⭐ as vagas para iniciantes quando aparecem.
**Aprendizado:** vagas remotas internacionais para iniciantes são raras nessa fonte. Uma melhoria futura é incluir fontes brasileiras com mais vagas de estágio e júnior.

### Mensagem longa demais para o Telegram
**Problema:** com todas as vagas, o texto gerado ficou próximo do limite de 4.096 caracteres por mensagem do Telegram.
**Solução:** limitei o resumo a no máximo 5 vagas no prompt.

### Timeout no envio para o Telegram
**Problema:** o nó do Telegram falhou com *connection timed out*.
**Solução:** ativei o *Retry On Fail* (3 tentativas, 3 segundos de intervalo).

## Limitações

- O n8n roda localmente: o resumo só é enviado se o computador estiver ligado com o n8n aberto no horário agendado. Execuções perdidas não são recuperadas.
- As vagas da API do Remotive aparecem com 24 horas de atraso em relação ao site.

## Melhorias futuras

- Fazer o deploy do n8n em um servidor na nuvem para rodar 24 horas
- Adicionar fontes brasileiras com vagas de estágio e júnior
- Evitar repetir no dia seguinte vagas que já foram enviadas

## Fonte dos dados e termos de uso

As vagas são obtidas pela [API pública do Remotive](https://github.com/remotive-com/remote-jobs-api). Conforme os termos de uso, cada vaga é exibida com o link original e o Remotive é citado como fonte. A API é consultada no máximo algumas vezes por dia, e este repositório não armazena nem redistribui as vagas.

## Autor

Gabriel Moreira Melo, estudante de Análise e Desenvolvimento de Sistemas (ULBRA)
