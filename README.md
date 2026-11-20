# 🔧 Automação de Oficina Mecânica (Telegram + IA + Google Sheets)

Fluxo desenvolvido no **n8n** para automatizar o registro de serviços e trocas de óleo/filtros em oficinas mecânicas por meio de mensagens de voz enviadas no **Telegram**.

---

## 🎯 Por que essa automação é importante no dia a dia?

No ambiente dinâmico de uma oficina mecânica, mecânicos e atendentes frequentemente estão com as mãos ocupadas ou sujas, tornando lento e impreciso parar para digitar ordens de serviço, preencher planilhas ou anotar peças utilizadas no papel.

Com este fluxo:
* **Agilidade no atendimento:** Basta gravar um áudio rápido de poucos segundos no Telegram relatando o serviço realizado.
* **Padronização automática:** A Inteligência Artificial interpreta a fala informal, extrai os dados técnicos, separa quantidades numéricas e identifica se a peça veio do **estoque** ou de **autopeças**.
* **Zero retrabalho administrativo:** Os dados são lançados diretamente no **Google Sheets** com a data atual e um resumo organizado em arquivo `.txt` é devolvido no próprio chat para conferência ou impressão.
* **Redução de erros e esquecimentos:** Evita perda de anotações em papel e garante rastreabilidade por placa, modelo e quilometragem.

---

## ⚙️ Como o fluxo funciona (Passo a Passo)

1. **Recebimento e Validação (`Telegram Trigger` + `Switch`):**
   * O bot recebe mensagens no Telegram e verifica se o conteúdo é um **áudio (`voice`)**.
   * Caso o usuário envie texto, foto ou outro formato, o bot responde orientando a enviar um áudio e encerra a execução.
2. **Download e Preparação do Áudio:**
   * Faz o download do arquivo `.ogg` do Telegram, confirma o recebimento ao usuário e prepara o binário para transcrição.
3. **Transcrição de Voz para Texto (`Groq - Whisper`):**
   * Envia o áudio para a API da Groq utilizando o modelo **`whisper-large-v3-turbo`**, convertendo a fala do mecânico em texto.
4. **Extração Estruturada de Dados (`Groq - LLM`):**
   * Um agente de IA (`openai/gpt-oss-120b`) analisa a transcrição e extrai um objeto JSON estrito com os seguintes campos:
     * `placa`, `modelo`, `quilometragem`
     * `oleo`, `quantidade_oleo`
     * `filtro_de_oleo`, `filtro_de_combustivel`, `filtro_de_cabine`, `filtro_de_ar`
     * `arruela` (`sim` ou `nao`)
     * `itens_extras` e `data`
   * *Regra de origem:* Se o mecânico mencionar a origem do insumo (ex.: *"óleo 5W30 do estoque"*), a IA formata automaticamente como `5W30 - estoque` ou `ITEM - autopecas`.
5. **Registro no Google Sheets (`Insere informacoes na planilha`):**
   * Adiciona uma nova linha na aba `ITENS_TROCAS` da planilha `Planilha_trocas` com todos os campos mapeados.
6. **Confirmação e Geração de Comprovante (`.txt`):**
   * Envia uma mensagem de confirmação no Telegram com o link direto para a planilha.
   * Formata os campos preenchidos (ignorando campos nulos), gera um arquivo `servicos_<MODELO>_<PLACA>.txt` e o envia como documento no chat.
7. **Tratamento de Erros:**
   * Cada etapa crítica (download, transcrição, estruturação JSON, gravação na planilha e geração do `.txt`) possui saídas de erro dedicadas que notificam o usuário no Telegram e interrompem o fluxo com logs claros (`Stop and Error`).

---

## 🛠️ Tecnologias e Integrações Utilizadas

* **Orquestração:** [n8n](https://n8n.io/)
* **Interface de Entrada/Saída:** Telegram Bot API
* **Transcrição de Áudio (STT):** Groq API (`whisper-large-v3-turbo`)
* **Processamento de Linguagem Natural (LLM):** Groq API (`openai/gpt-oss-120b`)
* **Banco de Dados / Planilha:** Google Sheets API

---

## 🚀 Como Configurar e Importar

1. **Importe o Workflow:**
   * No n8n, crie um novo workflow, clique no menu superior direito (`...`) > **Import from File** e selecione o arquivo `automacao-oficina.json`.
2. **Configure as Credenciais:**
   * **Telegram API:** Crie uma credencial com o token gerado pelo `@BotFather` e vincule a todos os nós do Telegram.
   * **Groq API:** Configure sua API Key na credencial `groqApi` do n8n. *(Certifique-se de remover qualquer header `Authorization` manual nos nós HTTP Request).*
   * **Google Sheets OAuth2:** Conecte sua conta Google e selecione a planilha e a aba de destino (`ITENS_TROCAS`) com as colunas correspondentes:
     `PLACA | MODELO | QUILOMETRAGEM | ÓLEO | QUANTIDADE ÓLEO | FILTRO DE ÓLEO | FILTRO DE COMBUSTÍVEL | FILTRO DE CABINE | FILTRO DE AR | ARRUELA | ITENS EXTRAS | DATA`
3. **Ative o Fluxo:**
   * Alterne o botão do workflow para **Active** e envie um áudio de teste para o seu bot no Telegram.

---

## 💬 Exemplo de Uso (Áudio Enviado pelo Mecânico)

> *"Troca de óleo do Onix placa ABC1D23, está com 85 mil km. Foi usado 4 litros de óleo 5W30 do estoque, filtro de óleo OC 1449 de autopeças, filtro de ar do estoque, trocou arruela e completou água do limpador."*

**Resultado salvo na planilha e no `.txt`:**
* **Placa:** ABC1D23
* **Modelo:** Onix
* **Quilometragem:** 85000
* **Óleo:** 5W30 - estoque
* **Quantidade de óleo:** 4
* **Filtro de óleo:** OC 1449 - autopecas
* **Filtro de ar:** estoque
* **Arruela:** sim
* **Itens Extras:** água do limpador