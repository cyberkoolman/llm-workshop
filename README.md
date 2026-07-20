# LLM Workshop

Hands-on workshop for learning Large Language Model (LLM) fundamentals using Azure OpenAI. Covers chat completions, embeddings, semantic search, rate limiting, and prompt caching.

## Prerequisites

- Python 3.14+
- [uv](https://docs.astral.sh/uv/) package manager
- Azure OpenAI resource with deployed models (e.g., `gpt-4.1`, embedding model)
- [VS Code REST Client extension](https://marketplace.visualstudio.com/items?itemName=humao.rest-client) (for `.http` files)

## Setup

1. **Clone the repository**

   ```bash
   git clone https://github.com/cyberkoolman/llm-workshop.git
   cd llm-workshop
   ```

2. **Install dependencies**

   ```bash
   uv sync
   ```

3. **Configure environment variables**

   Create a `.env` file in the project root (see `.env` example below):

   ```env
   OpenAiEndPoint=https://<your-resource>.openai.azure.com/
   OpenAiVersion=2024-12-01-preview
   OpenAiKey=<your-key>
   OpenAiEmbedding=<embedding-deployment-name>
   SearchService=<your-search-service>
   SearchIndex=<your-search-index>
   ```

## Workshop Contents

### 1. Chat Completions & Rate Limiting (`chatcompletion.http`)

Interactive HTTP requests demonstrating:
- Basic chat completion calls
- Token usage observation via response headers
- Triggering 429 rate limits with large/multiple completions
- **Prompt caching** — send the same large system prompt twice and observe `cached_tokens` in the response

### 2. Embeddings & Semantic Search (`Embeddings.ipynb`)

Jupyter notebook covering:
- Generating word embeddings with Azure OpenAI
- Computing cosine similarity between vectors
- Building a semantic search over words, sentences, and earnings call transcripts
- Using CSV datasets (`Data/CSV/`)

### 3. Azure AI Search (`azuresearch.http`)

REST requests to query an Azure AI Search index — useful for verifying indexed documents.

### 4. Request Parameters Reference (`llm_request.md`)

Detailed reference for all Azure OpenAI chat completion request parameters including `temperature`, `top_p`, `max_tokens`, `seed`, `response_format`, and more.

## Project Structure

```
├── chatcompletion.http      # REST Client demos (chat, rate limiting, caching)
├── azuresearch.http         # Azure AI Search queries
├── Embeddings.ipynb         # Jupyter notebook — embeddings & semantic search
├── Data/CSV/                # Sample datasets (words, earnings transcripts)
├── Utilities/               # Helper module for loading env vars
├── llm_request.md           # Chat completion parameters reference
├── llm_response.md          # Chat completion response structure reference
├── pyproject.toml           # Project config & dependencies (uv)
└── .env                     # Your local secrets (git-ignored)
```

## Running the Notebook

```bash
uv run jupyter notebook Embeddings.ipynb
```

## Authentication

The `.http` files use **Entra ID (Azure AD) bearer tokens**. Generate one with:

```bash
az account get-access-token --resource https://cognitiveservices.azure.com --query accessToken -o tsv
```

Paste the token into the `@token` variable at the top of each `.http` file.

## License

This project is for educational/workshop purposes.
