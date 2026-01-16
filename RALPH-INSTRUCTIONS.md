# Como Usar o Plugin Ralph Wiggum com LexRAG

## 🔄 O Plugin Ralph Wiggum

O plugin Ralph Wiggum permite desenvolvimento autônomo iterativo, onde o Claude Code trabalha continuamente em uma tarefa até completá-la.

## 📋 Pré-requisitos

1. **Plugin instalado** (já está em `~/.claude/plugins/repos/ralph-wiggum/`)
2. **Plugin ativado** (verifique em `~/.claude/plugins/config.json`)

## 🚀 Como Ativar o Plugin

### Opção 1: Reiniciar VSCode (Recomendado)

1. Feche completamente o VSCode
2. Abra o VSCode novamente
3. Abra o projeto LexRAG
4. O plugin estará carregado automaticamente

### Opção 2: Verificar se já está ativo

Tente executar:

```bash
/help ralph-wiggum
```

Se funcionar, o plugin já está ativo!

## 🎯 Como Executar o Ralph Loop

### Passo 1: Instalar Dependências

```bash
pip install -e .
```

### Passo 2: Executar o Ralph Loop

Existem **duas formas** de executar:

#### Forma A: Usando o Prompt Pré-Preparado

```bash
/ralph-loop --file .ralph-prompt.md --max-iterations 50 --completion-promise "FASE1_COMPLETE"
```

#### Forma B: Usando o Prompt Direto

```bash
/ralph-loop "Implemente o sistema LexRAG Fase 1 conforme .ralph-prompt.md. Siga a metodologia TDD: escreva testes primeiro, implemente depois, refatore quando necessário. Output <promise>FASE1_COMPLETE</promise> quando todas as tarefas estiverem completas." --max-iterations 50 --completion-promise "FASE1_COMPLETE"
```

## 📝 O Que Acontece Depois

O Ralph Wiggum irá:

1. **Ler o prompt** inicial
2. **Implementar** o código iterativamente
3. **Testar** cada módulo
4. **Corrigir erros** automaticamente
5. **Refatorar** quando necessário
6. **Repetir** até `<promise>FASE1_COMPLETE</promise>` ser outputado

## ⚠️ Regras Importantes

### Para o Claude Code (AI):
- **NUNCA** output `<promise>FASE1_COMPLETE</promise>` a menos que seja **VERDADEIRAMENTE** completo
- Não tente " escapar" do loop com promises falsas
- O loop é designed para continuar até conclusão genuína

### Para Você (Usuário):
- **Monitore** o progresso ocasionalmente
- **Intervenha** apenas se necessário (ex: parada por erro crítico)
- **Seja paciente** - o loop pode rodar por horas
- Use `/cancel-ralph` se precisar parar

## 🛑 Como Parar o Loop

### Cancelar Manualmente

```bash
/cancel-ralph
```

### Limite de Iterações

O loop para automaticamente após `--max-iterations` (ex: 50)

### Completion Promise

O loop para quando `<promise>FASE1_COMPLETE</promise>` é detectado

## 📊 O Que Será Implementado

### Fase 1: Setup Básico

1. ✅ pyproject.toml (já criado)
2. ⏳ config.py
3. ⏳ loader.py (PDF/DOCX/TXT)
4. ⏳ ocr.py (EasyOCR/Tesseract)
5. ⏳ chunker.py (LangChain splitter)
6. ⏳ embedder.py (sentence-transformers)
7. ⏳ vector_store.py (ChromaDB)
8. ⏳ Testes unitários para todos

## 🔍 Acompanhamento

### Ver Progresso

```bash
# Ver últimos commits
git log --oneline

# Ver arquivos modificados
git status

# Ver testes
pytest tests/ -v
```

### Ver Arquivos Criados

```bash
# Listar módulos Python
find src/ -name "*.py" | grep -v __pycache__

# Listar testes
find tests/ -name "test_*.py"
```

## 🐛 Troubleshooting

### Plugin não reconhece `/ralph-loop`

**Solução:** Reinicie o VSCode completamente

### Loop trava ou erro crítico

**Solução:** Execute `/cancel-ralph` e investigue o erro

### Testes falhando

**Solução:** O Ralph tentará corrigir automaticamente. Se travar, cancele e investigue manualmente

## 🎉 Após Conclusão

Quando ver `✅ Ralph loop: Detected <promise>FASE1_COMPLETE</promise>`:

1. **Verifique** todos os módulos foram criados
2. **Rode** os testes: `pytest tests/ -v`
3. **Verifique** imports funcionam
4. **Commit** as mudanças: `git add . && git commit -m "Complete Fase 1"`
5. **Push** para GitHub: `git push origin main`

## 📚 Recursos

- [Ralph Wiggum README](~/.claude/plugins/repos/ralph-wiggum/README.md)
- [CLAUDE.md](CLAUDE.md) - Documentação do projeto
- [prd-rag-juridico-local.md](prd-rag-juridico-local.md) - PRD completo

---

**Preparado para começar?**

1. Reinicie o VSCode
2. Execute: `/ralph-loop --file .ralph-prompt.md --max-iterations 50 --completion-promise "FASE1_COMPLETE"`
3. Relaxe e deixe o Ralph trabalhar! 🚀
