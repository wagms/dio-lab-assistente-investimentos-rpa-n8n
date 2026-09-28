# Criando um Assistente de Investimentos com RPA e IA Generativa

## Descrição

Aprenda na prática como criar um fluxo de automação inteligente combinando técnicas de RPA (Robotic Process Automation) com workflows de IA no N8N.

Neste desafio, você vai construir um assistente de investimentos automatizado. O fluxo começa com a extração de dados de clientes em uma página web usando Python, passa pela orquestração de um workflow no N8N e termina com a geração de mensagens personalizadas para cada perfil de investidor.

## Objetivo do Projeto

Desenvolver um pipeline de automação que:

- Coleta dados de clientes de uma página web simulada usando Python
- Processa as informações através de um workflow no N8N
- Cruza perfis de investidor com uma base de opções de investimento
- Gera mensagens personalizadas para cada cliente

## Tecnologias e Ferramentas

| Etapa | Ferramenta | Função |
|---|---|---|
| Hospedagem | GitHub Pages | Servir a página de clientes e o CSV de investimentos |
| Extração (RPA) | Python + BeautifulSoup | Coletar dados dos clientes via web scraping |
| Orquestração | N8N | Processar dados, cruzar perfis e gerar mensagens |
| Geração com IA | Agente de IA no N8N (Gemini) | Criar mensagens personalizadas com LLM |

---

## ✅ Minha implementação (Desafio Completo)

Este fork implementa o **Desafio Completo**: extração via RPA, workflow no N8N com cruzamento de perfil × opções de investimento, e geração de mensagens personalizadas por um Agente de IA (Google Gemini).

### Como o pipeline funciona, de ponta a ponta

1. **Extração (RPA em Python)** — `rpa/extrair_clientes.ipynb`, rodado no Google Colab, acessa a página de clientes publicada em GitHub Pages, usa `BeautifulSoup` para ler a tabela HTML (`#clientes tbody tr`) e monta uma lista de dicionários `{nome, email, saldo, perfil}`. Ao final, o script faz um `POST` dessa lista para o Webhook do N8N.
2. **Recepção (Webhook)** — o node **Webhook Clientes** recebe o JSON `{ "clientes": [...] }` e dispara o workflow.
3. **Busca das opções de investimento** — o node **Buscar Opções de Investimento** faz um `GET` no `docs/data.csv` (hospedado via GitHub Pages) e o node **Ler CSV** (Extract From File) converte o CSV em uma lista de objetos `{perfil, produto, minimo, rentabilidade}`. O node **Agregar Opções** junta todas as linhas em um único array.
4. **Cruzamento perfil × opções** — o node de código **Montar Recomendações** (JavaScript) pega a lista de clientes (vinda do Webhook) e a lista de opções (vinda do CSV) e, para cada cliente, filtra apenas as opções compatíveis com o perfil dele (Conservador, Moderado ou Arrojado), gerando um item por cliente já com seu contexto de perfil e as opções elegíveis.
5. **Geração da mensagem com IA** — o node **Gerar Mensagem Personalizada** é um Agente de IA do N8N conectado a um modelo de chat (**Google Gemini**, node "Modelo Gemini"). Ele recebe, por cliente, o nome, saldo, perfil e as opções compatíveis, e gera uma mensagem curta, consultiva e personalizada — nunca prometendo rentabilidade garantida nem sugerindo produtos fora da lista fornecida (isso está fixado no *system message* do agente).
6. **Formatação e resposta** — o node **Formatar Saída** monta o objeto final (`nome`, `email`, `perfil`, `mensagem`), o node **Agregar Resultado Final** junta todas as mensagens geradas em uma lista só, e o **Responder Webhook** devolve esse array como resposta HTTP ao script Python.

### Por que essas decisões técnicas

- **CSV lido via HTTP + Extract From File, em vez de hardcoded no workflow**: mantém o workflow desacoplado dos dados — trocar as opções de investimento significa só editar o `data.csv` publicado, sem tocar no fluxo do N8N.
- **Cruzamento feito em um node de Code, não em vários nodes de Filter/Merge**: com o volume de dados do desafio (poucos perfis, poucas opções por perfil), um único node de JavaScript é mais direto de ler e depurar do que uma cadeia de nodes visuais para essa lógica específica de filtro por perfil.
- **Agente de IA com *system message* restritivo**: a regra "nunca invente produtos fora da lista e nunca prometa rentabilidade garantida" foi colocada diretamente no *system message* do agente (não deixada apenas implícita no prompt do usuário), para reduzir a chance de alucinação do modelo.
- **Resposta síncrona via Respond to Webhook**: como o desafio pede uma demonstração de ponta a ponta (RPA → N8N → mensagens), a resposta HTTP devolve o resultado processado diretamente para quem chamou o Webhook (o script Python), o que facilita testar e conferir a saída sem precisar abrir o N8N para ver o resultado.

### Estrutura do repositório

```
dio-lab-assistente-investimentos-rpa-n8n/
├── README.md
├── rpa/
│   └── extrair_clientes.ipynb   # Script de RPA (Python + BeautifulSoup), com envio ao Webhook configurado
├── n8n/
│   └── workflow.json            # Workflow completo exportado do N8N (RPA → cruzamento → Agente de IA → resposta)
└── docs/
    ├── index.html               # Página de clientes (fornecida pelo desafio)
    └── data.csv                 # Opções de investimento por perfil (fornecida pelo desafio)
```

### Como rodar

1. Importe `n8n/workflow.json` em uma instância do N8N (Cloud ou local).
2. No node **Modelo Gemini**, configure sua própria credencial de API do Google Gemini (o workflow não traz nenhuma chave — cada pessoa usa a sua).
3. Ative o workflow e copie a URL do node **Webhook Clientes** (Test URL para testar, Production URL depois de publicado).
4. Abra `rpa/extrair_clientes.ipynb` no Google Colab, cole essa URL na variável `N8N_WEBHOOK` e rode todas as células.
5. O notebook imprime a lista de clientes extraída e, em seguida, a resposta do N8N com as mensagens personalizadas geradas para cada cliente.

---

## Referências

- [Documentação do N8N](https://docs.n8n.io/)
- [BeautifulSoup: Web Scraping com Python](https://www.crummy.com/software/BeautifulSoup/bs4/doc/)
- [GitHub Pages: Guia Rápido](https://pages.github.com/)
