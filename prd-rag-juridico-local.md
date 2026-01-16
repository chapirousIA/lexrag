# PRD — RAG Jurídico Local para Claude Code

**Projeto:** LexRAG - Sistema de Retrieval-Augmented Generation para Documentos Jurídicos  
**Versão:** 1.0  
**Data:** 15/01/2026  
**Stakeholder:** Pedrosa & Peixoto Advogados

---

## Resumo Executivo

Este PRD especifica a criação de um sistema RAG (Retrieval-Augmented Generation) local para fins jurídicos, integrado ao Claude Code via MCP (Model Context Protocol). O sistema permitirá consultas semânticas em documentos tributários (petições, pareceres, jurisprudência, legislação) armazenados localmente, sem envio de dados sensíveis para servidores externos.

**Resultado esperado:** Claude Code poderá responder perguntas sobre o acervo documental do escritório, citando fontes específicas com número de processo, tribunal e tese jurídica.

**Stack recomendada:** Python + ChromaDB + MCP Server + Claude Code

**Investimento estimado:** 8-16 horas de desenvolvimento inicial + manutenção contínua.

---

## 1. Contexto e Problema

### 1.1 Situação Atual

| Aspecto | Descrição |
|---------|-----------|
| Acervo | Petições, pareceres, jurisprudência em PDF/DOCX dispersos em pastas locais |
| Busca atual | Ctrl+F em arquivos individuais ou busca por nome de arquivo |
| Limitações | Sem busca semântica; contexto perdido entre documentos; retrabalho em pesquisas recorrentes |

### 1.2 Necessidades

1. **Busca semântica:** encontrar documentos por conceito jurídico, não apenas por palavra exata
2. **Citação precisa:** retornar número do processo, tribunal, data, tese com link para arquivo original
3. **Privacidade:** dados sensíveis (nomes de clientes, valores) não podem sair do ambiente local
4. **Integração Claude Code:** consultar o acervo diretamente pelo terminal/IDE durante a elaboração de peças

### 1.3 Escopo

**Incluído:**
- Indexação de documentos PDF, DOCX, TXT, MD
- Busca semântica via embeddings locais
- Servidor MCP para integração com Claude Code
- Metadados jurídicos estruturados
- Interface CLI para consultas e manutenção

**Excluído (v1.0):**
- Interface web/GUI
- OCR de documentos escaneados (requer pré-processamento separado)
- Embeddings via API externa (OpenAI, etc.)
- Multi-tenancy (apenas uso interno)

---

## 2. Requisitos Funcionais

### 2.1 Indexação de Documentos

| ID | Requisito | Prioridade |
|----|-----------|------------|
| RF01 | Indexar arquivos PDF com extração de texto | Alta |
| RF02 | Indexar arquivos DOCX preservando estrutura | Alta |
| RF03 | Indexar arquivos TXT e MD | Média |
| RF04 | Chunking inteligente respeitando parágrafos e seções | Alta |
| RF05 | Extração automática de metadados via regex (nº processo, tribunal) | Alta |
| RF06 | Metadados manuais via arquivo YAML companion | Média |
| RF07 | Re-indexação incremental (apenas arquivos modificados) | Média |
| RF08 | Exclusão de arquivos do índice sem remover fonte | Baixa |

### 2.2 Busca e Recuperação

| ID | Requisito | Prioridade |
|----|-----------|------------|
| RF09 | Busca semântica por similaridade vetorial | Alta |
| RF10 | Busca híbrida (semântica + keyword) | Média |
| RF11 | Filtros por tipo de documento, tribunal, data, tags | Alta |
| RF12 | Retornar chunks rankeados com score de relevância | Alta |
| RF13 | Retornar caminho do arquivo fonte para verificação | Alta |
| RF14 | Limitar resultados por quantidade (top_k) | Alta |

### 2.3 Integração Claude Code (MCP)

| ID | Requisito | Prioridade |
|----|-----------|------------|
| RF15 | Servidor MCP expondo tool `search_documents(query, filters)` | Alta |
| RF16 | Servidor MCP expondo tool `get_document_chunk(chunk_id)` | Alta |
| RF17 | Servidor MCP expondo tool `list_sources()` | Média |
| RF18 | Configuração via `~/.claude.json` ou `.mcp.json` local | Alta |
| RF19 | Logs de queries para auditoria | Baixa |

