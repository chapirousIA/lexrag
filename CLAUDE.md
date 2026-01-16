# LexRAG - Sistema RAG Jurídico Local

## Visão Geral

**Projeto:** LexRAG - Sistema de Retrieval-Augmented Generation para Documentos Jurídicos
**Versão:** 1.0
**Stakeholder:** Pedrosa & Peixoto Advogados
**Objetivo:** Sistema RAG local para busca semântica em documentos jurídicos, integrado ao Claude Code via MCP (Model Context Protocol)

Sistema que permite consultas semânticas em documentos tributários (petições, pareceres, jurisprudência, legislação) armazenados localmente, sem envio de dados sensíveis para servidores externos.

**Resultado esperado:** Claude Code poderá responder perguntas sobre o acervo documental do escritório, citando fontes específicas com número de processo, tribunal e tese jurídica.

## Contexto Jurídico

- **Área principal:** Direito Tributário Federal
- **Foco:** Transação Tributária (Lei 13.988/2020), regularização de passivo fiscal
- **Clientes:** MPEs e PF classe média com débitos federais R$ 50k-2M
- **Acervo:** Petições, pareceres, jurisprudência em PDF/DOCX dispersos em pastas locais

**Problema resolvido:** Substituir buscas Ctrl+F por busca semântica, recuperando contexto entre documentos e evitando retrabalho em pesquisas recorrentes.

## Stack Tecnológico

| Camada | Tecnologia | Justificativa |
|--------|------------|---------------|
| Linguagem | Python 3.11+ | Ecossistema NLP maduro; facilidade de manutenção |
| Vector DB | ChromaDB | Leve, local, sem servidor separado, persistência SQLite |
| Embeddings | `sentence-transformers/all-MiniLM-L6-v2` ou `nomic-embed-text` via Ollama | Trade-off qualidade/velocidade; 100% local |
| Extração PDF | `pymupdf` (fitz) | Mais rápido que PyPDF2; preserva layout |
| Extração DOCX | `python-docx` + `mammoth` | Estrutura + texto limpo |
| OCR | `easyocr` ou `tesseract` + `pytesseract` | Extração de texto de PDFs escaneados |
| Chunking | `langchain.text_splitter.RecursiveCharacterTextSplitter` | Configurável; respeita estrutura |
| MCP Server | `mcp` (SDK oficial Anthropic) | Integração nativa Claude Code |
| CLI | `typer` | Experiência moderna; autocompleção |

## Estrutura do Projeto

```
lexrag/
├── src/
│   ├── __init__.py
│   ├── cli.py                  # Comandos CLI (typer)
│   ├── config.py               # Configurações e constantes
│   ├── indexer/
│   │   ├── __init__.py
│   │   ├── loader.py           # DocumentLoader (PDF, DOCX, TXT)
│   │   ├── ocr.py              # OCR para PDFs escaneados (EasyOCR/Tesseract)
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
│   ├── test_ocr.py
│   ├── test_chunker.py
│   ├── test_retriever.py
│   └── fixtures/               # Documentos de teste
├── data/
│   ├── chroma/                 # Banco vetorial (gitignore)
│   └── sources/                # Documentos fonte (ou symlink)
├── scripts/
│   ├── setup_ollama.sh         # Script para instalar modelo local
│   └── setup_tesseract.sh      # Script para instalar Tesseract (opcional)
├── CLAUDE.md                   # Este arquivo
├── pyproject.toml              # Dependências e build
└── README.md                   # Documentação
```

## Comandos CLI Principais

### Indexar Documentos
```bash
lexrag index ~/Documentos/Juridico --recursive
```
- Indexa arquivos PDF, DOCX, TXT, MD
- OCR automático para PDFs escaneados (sem texto selecionável)
- Extração automática de metadados via regex
- Chunking inteligente respeitando parágrafos e seções
- Re-indexação incremental (apenas arquivos modificados)

### Buscar no Acervo
```bash
lexrag search "prescrição intercorrente execução fiscal"
```
- Busca semântica por similaridade vetorial
- Retorna chunks rankeados com score de relevância
- Mostra caminho do arquivo fonte para verificação

