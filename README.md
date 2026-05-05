# 🌤️ Chatbot de Clima — Telegram + N8N + OpenWeather

Workflow automatizado no N8N que responde consultas de temperatura via Telegram, integrando a API do OpenWeatherMap com tratamento de erros e normalização de entrada.

---

## 📋 Visão Geral

O usuário envia o nome de uma cidade pelo Telegram e o bot responde com a temperatura atual em Celsius.

```
Usuário: São Paulo,SP,BR
Bot: 🌤️ A temperatura na cidade de São Paulo é de 28°C.
```

---

## 🔄 Fluxo do Workflow

```
Telegram Trigger
      ↓
Edit Fields (normaliza o texto → variável: queue)
      ↓
HTTP Request (OpenWeatherMap API via credencial N8N)
      ↓
If (cod == 200?)
   ↓ Sim                    ↓ Não
Temperatura              RequestError
(valida name + main.temp)
   ↓                          ↓
Send Message           Send Message (erro)
```

---

## 🧩 Nodes

### 1. Telegram Trigger
Escuta mensagens recebidas no bot do Telegram.

- **Evento:** `message`
- **Campo usado:** `$json.message.text`

---

### 2. Edit Fields (Set)
Normaliza o texto enviado pelo usuário para compatibilidade com a API OpenWeather. O resultado é armazenado na variável `queue`.

**Expressão aplicada:**
```js
{{ $json.message.text
  .normalize("NFD")
  .replace(/[\u0300-\u036f]/g, "")
  .toLowerCase()
  .trim()
  .replace(/,\s*/g, ",") }}
```

| Transformação | Exemplo |
|---|---|
| Entrada do usuário | `São Paulo, SP, BR` |
| Após normalização (`queue`) | `sao paulo,sp,br` |

> ℹ️ A variável foi renomeada de `chatInput` para `queue` para refletir melhor sua semântica de fila de consulta à API.

---

### 3. HTTP Request
Consulta a API do OpenWeatherMap com o texto normalizado. A API Key é lida diretamente da credencial configurada no N8N — **não há nenhum valor hardcoded no workflow**.

- **URL:** `https://api.openweathermap.org/data/2.5/weather`
- **Parâmetros:**

| Parâmetro | Valor |
|---|---|
| `appid` | `={{ $credentials.openWeatherMapApi.apiKey }}` |
| `q` | `={{ $json.queue }}` |
| `lang` | `pt_br` |
| `units` | `metric` |

> ℹ️ `units=metric` retorna a temperatura em **Celsius**.

---

### 4. If
Verifica se a API retornou sucesso.

- **Condição:** `cod == 200`
- ✅ **True** → node `Temperatura`
- ❌ **False** → node `RequestError`

---

### 5. Temperatura (Set)
Monta a mensagem de resposta com os dados da API. Antes de formatar, **valida se os campos `name` e `main.temp` estão presentes** na resposta para evitar erros silenciosos.

```js
{{
  ($json.name && $json.main && $json.main.temp !== undefined)
    ? '🌤️ A temperatura na cidade de ' + $json.name + ' é de ' + Math.round($json.main.temp) + '°C.'
    : '⚠️ Resposta incompleta da API. Tente novamente.'
}}
```

| Situação | Mensagem exibida |
|---|---|
| Campos presentes | `🌤️ A temperatura na cidade de São Paulo é de 28°C.` |
| Campos ausentes | `⚠️ Resposta incompleta da API. Tente novamente.` |

---

### 6. RequestError (Set)
Monta a mensagem de erro quando a cidade não é encontrada (API retorna `cod != 200`).

```
❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).
```

---

### 7. Send a text message / Send a text message1
Envia a resposta de volta ao usuário no Telegram.

- **Chat ID:** `{{ $('Telegram Trigger').item.json.message.chat.id }}`
- **Texto (sucesso):** `{{ $json.mensagem }}`
- **Texto (erro):** `{{ $json.Erro }}`

---

## ⚙️ Configuração

### Pré-requisitos