### 2.4 Manutenção

| ID | Requisito | Prioridade |
|----|-----------|------------|
| RF20 | CLI: `lexrag index <pasta>` para indexar documentos | Alta |
| RF21 | CLI: `lexrag search "<query>"` para testes | Alta |
| RF22 | CLI: `lexrag status` para estatísticas do índice | Média |
| RF23 | CLI: `lexrag serve` para iniciar servidor MCP | Alta |

---

## 3. Requisitos Não-Funcionais

| ID | Categoria | Requisito | Métrica |
|----|-----------|-----------|---------|
| RNF01 | Performance | Busca em < 2s para acervo de até 10.000 chunks | Latência p95 |
| RNF02 | Performance | Indexação de 1 PDF (50 páginas) em < 30s | Tempo médio |
| RNF03 | Escalabilidade | Suportar até 50.000 chunks sem degradação | Limite testado |
| RNF04 | Disponibilidade | Servidor MCP deve iniciar em < 5s | Tempo de boot |
| RNF05 | Segurança | Zero dados enviados para servidores externos | Auditável |
| RNF06 | Segurança | Embeddings gerados 100% localmente | Verificável |
| RNF07 | Manutenibilidade | Código documentado com docstrings | Cobertura 100% |
| RNF08 | Portabilidade | Funcionar em macOS, Linux, Windows (WSL) | Testado |

---

## 4. Arquitetura Técnica

### 4.1 Diagrama de Componentes

```
┌─────────────────────────────────────────────────────────────────┐
│                        Claude Code (IDE/Terminal)                │
└─────────────────────────────┬───────────────────────────────────┘
                              │ MCP Protocol (stdio/SSE)
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    MCP Server (lexrag serve)                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │ search_docs │  │ get_chunk   │  │ list_sources            │  │
│  └──────┬──────┘  └──────┬──────┘  └───────────┬─────────────┘  │
└─────────┼────────────────┼─────────────────────┼────────────────┘
          │                │                     │
          ▼                ▼                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Core RAG Engine (Python)                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │ Retriever   │  │ Embedder    │  │ DocumentLoader          │  │
│  │ (ChromaDB)  │  │ (Local LLM) │  │ (PDF/DOCX/TXT)          │  │
│  └──────┬──────┘  └──────┬──────┘  └───────────┬─────────────┘  │
└─────────┼────────────────┼─────────────────────┼────────────────┘
          │                │                     │
          ▼                ▼                     ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────────────┐
│   ChromaDB      │ │  Modelo Local   │ │    Sistema de Arquivos  │
│   (SQLite)      │ │  (Ollama/etc)   │ │    ~/lexrag/sources/    │
└─────────────────┘ └─────────────────┘ └─────────────────────────┘
```

### 4.2 Stack Tecnológica

| Camada | Tecnologia | Justificativa |
|--------|------------|---------------|
| Linguagem | Python 3.11+ | Ecossistema NLP maduro; facilidade de manutenção |
| Vector DB | ChromaDB | Leve, local, sem servidor separado, persistência SQLite |
| Embeddings | `sentence-transformers/all-MiniLM-L6-v2` ou `nomic-embed-text` via Ollama | Trade-off qualidade/velocidade; 100% local |
| Extração PDF | `pymupdf` (fitz) | Mais rápido que PyPDF2; preserva layout |
| Extração DOCX | `python-docx` + `mammoth` | Estrutura + texto limpo |
| Chunking | `langchain.text_splitter.RecursiveCharacterTextSplitter` | Configurável; respeita estrutura |
| MCP Server | `mcp` (SDK oficial Anthropic) | Integração nativa Claude Code |
| CLI | `typer` | Experiência moderna; autocompleção |

### 4.3 Modelo de Dados

#### 4.3.1 Estrutura de Metadados Jurídicos