### Status do Índice
```bash
lexrag status
```
- Estatísticas do índice (quantidade de documentos, chunks)
- Informações sobre modelo de embeddings
- Espaço utilizado

### Iniciar Servidor MCP
```bash
lexrag serve
```
- Inicia servidor MCP para integração com Claude Code
- Exponha tools: search_documents, get_document_chunk, list_sources
- Stdio ou SSE conforme configuração

## Tools MCP Disponíveis

### search_documents

Busca documentos jurídicos por similaridade semântica.

**Parâmetros:**
- `query` (str, obrigatório): Pergunta ou termos de busca em linguagem natural
- `top_k` (int, opcional): Número máximo de resultados (padrão: 5)
- `doc_type` (str, opcional): Filtrar por tipo (peticao, acordao, parecer, legislacao)
- `tribunal` (str, opcional): Filtrar por tribunal (STF, STJ, TRF5, CARF, etc.)
- `area` (str, opcional): Filtrar por área (tributario, civil, administrativo)
- `tags` (list[str], opcional): Filtrar por tags específicas

**Retorna:**
```python
[
    {
        "chunk_id": "uuid",
        "content": "texto do chunk",
        "score": 0.85,
        "source": "nome_do_arquivo.pdf",
        "path": "/caminho/completo/arquivo.pdf",
        "doc_type": "acordao",
        "processo": "0000000-00.0000.0.00.0000",
        "tribunal": "STJ",
        "tese": "resumo da tese jurídica",
        "page": 15
    }
]
```

### get_document_chunk

Recupera um chunk específico pelo ID para contexto adicional.

**Parâmetros:**
- `chunk_id` (str, obrigatório): ID único do chunk

**Retorna:**
```python
{
    "chunk_id": "uuid",
    "content": "texto completo do chunk",
    "document_id": "doc_uuid",
    "chunk_index": 5,
    "page_number": 15,
    "metadata": {
        "file_name": "arquivo.pdf",
        "file_path": "/caminho/completo/arquivo.pdf",
        "doc_type": "acordao",
        "processo": "0000000-00.0000.0.00.0000",
        "tribunal": "STJ",
        "tese": "resumo da tese",
        "tags": ["tag1", "tag2"]
    }
}
```

### list_sources

Lista documentos indexados no sistema.

**Parâmetros:**
- `doc_type` (str, opcional): Filtrar por tipo de documento
- `limit` (int, opcional): Número máximo de resultados (padrão: 50)

**Retorna:**
```python
[
    {
        "document_id": "doc_uuid",
        "file_name": "arquivo.pdf",
        "doc_type": "acordao",
        "processo": "0000000-00.0000.0.00.0000",
        "tribunal": "STJ",
        "chunk_count": 12,
        "indexed_at": "2026-01-15T10:30:00"
    }
]
```

## Padrões de Código e Desenvolvimento

### Requisitos de Código
- **Python 3.11+** - Uso de features modernas (match/case, type params)
- **Type hints obrigatórios** - Todas as funções devem ter anotações de tipo
- **Docstrings Google** - Documentação no formato Google Style
- **Async/await** - Operações I/O devem ser assíncronas
- **Testes pytest** - Cobertura > 70% com pytest + pytest-asyncio

### Exemplo de Padrão

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class DocumentMetadata:
    """Metadados jurídicos de um documento indexado.

    Attributes:
        file_path: Caminho absoluto do arquivo fonte
        file_name: Nome do arquivo
        file_hash: SHA256 para detectar alterações
        doc_type: Tipo do documento (peticao, acordao, parecer)
        numero_processo: Número do processo no formato CNJ
        tribunal: Tribunal responsável (STF, STJ, TRF5, etc.)
    """
    file_path: str
    file_name: str
    file_hash: str
    doc_type: str
    numero_processo: Optional[str] = None
    tribunal: Optional[str] = None
