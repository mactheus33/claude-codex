---
name: codex
description: Use esta skill quando o usuário invocar "/codex", ou pedir para "encerrar a sessão", "fechar", "salvar handoff", "atualizar memória do projeto", "compactar memória", "escrever resumo de sessão", "limpar memory", ou qualquer frase indicando que a sessão deve ser encerrada. Faz faxina completa do memory/ do projeto, sintetiza conteúdo (regras → mestre, lições → aprendizados, notas granulares → changelog), sobrescreve handoff e atualiza documento_mestre cirurgicamente. Escreve apenas após aprovação.
version: 0.6.0
---

# Codex — Faxineiro de Sessão + Rolling Digest

Encerra uma sessão de trabalho fazendo **faxina completa** do `memory/` do projeto. Não acumula lixo: tudo que foi criado durante a sessão é triado, promovido para o lugar certo na camada permanente (raiz do projeto) ou descartado. O `changelog.md` é o backup final — nada se perde.

## Filosofia

Memória que só cresce vira ruído. Esta skill aplica **faxina por sessão**:

1. **Pequeno e curado** — `documento_mestre.md`, `aprendizados.md`, `handoff.md` ficam pequenos e atualizados.
2. **Histórico preservado** — `changelog.md` é append-only, nunca apaga, capta tudo que sai dos outros arquivos.
3. **memory/ vira espaço de rascunho** — agente cria livre durante a sessão; /codex faxina no fim.
4. **Promoção para a camada certa** — regras vão pro mestre, lições pro aprendizados, notas granulares pro changelog, fonte de verdade de tema grande pra `<tema>_mestre.md` na raiz.

## Estrutura de duas camadas

```
<projeto>/
├── CLAUDE.md                # 0. institucional + arquitetura
├── documento_mestre.md      # 1. PERMANENTE: status + pendências + regras (BOOTSTRAP)
├── aprendizados.md          # 2. PERMANENTE: lições curadas (BOOTSTRAP)
├── changelog.md             # 3. PERMANENTE: histórico append-only (NÃO bootstrap)
├── <tema>_mestre.md         # PERMANENTE: satélites do mestre (lidos sob demanda)
└── memory/
    ├── handoff.md           # TRANSIENTE: estado última sessão (BOOTSTRAP)
    └── *.md                 # TRANSIENTE: rascunhos da sessão (alvo da faxina)
```

## Workflow

### Passo 1 — Confirmar projeto

Identificar o projeto ativo a partir do diretório de trabalho. Se ambíguo, pedir confirmação. Se `<projeto>/documento_mestre.md` não existir, parar e sugerir bootstrap via templates em `~/.claude/shared/memory-template/raiz/`.

### Passo 2 — Ler camada permanente

Ler em ordem (todos da raiz do projeto):
- `<projeto>/documento_mestre.md`
- `<projeto>/aprendizados.md`
- `<projeto>/memory/handoff.md`

`<projeto>/changelog.md` **não** é lido automaticamente — só se a sessão pediu busca histórica.
`<projeto>/<tema>_mestre.md` **não** são lidos automaticamente — só os relevantes ao trabalho da sessão.

### Passo 3 — Escanear memory/

Listar todos os arquivos em `<projeto>/memory/` (exceto `handoff.md`). Para cada um, identificar:
- **Origem temporal:** criado/modificado nesta sessão? Em sessão anterior?
- **Tipo de conteúdo:** rascunho, decisão, gotcha técnico, status update, registro de ação
- **Relevância atual:** ainda válido? Já obsoleto?

### Passo 4 — Ler conversa da sessão

Escanear a conversa em busca de:
- O que foi entregue (estados finais, testes passando, mudanças deployadas).
- Decisões tomadas (pequenas e grandes).
- Lições não-óbvias aprendidas.
- Regras permanentes descobertas ou ratificadas.
- Pendências resolvidas (do mestre) e novas.
- Notas granulares do tipo "tentamos X, voltamos pra Y porque Z".

### Passo 5 — Triagem do memory/

Para cada arquivo em memory/ (exceto handoff.md), classificar e propor destino:

| Conteúdo | Destino | Operação |
|---|---|---|
| Regra permanente "sempre/nunca" | `documento_mestre.md` (seção "Regras permanentes") | Promover + apagar satélite |
| Lição genérica reutilizável | `aprendizados.md` (categoria correspondente) | Consolidar + apagar satélite |
| Fonte de verdade de tema grande (50+ linhas, persistente) | Promover para `<tema>_mestre.md` na **raiz** do projeto, com header `> Documento pai: documento_mestre.md` + ponteiro do mestre | Mover + apagar satélite |
| Nota granular histórica ("decidimos X em Y porque Z") | Registrar no `changelog.md` | Append + apagar satélite |
| Lixo (rascunho, redundância, decisão revertida) | `memory/arquivo/` (retenção 90d) ou apagar direto | Mover ou apagar |

