
# Guia de Solução de Problemas

Problemas comuns e soluções para o plugin Everything Claude Code (ECC).

## Tabela de Conteúdos

- [Problemas de Memória e Contexto](#problemas-de-memória--contexto)
- [Falhas no Arnês do Agente](#falhas-no-arnês-do-agente)
- [Erros de Hook e Fluxo de Trabalho](#erros-de-hook--fluxo-de-trabalho)
- [Instalação e Configuração](#instalação--configuração)
- [Problemas de Desempenho](#problemas-de-desempenho)
- [Mensagens de Erro Comuns](#mensagens-de-erro-comuns)
- [Obtendo Ajuda](#obtendo-ajuda)

---

## Problemas de Memória e Contexto

### Excesso de Janela de Contexto

**Sintoma:** Erros de "Contexto muito longo" ou respostas incompletas

**Causas:**
- Upload de arquivos grandes excedendo os limites de token
- Histórico de conversas acumulado
- Múltiplas saídas de ferramentas grandes em uma única sessão

**Soluções:**
```bash
# 1. Limpe o histórico de conversas e comece de novo
# Use o Claude Code: "Novo Chat" ou Cmd/Ctrl+Shift+N

# 2. Reduza o tamanho do arquivo antes da análise
head -n 100 arquivo-grande.log > amostra.log

# 3. Use streaming para saídas grandes
head -n 50 arquivo-grande.txt

# 4. Divida as tarefas em partes menores
# Em vez de: "Analisar todos os 50 arquivos"
# Use: "Analisar arquivos no diretório src/components/"
```

### Falhas na Persistência da Memória

**Sintoma:** O agente não se lembra do contexto ou das observações anteriores

**Causas:**
- Hooks de aprendizado contínuo desativados
- Arquivos de observação corrompidos
- Falhas na detecção de projetos

**Soluções:**
```bash
# Verifique se as observações estão sendo gravadas
ls ~/.claude/homunculus/projects/*/observations.jsonl

# Encontre o id de hash do projeto atual
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

# Veja as observações recentes para esse projeto
tail -20 ~/.claude/homunculus/projects/<hash-do-projeto>/observations.jsonl

# Faça backup de um arquivo de observações corrompido antes de recriá-lo
mv ~/.claude/homunculus/projects/<hash-do-projeto>/observations.jsonl 
  ~/.claude/homunculus/projects/<hash-do-projeto>/observations.jsonl.bak.$(date +%Y%m%d-%H%M%S)

# Verifique se os hooks estão habilitados
grep -r "observe" ~/.claude/settings.json
```

---

## Falhas no Arnês do Agente

### Agente Não Encontrado

**Sintoma:** Erros de "Agente não carregado" ou "Agente desconhecido"

**Causas:**
- Plugin não instalado corretamente
- Má configuração do caminho do agente
- Incompatibilidade de instalação do marketplace vs. manual

**Soluções:**
```bash
# Verifique a instalação do plugin
ls ~/.claude/plugins/cache/

# Verifique se o agente existe (instalação do marketplace)
ls ~/.claude/plugins/cache/*/agents/

# Para instalação manual, os agentes devem estar em:
ls ~/.claude/agents/  # Apenas agentes personalizados

# Recarregue o plugin
# Claude Code → Configurações → Extensões → Recarregar
```

### Execução do Fluxo de Trabalho Travada

**Sintoma:** O agente inicia, mas nunca termina

**Causas:**
- Loops infinitos na lógica do agente
- Bloqueado na entrada do usuário
- Tempo de espera da rede aguardando a API

**Soluções:**
```bash
# 1. Verifique se há processos travados
ps aux | grep claude

# 2. Habilite o modo de depuração
export CLAUDE_DEBUG=1

# 3. Defina tempos de espera mais curtos
export CLAUDE_TIMEOUT=30

# 4. Verifique a conectividade da rede
curl -I https://api.anthropic.com
```

### Erros de Uso da Ferramenta

**Sintoma:** "Falha na execução da ferramenta" ou permissão negada

**Causas:**
- Dependências ausentes (npm, python, etc.)
- Permissões de arquivo insuficientes
- Caminho não encontrado

**Soluções:**
```bash
# Verifique se as ferramentas necessárias estão instaladas
which node python3 npm git

# Corrija as permissões nos scripts de hook
chmod +x ~/.claude/plugins/cache/*/hooks/*.sh
chmod +x ~/.claude/plugins/cache/*/skills/*/hooks/*.sh

# Verifique se o PATH inclui os binários necessários
echo $PATH
```

---

## Erros de Hook e Fluxo de Trabalho

### Hooks Não Disparando

**Sintoma:** Hooks pré/pós não executam

**Causas:**
- Hooks não registrados em settings.json
- Sintaxe de hook inválida
- Script de hook não executável

**Soluções:**
```bash
# Verifique se os hooks estão registrados
grep -A 10 '"hooks"' ~/.claude/settings.json

# Verifique se os arquivos de hook existem e são executáveis
ls -la ~/.claude/plugins/cache/*/hooks/

# Teste o hook manualmente
bash ~/.claude/plugins/cache/*/hooks/pre-bash.sh <<< '{"command":"echo test"}'

# Registre novamente os hooks (se estiver usando plugin)
# Desabilite e reabilite o plugin nas configurações do Claude Code
```

### Incompatibilidades de Versão do Python/Node

**Sintoma:** "python3 não encontrado" ou "node: comando não encontrado"

**Causas:**
- Instalação ausente do Python/Node
- PATH não configurado
- Versão errada do Python (Windows)

**Soluções:**
```bash
# Instale o Python 3 (se ausente)
# macOS: brew install python3
# Ubuntu: sudo apt install python3
# Windows: Baixe em python.org

# Instale o Node.js (se ausente)
# macOS: brew install node
# Ubuntu: sudo apt install nodejs npm
# Windows: Baixe em nodejs.org

# Verifique as instalações
python3 --version
node --version
npm --version

# Windows: Certifique-se de que python (não python3) funciona
python --version
```

### Falsos Positivos do Bloqueador do Servidor de Desenvolvimento

**Sintoma:** O hook bloqueia comandos legítimos que mencionam "dev"

**Causas:**
- O conteúdo do Heredoc aciona a correspondência de padrões
- Comandos não-dev com "dev" nos argumentos

**Soluções:**
```bash
# Isso foi corrigido na v1.8.0+ (PR #371)
# Atualize o plugin para a versão mais recente

# Solução alternativa: Envolva os servidores de desenvolvimento no tmux
tmux new-session -d -s dev "npm run dev"
tmux attach -t dev

# Desabilite o hook temporariamente, se necessário
# Edite ~/.claude/settings.json e remova o hook pre-bash
```

---

## Instalação e Configuração

### Plugin Não Carregando

**Sintoma:** Recursos do plugin indisponíveis após a instalação

**Causas:**
- Cache do marketplace não atualizado
- Incompatibilidade de versão do Claude Code
- Arquivos de plugin corrompidos

**Soluções:**
```bash
# Inspecione o cache do plugin antes de alterá-lo
ls -la ~/.claude/plugins/cache/

# Faça backup do cache do plugin em vez de excluí-lo no local
mv ~/.claude/plugins/cache ~/.claude/plugins/cache.backup.$(date +%Y%m%d-%H%M%S)
mkdir -p ~/.claude/plugins/cache

# Reinstale a partir do marketplace
# Claude Code → Extensões → Everything Claude Code → Desinstalar
# Em seguida, reinstale a partir do marketplace

# Verifique a versão do Claude Code
claude --version
# Requer Claude Code 2.0+

# Instalação manual (se o marketplace falhar)
git clone https://github.com/affaan-m/everything-claude-code.git
cp -r everything-claude-code ~/.claude/plugins/ecc
```

### Falha na Detecção do Gerenciador de Pacotes

**Sintoma:** Gerenciador de pacotes errado usado (npm em vez de pnpm)

**Causas:**
- Nenhum arquivo de bloqueio presente
- CLAUDE_PACKAGE_MANAGER não definido
- Vários arquivos de bloqueio confundindo a detecção

**Soluções:**
```bash
# Defina o gerenciador de pacotes preferido globalmente
export CLAUDE_PACKAGE_MANAGER=pnpm
# Adicione a ~/.bashrc ou ~/.zshrc

# Ou defina por projeto
echo '{"packageManager": "pnpm"}' > .claude/package-manager.json

# Ou use o campo package.json
npm pkg set packageManager="pnpm@8.15.0"

# Aviso: remover arquivos de bloqueio pode alterar as versões das dependências instaladas.
# Comite ou faça backup do arquivo de bloqueio primeiro, depois execute uma nova instalação e execute novamente o CI.
# Faça isso apenas ao trocar intencionalmente de gerenciador de pacotes.
rm package-lock.json  # Se estiver usando pnpm/yarn/bun
```

---

## Problemas de Desempenho

### Tempos de Resposta Lentos

**Sintoma:** O agente leva mais de 30 segundos para responder

**Causas:**
- Arquivos de observação grandes
- Muitos hooks ativos
- Latência de rede para a API

**Soluções:**
```bash
# Arquive observações grandes em vez de excluí-las
archive_dir="$HOME/.claude/homunculus/archive/$(date +%Y%m%d)"
mkdir -p "$archive_dir"
find ~/.claude/homunculus/projects -name "observations.jsonl" -size +10M -exec sh -c '
  for file do
    base=$(basename "$(dirname "$file")")
    gzip -c "$file" > "'"$archive_dir"'/${base}-observations.jsonl.gz"
    : > "$file"
  done
' sh {} +

# Desabilite hooks não utilizados temporariamente
# Edite ~/.claude/settings.json

# Mantenha os arquivos de observação ativos pequenos
# Arquivos grandes devem ficar em ~/.claude/homunculus/archive/
```

### Alto Uso da CPU

**Sintoma:** O Claude Code consumindo 100% da CPU

**Causas:**
- Loops de observação infinitos
- Observação de arquivos em diretórios grandes
- Vazamentos de memória em hooks

**Soluções:**
```bash
# Verifique se há processos descontrolados
top -o cpu | grep claude

# Desabilite o aprendizado contínuo temporariamente
touch ~/.claude/homunculus/disabled

# Reinicie o Claude Code
# Cmd/Ctrl+Q e reabra

# Verifique o tamanho do arquivo de observação
du -sh ~/.claude/homunculus/*/
```

---

## Mensagens de Erro Comuns

### "EACCES: permissão negada"

```bash
# Corrija as permissões do hook
find ~/.claude/plugins -name "*.sh" -exec chmod +x {} \;

# Corrija as permissões do diretório de observação
chmod -R u+rwX,go+rX ~/.claude/homunculus
```

### "MODULE_NOT_FOUND"

```bash
# Instale as dependências do plugin
cd ~/.claude/plugins/cache/everything-claude-code
npm install

# Ou para instalação manual
cd ~/.claude/plugins/ecc
npm install
```

### "spawn UNKNOWN"

```bash
# Específico do Windows: Certifique-se de que os scripts usem as terminações de linha corretas
# Converta CRLF para LF
find ~/.claude/plugins -name "*.sh" -exec dos2unix {} \;

# Ou instale o dos2unix
# macOS: brew install dos2unix
# Ubuntu: sudo apt install dos2unix
```

---

## Obtendo Ajuda

 Se você ainda estiver com problemas:

1. **Verifique as Issues do GitHub**: [github.com/affaan-m/everything-claude-code/issues](https://github.com/affaan-m/everything-claude-code/issues)
2. **Habilite o Logging de Depuração**:
   ```bash
   export CLAUDE_DEBUG=1
   export CLAUDE_LOG_LEVEL=debug
   ```
3. **Colete Informações de Diagnóstico**:
   ```bash
   claude --version
   node --version
   python3 --version
   echo $CLAUDE_PACKAGE_MANAGER
   ls -la ~/.claude/plugins/cache/
   ```
4. **Abra uma Issue**: Inclua logs de depuração, mensagens de erro e informações de diagnóstico

---

## Documentação Relacionada

- [README.md](./README.md) - Instalação e recursos
- [CONTRIBUTING.md](./CONTRIBUTING.md) - Diretrizes de desenvolvimento
- [docs/](./docs/) - Documentação detalhada
- [examples/](./examples/) - Exemplos de uso
