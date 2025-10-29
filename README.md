# 🤖 Agente de Agendamento de Consultas via WhatsApp com n8n e OpenAI

Este projeto é um sistema automatizado de agendamento de consultas, construído inteiramente na plataforma de automação de fluxo de trabalho **n8n**. Ele utiliza a **API da OpenAI** para criar um agente de IA conversacional que interage com clientes via **WhatsApp** (através da Evolution API) para agendar, reagendar, cancelar e sugerir horários de consulta diretamente no **Google Calendar**.

## ✨ Principais Funcionalidades

* **Processamento Multi-Modal:** O agente entende não apenas mensagens de **texto**, mas também processa **mensagens de áudio** (transcrevendo-as com a OpenAI) e **imagens** (analisando-as com o GPT-4o-mini).
* **IA Conversacional com Memória:** Utiliza um Agente da OpenAI (GPT) com um *System Prompt* detalhado que define sua persona, regras de negócio (horários de atendimento) e instruções. A memória conversacional é armazenada em um banco **PostgreSQL**, permitindo que o agente se lembre do histórico da conversa com cada cliente.
* **Gerenciamento de Agenda no Google Calendar:** O agente pode:
    * Sugerir horários disponíveis com base nas regras de negócio.
    * Verificar a disponibilidade de um horário específico.
    * Criar novos eventos (agendamentos).
    * Atualizar eventos existentes (reagendamentos).
    * Excluir eventos (cancelamentos).
* **Enfileiramento de Mensagens:** Utiliza o **Redis** para criar uma fila de mensagens. Isso agrupa mensagens rápidas do mesmo usuário antes de processá-las, evitando que a IA seja "bombardeada" e permitindo que ela responda ao contexto completo.
* **Gerenciamento de Clientes:** Integra-se ao **Baserow** (um banco de dados NoSQL) para consultar e cadastrar informações de clientes (Nome, Número, Data, etc.).

## 🛠️ Tecnologias Utilizadas

