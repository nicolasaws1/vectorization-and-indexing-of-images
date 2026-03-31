# Fase 3 — Vetorização e Indexação de Descrições de Imagens

## Visão Geral

Este script é a **terceira e última etapa** do pipeline multimodal do projeto Agrônomo Virtual (SB100). Ele lê as descrições de figuras geradas pelo Gemini na Fase 2, converte cada descrição em vetores densos e esparsos utilizando os mesmos modelos de embedding do pipeline de texto, e faz o upsert no banco vetorial Qdrant.

O resultado é que figuras, gráficos e fotos de artigos científicos passam a ser recuperáveis por busca semântica, lado a lado com os chunks de texto — tudo no mesmo espaço vetorial.

## Contexto no Pipeline

| Etapa | O que faz | Consome API externa | Carrega modelos ML |
|-------|-----------|--------------------|--------------------|
| **Código 1** — Extração | PDFs → texto + chunks + imagens PNG + JSON | Não | Sim (Qwen3, OpenSearch) |
| **Código 2** — Gemini | Imagens → descrições textuais no JSON | Sim (Gemini) | Não |
| **Código 3** — Vetorização *(este)* | Descrições → vetores → Qdrant | Não | Sim (Qwen3, OpenSearch) |

A separação garante que este script nunca gasta créditos de API externa. Ele apenas lê os JSONs que já contêm descrições e converte texto em vetores localmente.

## Como Funciona

### Entrada

O script lê os arquivos `*-imagens-meta.json` em `data/Imagens/`. Ele processa apenas páginas que atendam a duas condições simultaneamente: `"gemini_processado": true` (a Fase 2 já gerou descrições) e `"qdrant_vetorizado": false` ou ausente (ainda não foi indexada).

Exemplo de entrada (JSON após a Fase 2):

```json
{
  "arquivo_pdf": "Silva2020-adubacao-citros.pdf",
  "titulo": "Adubação nitrogenada em citros no estado de São Paulo",
  "autores": "Silva, J. A.; Souza, M. R.",
  "cultura": "Citricultura",
  "paginas": [
    {
      "arquivo_imagem": "Silva2020-adubacao-citros-pagina-005.png",
      "pagina_pdf": 5,
      "contexto_textual": "As doses de nitrogênio aplicadas variaram de 0 a 200 kg N ha-1...",
      "gemini_processado": true,
      "figuras": [
        {
          "tipo": "grafico_linhas",
          "descricao": "Gráfico de linhas mostrando a resposta da produtividade...",
          "variaveis": ["Dose de N (kg ha-1)", "Produtividade (t ha-1)"],
          "unidades": ["kg ha-1", "t ha-1"],
          "conclusao_principal": "Produtividade máxima atingida em 120 kg N ha-1."
        }
      ]
    }
  ]
}
```

### Composição do Texto para Vetorização

Para cada figura descrita pelo Gemini, o script monta um texto que combina todas as informações semânticas relevantes:

```
[IMAGEM - grafico_linhas] Gráfico de linhas mostrando a resposta da produtividade
de frutos de laranjeira Valência à adubação nitrogenada ao longo de 5 safras.
Variáveis: Dose de N (kg ha-1), Produtividade (t ha-1).
Unidades: kg ha-1, t ha-1.
Produtividade máxima atingida em 120 kg N ha-1.
```

O prefixo `[IMAGEM - tipo]` permite que o sistema de RAG downstream identifique que o chunk é proveniente de uma figura. As variáveis, unidades e conclusão são incluídas para maximizar a relevância semântica na busca vetorial.

### Vetorização Híbrida

Cada texto é vetorizado por dois modelos, os mesmos utilizados pelo Código 1 para chunks de texto:

- **Vetor denso** — `Qwen/Qwen3-Embedding-0.6B` (1024 dimensões, distância Cosine). Captura semântica geral do conteúdo.
- **Vetor esparso** — `opensearch-project/opensearch-neural-sparse-encoding-doc-v2-distill`. Captura termos-chave exatos (nomes de variáveis, unidades, culturas).

Isso garante que chunks de imagem e chunks de texto estejam no mesmo espaço vetorial. Quando o Squad 4 faz uma query como "gráfico de produtividade de citros versus dose de nitrogênio", a busca híbrida recupera tanto chunks de texto que mencionam esses dados quanto as descrições de figuras que mostram exatamente esses gráficos.

