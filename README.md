Segue o README pronto para colar no seu repositório. Ele descreve o workflow como está salvo agora (com o Agente de IA da Etapa 5).

# 🤖 Assistente de Investimentos com RPA + n8n + IA Generativa

Projeto final do curso **"Criando um Assistente de Investimentos com RPA e IA Generativa"** da [DIO](https://www.dio.me/).

O objetivo é automatizar, de ponta a ponta, o envio de recomendações de investimento personalizadas para clientes de uma corretora fictícia:

1. **RPA (Python/Colab)** extrai a lista de clientes de uma página web.
2. **n8n** recebe esses clientes, cruza com a base de investimentos e organiza os dados.
3. **IA generativa (Google Gemini)** escreve uma mensagem personalizada para cada cliente.

---

## 🗺️ Arquitetura

```
┌──────────────────────┐     POST JSON      ┌──────────────────────────────────────────────┐
│  Colab (Python/RPA)  │ ─────────────────▶ │                   n8n                         │
│  extrai #clientes    │                    │                                              │
│  da página da DIO    │ ◀───────────────── │ Webhook → CSV → Cruzamento → Agente IA →     │
└──────────────────────┘  JSON c/ mensagens │ Montar Resposta → Responder                  │
                                            └──────────────────────────────────────────────┘
```

---

## 📁 Estrutura do repositório

```
├── docs/
│   └── data.csv               # Base de investimentos (perfil, produto, mínimo, rentabilidade)
├── src/
│   └── extrair_clientes.ipynb # Notebook de RPA (Colab) que extrai e envia os clientes
├── n8n/
│   └── workflow.json          # Workflow exportado do n8n
└── README.md
```

---

## 🧩 Etapas do projeto

### Etapa 1 – Extração dos clientes (RPA)
O notebook usa `requests` + `BeautifulSoup` para ler a tabela `#clientes` da
[página do laboratório](https://digitalinnovationone.github.io/dio-lab-assistente-investimentos-rpa-n8n/)
e montar uma lista com `nome`, `email`, `saldo` e `perfil` de cada cliente.

### Etapa 2 – Envio para o n8n
A lista é enviada via `POST` para o webhook do n8n no formato:

```json
{ "clientes": [ { "nome": "...", "email": "...", "saldo": "R$ 12.500,00", "perfil": "Moderado" } ] }
```

### Etapa 3 – Base de investimentos
O n8n baixa o arquivo [`docs/data.csv`](docs/data.csv) direto do GitHub (raw) e converte
cada linha em um item com `perfil`, `produto`, `minimo` e `rentabilidade`.

### Etapa 4 – Cruzamento cliente × investimento
Um nó de código (JavaScript) faz duas coisas para cada cliente:
- converte o saldo do formato brasileiro (`"R$ 12.500,00"`) para número (`12500`);
- seleciona só os produtos **do mesmo perfil** cujo **valor mínimo cabe no saldo**.

### Etapa 5 – Mensagem personalizada com IA
Um **Agente de IA** com o modelo **Google Gemini** recebe nome, perfil, saldo e a lista
de investimentos compatíveis, e escreve uma mensagem curta e personalizada em português.
O resultado volta para o Colab como resposta do webhook.

---

## ⚙️ Workflow no n8n

| # | Nó | Função |
|---|----|--------|
| 1 | **Receber Clientes (Webhook)** | Recebe o `POST` do Colab em `/webhook/assistente-investimentos` |
| 2 | **Baixar Investimentos (CSV)** | Baixa o `data.csv` do repositório no GitHub |
| 3 | **Ler Linhas do CSV** | Converte o arquivo em itens (usando a primeira linha como cabeçalho) |
| 4 | **Cruzar Clientes x Investimentos** | Filtra os produtos por perfil e saldo para cada cliente |
| 5 | **Gerar Mensagem (Agente IA)** | Gemini escreve a recomendação personalizada |
| 6 | **Montar Resposta** | Junta `nome`, `email`, `perfil` e `mensagem` |
| 7 | **Responder com Mensagens** | Devolve todas as mensagens em JSON para o Colab |

Para importar: no n8n, **Workflows → Import from File** e selecione `n8n/workflow.json`.

---

## 🧠 Decisões técnicas

- **Webhook com resposta síncrona (`Respond to Webhook`)**: o Colab envia os clientes e recebe as
  mensagens na mesma requisição, sem precisar de banco de dados nem de uma segunda chamada.
  Isso deixa o fluxo simples e fácil de testar.
- **CSV lido direto do GitHub**: a base de investimentos fica versionada no repositório. Para
  alterar produtos ou rentabilidades, basta editar o `data.csv`, sem mexer no workflow.
- **Filtro de regras antes da IA**: a escolha dos produtos (perfil + valor mínimo) é feita
  por código, de forma **determinística**. A IA só redige o texto. Assim ela não "inventa"
  produtos nem recomenda algo que o cliente não pode comprar.
- **Prompt com regras claras**: o *system message* limita a mensagem a 4 frases, exige o
  primeiro nome do cliente, explica o que cada perfil significa e proíbe produtos ou números
  fora da lista recebida. Se nenhum produto couber no saldo, a IA incentiva o cliente a
  começar aos poucos.
- **Modelo Gemini Flash Lite (temperatura 0.7)**: rápido e barato, suficiente para textos
  curtos. A temperatura 0.7 dá variedade às mensagens sem perder coerência.
- **Tratamento do saldo em formato brasileiro**: o código remove `R$` e os pontos de milhar e
  troca a vírgula decimal por ponto, para comparar o saldo com o valor mínimo de cada produto.
- **Entrada flexível no webhook**: o código aceita tanto `{"clientes": [...]}` quanto uma lista
  direta, o que facilita testar com outras ferramentas (Postman, curl etc.).

---

## ▶️ Como executar

1. Importe `n8n/workflow.json` no seu n8n e configure a credencial do Google Gemini.
2. **Teste:** clique em *Execute workflow* e use a URL `.../webhook-test/assistente-investimentos`.
   **Produção:** publique o workflow e use `.../webhook/assistente-investimentos`.
3. No Colab, rode o notebook `src/extrair_clientes.ipynb` com a URL do webhook.
4. Para exibir as mensagens geradas:

```python
resposta = requests.post(URL_WEBHOOK, json={"clientes": clientes})
for item in resposta.json():
    print(f"{item['nome']} ({item['perfil']}) - {item['email']}")
    print(item['mensagem'])
    print("-" * 60)
```

---

## 📌 Exemplo de saída

Saída real impressa no Colab após a execução do workflow (resposta `200`):

```
Ana Silva (Conservador) - ana@email.com
Oi, Ana! Com o seu perfil conservador e esse saldo de R$ 12.500,00, temos ótimas opções para
valorizar seu patrimônio com segurança. O Tesouro Selic, com rentabilidade de 13%, é uma excelente
escolha para o seu momento, mas o CDB de Liquidez Diária a 12,5% também oferece uma ótima
alternativa para manter a flexibilidade. Vamos aproveitar essas taxas para colocar seu dinheiro
para trabalhar da melhor forma?

Carla Souza (Arrojado) - carla@email.com
Carla, com esse saldo e seu perfil arrojado, temos ótimas oportunidades para potencializar seus
retornos! Para diversificar sua carteira, sugiro focarmos em Ações ETF, Fundo de Ações e
Criptomoedas. Todos esses ativos oferecem o dinamismo que você busca para o seu patrimônio.
Vamos aproveitar esse momento de mercado para montar essa estratégia juntos?

Daniel Oliveira (Conservador) - daniel@email.com
Daniel, que prazer ver você dando esse passo importante para cuidar do seu patrimônio! Como seu
perfil é conservador, minha recomendação é aproveitar o Tesouro Selic, que entrega uma
rentabilidade bem superior à da poupança com total segurança. Com esses R$ 850,00, você já
consegue garantir uma excelente base para sua reserva de emergência. Vamos colocar esse valor
para render de verdade hoje mesmo?
```

Formato JSON devolvido pelo webhook (um item por cliente):

```json
{
  "nome": "Ana Silva",
  "email": "ana@email.com",
  "perfil": "Conservador",
  "mensagem": "Oi, Ana! Com o seu perfil conservador e esse saldo de R$ 12.500,00, ..."
}
```

**Observações sobre o resultado**
- O filtro por saldo funciona: o Daniel (R$ 850,00) recebeu só o Tesouro Selic, porque os outros
  produtos conservadores têm valor mínimo maior que o saldo dele.
- Cada mensagem usa o primeiro nome, cita o perfil e recomenda só os produtos da lista.
- Limitação: o modelo às vezes simplifica a rentabilidade (ex.: escreveu "Tesouro IPCA+ a 6,0%"
  em vez de "IPCA + 6,0%"). Para uso real, as mensagens precisariam de revisão humana ou de um
  prompt mais rígido.

---

## 🚀 Melhorias futuras

- Enviar as mensagens por e-mail (Gmail) automaticamente para cada cliente.
- Registrar o histórico de recomendações em uma planilha ou Data Table.
- Agendar a extração dos clientes para rodar periodicamente.

---

## 👤 Autor

**Tiago Musardo** – [GitHub](https://github.com/musardotiago)