* **Plataforma de Automação:** [n8n](https://n8n.io/)
* **Inteligência Artificial:** [OpenAI](https://openai.com/) (GPT-4.1-mini, Whisper)
* **Mensageria:** [Evolution API](https://evolution-api.com/) (para WhatsApp)
* **Agenda:** [Google Calendar API](https://developers.google.com/calendar)
* **Banco de Dados (Clientes):** [Baserow](https://baserow.io/)
* **Banco de Dados (Memória da IA):** [PostgreSQL](https://www.postgresql.org/)
* **Fila/Cache:** [Redis](https://redis.io/)

## ⚙️ Como Funciona: Arquitetura dos Workflows

O sistema é dividido em dois workflows principais do n8n para uma arquitetura de "Agente e Ferramentas" (Agent and Tools).

### 1. `BASE_AGENDADOR_GOOGLE_CALENDAR.json` (Workflow Principal do Agente)

Este é o workflow principal que gerencia a interação com o usuário.

1.  **Webhook (Recebe Mensagem):** Aguarda por novas mensagens do WhatsApp (via Evolution API).
2.  **Verificação Inicial:** Checa se a mensagem não é do próprio agente (`Verifica se sou eu`).
3.  **Filtragem de Dados:** Extrai e formata os dados do cliente (nome, número, etc.).
4.  **Banco de Dados (Baserow):** Consulta se o cliente já existe. Se não, cadastra-o.
5.  **Processamento de Tipo de Mensagem (Switch):**
    * **Áudio:** Converte para Base64, salva o arquivo e usa o nó `Transcreve Audio` (OpenAI) para obter o texto.
    * **Imagem:** Converte para Base64, salva o arquivo e usa o nó `Transcreve Imagem` (OpenAI GPT-4o-mini) para descrever o conteúdo.
    * **Texto:** A mensagem é usada diretamente.
6.  **Fila de Mensagens (Redis):**
    * A mensagem processada é adicionada a uma fila no Redis, usando o número do cliente como chave.
    * O sistema aguarda 5 segundos (`Aguarda 5 seg`) para "agrupar" mensagens subsequentes do mesmo usuário.
    * Após a espera, ele busca todas as mensagens da fila, organiza-as em uma única string e limpa a fila.
7.  **Agente de IA (AI Agent):**
    * A mensagem organizada é enviada ao agente.
    * O agente usa o **Postgres Chat Memory** para recuperar o histórico da conversa.
    * O agente utiliza um **Modelo OpenAI (gpt-4.1-mini)**.
    * Ele tem acesso a quatro ferramentas: `agendamento`, `reagendamento`, `cancelamento` e `sugestao`.
8.  **Chamada de Ferramentas:** Quando o agente decide usar uma ferramenta (ex: "agendamento"), ele chama o segundo workflow (`BASE_AGENDADOR_WORKFLOW.json`).
9.  **Envio de Resposta:** A resposta final do agente (seja texto puro ou a resposta da ferramenta) é formatada, dividida em várias mensagens (se necessário) e enviada de volta ao cliente via Evolution API.

### 2. `BASE_AGENDADOR_WORKFLOW.json` (Workflow de Ferramentas)

Este workflow funciona como um "endpoint" de ferramentas para o agente principal. Ele nunca é executado sozinho, apenas quando chamado pelo Agente de IA.

1.  **Trigger (Execute Workflow Trigger):** Recebe a chamada do agente principal, contendo o nome da ferramenta a ser usada (`Evento`) e os dados (ex: `nome`, `email`, `start`, `end`).
2.  **Roteamento de Ferramenta (Switch):** Direciona o fluxo com base no `Evento`:
    * **`agendamento`:**
        1.  Verifica se o cliente já possui um evento marcado.
        2.  Se não, verifica a disponibilidade (`Disponibilidade`) no Google Calendar para o horário solicitado.
        3.  Se estiver disponível, cria o evento (`Marca`) no Google Calendar e retorna uma mensagem de sucesso.
        4.  Se estiver ocupado, retorna uma mensagem de "horário não disponível".
    * **`reagendamento`:**
        1.  Verifica a disponibilidade do *novo* horário.
        2.  Se disponível, localiza o evento antigo (`Verifica evento1`) e o atualiza (`Google Calendar3`) para o novo horário.
        3.  Retorna uma mensagem de sucesso.
    * **`cancelamento`:**
        1.  Localiza o evento do cliente (`Verifica evento`).
        2.  Exclui o evento (`Google Calendar1`).
        3.  Retorna uma mensagem de confirmação de cancelamento.
    * **`sugestao`:**
        1.  Busca todos os eventos futuros (`Get many events1`) no Google Calendar.
        2.  Executa um nó `Code` que calcula todos os horários livres nos próximos 7 dias, com base nas regras de negócio (horários de atendimento da clínica).
        3.  Retorna a lista de horários disponíveis para o agente.

## 🚀 Como Configurar

1.  **Clonar o Repositório:**
    ```bash
    git clone https://github.com/LucasPinottiBrito/scheduling_agent_n8n.git
    ```
2.  **Importar Workflows no n8n:**
    * Faça o upload dos dois arquivos `.json` para sua instância do n8n.
3.  **Configurar Credenciais:**
    Você precisará criar e configurar as seguintes credenciais no n8n:
    * `Baserow account`
    * `Evolution API` (HttpHeaderAuth para a sua instância do Evolution)
    * `OpenAi account`
    * `Redis account`
    * `Postgres account` (para a memória do agente)
    * `Google Calendar Dra Karina` (OAuth2 para a conta do Google Calendar)
4.  **Ativar os Workflows:**
    * Ative o workflow `BASE_AGENDADOR_GOOGLE_CALENDAR`.
    * O workflow `BASE_AGENDADOR_WORKFLOW` **não** precisa ser ativado, pois ele é chamado diretamente pelo primeiro.
5.  **Configurar Webhook:**
    * No workflow `BASE_AGENDADOR_GOOGLE_CALENDAR`, copie a URL de "Test" ou "Production" do nó `Recebe Mensagem`.
    * Configure esta URL na sua instância da Evolution API para receber as mensagens do WhatsApp.