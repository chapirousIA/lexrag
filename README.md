# LexRAG - Sistema RAG Jurídico Local

Sistema de Retrieval-Augmented Generation (RAG) para documentos jurídicos, integrado ao Claude Code via MCP (Model Context Protocol).

## 🎯 Objetivo

Sistema local para busca semântica em documentos tributários (petições, pareceres, jurisprudência, legislação) sem envio de dados sensíveis para servidores externos.

## 📋 Status do Projeto

- [x] Estrutura do projeto
- [x] Documentação (CLAUDE.md, PRD)
- [ ] Setup básico (Fase 1)
- [ ] Extração jurídica (Fase 2)
- [ ] MCP Server (Fase 3)
- [ ] CLI completa (Fase 4)
- [ ] Testes e validação (Fase 5)

## 🚀 Setup

### Pré-requisitos

- Python 3.11+
- pip ou poetry

### Instalação

```bash
# Clone o repositório
git clone https://github.com/seu-usuario/lexrag.git
cd lexrag

# Instale as dependências
pip install -e .

# Ou com poetry
poetry install
```

### Configuração

```bash
# Configure o diretório de dados
export LEXRAG_DATA_DIR="/caminho/para/lexrag/data"

# Baixe o modelo de embeddings (automático na primeira execução)
python -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('all-MiniLM-L6-v2')"
```

## 📖 Documentação

- [CLAUDE.md](CLAUDE.md) - Documentação completa para Claude Code
- [prd-rag-juridico-local.md](prd-rag-juridico-local.md) - Product Requirements Document

## 🔧 Como Conectar ao GitHub

### Opção 1: Via Web UI

1. Crie um repositório no GitHub: https://github.com/new
   - Nome: `lexrag` (ou outro de sua preferência)
   - Descrição: Sistema RAG Jurídico Local
   - Não inicialize com README (já temos um)
   - Marque como Private se desejar

2. Após criar, o GitHub mostrará comandos para conectar. Use:

```bash
# Adicione o remote
git remote add origin https://github.com/SEU_USUARIO/lexrag.git

# Renomeie branch para main (opcional, mas recomendado)
git branch -M main

# Push para o GitHub
git push -u origin main
```

### Opção 2: Instalar GitHub CLI

```bash
# Windows (via winget)
winget install GitHub.cli

# macOS
brew install gh

# Linux
# Verifique https://github.com/cli/cli/blob/trunk/docs/install_linux.md
```

Depois faça login:

```bash
gh auth login
```

E crie o repositório:

```bash
gh repo create lexrag --public --source=. --remote=origin --push
```

## 📝 Próximos Passos

1. Conectar este repositório ao GitHub
2. Continuar implementação seguindo o PRD
3. Implementar Fase 1: Setup Básico

## 👥 Stakeholder

**Pedrosa & Peixoto Advogados**
- Área: Direito Tributário Federal
- Foco: Transação Tributária (Lei 13.988/2020)

## 📄 Licença

Copyright © 2026 Pedrosa & Peixoto Advogados. Todos os direitos reservados.

---

**Desenvolvido com Claude Code + Ralph Wiggum Plugin** 🤖
