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
Edit Fields (normaliza o texto)
      ↓
HTTP Request (OpenWeatherMap API)
      ↓
If (cod == 200?)
   ↓ Sim              ↓ Não
Temperatura        RequestError
   ↓                    ↓
Send Message      Send Message (erro)
```

---

## 🧩 Nodes

### 1. Telegram Trigger
Escuta mensagens recebidas no bot do Telegram.

- **Evento:** `message`
- **Campo usado:** `$json.message.text`

---

### 2. Edit Fields (Set)
Normaliza o texto enviado pelo usuário para compatibilidade com a API OpenWeather.

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
| Após normalização | `sao paulo,sp,br` |

---

### 3. HTTP Request
Consulta a API do OpenWeatherMap com o texto normalizado.

- **URL:** `https://api.openweathermap.org/data/2.5/weather`
- **Parâmetros:**

| Parâmetro | Valor |
|---|---|
| `appid` | `SUA_API_KEY` |
| `q` | `{{ $json.chatInput }}` |
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
Monta a mensagem de resposta com os dados da API.

```json
{
  "mensagem": "🌤️ A temperatura na cidade de {{ $json.name }} é de {{ Math.round($json.main.temp) }}°C."
}
```

---

### 6. RequestError (Set)
Monta a mensagem de erro quando a cidade não é encontrada.

```json
{
  "Erro": "❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR)."
}
```

---

### 7. Send a text message / Send a text message1
Envia a resposta de volta ao usuário no Telegram.

- **Chat ID:** `{{ $('Telegram Trigger').item.json.message.chat.id }}`
- **Texto:** `{{ $json.mensagem }}` ou `{{ $json.Erro }}`

---

## ⚙️ Configuração

### Pré-requisitos

- [N8N](https://n8n.io/) instalado (self-hosted ou cloud)
- Bot do Telegram criado via [@BotFather](https://t.me/BotFather)
- Chave de API do [OpenWeatherMap](https://openweathermap.org/api)

### Passo a passo

1. **Importe o workflow** no N8N via `Settings → Import Workflow` e cole o JSON.

2. **Configure as credenciais do Telegram:**
   - Vá em `Credentials → New → Telegram`
   - Insira o token do bot gerado pelo BotFather

3. **Substitua a API Key do OpenWeather:**
   - No node `HTTP Request`, troque o valor do parâmetro `appid` pela sua chave

4. **Ative o workflow** pelo toggle no canto superior direito.

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

> A entrada é case-insensitive e aceita acentos — a normalização é feita automaticamente.

---

## ⚠️ Observações

- **Telegram Trigger em teste:** O N8N não consegue escutar execuções de teste e produção simultaneamente. Para testar, desative o workflow antes de usar o botão "Test Workflow", ou deixe ativo e verifique em `Executions`.
- **API Key:** Não exponha sua chave do OpenWeather publicamente. Use as credenciais do N8N para armazená-la com segurança.
- **Rate limit:** O plano gratuito do OpenWeather permite até 60 requisições/minuto.

---

## 📁 Estrutura do JSON

```
nodes
├── Telegram Trigger       → recebe a mensagem
├── Edit Fields            → normaliza o texto
├── HTTP Request           → consulta a API
├── If                     → verifica o retorno
├── Temperatura            → monta resposta de sucesso
├── RequestError           → monta resposta de erro
├── Send a text message    → envia sucesso ao Telegram
└── Send a text message1   → envia erro ao Telegram
```

---

## 📄 Licença

MIT — livre para uso e modificação.