- [N8N](https://n8n.io/) instalado (self-hosted ou cloud)
- Bot do Telegram criado via [@BotFather](https://t.me/BotFather)
- Chave de API do [OpenWeatherMap](https://openweathermap.org/api)

---

### Passo a passo

#### 1. Importe o workflow
No N8N, vá em **Settings → Import Workflow** e cole o conteúdo do arquivo `workflow-chatbot-telegram.json`.

#### 2. Configure a credencial do Telegram
1. Vá em **Credentials → New → Telegram**
2. Insira o token do bot gerado pelo BotFather
3. Salve como `Telegram account`

#### 3. Configure a credencial do OpenWeather (recomendado)

> ⚠️ **Nunca cole sua API Key diretamente no campo do node.** Use o sistema de credenciais do N8N para manter a chave segura.

**Opção A — Credencial nativa (recomendado):**
1. Vá em **Credentials → New → Header Auth** (ou **Query Auth**)
2. Configure:
   - **Name:** `OpenWeather API`
   - **Name (parâmetro):** `appid`
   - **Value:** `sua_api_key_aqui`
3. No node **HTTP Request**, vínculo a credencial criada em **Authentication → Predefined Credential Type → OpenWeather API**

**Opção B — Variável de ambiente:**
1. No arquivo `.env` da sua instância N8N, adicione:
   ```env
   OPENWEATHER_API_KEY=sua_api_key_aqui
   ```
2. No node **HTTP Request**, use a expressão:
   ```js
   {{ $env.OPENWEATHER_API_KEY }}
   ```

#### 4. Ative o workflow
Clique no toggle **Active** no canto superior direito.

---

## 💬 Como usar

Envie uma mensagem para o bot no seguinte formato:

```
Cidade,UF,BR
```

**Exemplos:**

```
São Paulo,SP,BR
Rio de Janeiro,RJ,BR
Brasília,DF,BR
Belo Horizonte,MG,BR
```

> A entrada é case-insensitive e aceita acentos — a normalização é feita automaticamente pelo node **Edit Fields**, que converte o texto para a variável `queue`.

---

## ⚠️ Observações

- **Telegram Trigger em teste:** O N8N não consegue escutar execuções de teste e produção simultaneamente. Para testar, desative o workflow antes de usar o botão "Test Workflow", ou deixe ativo e verifique em `Executions`.
- **Segurança da API Key:** Nunca exponha sua chave do OpenWeather no JSON do workflow. Use sempre o sistema de credenciais do N8N (opção A) ou variável de ambiente (opção B) conforme descrito acima.
- **Rate limit:** O plano gratuito do OpenWeather permite até 60 requisições/minuto.
- **Validação de campos:** O node `Temperatura` valida a presença de `name` e `main.temp` antes de montar a mensagem. Caso a API retorne um objeto inesperado mesmo com `cod 200`, o usuário receberá um aviso de resposta incompleta em vez de uma mensagem quebrada.

---

## 📁 Estrutura do JSON

```
nodes
├── Telegram Trigger       → recebe a mensagem
├── Edit Fields            → normaliza o texto → variável: queue
├── HTTP Request           → consulta a API (credencial N8N)
├── If                     → verifica cod == 200
├── Temperatura            → valida campos e monta resposta de sucesso
├── RequestError           → monta resposta de erro
├── Send a text message    → envia sucesso ao Telegram
└── Send a text message1   → envia erro ao Telegram
```

---

## 🔄 Changelog

| Versão | Alteração |
|---|---|
| 1.1 | Variável `chatInput` renomeada para `queue` |
| 1.1 | API Key migrada de valor hardcoded para referência de credencial N8N |
| 1.1 | Adicionada validação dos campos `name` e `main.temp` antes de formatar a mensagem |
| 1.1 | Documentação de credenciais expandida com Opção A e Opção B |
| 1.0 | Versão inicial |

---

## 📄 Licença

Livre para uso e modificação.

## Exemplo de execução
https://ibb.co/84TkZZ72