### Payload no Qdrant

Cada ponto indexado no Qdrant carrega um payload completo:

```json
{
  "content": "[IMAGEM - grafico_linhas] Gráfico de linhas mostrando...",
  "tipo": "imagem",
  "tipo_imagem": "grafico_linhas",
  "descricao_gemini": "Gráfico de linhas mostrando...",
  "variaveis": ["Dose de N (kg ha-1)", "Produtividade (t ha-1)"],
  "unidades": ["kg ha-1", "t ha-1"],
  "conclusao": "Produtividade máxima atingida em 120 kg N ha-1.",

  "titulo": "Adubação nitrogenada em citros no estado de São Paulo",
  "autores": "Silva, J. A.; Souza, M. R.",
  "ano": "2020",
  "doi": "10.1234/example",
  "periodico": "Revista Brasileira de Citricultura",
  "cultura": "Citricultura",
  "palavras_chave": "nitrogênio, citros, adubação, produtividade",

  "file": "Silva2020-adubacao-citros.pdf",
  "pagina_pdf": 5,
  "arquivo_imagem": "Silva2020-adubacao-citros-pagina-005.png",
  "contexto_textual": "As doses de nitrogênio aplicadas variaram...",
  "ingestao": "2026-03-31T15:30:00.123456"
}
```

O campo `"tipo": "imagem"` distingue esses pontos dos chunks de texto (`"tipo": "texto"` ou `"tipo": "tabela"`), permitindo filtragem no Qdrant se necessário.

### Saída

O JSON é atualizado in-place com os campos de controle da vetorização:

```json
{
  "arquivo_imagem": "Silva2020-adubacao-citros-pagina-005.png",
  "gemini_processado": true,
  "qdrant_vetorizado": true,
  "qdrant_data": "2026-03-31T15:30:00.123456",
  "figuras": [...]
}
```

## Controle de Duplicatas

Dois campos no JSON garantem que nenhuma etapa é repetida:

| Campo | Quem marca | Significado |
|-------|-----------|-------------|
| `gemini_processado` | Código 2 | Gemini já descreveu esta página |
| `qdrant_vetorizado` | Código 3 | Descrições já foram vetorizadas e indexadas |

O script classifica cada página em uma de quatro categorias:

- `gemini_processado: false` → **aguardando Código 2** (contabilizada mas não processada)
- `gemini_processado: true` + sem figuras → **nada a vetorizar** (marcada como feita automaticamente)
- `gemini_processado: true` + figuras + `qdrant_vetorizado: false` → **pendente** (será processada)
- `qdrant_vetorizado: true` → **já feita** (ignorada)

Se a execução for interrompida, o campo `qdrant_vetorizado` só é marcado como `true` quando todas as figuras de uma página são indexadas com sucesso. Em caso de erro parcial, a página inteira será reprocessada na próxima execução.

## Configuração

```python
QDRANT_URL       = "https://...sa-east-1-0.aws.cloud.qdrant.io:6333"
QDRANT_API_KEY   = "..."
COLLECTION_NAME  = "sb100"
```

A comunicação com o Qdrant é feita via REST direto (biblioteca `requests`), evitando o problema de IPv6 do `qdrant-client` no Google Colab.

## Dependências

```bash
pip install sentence-transformers transformers torch requests
```

Os modelos são baixados automaticamente do Hugging Face na primeira execução (~1.5 GB total).

## Execução

```
📁 Buscando JSONs em .../data/Imagens...
📄 45 JSONs encontrados.
📊 Qdrant atual: 2847 pontos

============================================================
📄 Silva2020-adubacao-citros.pdf
  2 páginas para vetorizar
============================================================
  📄 p.5 — 2 figura(s)
    [1] grafico_linhas → 🧠 🔍(847) → 📤 completed
    [2] grafico_barras → 🧠 🔍(912) → 📤 completed
  📄 p.8 — 1 figura(s)
    [1] foto → 🧠 🔍(634) → 📤 completed
  💾 JSON atualizado: Silva2020-adubacao-citros-imagens-meta.json

🎉 Código 3 concluído!
  📤 Figuras vetorizadas agora: 3
  ⏭️  Já vetorizadas antes: 87
  ⏸️  Aguardando Gemini (código 2): 12
  ❌ Erros: 0
  📊 Total no Qdrant: 2850 pontos
```
