# Desafio Técnico - 4blue: Automação de Matrículas

## 📌 Visão Geral

Este repositório contém a resolução do Desafio Técnico de Automação de Matrículas e Boas-vindas proposto pela 4blue. O objetivo é processar dados de novos alunos via Webhook, realizar o tratamento técnico das informações, armazená-las no banco de dados e enviar um e-mail transacional de confirmação.

## 🎥 Apresentação em Vídeo

[**▶️ Clique aqui para assistir à explicação detalhada do fluxo**](https://drive.google.com/file/d/1-aV0V2rACZuB-SasDhU5ibyE9x4bgJ_G/view?usp=sharing)

## ⚙️ Arquitetura e Decisões Técnicas

A automação foi desenvolvida no n8n utilizando uma arquitetura modular. Em vez de um fluxo monolítico gigante, o sistema é dividido em sub-workflows (`Execute Workflow`), garantindo que o fluxo principal (`Main.json`) atue como um orquestrador limpo e escalável.

### 1. Recebimento e Validação de Dados (DX & Segurança)

* **Webhook (POST):** Ponto de entrada configurado para responder dinamicamente usando o nó `Respond to Webhook`.
* **Validação de Chaves (`ValidateJSONKeys.json`):** Verifica se o payload contém estritamente todos os campos obrigatórios. Se faltar algo, o fluxo é interrompido e retorna um `400 Bad Request` informando exatamente qual chave está ausente.
* **Validação de Tipagem e Formato (`ValidateJSONData.json`):** Utiliza expressões regulares (Regex) em JavaScript puro para garantir que e-mails sejam válidos, datas sigam o calendário real, e telefones obedeçam ao padrão esperado antes de qualquer processamento. Retorna `400 Bad Request` com o detalhe do erro caso o padrão não seja atendido.

### 2. Tratamento e Formatação (`FormatWebhookData.json`)

* **Nome:** Manipulado via JavaScript (`.split().map()`) para garantir a capitalização correta de cada palavra (Ex: "john smith" -> "John Smith").
* **Telefone:** Utilizada Regex (`/[() \-]/g`) com `.replaceAll()` para remover caracteres especiais, mantendo exclusivamente os números.
* **Data de Nascimento:** Convertida do padrão brasileiro (`DD/MM/YYYY`) para o formato ISO (`YYYY-MM-DD`) utilizando a biblioteca embutida Luxon.

### 3. Lógica de Roteamento e Banco de Dados (`InsertDataIntoDB.json`)

O fluxo utiliza o nó `Switch` para direcionar a inserção dos dados com base na variável `curso_escolhido`.

* **Decisão de Infraestrutura:** Conforme as orientações recebidas por e-mail, a inserção dos dados foi configurada utilizando os nós nativos de **Data Table** do n8n (`matriculas_tech` e `matriculas_biz`). Mesmo que o PDF enviado pedia como requisito utilizar nós PostgreSQL não tiveram instruções mais detalhadas de como criar/instanciar o banco de dados, logo foi seguido o vídeo de criação dos Data Table pelos arquivos .csv e persistido os dados, mas, mesmo assim, o workflow está preparado para receber nós PostgreSQL.  Isso garante a aderência às instruções e mantém a operação 100% contida e segura no ambiente da empresa.
  * **Rotas:**
    * "Tech" -> `matriculas_tech`.
    * "Business" -> `matriculas_biz`.

### 4. O Desafio Oculto 🕵️‍♂️

* **O Risco Identificado:** O escopo exigia o direcionamento para tabelas específicas. Contudo, se o Webhook recebesse um payload com um curso não mapeado, a automação falharia silenciosamente ou inseriria dados inconsistentes.
* **A Solução:** Foi implementada uma dupla verificação de segurança. No fluxo principal (`Main.json`), o nó **IsCourseValid?** atua como um *gatekeeper*. Se o curso não for exatamente "Tech" ou "Business", o webhook devolve imediatamente um `400 Bad Request` informando que o curso é inválido, prevenindo a poluição do banco e travando envios incorretos de e-mail. Há também tratamento de erros mapeados internamente nos sub-workflows.

### 5. Comunicação

Após a validação e inserção bem-sucedida, um e-mail transacional é disparado contendo variáveis dinâmicas (*"Olá {{ Nome }}, sua matrícula no curso {{ Curso }} foi recebida com sucesso!"*), e o Webhook devolve um status `200 OK` finalizando o ciclo.

---

## 📂 Arquivos do Repositório

* `Main.json`: Orquestrador principal contendo o Webhook, decisões lógicas HTTP e o envio de e-mail.
* `ValidateJSONKeys.json`: Sub-workflow de verificação de integridade do payload.
* `ValidateJSONData.json`: Sub-workflow de validação de dados via Regex.
* `FormatWebhookData.json`: Sub-workflow de sanitização e tipagem de strings/datas.
* `InsertDataIntoDB.json`: Sub-workflow de inserção nas Data Tables.

---

## 🚀 Como testar localmente

### 1. Configurando o Ambiente

1. Importe os arquivos JSON para o seu n8n. **Atenção:** Importe primeiro os sub-workflows, pegue os IDs gerados pela sua instância e atualize os nós `Execute Workflow` dentro do `Main.json`.
2. Configure as credenciais do Gmail e crie as Data Tables correspondentes.
3. Ative o fluxo `Main`.

### 2. Disparando a Requisição

Você pode testar a automação de duas formas:

**Opção A: Via terminal (cURL)**
Utilize o comando abaixo no seu terminal para disparar uma requisição real para o Webhook:

```bash
curl --request POST \
  --url https://ttc.4blue.com.br/webhook/0187496d-c4b0-4036-8c3a-6ee3f66809b2 \
  --header 'content-type: application/json' \
  --data '{
  "nome_completo": "john smith",
  "email_aluno": "john@4blue.com.br",
  "data_nascimento": "28/02/2020",
  "telefone": "(11) 99999-9999",
  "curso_escolhido": "Business"
}'
```

**Opção B: Via interface do n8n (Mock Data)**
Se preferir rodar manualmente por dentro do n8n para debugar os nós, insira o JSON abaixo na aba de **Pinned Data** do nó Webhook

```json
[
  {
    "body": {
      "nome_completo": "john smith",
      "email_aluno": "john@4blue.com.br",
      "data_nascimento": "28/02/2020",
      "telefone": "(11) 99999-9999",
      "curso_escolhido": "Business"
    }
  }
]
```
