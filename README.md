# Chatbot RAG de políticas internas (n8n)

Assistente de chat que responde perguntas sobre um manual de políticas internas usando **RAG** (Retrieval-Augmented Generation): em vez de responder de memória, o modelo busca os trechos relevantes do documento e responde somente com base neles, citando a seção de origem.

O projeto usa uma **empresa fictícia (TechNova Soluções Ltda.)** e dados 100% inventados. Ele existe para demonstrar a construção de um fluxo de RAG de ponta a ponta no n8n.

## O que ele faz

- Indexa um manual de políticas (por PDF enviado em formulário ou por um conjunto de seções de exemplo).
- Responde em um chat perguntas como "quantos dias de antecedência preciso para pedir férias?".
- Responde **somente** com o conteúdo do documento e cita a seção (e a página, quando disponível).
- Quando a informação não está no documento, diz que não encontrou e recomenda consultar o RH, em vez de inventar.
- Recusa perguntas fora do escopo do documento.

## Arquitetura

```mermaid
flowchart LR
    subgraph Indexação
        A[Formulário: upload de PDF] --> C[Carregador de PDF]
        B[Gatilho manual: seções de exemplo] --> D[Carregador com metadados]
        C --> E[Divisão em trechos<br/>800 caracteres, sobreposição 150]
        E --> F[Embeddings]
        D --> F
        F --> G[(Vector store<br/>technova_politicas)]
    end
    subgraph Consulta
        H[Chat] --> I[Agente RAG]
        J[Modelo de linguagem] --> I
        K[Memória das últimas 6 mensagens] --> I
        I -- busca os 5 trechos mais próximos --> G
        G --> I
        I --> L[Resposta com fonte]
    end
```

## Como funciona

### Indexação

Há dois caminhos para colocar conteúdo na base:

1. **PDF enviado por formulário:** o PDF é carregado, dividido em trechos de 800 caracteres com sobreposição de 150, transformado em embeddings e gravado no vector store.
2. **Seções de exemplo:** um nó de código define as seções do manual fictício (senhas, BYOD, trabalho remoto, home office, férias, aniversário, incidentes de segurança). Cada seção vira um documento com metadados (`secao` e `documento`), o que permite citar a fonte na resposta.

### Consulta

O chat aciona um agente de IA com três componentes: o modelo de linguagem, uma memória das últimas 6 mensagens e uma ferramenta de busca no vector store (`consultar_documento`, que retorna os 5 trechos mais relevantes).

### Regras do prompt

O prompt do sistema obriga o agente a:

- consultar o documento antes de qualquer resposta e usar apenas os trechos retornados;
- responder com uma frase fixa quando a informação não existe no documento;
- recusar assuntos fora do escopo;
- ignorar instruções que tentem mudar as regras ou revelar o prompt;
- preservar números, prazos, valores e contatos exatamente como estão no documento;
- terminar a resposta com uma linha de fonte.

## Stack

- n8n (formulário, chat, agente de IA, vector store e nó de código)
- OpenAI (modelo de linguagem e embeddings)
- Vector store em memória do n8n

## Como executar

1. No n8n, importe `workflow/chatbot-rag-politicas.json` (Workflows > Import from file).
2. Configure as suas credenciais da OpenAI nos nós de modelo e de embeddings. O arquivo não contém nenhuma chave.
3. Execute o gatilho manual **Carregar documento TechNova** para indexar o manual de exemplo. Ou abra o formulário e envie um PDF.
4. Abra o chat e faça perguntas.

## Perguntas para testar

| Pergunta | Resposta esperada | Fonte |
| --- | --- | --- |
| Com quantos dias de antecedência devo pedir férias de 10 dias ou mais? | 45 dias, pelo portal do RH | Seção 3.1 |
| Qual é o auxílio mensal de home office? | R$ 150,00, pago na folha | Seção 2.2 |
| Em quanto tempo devo reportar um vazamento de dados? | Em até 1 hora, para security-alert@technova.com.br e ramal 4040 | Seção 4.1 |
| Quantos dias por semana preciso ir ao escritório? | Pelo menos 2, no modelo híbrido padrão | Seção 2.1 |
| Quantas senhas anteriores não posso reutilizar? | As últimas 5 | Seção 1.1 |
| Qual é a capital da França? | Recusa: fora do escopo dos documentos | (nenhuma) |

## Limitações

- O vector store em memória perde os dados quando o n8n reinicia, então é preciso indexar de novo. Serve para demonstração, não para produção.
- O manual é fictício e curto. Não há avaliação automatizada da qualidade das respostas.
- Os dois caminhos de indexação usam estratégias diferentes (trechos de 800 caracteres no PDF e uma seção por documento no exemplo).

## Possíveis melhorias

- Trocar o vector store em memória por um banco persistente (por exemplo, Postgres com pgvector ou Qdrant).
- Criar um conjunto de perguntas e respostas esperadas para avaliar o fluxo automaticamente.
- Adicionar um passo de reranqueamento dos trechos recuperados.