```python
@dataclass
class DocumentMetadata:
    # Identificação
    file_path: str              # Caminho absoluto do arquivo fonte
    file_name: str              # Nome do arquivo
    file_hash: str              # SHA256 para detectar alterações
    
    # Classificação jurídica
    doc_type: str               # "peticao", "acordao", "parecer", "legislacao", "doutrina"
    area: str                   # "tributario", "civil", "administrativo"
    
    # Processo (quando aplicável)
    numero_processo: str | None # Ex: "0000000-00.0000.0.00.0000"
    tribunal: str | None        # "TRF5", "STJ", "CARF", etc.
    vara_turma: str | None      # "1ª Turma", "2ª Vara Federal"
    
    # Conteúdo
    tese_principal: str | None  # Resumo da tese em até 200 caracteres
    tags: list[str]             # ["transacao_tributaria", "pgfn", "desconto"]
    
    # Temporal
    data_documento: date | None # Data do documento/decisão
    data_indexacao: datetime    # Quando foi indexado
    
    # Técnico
    chunk_count: int            # Quantidade de chunks gerados
    embedding_model: str        # Modelo usado para embeddings
```

#### 4.3.2 Estrutura de Chunk

```python
@dataclass
class Chunk:
    chunk_id: str               # UUID único
    document_id: str            # Referência ao documento pai
    content: str                # Texto do chunk
    chunk_index: int            # Posição no documento (0-based)
    page_number: int | None     # Página no PDF original
    section_title: str | None   # Título da seção (se detectável)
    embedding: list[float]      # Vetor de embedding
    metadata: DocumentMetadata  # Metadados herdados + específicos
```

### 4.4 Estrutura de Diretórios do Projeto

```
lexrag/
├── src/
│   ├── __init__.py
│   ├── cli.py                  # Comandos CLI (typer)
│   ├── config.py               # Configurações e constantes
│   ├── indexer/
│   │   ├── __init__.py
│   │   ├── loader.py           # DocumentLoader (PDF, DOCX, TXT)
│   │   ├── chunker.py          # Estratégias de chunking
│   │   ├── extractor.py        # Extração de metadados jurídicos
│   │   └── embedder.py         # Geração de embeddings
│   ├── retriever/
│   │   ├── __init__.py
│   │   ├── vector_store.py     # Interface ChromaDB
│   │   └── hybrid_search.py    # Busca híbrida (opcional)
│   ├── mcp/
│   │   ├── __init__.py
│   │   ├── server.py           # MCP Server principal
│   │   └── tools.py            # Definição das tools
│   └── utils/
│       ├── __init__.py
│       ├── legal_patterns.py   # Regex para nº processo, tribunais
│       └── text_utils.py       # Limpeza e normalização
├── tests/
│   ├── __init__.py
│   ├── test_loader.py
│   ├── test_chunker.py
│   ├── test_retriever.py
│   └── fixtures/               # Documentos de teste
├── data/
│   ├── chroma/                 # Banco vetorial (gitignore)
│   └── sources/                # Documentos fonte (ou symlink)
├── scripts/
│   └── setup_ollama.sh         # Script para instalar modelo local
├── CLAUDE.md                   # Instruções para Claude Code
├── pyproject.toml              # Dependências e build
├── README.md                   # Documentação
└── .mcp.json                   # Configuração MCP local (opcional)
```

---

## 5. Especificações Detalhadas

### 5.1 Extração de Metadados Jurídicos (Regex)