```

## Metadados Jurídicos

### Estrutura de Metadados

Cada documento indexado contém:
- `numero_processo`: formato CNJ (NNNNNNN-DD.AAAA.J.TR.OOOO) ou antigo
- `tribunal`: STF, STJ, TRF1-6, CARF, CSRF
- `doc_type`: peticao_inicial, contestacao, recurso, sentenca, acordao, parecer
- `area`: tributario, civil, administrativo
- `tese_principal`: resumo em até 200 caracteres
- `tags`: classificação livre (ex: ["transacao_tributaria", "pgfn", "desconto"])

### Tipos de Documento

| Tipo | Descrição | Padrão de Detecção |
|------|-----------|-------------------|
| peticao_inicial | Petição inicial/exordial | Conteúdo ou nome |
| contestacao | Contestação à ação | Conteúdo |
| recurso | Apelação, agravo, RE/REX | Conteúdo |
| sentenca | Sentença ou decisão de mérito | Conteúdo |
| acordao | Acórdão de tribunal | Conteúdo |
| parecer | Parecer/opinião legal | Conteúdo ou nome |
| legislacao | Leis, decretos, normas | Manual |

## Regex para Extração Jurídica

### Números de Processo

**Formato CNJ:** `\d{7}-\d{2}\.\d{4}\.\d\.\d{2}\.\d{4}`
Exemplo: `0000000-00.0000.0.00.0000`

**Formato Antigo:** `\d{4}\.\d{2}\.\d{2}\.\d{6}-\d`
Exemplo: `1234.56.78.901234-5`

### Tribunais

```python
TRIBUNAIS = {
    'STF': r'\bSTF\b|Supremo Tribunal Federal',
    'STJ': r'\bSTJ\b|Superior Tribunal de Justiça',
    'TRF1': r'\bTRF[ -]?1\b|TRF da 1ª Região',
    'TRF2': r'\bTRF[ -]?2\b|TRF da 2ª Região',
    'TRF3': r'\bTRF[ -]?3\b|TRF da 3ª Região',
    'TRF4': r'\bTRF[ -]?4\b|TRF da 4ª Região',
    'TRF5': r'\bTRF[ -]?5\b|TRF da 5ª Região',
    'TRF6': r'\bTRF[ -]?6\b|TRF da 6ª Região',
    'CARF': r'\bCARF\b|Conselho Administrativo',
    'CSRF': r'\bCSRF\b|Câmara Superior',
}
```

### Detecção de Tipo de Documento

```python
DOC_TYPES = {
    'peticao_inicial': r'petição inicial|exordial',
    'contestacao': r'contestação',
    'recurso': r'apelação|agravo|recurso especial|recurso extraordinário',
    'sentenca': r'sentença|decisão de mérito',
    'acordao': r'acórdão|acordam os',
    'parecer': r'parecer|opinião legal',
}
```

## Configuração MCP

### Claude Code Config (~/.claude.json)

```json
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

### Configuração com Ollama (Alternativa)

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

### Variáveis de Ambiente

- `LEXRAG_DATA_DIR`: Diretório para dados (ChromaDB, sources)
- `LEXRAG_EMBEDDING_MODEL`: Modelo de embeddings (all-MiniLM-L6-v2, nomic-embed-text)
- `LEXRAG_EMBEDDING_PROVIDER`: Provider (sentence-transformers, ollama)
- `OLLAMA_HOST`: URL do Ollama (quando aplicável)

## Processamento de PDFs com OCR

### Detecção Automática

O sistema detecta automaticamente se um PDF precisa de OCR:
- **PDF com texto selecionável**: Usa extração direta com pymupdf
- **PDF escaneado**: Aplica OCR para extrair texto das imagens

### Engines de OCR

**EasyOCR (Recomendado)**
```python
# Instalação
pip install easyocr

# Uso
import easyocr
reader = easyocr.Reader(['pt'], gpu=False)  # Português
result = reader.readtext('image.png')
```
- ✅ Mais fácil de instalar (pip only)
- ✅ Suporte GPU para processamento mais rápido
- ✅ Melhor para documentos jurídicos em português