**Regra crítica:** antes de apagar qualquer arquivo, garantir que o conteúdo foi capturado em pelo menos um destino acima. Lixo só é deletado direto se for verificadamente redundante (ex: rascunho de algo que já está no mestre).

### Passo 6 — Compor entrada do changelog.md

`changelog.md` é **append-only** — adicionar entrada no final do arquivo, NUNCA editar entradas anteriores.

Estrutura da entrada:

```markdown
## YYYY-MM-DD — <título da sessão>

### Handoff anterior (substituído)
<conteúdo integral do handoff.md que vai ser sobrescrito>

### Decisões / mudanças
- <decisão tomada nesta sessão>
- <mudança implementada com link para PR/commit>

### Notas granulares
- <conteúdo dos satélites apagados>
- <decisões intermediárias do tipo "tentamos X, voltamos pra Y">
- <contexto de relacionamento, status atualizado de cliente, etc>

### Lições promovidas
- Para `documento_mestre.md`: <regras que subiram>
- Para `aprendizados.md`: <lições que viraram parte do digest>
- Para `<tema>_mestre.md`: <se algum satélite na raiz foi criado/atualizado>
```

### Passo 7 — Compor novo handoff.md

Sobrescrever `memory/handoff.md` com sessão atual. Cap <50 linhas. Estrutura:

```markdown
# Sessão YYYY-MM-DD — <nome-do-projeto>

## Handoff
- <1 linha: o que foi feito, estado final>

## Próximos passos
- <bullets acionáveis, priorizados>

## Bloqueios
- <só se houver — omitir seção senão>
```

### Passo 8 — Atualizar aprendizados.md

`aprendizados.md` (raiz) é a memória curada persistente, lida no bootstrap. Cap <150 linhas.

Operações:
- **Adicionar** lições novas no topo da categoria correspondente (Segurança / Infra / Negócio / Frontend / Process / etc).
- **Atualizar sumário** de "Áreas cobertas" no topo se houve mudança de categoria.
- **Remover** lições que viraram regra permanente (subiram pro mestre).
- **Não inflar com nuance granular** — isso vai pro changelog.

Se passar de 150 linhas após edição, sugerir rodar `/otimizar-projeto` na próxima sessão para compactação.

### Passo 9 — Atualizar documento_mestre.md cirurgicamente

Não sobrescrever — aplicar mudanças cirúrgicas:
- **Header:** atualizar "Última atualização" para hoje.
- **Status atual:** 1-3 frases sobre estado pós-sessão.
- **Pendências:** marcar resolvidas com `~~[x]~~ → ver changelog YYYY-MM-DD`; adicionar novas com `[ ]`.
- **Próximos passos:** atualizar conforme handoff.
- **Regras permanentes:** adicionar regras promovidas nesta sessão.
- **Decisões-chave:** adicionar decisões estruturais novas.
- **Satélites:** se algum `<tema>_mestre.md` foi criado/atualizado, atualizar o ponteiro.

**Health check:**

| Linhas do mestre | Ação |
|---|---|
| <300 | 🟢 |
| 300-500 | 🟡 sugerir `/otimizar-projeto` na próxima sessão |
| >500 | 🔴 sugerir `/otimizar-projeto` agora antes de continuar |

### Passo 10 — Mostrar drafts e pedir aprovação

Antes de gravar **qualquer coisa**, exibir ao usuário:

1. **Lista de operações no memory/** — para cada arquivo: ação proposta (promover/consolidar/registrar/apagar) + destino + 1 linha de justificativa
2. **Entrada do changelog.md** completa
3. **Novo handoff.md** completo
4. **Diff do aprendizados.md** (linhas adicionadas/removidas)
5. **Diff do documento_mestre.md** (mudanças cirúrgicas)
6. **Resumo numérico:** "Triados N satélites: K promovidos, L consolidados, M registrados no changelog, P apagados. Mestre agora com X linhas (🟢/🟡/🔴)."

Perguntar: **"Aprova para gravar?"**

### Passo 11 — Gravar apenas após aprovação

- **Aprovação total:** gravar todos os arquivos. Apagar satélites de memory/ na ordem após registrar no changelog.
- **Aprovação parcial:** ("aprovo 1, 3, 5 mas não 2"): só executar aprovados. Mostrar plano restante.
- **Redirecionamento:** revisar conforme feedback, mostrar de novo. Não gravar até aprovação explícita.
- **Rejeição:** parar. Não gravar parcialmente.

Se mestre passou para 🔴 (>500 linhas), terminar com:
> "documento_mestre.md está em [N] linhas (🔴). Recomendo rodar `/otimizar-projeto` agora para split em satélites antes de continuar."

### Passo 12 — Commit + push em dev

Após gravação aprovada (Passo 11), fechar a sessão no Git. Resolve o empilhamento de mudanças locais não pushadas entre sessões.

#### Detecção de escopo

Antes de qualquer commit, **detectar quantos repos estão dirty** e classificar:

1. **Repo do projeto ativo** — sempre o primeiro candidato. `git status` lá.
2. **Outros repos da mesma empresa** — descobrir via frontmatter `empresa: <slug>` no `<projeto>/CLAUDE.md` (convenção em `~/.claude/shared/memory-convention.md`). Escanear `~/projetos-claude/*/CLAUDE.md`, filtrar pelo slug da empresa ativa, rodar `git status` em cada.
3. **Cross-empresa** — se `git status` revelar dirty em repo de outra empresa, **bloquear**: "Detectei dirty em `<repo>` que pertence a `<outra-empresa>`. /codex não cruza empresas. Resolva manualmente."
4. **Meta-projeto (`~/.claude/`)** — caso especial. Se sessão é modo claude-identity, ele é o repo "ativo" e sai pelo fluxo padrão (sem multi-repo). Se sessão é de projeto e `~/.claude/` aparece dirty, isso é cross-escopo (não cross-empresa) — pausar e pedir orientação ao usuário.

#### Branch enforcement

Para **cada** repo a tocar (projeto ativo + repos da mesma empresa):
- `git branch --show-current` em cada
- Se NÃO for `dev`, **bloquear** apenas aquele repo: "Repo `<nome>` está em `<branch>`. /codex commita apenas em `dev`. Esse repo fica fora — resolva manualmente. Sigo com os outros?"
- Nunca trocar branch automaticamente. Nunca commitar em `main`.

#### Modo single-repo (escopo padrão)

Apenas o repo do projeto ativo está dirty. Fluxo simples:

1. **Mostrar diff resumido.** `git status` + lista de arquivos. Diff completo só se usuário pedir.

2. **Compor mensagem de commit** a partir do handoff e do changelog desta sessão:
   - **Título:** `chore(memory): <título da sessão do handoff>` — mesmo título usado no changelog (Passo 6).
   - **Body:** os 5 primeiros bullets da seção "Decisões / mudanças" da entrada do changelog. Se houver menos, listar todos. Se houver mais, truncar a 5 e omitir o resto.

3. **Pedir aprovação explícita:**
   > "Vou commitar TUDO acima em `dev` e dar push com a mensagem:
   > ```
   > <título>
   >
   > <body>
   > ```
   > Aprova?"

   Default seguro: sem "sim" explícito, **não commita**.

4. **Após aprovação:**
   - `git add -A`
   - `git commit -m "<título>" -m "<body>"`
   - `git push` — se branch sem upstream, `git push -u origin dev`

5. **Confirmar resultado:** SHA curto + branch remoto.

#### Modo multi-repo (mesma empresa)

Múltiplos repos da mesma empresa estão dirty (ex: sessão tocou `projeto-fullcred` E `fullcred-institucional`). Fluxo expandido:

1. **Listar todos os repos dirty da empresa**, com `git status --short` resumido em cada.

2. **Classificar cada repo por origem do dirty:**
   - **Editado nesta sessão** — recebe commit normal com mensagem do changelog/handoff.
   - **Dirty drift pré-existente** — recebe commit de saneamento com mensagem clara ("chore(saneamento): commit de dirty drift detectado em `/codex` da sessão YYYY-MM-DD" + lista dos arquivos). Cobertura da exceção cirúrgica do CLAUDE.md global.

3. **Compor mensagem por repo:**
   - Repos editados na sessão: mesmo formato do single-repo (título + 5 bullets), mas **filtrar bullets relevantes àquele repo** quando possível (se a entrada do changelog tem decisões que tocam só um repo, a mensagem dele só leva esses bullets).
   - Repos com dirty drift: título "chore(saneamento): dirty drift detectado em bootstrap" + body listando os arquivos commitados.

4. **Pedir aprovação consolidada:**
   > "Vou commitar **N repos** da empresa `<slug>` em sequência:
   >
   > **Repo 1: `<nome>`** (editado nesta sessão)
   > Mensagem: `<título>`
   > <body>
   >
   > **Repo 2: `<nome>`** (dirty drift, saneamento)
   > Mensagem: `<título saneamento>`
   > <body>
   >
   > Aprova todos? (ou diga quais aprovar — `aprovo 1, 3` / `só 1` / `cancela tudo`)"

5. **Após aprovação:** loop sequencial, um repo por vez:
   - `cd <repo>`
   - `git add -A`
   - `git commit -m "<título>" -m "<body>"`
   - `git push` (ou `push -u origin dev` se sem upstream)
   - Reportar SHA curto + nome do repo
   - Se algum push falhar: parar o loop, reportar qual repo, perguntar se segue ou pausa. Nunca force push.

6. **Confirmar resultado consolidado:** lista final de SHAs por repo.

#### Aprovação parcial / rejeição

- **Aprovação parcial:** ("aprovo 1, 3 mas não 2") → executar só os aprovados, reportar os pulados como pendentes pra próxima sessão.
- **Rejeição total:** parar. Working tree fica como está. Reportar como pendência no próximo bootstrap.

## Regras invioláveis

1. **Um projeto ativo por invocação para edição de memória.** Triagem de `memory/`, edição de mestre/aprendizados/handoff/changelog acontecem **apenas no projeto ativo da sessão**. Nunca gravar em memória de outro projeto. (Exceção do Passo 12 é apenas commit/push de dirty pré-existente, sem editar conteúdo.)
2. **`changelog.md` é append-only.** NUNCA editar entradas antigas. NUNCA apagar.
3. **`<tema>_mestre.md` na raiz não são tocados.** Apenas `/otimizar-projeto` faz split do mestre criando-os; depois disso, são responsabilidade humana ou de `/otimizar-projeto`.
4. **Apagar satélite só depois de capturar em changelog ou outro destino.** Lixo verificadamente redundante pode ir direto.
5. **Triagem sempre passa por aprovação.** Sem aprovação, não apaga, não grava.
6. **Anti-fragmentação na criação de `<tema>_mestre.md`.** Promover satélite ad-hoc para a raiz só quando o conteúdo é fonte de verdade de um tema grande (50+ linhas, escopo único, persistente). Não promover por impulso.
7. **Aprendizados cross-project** vão para `<projeto-destino>/memory/incoming-learnings.md`, não direto edit.
8. **Idioma:** todos os arquivos de memória em PT-BR. Comunicação técnica com Claude Code/SDK/MCP/docs Anthropic em EN naturalmente.
9. **Commit final em `dev`, sempre com aprovação explícita.** /codex nunca commita em `main`. Nunca commita sem mostrar `git status` e pedir "aprova?". Sem flag de auto-commit. Sem force push. **Multi-repo da mesma empresa permitido** (Passo 12 modo multi-repo) — cross-empresa **bloqueia**.

## Exemplo de saída

### Entrada appendada ao changelog.md

```markdown
## 2026-04-29 — Refactor para nova convenção de memória