```python
# legal_patterns.py

import re
from typing import Optional

# Padrão CNJ: NNNNNNN-DD.AAAA.J.TR.OOOO
PATTERN_CNJ = re.compile(
    r'\b(\d{7}-\d{2}\.\d{4}\.\d\.\d{2}\.\d{4})\b'
)

# Padrão antigo (TRFs): NNNN.NN.NN.NNNNNN-N
PATTERN_ANTIGO = re.compile(
    r'\b(\d{4}\.\d{2}\.\d{2}\.\d{6}-\d)\b'
)

# Tribunais
TRIBUNAIS = {
    'STF': re.compile(r'\bSTF\b|Supremo Tribunal Federal', re.I),
    'STJ': re.compile(r'\bSTJ\b|Superior Tribunal de Justiça', re.I),
    'TRF1': re.compile(r'\bTRF[ -]?1\b|TRF da 1ª Região', re.I),
    'TRF2': re.compile(r'\bTRF[ -]?2\b|TRF da 2ª Região', re.I),
    'TRF3': re.compile(r'\bTRF[ -]?3\b|TRF da 3ª Região', re.I),
    'TRF4': re.compile(r'\bTRF[ -]?4\b|TRF da 4ª Região', re.I),
    'TRF5': re.compile(r'\bTRF[ -]?5\b|TRF da 5ª Região', re.I),
    'TRF6': re.compile(r'\bTRF[ -]?6\b|TRF da 6ª Região', re.I),
    'CARF': re.compile(r'\bCARF\b|Conselho Administrativo', re.I),
    'CSRF': re.compile(r'\bCSRF\b|Câmara Superior', re.I),
}

# Tipos de documento
DOC_TYPES = {
    'peticao_inicial': re.compile(r'petição inicial|exordial', re.I),
    'contestacao': re.compile(r'contestação', re.I),
    'recurso': re.compile(r'apelação|agravo|recurso especial|recurso extraordinário', re.I),
    'sentenca': re.compile(r'sentença|decisão de mérito', re.I),
    'acordao': re.compile(r'acórdão|acordam os', re.I),
    'parecer': re.compile(r'parecer|opinião legal', re.I),
}


def extract_processo(text: str) -> Optional[str]:
    """Extrai primeiro número de processo encontrado."""
    match = PATTERN_CNJ.search(text)
    if match:
        return match.group(1)
    match = PATTERN_ANTIGO.search(text)
    if match:
        return match.group(1)
    return None


def extract_tribunal(text: str) -> Optional[str]:
    """Identifica tribunal mencionado no texto."""
    for tribunal, pattern in TRIBUNAIS.items():
        if pattern.search(text):
            return tribunal
    return None


def extract_doc_type(text: str, filename: str) -> str:
    """Identifica tipo de documento por conteúdo ou nome."""
    # Primeiro tenta pelo conteúdo
    for doc_type, pattern in DOC_TYPES.items():
        if pattern.search(text[:5000]):  # Primeiros 5k chars
            return doc_type
    
    # Fallback pelo nome do arquivo
    filename_lower = filename.lower()
    if 'peticao' in filename_lower or 'inicial' in filename_lower:
        return 'peticao_inicial'
    if 'acordao' in filename_lower:
        return 'acordao'
    if 'parecer' in filename_lower:
        return 'parecer'
    
    return 'outro'
```

### 5.2 Configuração do Chunking

```python
# chunker.py

from langchain.text_splitter import RecursiveCharacterTextSplitter

def get_legal_chunker(
    chunk_size: int = 1000,
    chunk_overlap: int = 200,
) -> RecursiveCharacterTextSplitter:
    """
    Chunker otimizado para documentos jurídicos.
    
    Separadores ordenados por prioridade:
    1. Seções numeradas (Art., §, inciso)
    2. Parágrafos duplos
    3. Parágrafos simples
    4. Sentenças
    5. Palavras
    """
    return RecursiveCharacterTextSplitter(
        chunk_size=chunk_size,
        chunk_overlap=chunk_overlap,
        separators=[
            # Artigos e parágrafos legais
            r"\nArt\. \d+",
            r"\n§ \d+",
            r"\n[IVX]+[ –-]",
            # Estrutura de petições
            r"\n[IVX]+\. ",
            r"\nDOS? ",
            r"\nDA ",
            # Parágrafos
            "\n\n",
            "\n",
            # Sentenças
            ". ",
            # Fallback
            " ",
        ],
        length_function=len,
        is_separator_regex=True,
    )
```

### 5.3 MCP Server - Tools