**Tesseract (Alternativa)**
```python
# Instalação
# Windows: baixar instalador em https://github.com/UB-Mannheim/tesseract/wiki
# Linux: sudo apt install tesseract-ocr
# macOS: brew install tesseract

pip install pytesseract pillow

# Uso
import pytesseract
from PIL import Image

text = pytesseract.image_to_string(Image.open('image.png'), lang='por')
```
- ⚠️ Requer instalação separada do binário Tesseract
- ✅ Engine madura e estável
- ✅ Múltiplos idiomas suportados

### Estratégia de OCR no DocumentLoader

```python
def extract_text_from_pdf(file_path: str) -> str:
    """Extrai texto de PDF com ou sem OCR.

    1. Tenta extração direta com pymupdf
    2. Se < 50 caracteres, assume PDF escaneado
    3. Aplica OCR página por página
    4. Combina texto extraído com metadados
    """
    doc = fitz.open(file_path)
    text = ""

    for page in doc:
        # Tenta extração direta primeiro
        page_text = page.get_text()

        if len(page_text.strip()) < 50:
            # Aplica OCR
            pix = page.get_pixmap()
            img_bytes = pix.tobytes("png")
            page_text = apply_ocr(img_bytes)

        text += f"\n--- Página {page.number + 1} ---\n"
        text += page_text

    return text
```

### Configuração de OCR

**Variáveis de ambiente:**
```bash
# Engine de OCR (easyocr ou tesseract)
LEXRAG_OCR_ENGINE=easyocr

# Usar GPU se disponível (EasyOCR)
LEXRAG_OCR_GPU=true

# Idioma (por padrão: por)
LEXRAG_OCR_LANGUAGE=por
```

**Configuração no pyproject.toml:**
```bash
lexrag index --ocr-engine easyocr --ocr-gpu
```

### Performance de OCR

| Engine | Velocidade (pág) | Qualidade | Instalação |
|--------|-----------------|-----------|------------|
| EasyOCR (CPU) | ~5-10s | Boa | Simples |
| EasyOCR (GPU) | ~1-2s | Boa | Simples |
| Tesseract | ~3-8s | Excelente | Complexa |

### Limitações do OCR

- Documentos muito antigos ou danificados podem ter erros
- Tabelas complexas podem ser mal interpretadas
- Handwriting (texto manuscrito) não é suportado
- Requer processamento adicional para documentos com imagens e texto mistos

## Configuração de Chunking

Chunking otimizado para documentos jurídicos com separadores em ordem de prioridade:

```python
separators = [
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
]
```

**Parâmetros padrão:**
- `chunk_size`: 1000 caracteres
- `chunk_overlap`: 200 caracteres

## Especificações de Performance

| Métrica | Meta | Como Medir |
|---------|------|------------|
| Busca | < 2s para até 10.000 chunks | Latência p95 |
| Indexação PDF | < 30s para 50 páginas | Tempo médio |
| Escalabilidade | Até 50.000 chunks | Limite testado |
| Boot MCP Server | < 5s | Tempo de inicialização |

## Limitações Conhecidas

1. **Qualidade de embeddings locais** - Pode ser inferior a APIs comerciais (OpenAI)
2. **Chunking pode cortar argumentos** - Argumentos longos podem ser divididos
3. **Sem interface web/GUI** - Apenas CLI e integração MCP na v1.0
4. **Sem multi-tenancy** - Apenas uso interno do escritório
5. **Extração de metadados falha** - Regex pode não capturar todos os casos (mitigação: YAML companion)
6. **OCR pode falhar em documentos de baixa qualidade** - Documentos muito antigos ou danificados podem ter erros de reconhecimento

## Compliance e Segurança

### Privacidade de Dados
- ✅ Zero dados enviados para servidores externos
- ✅ Embeddings gerados 100% localmente
- ✅ Nomes de clientes e valores sensíveis permanecem local
- ✅ Logs registram apenas queries, não conteúdo de documentos

### Auditoria
- Logs de queries para fins de auditoria (sem conteúdo sensível)
- Rastreabilidade de fontes via caminho completo do arquivo
- SHA256 hash para detectar alterações nos documentos

### Verificação
```bash
# Verificar que não há chamadas externas
pytest tests/test_external_calls.py

# Auditoria de logs
lexrag status --audit
```

## Dependências Principais

