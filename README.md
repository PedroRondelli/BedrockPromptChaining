# BedrockPromptChaining
Este repositório contém o template de uma máquina de estados do AWS Step Functions que implementa o padrão de arquitetura Prompt Chaining usando o Amazon Bedrock.
# AWS Step Functions & Bedrock: Prompt Chaining Pattern

Este repositório contém o template de uma máquina de estados do **AWS Step Functions** que implementa o padrão de arquitetura **Prompt Chaining** usando o **Amazon Bedrock**.

---

## 📌 1. Resumo Geral

Este workflow orquestra a execução sequencial de uma lista de prompts enviada no payload de entrada. Ele garante que a resposta gerada pelo modelo de IA em cada etapa seja anexada como contexto para a próxima chamada, criando uma cadeia de refinamento contínuo.

> **Objetivo Macro:** Encadeia uma lista sequencial de prompts no Amazon Bedrock (modelo Amazon Nova Lite), acumulando e reutilizando as respostas anteriores como contexto dinâmico para a geração do passo seguinte.

---

## 🛠️ 2. Ferramentas e Tecnologias AWS

- **AWS Step Functions**: Servidor de orquestração responsável por gerenciar o ciclo de vida do workflow, controle de estado, iterações e escopo de variáveis.
- **Amazon Bedrock**: Serviço totalmente gerenciado de IA generativa. Utiliza o modelo `amazon.nova-lite-v1:0` para inferência de texto.
- **JSONata (`QueryLanguage`)**: Linguagem declarativa de consulta e transformação ativada no workflow (`"QueryLanguage": "JSONata"`). Ela substitui o legado JSONPath e permite manipulação de listas, concatenação de strings e operações matemáticas diretamente nos estados.
- **Optimized Integrations**: Integração nativa entre o Step Functions e as APIs do Bedrock (`states:::bedrock:invokeModel`), sem necessidade de intermediar com funções AWS Lambda.

---

## 🔍 3. Explicação dos Principais Trechos do Código

### 3.1. Estado `Initialize` (Pass)
Prepara e inicializa as variáveis de escopo local (bloco `Assign`).

```json
"Initialize": {
  "Type": "Pass",
  "Next": "Choice",
  "Assign": {
    "counter": "{% $count($states.input.prompts) %}",
    "conversation_history": [ "" ],
    "input_prompts": "{% $reverse($states.input.prompts) %}"
  }
}
```

* **`$states.input.prompts`**: Acessa o array de strings passado na requisição de entrada.
* **`counter`**: Conta o total de prompts enviados usando a função `$count()`.
* **`conversation_history`**: Inicializa a lista de histórico com um elemento vazio (`[""]`).
* **`input_prompts`**: Inverte a ordem do array original via `$reverse()`. Como o loop processa decrementando o contador (`counter - 1`), inverter a lista garante que os prompts sejam executados na ordem correta enviada pelo usuário.

### 3.2. Estado Choice (Controle de Loop)
Atua como a condição de parada (while loop).
```json
"Choice": {
  "Type": "Choice",
  "Choices": [
    {
      "Next": "Invoke model with prompt",
      "Condition": "{% $counter > 0 %}"
    }
  ],
  "Default": "Success"
}
```
* Se **`$counter > 0`**: Avança para chamar o Bedrock.
* Quando **`$counter == 0`**: O fluxo finaliza e é redirecionado para o estado Success.

### 3.3. Estado Invoke model with prompt (Task)
Relação com a API do Bedrock, montagem do prompt com histórico e atualização das variáveis.
```json
"text": "{% $conversation_history[-1] & '.' & $input_prompts[$counter - 1] %}"
```
* **`$conversation_history[-1]`**: Recupera o último elemento do array de histórico (a resposta imediatamente anterior do Bedrock).

* **`&`**: Operador JSONata de concatenação de strings.

* **`$input_prompts[$counter - 1]`**: Busca o prompt atual no array de entrada com base no contador.
```json
"Assign": {
  "conversation_history": "{% $append($conversation_history,$states.result.Body.output.message.content[0].text) %}",
  "counter": "{% $counter - 1 %}"
}
```
* **`$states.result`**: Lê o retorno da execução da chamada atual do Bedrock.
* **`$append()`**: Adiciona a nova resposta gerada ao final do histórico $conversation_history.
* **`$counter - 1`**: Decrementa o contador para avançar a iteração.

### 3.4. Estado Success (Succeed)
Consolidação do resultado final da máquina de estados.
```json
"Success": {
  "Type": "Succeed",
  "Output": "{% $join($conversation_history[[1..$count($conversation_history)]], '.') %}"
}
```
* **`$conversation_history[[1..$count(...)]]`**: Aplica um fatiamento (slice) para remover o índice 0 (a string vazia criada no Initialize).
* **`$join(..., '.')`**: Une todas as respostas geradas em uma única string final.
---
## 4. Usos Práticos
Geração Multietapa de Conteúdo: Criação sequencial de tópicos para artigos, postagens ou documentos onde o tom do segundo capítulo depende do primeiro.

Pipelines de Refinamento de Código:

Passo 1: Gerar uma função em Python a partir de um requisito.

Passo 2: Gerar os testes unitários em PyTest com base no código gerado no Passo 1.

Passo 3: Gerar a documentação (Docstrings/README) analisando os dois passos anteriores.

Análise Sequencial de Documentos (Chain-of-Thought): Dividir a interpretação de contratos longos em: extração de dados → identificação de cláusulas de risco → geração de recomendações.