```python
# mcp/tools.py

from mcp.server import Server
from mcp.types import Tool, TextContent
from typing import Any

app = Server("lexrag")

@app.tool()
async def search_documents(
    query: str,
    top_k: int = 5,
    doc_type: str | None = None,
    tribunal: str | None = None,
    area: str | None = None,
    tags: list[str] | None = None,
) -> list[dict[str, Any]]:
    """
    Busca documentos jurídicos por similaridade semântica.
    
    Args:
        query: Pergunta ou termos de busca em linguagem natural
        top_k: Número máximo de resultados (padrão: 5)
        doc_type: Filtrar por tipo (peticao, acordao, parecer, etc.)
        tribunal: Filtrar por tribunal (STF, STJ, TRF5, CARF, etc.)
        area: Filtrar por área (tributario, civil, administrativo)
        tags: Filtrar por tags específicas
    
    Returns:
        Lista de chunks relevantes com metadados e score
    """
    # Implementação conecta ao retriever
    from ..retriever.vector_store import search
    
    filters = {}
    if doc_type:
        filters["doc_type"] = doc_type
    if tribunal:
        filters["tribunal"] = tribunal
    if area:
        filters["area"] = area
    if tags:
        filters["tags"] = {"$contains_any": tags}
    
    results = await search(
        query=query,
        top_k=top_k,
        filters=filters if filters else None,
    )
    
    return [
        {
            "chunk_id": r.chunk_id,
            "content": r.content,
            "score": r.score,
            "source": r.metadata.file_name,
            "path": r.metadata.file_path,
            "doc_type": r.metadata.doc_type,
            "processo": r.metadata.numero_processo,
            "tribunal": r.metadata.tribunal,
            "tese": r.metadata.tese_principal,
            "page": r.page_number,
        }
        for r in results
    ]


@app.tool()
async def get_document_chunk(chunk_id: str) -> dict[str, Any]:
    """
    Recupera um chunk específico pelo ID.
    
    Útil para obter contexto adicional de um resultado de busca.
    
    Args:
        chunk_id: ID único do chunk
    
    Returns:
        Chunk completo com conteúdo e metadados
    """
    from ..retriever.vector_store import get_by_id
    
    chunk = await get_by_id(chunk_id)
    if not chunk:
        return {"error": "Chunk não encontrado"}
    
    return {
        "chunk_id": chunk.chunk_id,
        "content": chunk.content,
        "document_id": chunk.document_id,
        "chunk_index": chunk.chunk_index,
        "page_number": chunk.page_number,
        "metadata": {
            "file_name": chunk.metadata.file_name,
            "file_path": chunk.metadata.file_path,
            "doc_type": chunk.metadata.doc_type,
            "processo": chunk.metadata.numero_processo,
            "tribunal": chunk.metadata.tribunal,
            "tese": chunk.metadata.tese_principal,
            "tags": chunk.metadata.tags,
        }
    }


@app.tool()
async def list_sources(
    doc_type: str | None = None,
    limit: int = 50,
) -> list[dict[str, Any]]:
    """
    Lista documentos indexados no sistema.
    
    Args:
        doc_type: Filtrar por tipo de documento
        limit: Número máximo de resultados
    
    Returns:
        Lista de documentos com metadados básicos
    """
    from ..retriever.vector_store import list_documents
    
    docs = await list_documents(doc_type=doc_type, limit=limit)
    
    return [
        {
            "document_id": d.document_id,
            "file_name": d.file_name,
            "doc_type": d.doc_type,
            "processo": d.numero_processo,
            "tribunal": d.tribunal,
            "chunk_count": d.chunk_count,
            "indexed_at": d.data_indexacao.isoformat(),
        }
        for d in docs
    ]
```

### 5.4 Configuração MCP para Claude Code

```json
// ~/.claude.json (adicionar ao arquivo existente)

{
  "mcpServers": {
    "lexrag": {
      "command": "python",
      "args": ["-m", "lexrag.mcp.server"],
      "cwd": "/caminho/para/lexrag",
      "env": {
        "LEXRAG_DATA_DIR": "/caminho/para/lexrag/data",
        "LEXRAG_EMBEDDING_MODEL": "all-MiniLM-L6-v2"
      }
    }
  }
}
```

Alternativa com Ollama para embeddings:

```json
{
  "mcpServers": {
    "lexrag": {
      "command": "python",
      "args": ["-m", "lexrag.mcp.server"],
      "cwd": "/caminho/para/lexrag",
      "env": {
        "LEXRAG_DATA_DIR": "/caminho/para/lexrag/data",
        "LEXRAG_EMBEDDING_MODEL": "nomic-embed-text",
        "LEXRAG_EMBEDDING_PROVIDER": "ollama",
        "OLLAMA_HOST": "http://localhost:11434"
      }
    }
  }
}
```

---

## 6. CLAUDE.md para o Projeto

```markdown
# LexRAG - Sistema RAG Jurídico Local

## Visão Geral

Sistema de Retrieval-Augmented Generation para documentos jurídicos do escritório Pedrosa & Peixoto Advogados. Permite busca semântica em petições, jurisprudência, pareceres e legislação.

## Contexto Jurídico

- **Área principal:** Direito Tributário Federal
- **Foco:** Transação Tributária (Lei 13.988/2020), regularização de passivo fiscal
- **Clientes:** MPEs e PF classe média com débitos federais R$ 50k-2M

## Comandos Principais

```bash
# Indexar documentos
lexrag index ~/Documentos/Juridico --recursive