**Core:**
- chromadb>=0.4.0 - Vector database
- sentence-transformers>=2.2.0 - Embeddings locais

**Document Processing:**
- pymupdf>=1.23.0 - Extração de PDF
- python-docx>=1.1.0 - Extração de DOCX
- mammoth>=1.6.0 - DOCX para texto
- easyocr>=1.7.0 - OCR para PDFs escaneados (recomendado)
- pytesseract>=0.3.10 - Alternativa OCR (requer Tesseract instalado)
- pillow>=10.0.0 - Processamento de imagens para OCR

**Chunking:**
- langchain-text-splitters>=0.0.1 - Splitter inteligente

**MCP:**
- mcp>=1.0.0 - SDK oficial Anthropic

**CLI:**
- typer>=0.9.0 - CLI moderna
- rich>=13.0.0 - Output formatado

**Utilities:**
- pydantic>=2.0.0 - Validação de dados
- python-dotenv>=1.0.0 - Variáveis de ambiente

**Dev:**
- pytest>=7.0.0 - Testes
- pytest-asyncio>=0.21.0 - Testes assíncronos
- ruff>=0.1.0 - Linting

## Queries de Validação

Use estas queries para testar o sistema após implementação:

1. "Qual o prazo para adesão à transação tributária?"
2. "Jurisprudência sobre prescrição intercorrente no STJ"
3. "Requisitos para transação individual PGFN"
4. "Desconto máximo na transação de pequeno valor"
5. "Fundamentação para exclusão de multa qualificada"

## Plano de Implementação

### Fase 1: Setup Básico (4-6h)
- Estrutura do projeto + pyproject.toml
- DocumentLoader para PDF e DOCX
- OCR para PDFs escaneados (EasyOCR/Tesseract)
- Chunker configurado
- Embedder local (sentence-transformers)
- ChromaDB básico

### Fase 2: Extração Jurídica (2-3h)
- Regex para nº processo CNJ
- Detecção de tribunal
- Classificação doc_type
- Estrutura de metadados

### Fase 3: MCP Server (2-3h)
- Servidor MCP básico
- Tool search_documents
- Tool get_document_chunk
- Tool list_sources
- Configuração ~/.claude.json

### Fase 4: CLI e Polimento (2-3h)
- CLI index (recursive)
- CLI search (output formatado)
- CLI status (estatísticas)
- CLI serve (MCP server)
- README completo

### Fase 5: Testes e Validação (2h)
- Testes unitários (>70% cobertura)
- Teste E2E
- Validação com acervo real (10 queries)

**Tempo total estimado:** 8-16 horas

## Próximos Passos (v2.0)

- [ ] Interface web para consultas (Streamlit/Gradio)
- [ ] Integração com PJe para download automático
- [ ] RAG com reranking (cross-encoder)
- [ ] Geração de resumos automáticos de acórdãos
- [ ] Alertas de nova jurisprudência relevante
- [ ] OCR aprimorado com modelos especializados em documentos jurídicos
- [ ] Detecção automática de layout/tabelas em PDFs

## Troubleshooting

### Problemas Comuns

**Erro: "Modelo não encontrado"**
```bash
# Baixar modelo sentence-transformers
python -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('all-MiniLM-L6-v2')"
```

**Erro: "MCP server não conecta"**
- Verificar caminho em ~/.claude.json
- Confirmar que LEXRAG_DATA_DIR existe
- Testar com: `lexrag serve --verbose`

**Busca muito lenta**
- Reduzir top_k (ex: top_k=3)
- Verificar uso de CPU/disco
- Considerar modelo de embedding menor

**Metadados não extraídos**
- Verificar se padrão CNJ está correto
- Usar arquivo YAML companion para metadados manuais
- Melhorar regex em utils/legal_patterns.py

## Contato e Suporte

**Projeto mantido por:** Pedrosa & Peixoto Advogados
**Contexto:** Direito Tributário Federal - Transação Tributária
**Data de criação:** 15/01/2026
**Versão:** 1.0

Para questões sobre desenvolvimento ou melhorias, consultar o PRD original em `prd-rag-juridico-local.md`.