### Handoff anterior (substituído)
# Sessão 2026-04-27 — projeto-fullcred
## Handoff
- PR #32 mergeado (v1.2.3): system metrics dashboard...
[...]

### Decisões / mudanças
- Migrado mestre + aprendizados para a raiz do projeto
- Criado changelog.md (este arquivo) — append-only
- Triados 39 satélites legacy do memory/ (ver "Notas granulares" abaixo)

### Notas granulares
- project_app_jsx_refactor_pending.md → conteúdo já refletido na pendência "App.jsx routing refactor" do mestre, satélite apagado
- feedback_no_auto_git.md → regra já promovida no mestre seção "Regras permanentes > Operacional", satélite apagado
[...39 entradas similares...]

### Lições promovidas
- Para documento_mestre.md: nenhuma (todas já estavam)
- Para aprendizados.md: "memory/ é working space transiente, nunca persistir info crítica lá"
```

### Novo handoff.md

```markdown
# Sessão 2026-04-29 — projeto-fullcred

## Handoff
- Refactor concluído: convenção de duas camadas aplicada (raiz permanente + memory/ transiente). 39 satélites legacy triados.

## Próximos passos
- ZIP download de documentos
- Histórico do cliente na proposta
- Validar checkbox 4×4 da landing page
```

### Diff cirúrgico do documento_mestre.md

```diff
 # projeto-fullcred — Documento Mestre
-> Última atualização: 27/04/2026
+> Última atualização: 29/04/2026
 > Status: produção (v1.2.3)

 ## Status atual
-v1.2.3 em produção. Mason audit cycle ativo. LP rewrite concluído.
+v1.2.3 em produção. Refactor de memória concluído (29/04). Mason audit cycle ativo.

 ## Satélites
+- `<tema>_mestre.md`: nenhum criado nesta sessão.
```