# Buscar no acervo
lexrag search "prescrição intercorrente execução fiscal"

# Iniciar servidor MCP
lexrag serve

# Status do índice
lexrag status
```

## Tools MCP Disponíveis

### search_documents
Busca semântica no acervo. Parâmetros:
- `query`: texto da busca
- `top_k`: quantidade de resultados (padrão: 5)
- `doc_type`: filtro por tipo (peticao, acordao, parecer)
- `tribunal`: filtro por tribunal (STJ, TRF5, CARF)
- `tags`: filtro por tags

### get_document_chunk
Recupera chunk específico por ID para contexto adicional.

### list_sources
Lista documentos indexados com metadados.

## Estrutura do Projeto

```
src/
├── cli.py           # Comandos CLI
├── indexer/         # Carga e processamento de documentos
├── retriever/       # Busca vetorial (ChromaDB)
├── mcp/             # Servidor MCP e tools
└── utils/           # Padrões jurídicos e utilidades
```

## Padrões de Código

- Python 3.11+
- Type hints obrigatórios
- Docstrings no formato Google
- Testes com pytest
- Async/await para operações I/O

## Metadados Jurídicos

Cada documento indexado contém:
- `numero_processo`: formato CNJ ou antigo
- `tribunal`: STF, STJ, TRF1-6, CARF, etc.
- `doc_type`: peticao, acordao, parecer, legislacao
- `area`: tributario, civil, administrativo
- `tese_principal`: resumo em até 200 chars
- `tags`: classificação livre

## Regex para Extração

- Número CNJ: `\d{7}-\d{2}\.\d{4}\.\d\.\d{2}\.\d{4}`
- Número antigo: `\d{4}\.\d{2}\.\d{2}\.\d{6}-\d`

## Limitações Conhecidas

1. Não processa PDFs escaneados (sem OCR)
2. Embeddings locais podem ter qualidade inferior a APIs comerciais
3. Chunking pode cortar argumentos longos

## Compliance

- Dados sensíveis de clientes NÃO são enviados para APIs externas
- Embeddings gerados 100% localmente
- Logs não registram conteúdo, apenas queries
```

---

## 7. Plano de Implementação

### Fase 1: Setup Básico (4-6h)

| Tarefa | Entregável | Critério de Aceite |
|--------|------------|-------------------|
| 1.1 | Estrutura do projeto + pyproject.toml | `pip install -e .` funciona |
| 1.2 | DocumentLoader para PDF | Extrai texto de PDF de teste |
| 1.3 | DocumentLoader para DOCX | Extrai texto de DOCX de teste |
| 1.4 | Chunker configurado | Gera chunks de ~1000 chars |
| 1.5 | Embedder local (sentence-transformers) | Gera embedding de 384 dims |
| 1.6 | ChromaDB básico | Indexa e busca 1 documento |

### Fase 2: Extração Jurídica (2-3h)

| Tarefa | Entregável | Critério de Aceite |
|--------|------------|-------------------|
| 2.1 | Regex para nº processo CNJ | Extrai de 10 documentos teste |
| 2.2 | Detecção de tribunal | Identifica corretamente tribunal |
| 2.3 | Classificação doc_type | Classifica 80%+ corretamente |
| 2.4 | Estrutura de metadados | Dataclass completa |

### Fase 3: MCP Server (2-3h)

| Tarefa | Entregável | Critério de Aceite |
|--------|------------|-------------------|
| 3.1 | Servidor MCP básico | Inicia sem erros |
| 3.2 | Tool search_documents | Retorna resultados no Claude Code |
| 3.3 | Tool get_document_chunk | Recupera chunk por ID |
| 3.4 | Tool list_sources | Lista documentos indexados |
| 3.5 | Configuração ~/.claude.json | Claude Code conecta ao servidor |

### Fase 4: CLI e Polimento (2-3h)

| Tarefa | Entregável | Critério de Aceite |
|--------|------------|-------------------|
| 4.1 | CLI index | Indexa pasta recursivamente |
| 4.2 | CLI search | Busca com output formatado |
| 4.3 | CLI status | Mostra estatísticas |
| 4.4 | CLI serve | Inicia MCP server |
| 4.5 | README completo | Instruções de instalação |
| 4.6 | CLAUDE.md | Contexto para Claude Code |

