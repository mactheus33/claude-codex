---
name: codex
description: Use esta skill quando o usuário invocar "/codex", ou pedir para "encerrar a sessão", "fechar", "salvar handoff", "atualizar memória do projeto", "compactar memória", "escrever resumo de sessão", ou qualquer frase indicando que a sessão de trabalho deve ser encerrada. Lê a conversa atual, compacta o handoff.md e aprendizados.md do projeto ativo em digests rolantes concisos, e escreve apenas após aprovação do usuário.
version: 0.2.0
---

# Codex — Rolling Digest

Encerra uma sessão de trabalho do Claude Code compactando a memória do projeto em um digest pequeno, atual e carregando peso. Previne os arquivos de `memory/` de crescerem sem limite, preservando lições que ainda importam.

## Filosofia

Memória que só cresce vira ruído. Esta skill aplica **rolling digest**: a cada fim de sessão, a memória é relida, mesclada com o trabalho de hoje e reescrita concisa. Três princípios:

1. **Pequeno** — tamanho limitado, cabe no contexto sem queimar tokens.
2. **Atual** — reflete estado mais recente, não histórico. Histórico vive no git.
3. **Carrega peso** — toda linha justifica o lugar evitando um erro futuro ou esclarecendo estado atual.

## Arquivos gerenciados

Relativos à raiz do projeto ativo:

- **`memory/handoff.md`** — estado atual. Sempre sobrescrito.
- **`memory/aprendizados.md`** — lições curadas. Sempre sobrescrito, depois de mesclar antigas + novas.

A auto-memória global (`~/.claude/projects/.../MEMORY.md`) NÃO é tocada por esta skill — ela tem disciplina própria.

## Workflow

Seguir os passos na ordem quando a skill for acionada.

### Passo 1: Confirmar escopo do projeto

Identificar o projeto ativo a partir do diretório de trabalho atual e do contexto da conversa. Se ambíguo, ou se o diretório não tem pasta `memory/` e criar uma não é obviamente correto, pedir para o usuário confirmar projeto e caminho antes de prosseguir. Não assumir.

### Passo 2: Ler contexto da sessão atual

Escanear a conversa em busca de:
- O que foi entregue (estados finais, testes passando, mudanças deployadas).
- Lições não-óbvias aprendidas — técnicas, de negócio, de processo.
- Evoluções estruturais (mudanças de stack, decisões de arquitetura, novas integrações).
- O que está em progresso, o que vem a seguir, o que está bloqueado.

### Passo 3: Carregar memória existente

Ler (se existir):
- `<raiz-do-projeto>/memory/handoff.md`
- `<raiz-do-projeto>/memory/aprendizados.md`

Se algum deles não existir, tratar como vazio e planejar criar.

### Passo 4: Compor novo handoff.md

handoff.md sempre é sobrescrito. Usar exatamente esta estrutura:

```markdown
# Sessão YYYY-MM-DD — <nome-do-projeto>

## Handoff
- <1 linha: o que foi feito, estado final>

## Próximos passos
- <bullets, acionáveis, priorizados>

## Bloqueios
- <somente se houver bloqueios — omitir a seção inteira caso contrário>
```

Regras:
- Uma data por arquivo. Conteúdo anterior do handoff é substituído, não anexado.
- Sem enchimento. Se "Próximos passos" estiver vazio, escrever "Nenhum.". Não inventar itens.
- Linguagem direta, declarativa. Sem hedge.

### Passo 5: Compor novo aprendizados.md (digest)

aprendizados.md também é sobrescrito, mas apenas depois de mesclar antigos + novos.

**Manter dos aprendizados existentes:**
- Não-óbvios E ainda acionáveis.
- Lições que, se esquecidas, causariam repetição de erro.

**Descartar dos aprendizados existentes:**
- Bug fixes pontuais já refletidos no código.
- Lições que viraram convenção do código (vivem no código agora).
- Qualquer coisa genérica já coberta em arquivos de identidade global (ex: `~/.claude/soul.md`, CLAUDE.md).
- Itens duplicados — mesclar em um.

**Adicionar do que veio da sessão de hoje:**
- Aprendizados não-óbvios novos.

**Cap:** ~150 linhas no total. Se exceder, comprimir mais mesclando itens similares.

Formato:

```markdown
# Aprendizados — <nome-do-projeto>

## <Categoria>
- <lição direta de 1 linha>. Por quê: <opcional, mas útil para itens não-óbvios>.
```

Categorias emergem organicamente do conteúdo (ex: Segurança, Infra, Negócio, Performance). Não criar categorias vazias.

### Passo 6: Mostrar drafts, pedir aprovação

Antes de escrever, exibir ao usuário:

1. O `handoff.md` proposto (conteúdo completo).
2. O `aprendizados.md` proposto (conteúdo completo).
3. Resumo do diff: "Mantidos N itens dos aprendizados existentes, descartados M, adicionados K novos."

Perguntar explicitamente: "Aprova para gravar?"

### Passo 7: Gravar apenas após aprovação

- Aprovação: gravar ambos os arquivos com a tool Write (sobrescrever).
- Redirecionamento: revisar com base no feedback do usuário, mostrar de novo. Não gravar até aprovação explícita.
- Rejeição: parar. Não gravar parcialmente.

## Regras

- **Um projeto por invocação.** Nunca gravar em memória de múltiplos projetos numa execução.
- **Nunca anexar.** Sempre ler → mesclar → sobrescrever.
- **Sem fabricação.** Se a sessão não produziu aprendizado, deixar o aprendizados.md intacto. Vazio é melhor que inventado.
- **Aprendizados cross-project** vão para `memory/incoming-learnings.md` do projeto destino, nunca direto edit.
- **Idioma:** escrever os arquivos de memória em PT-BR. Headers, estrutura e exemplos em PT-BR independente da língua de trabalho da sessão (a comunicação técnica com Claude Code/SDK/MCP segue em EN naturalmente).

## Exemplo de saída

### handoff.md

```markdown
# Sessão 2026-04-17 — viabox

## Handoff
- Validação dos testes e2e para features de upload do MVP. 15/15 passaram.

## Próximos passos
- Deploy em staging.
- Agendar revisão de segurança com auditor externo.

## Bloqueios
- Aguardando API key do provedor de auth third-party.
```

### aprendizados.md

```markdown
# Aprendizados — viabox

## Segurança
- Bypass de file size no pentest de upload. Corrigido via max upload sizing. Por quê: atacante conseguia burlar o check de MIME spoofando a extensão num arquivo que excedia checks de content-type só depois do estágio de descompressão.

## Infra
- Hetzner CX33 chega a 85% CPU em rebuilds Docker. Por quê: instância sizeada para steady-state, não burst.

## Negócio
- Conversão da landing page +40% quando o campo telefone vira opcional. Por quê: leads de real estate resistem a dar telefone de cara.
```
