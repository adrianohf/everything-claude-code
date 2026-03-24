# Guia de Solução de Problemas

Problemas comuns e soluções para o plugin Everything Claude Code (ECC).

## Sumário

- [Problemas de Memória e Contexto](#problemas-de-memória-e-contexto)
- [Falhas do Harness de Agentes](#falhas-do-harness-de-agentes)
- [Erros de Hooks e Workflows](#erros-de-hooks-e-workflows)
- [Instalação e Configuração](#instalação-e-configuração)
- [Problemas de Desempenho](#problemas-de-desempenho)
- [Mensagens de Erro Comuns](#mensagens-de-erro-comuns)
- [Obter Ajuda](#obter-ajuda)

---

## Problemas de Memória e Contexto

### Estouro da Janela de Contexto

**Sintoma:** Erros de "contexto muito longo" ou respostas incompletas

**Causas:**
- Uploads de arquivos grandes excedendo limites de tokens
- Histórico de conversa acumulado
- Múltiplas saídas grandes de ferramentas em uma única sessão

**Soluções:**
```bash
# 1. Limpar histórico de conversa e começar fresco
# Use no Claude Code: "Nova Conversa" ou Cmd/Ctrl+Shift+N

# 2. Reduzir tamanho do arquivo antes da análise
head -n 100 large-file.log > sample.log

# 3. Usar streaming para grandes saídas
head -n 50 large-file.txt

# 4. Dividir tarefas em blocos menores
# Em vez de: "Analisar todos os 50 arquivos"
# Use: "Analisar arquivos no diretório src/components/"
```

### Falhas de Persistência de Memória

**Sintoma:** Agente não lembra contexto ou observações anteriores

**Causas:**
- Hooks de aprendizado contínuo desabilitados
- Arquivos de observação corrompidos
- Falhas na detecção do projeto

**Soluções:**
```bash
# Verificar se observações estão sendo registradas
ls ~/.claude/homunculus/projects/*/observations.jsonl

# Encontrar o ID de hash do projeto atual
python3 - <<'PY'
import json, os
registry_path = os.path.expanduser("~/.claude/homunculus/projects.json")
with open(registry_path) as f:
    registry = json.load(f)
for project_id, meta in registry.items():
    if meta.get("root") == os.getcwd():
        print(project_id)
        break
else:
    raise SystemExit("Hash do projeto não encontrado em ~/.claude/homunculus/projects.json")
PY

# Visualizar observações recentes para esse projeto
tail -20 ~/.claude/homunculus/projects/<project-hash>/observations.jsonl

# Fazer backup de um arquivo de observações corrompido antes de recriá-lo
mv ~/.claude/homunculus/projects/<project-hash>/observations.jsonl \
  ~/.claude/homunculus/projects/<project-hash>/observations.jsonl.bak.$(date +%Y%m%d-%H%M%S)

# Verificar se hooks estão habilitados
grep -r "observe" ~/.claude/settings.json
```

---

## Falhas do Harness de Agentes

### Agente Não Encontrado

**Sintoma:** Erros de "agente não carregado" ou "agente desconhecido"

**Causas:**
- Plugin não instalado corretamente
- Caminho do agente mal configurado
- Incompatibilidade entre instalação via marketplace e manual

**Soluções:**
```bash
# Verificar instalação do plugin
ls ~/.claude/plugins/cache/

# Verificar se agente existe (instalação via marketplace)
ls ~/.claude/plugins/cache/*/agents/

# Para instalação manual, agentes devem estar em:
ls ~/.claude/agents/  # Apenas agentes personalizados

# Recarregar plugin
# Claude Code → Configurações → Extensões → Recarregar
```

### Execução de Workflow Trava

**Sintoma:** Agente inicia mas nunca completa

**Causas:**
- Loops infinitos na lógica do agente
- Bloqueado em entrada do usuário
- Timeout de rede esperando pela API

**Soluções:**
```bash
# 1. Verificar processos travados
ps aux | grep claude

# 2. Habilitar modo debug
export CLAUDE_DEBUG=1

# 3. Definir timeouts mais curtos
export CLAUDE_TIMEOUT=30

# 4. Verificar conectividade de rede
curl -I https://api.anthropic.com
```

### Erros de Uso de Ferramentas

**Sintoma:** "Execução de ferramenta falhou" ou permissão negada

**Causas:**
- Dependências faltantes (npm, python, etc.)
- Permissões de arquivo insuficientes
- Caminho não encontrado

**Soluções:**
```bash
# Verificar se ferramentas necessárias estão instaladas
which node python3 npm git

# Corrigir permissões em scripts de hooks
chmod +x ~/.claude/plugins/cache/*/hooks/*.sh
chmod +x ~/.claude/plugins/cache/*/skills/*/hooks/*.sh

# Verificar se PATH inclui binários necessários
echo $PATH
```

---

## Erros de Hooks e Workflows

### Hooks Não Disparam

**Sintoma:** Hooks pre/post não executam

**Causas:**
- Hooks não registrados em settings.json
- Sintaxe de hook inválida
- Script de hook não executável

**Soluções:**
```bash
# Verificar se hooks estão registrados
grep -A 10 '"hooks"' ~/.claude/settings.json

# Verificar se arquivos de hook existem e são executáveis
ls -la ~/.claude/plugins/cache/*/hooks/

# Testar hook manualmente
bash ~/.claude/plugins/cache/*/hooks/pre-bash.sh <<< '{"command":"echo test"}'

# Re-registrar hooks (se usando plugin)
# Desabilitar e re-habilitar plugin nas configurações do Claude Code
```

### Incompatibilidades de Versão Python/Node

**Sintoma:** "python3 não encontrado" ou "comando node não encontrado"

**Causas:**
- Instalação Python/Node faltando
- PATH não configurado
- Versão errada do Python (Windows)

**Soluções:**
```bash
# Instalar Python 3 (se faltando)
# macOS: brew install python3
# Ubuntu: sudo apt install python3
# Windows: Baixar de python.org

# Instalar Node.js (se faltando)
# macOS: brew install node
# Ubuntu: sudo apt install nodejs npm
# Windows: Baixar de nodejs.org

# Verificar instalações
python3 --version
node --version
npm --version

# Windows: Garantir que python (não python3) funciona
python --version
```

### Falsos Positivos do Blocker de Servidor Dev

**Sintoma:** Hook bloqueia comandos legítimos contendo "dev"

**Causas:**
- Conteúdo de heredoc disparando correspondência de padrão
- Comandos não-dev com "dev" nos argumentos

**Soluções:**
```bash
# Isso foi corrigido na v1.8.0+ (PR #371)
# Atualizar plugin para a versão mais recente

# Solução alternativa: Colocar servidores dev no tmux
tmux new-session -d -s dev "npm run dev"
tmux attach -t dev

# Desabilitar hook temporariamente se necessário
# Editar ~/.claude/settings.json e remover hook pre-bash
```

---

## Instalação e Configuração

### Plugin Não Carrega

**Sintoma:** Recursos do plugin indisponíveis após instalação

**Causas:**
- Cache do marketplace não atualizado
- Incompatibilidade de versão do Claude Code
- Arquivos do plugin corrompidos

**Soluções:**
```bash
# Inspecionar o cache do plugin antes de alterá-lo
ls -la ~/.claude/plugins/cache/

# Fazer backup do cache do plugin em vez de deletá-lo
mv ~/.claude/plugins/cache ~/.claude/plugins/cache.backup.$(date +%Y%m%d-%H%M%S)
mkdir -p ~/.claude/plugins/cache

# Reinstalar do marketplace
# Claude Code → Extensões → Everything Claude Code → Desinstalar
# Depois reinstalar do marketplace

# Verificar versão do Claude Code
claude --version
# Requer Claude Code 2.0+

# Instalação manual (se marketplace falhar)
git clone https://github.com/affaan-m/everything-claude-code.git
cp -r everything-claude-code ~/.claude/plugins/ecc
```

### Detecção do Gerenciador de Pacotes Falha

**Sintoma:** Gerenciador de pacotes errado usado (npm em vez de pnpm)

**Causas:**
- Arquivo lock ausente
- CLAUDE_PACKAGE_MANAGER não definido
- Múltiplos arquivos lock confundindo a detecção

**Soluções:**
```bash
# Definir gerenciador de pacotes preferido globalmente
export CLAUDE_PACKAGE_MANAGER=pnpm
# Adicionar a ~/.bashrc ou ~/.zshrc

# Ou definir por projeto
echo '{"packageManager": "pnpm"}' > .claude/package-manager.json

# Ou usar campo no package.json
npm pkg set packageManager="pnpm@8.15.0"

# Aviso: remover arquivos lock pode alterar versões de dependências instaladas.
# Faça commit ou backup do arquivo lock primeiro, depois execute uma instalação
# limpa e reexecute o CI. Apenas faça isso ao trocar intencionalmente de
# gerenciador de pacotes.
rm package-lock.json  # Se usando pnpm/yarn/bun
```

---

## Problemas de Desempenho

### Tempos de Resposta Lentos

**Sintoma:** Agente leva 30+ segundos para responder

**Causas:**
- Arquivos de observação grandes
- Muitos hooks ativos
- Latência de rede para API

**Soluções:**
```bash
# Arquivar observações grandes em vez de deletá-las
archive_dir="$HOME/.claude/homunculus/archive/$(date +%Y%m%d)"
mkdir -p "$archive_dir"
find ~/.claude/homunculus/projects -name "observations.jsonl" -size +10M -exec sh -c '
  for file do
    base=$(basename "$(dirname "$file")")
    gzip -c "$file" > "'"$archive_dir"'/${base}-observations.jsonl.gz"
    : > "$file"
  done
' sh {} +

# Desabilitar hooks não utilizados temporariamente
# Editar ~/.claude/settings.json

# Manter arquivos de observação ativos pequenos
# Arquivos mortos grandes devem ficar em ~/.claude/homunculus/archive/
```

### Alto Uso de CPU

**Sintoma:** Claude Code consumindo 100% de CPU

**Causas:**
- Loops infinitos de observação
- Monitoramento de arquivos em diretórios grandes
- Vazamentos de memória em hooks

**Soluções:**
```bash
# Verificar processos descontrolados
top -o cpu | grep claude

# Desabilitar aprendizado contínuo temporariamente
touch ~/.claude/homunculus/disabled

# Reiniciar Claude Code
# Cmd/Ctrl+Q e reabrir

# Verificar tamanho do arquivo de observação
du -sh ~/.claude/homunculus/*/
```

---

## Mensagens de Erro Comuns

### "EACCES: permission denied"

```bash
# Corrigir permissões de hooks
find ~/.claude/plugins -name "*.sh" -exec chmod +x {} \;

# Corrigir permissões do diretório de observações
chmod -R u+rwX,go+rX ~/.claude/homunculus
```

### "MODULE_NOT_FOUND"

```bash
# Instalar dependências do plugin
cd ~/.claude/plugins/cache/everything-claude-code
npm install

# Ou para instalação manual
cd ~/.claude/plugins/ecc
npm install
```

### "spawn UNKNOWN"

```bash
# Específico do Windows: Garantir que scripts usem quebras de linha corretas
# Converter CRLF para LF
find ~/.claude/plugins -name "*.sh" -exec dos2unix {} \;

# Ou instalar dos2unix
# macOS: brew install dos2unix
# Ubuntu: sudo apt install dos2unix
```

---

## Obter Ajuda

Se você ainda estiver enfrentando problemas:

1. **Verificar Issues do GitHub**: [github.com/affaan-m/everything-claude-code/issues](https://github.com/affaan-m/everything-claude-code/issues)
2. **Habilitar Logs de Debug**:
   ```bash
   export CLAUDE_DEBUG=1
   export CLAUDE_LOG_LEVEL=debug
   ```
3. **Coletar Informações de Diagnóstico**:
   ```bash
   claude --version
   node --version
   python3 --version
   echo $CLAUDE_PACKAGE_MANAGER
   ls -la ~/.claude/plugins/cache/
   ```
4. **Abrir uma Issue**: Incluir logs de debug, mensagens de erro e informações de diagnóstico

---

## Documentação Relacionada

- [README.md](./README.md) - Instalação e funcionalidades
- [CONTRIBUTING.md](./CONTRIBUTING.md) - Diretrizes de desenvolvimento
- [docs/](./docs/) - Documentação detalhada
- [examples/](./examples/) - Exemplos de uso