### Fase 5: Testes e Validação (2h)

| Tarefa | Entregável | Critério de Aceite |
|--------|------------|-------------------|
| 5.1 | Testes unitários | Cobertura > 70% |
| 5.2 | Teste E2E | Fluxo completo funciona |
| 5.3 | Validação com acervo real | 10 queries corretas |

---

## 8. Dependências (pyproject.toml)

```toml
[project]
name = "lexrag"
version = "0.1.0"
description = "RAG jurídico local para Claude Code"
requires-python = ">=3.11"
dependencies = [
    # Core
    "chromadb>=0.4.0",
    "sentence-transformers>=2.2.0",
    
    # Document processing
    "pymupdf>=1.23.0",          # PDF extraction
    "python-docx>=1.1.0",       # DOCX extraction
    "mammoth>=1.6.0",           # DOCX to text
    
    # Chunking (opcional, pode usar implementação própria)
    "langchain-text-splitters>=0.0.1",
    
    # MCP
    "mcp>=1.0.0",
    
    # CLI
    "typer>=0.9.0",
    "rich>=13.0.0",
    
    # Utilities
    "pydantic>=2.0.0",
    "python-dotenv>=1.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0.0",
    "pytest-asyncio>=0.21.0",
    "ruff>=0.1.0",
]
ollama = [
    "ollama>=0.1.0",            # Para embeddings via Ollama
]

[project.scripts]
lexrag = "lexrag.cli:app"
```

---

## 9. Riscos e Mitigações

| Risco | Probabilidade | Impacto | Mitigação |
|-------|---------------|---------|-----------|
| Qualidade de embeddings locais inferior a APIs | Alta | Médio | Testar com nomic-embed-text (Ollama) que tem qualidade próxima ao OpenAI; ajustar chunk_size |
| PDFs mal formatados (escaneados, tabelas) | Alta | Alto | Pré-requisito: apenas PDFs com texto selecionável; documentar limitação |
| ChromaDB não escala para acervo grande | Média | Médio | Monitorar performance; migrar para Qdrant se necessário |
| Extração de metadados falha (regex) | Média | Baixo | Permitir metadados manuais via YAML; melhorar regex iterativamente |
| MCP instável entre versões Claude Code | Baixa | Alto | Fixar versão do SDK MCP; testar antes de atualizar Claude Code |

---

## 10. Métricas de Sucesso

| Métrica | Meta | Como Medir |
|---------|------|------------|
| Precisão de busca | > 80% relevante no top-3 | Avaliação manual de 20 queries |
| Tempo de resposta | < 2s para busca | Logs de latência |
| Cobertura de metadados | > 70% docs com tribunal identificado | Query no ChromaDB |
| Adoção | Uso diário no Claude Code | Auto-relato |

---

## 11. Checklist de QA

### Antes do Deploy

- [ ] Todos os testes passam (`pytest`)
- [ ] Indexação de 100+ documentos sem erro
- [ ] Busca retorna resultados em < 2s
- [ ] MCP server conecta ao Claude Code
- [ ] Tools funcionam no Claude Code
- [ ] README documenta instalação
- [ ] CLAUDE.md presente e completo
- [ ] Dados sensíveis não vazam (verificar logs)

### Queries de Validação

1. "Qual o prazo para adesão à transação tributária?"
2. "Jurisprudência sobre prescrição intercorrente no STJ"
3. "Requisitos para transação individual PGFN"
4. "Desconto máximo na transação de pequeno valor"
5. "Fundamentação para exclusão de multa qualificada"

---

## 12. Próximos Passos (v2.0 - Futuro)

- [ ] OCR para PDFs escaneados (Tesseract/EasyOCR)
- [ ] Interface web para consultas (Streamlit/Gradio)
- [ ] Integração com PJe para download automático
- [ ] RAG com reranking (cross-encoder)
- [ ] Geração de resumos automáticos de acórdãos
- [ ] Alertas de nova jurisprudência relevante

---

**Documento elaborado conforme boas práticas de PRD para projetos de software, adaptado ao contexto jurídico-tributário do escritório Pedrosa & Peixoto Advogados.**
