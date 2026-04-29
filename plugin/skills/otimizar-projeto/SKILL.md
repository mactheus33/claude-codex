---
name: otimizar-projeto
description: Use esta skill quando o usuário invocar "/otimizar-projeto", ou pedir para "otimizar memória do projeto", "split do mestre", "compactar aprendizados", "limpar memory antigo", "higiene do projeto". Operações pesadas de manutenção da memória do projeto: split do documento_mestre quando >300 linhas, compactação do aprendizados quando >150 linhas, scan e arquivamento de arquivos antigos no memory/. Plano antes de execução. NUNCA toca em changelog.md.
version: 0.5.0
---

# Otimizar Projeto — Higiene Macro da Memória

Faz manutenção pesada de médio/longo prazo na memória do projeto. Complementa o `/codex` (que faz faxina por sessão). Esta skill é invocada **sob demanda** quando arquivos crescem além dos limites e precisam de operação estrutural (split, compactação, arquivamento).

## Filosofia

`/codex` mantém digest curto a cada sessão. Esta skill faz manutenção de **médio prazo**: split do mestre quando ele cresce (criando `<tema>_mestre.md` na raiz), compactação do `aprendizados.md` quando passa de 150 linhas, arquivamento de conteúdo antigo no `memory/`, e atualização da estrutura quando a árvore do projeto evolui.

Quatro princípios:

1. **Plano antes de execução.** Sempre apresentar lista numerada por categoria, com aprovação por item ou em bloco.
2. **Nunca apaga.** Conteúdo obsoleto vai para `memory/arquivo/` (retenção 90d) — git preserva histórico se for revertido.
3. **Limites numéricos como gatilho.** Não otimizar por intuição — usar os semáforos da convenção.
4. **Anti-fragmentação.** Máximo 3 satélites novos por execução.

## Estrutura de duas camadas

```
<projeto>/
├── CLAUDE.md
├── documento_mestre.md      # OPERA: split em <tema>_mestre.md quando >300 linhas
├── aprendizados.md          # OPERA: compactação quando >150 linhas
├── changelog.md             # NUNCA TOCA — imutável total
├── <tema>_mestre.md         # OPERA: sub-split se >500 linhas
└── memory/
    ├── handoff.md           # NÃO TOCA (responsabilidade do /codex)
    ├── arquivo/             # destino do conteúdo arquivado
    └── *.md                 # OPERA: scan por idade, arquivamento
```

## Workflow

### Passo 1 — Confirmar projeto

Identificar o projeto ativo a partir do diretório de trabalho. Se ambíguo, pedir confirmação. Se `<projeto>/documento_mestre.md` não existir, parar e sugerir bootstrap via templates em `~/.claude/shared/memory-template/`.

### Passo 2 — Scan e relatório de saúde

Escanear todos os arquivos `.md` em `<projeto>/` (raiz) e `<projeto>/memory/`. Para cada arquivo coletar: nome, número de linhas, tamanho em KB, data da última modificação.

Apresentar tabela com semáforos:

```
Relatório de saúde — <projeto>
Data: DD/MM/AAAA

RAIZ
| Arquivo | Linhas | Tamanho | Modificado | Status |
|---|---|---|---|---|
| documento_mestre.md | 487 | 28KB | hoje | 🔴 grande |
| aprendizados.md | 178 | 12KB | hoje | 🟡 atenção |
| changelog.md | 2400 | 145KB | hoje | — (sem cap) |
| stripe_mestre.md | 240 | 15KB | DD/MM | 🟢 ok |

MEMORY/
| Arquivo | Linhas | Tamanho | Modificado | Status |
|---|---|---|---|---|
| handoff.md | 45 | 4KB | hoje | 🟢 ok |
| feedback_old_thing.md | 32 | 1KB | há 100 dias | 🔴 antigo |
```

**Regras de semáforo:**

| Arquivo | 🟢 OK | 🟡 Atenção | 🔴 Ação |
|---|---|---|---|
| `documento_mestre.md` | <300 linhas | 300-500 | >500 → split obrigatório |
| `<tema>_mestre.md` | <25KB / 500 linhas | 25-40KB | >40KB → sub-split |
| `aprendizados.md` | <150 linhas | 150-200 | >200 → compactar |
| `handoff.md` | — | — | (responsabilidade do /codex, ignorar) |
| `changelog.md` | — | — | **NUNCA TOCA — sem cap** |
| Arquivos em `memory/` | <60 dias sem mod | 60-90 dias | >90 dias |

### Passo 3 — Análise profunda

Para cada arquivo 🔴 ou 🟡, analisar conteúdo e propor ação.

**Para `documento_mestre.md` >300 linhas:**
- Identificar seções com 50+ linhas sobre tema único.
- Cada candidato vira proposta de satélite `<tema>_mestre.md` na **raiz** (não em memory/).

**Para `aprendizados.md` >150 linhas:**
- Identificar entradas duplicadas ou similares para mesclar.
- Identificar lições antigas (referenciadas no changelog há >90 dias) candidatas a remover.
- Identificar lições que viraram regra permanente (deveriam ter subido pro mestre via /codex mas escaparam).

**Para arquivos em `memory/` >90 dias:**
- Avaliar se conteúdo virou regra (deveria ter sido promovido) ou é histórico (vai para `memory/arquivo/`).
- Se deveria ter virado regra, propor promoção pro mestre + apagamento.
- Se é histórico não-essencial, propor mover para `memory/arquivo/`.

**Para `<tema>_mestre.md` >500 linhas:**
- Identificar sub-temas para split em sub-satélite (mesma convenção, ex: `stripe_webhooks_mestre.md` saindo de `stripe_mestre.md`).

### Passo 4 — Plano de ação

Apresentar lista numerada agrupada por categoria:

```
Plano de organização — <projeto>

SPLIT DO MESTRE (X ações)
1. [documento_mestre.md] Seção "Decisões Stripe" (62 linhas) → criar stripe_mestre.md na raiz
2. [documento_mestre.md] Seção "Compliance Bacen" (55 linhas) → criar compliance_mestre.md na raiz

COMPACTAÇÃO DE APRENDIZADOS (X ações)
3. [aprendizados.md] Mesclar 4 entradas similares na seção Segurança em 1 entrada consolidada
4. [aprendizados.md] Promover regra "X" da seção Process para o mestre (deveria ter ido via /codex)

ARQUIVAMENTO DE MEMORY/ (X ações)
5. [memory/feedback_old_thing.md] >100 dias, conteúdo já refletido em código → memory/arquivo/
6. [memory/project_dead.md] decisão revertida em sessão posterior → memory/arquivo/

SUB-SPLIT (X ações)
7. [stripe_mestre.md] Seção "Webhooks" (110 linhas) → criar stripe_webhooks_mestre.md na raiz

MANTER (sem ação) — X arquivos
- changelog.md: imutável, fora do escopo
- handoff.md: responsabilidade do /codex
- demais arquivos dentro do limite
```

Perguntar:
> "Posso executar o plano completo? Ou prefere aprovar item por item?"

**NUNCA executar sem resposta explícita.**

### Passo 5 — Execução

Aceitar respostas como "tudo sim", "só 1, 3 e 5", "não quero arquivar nada". Executar apenas itens aprovados, nesta ordem:

#### 5a. Split do mestre

Para cada split aprovado:
1. Ler a seção original do `documento_mestre.md`.
2. Criar arquivo `<projeto>/<tema>_mestre.md` (raiz, **não memory/**) com cabeçalho:
   ```markdown
   > Documento pai: documento_mestre.md
   > Criado em: DD/MM/AAAA | Extraído de: documento_mestre.md

   # <Título do tema>

   <conteúdo completo da seção>
   ```
3. No `documento_mestre.md`, substituir a seção original por ponteiro de 2-3 linhas:
   ```markdown
   > **<TEMA>:** <resumo de 1-2 linhas>. Ver `<tema>_mestre.md`.
   ```

**Convenção de nomes:**
- Tudo em minúsculas: `<tema>_mestre.md`
- Exemplos: `stripe_mestre.md`, `auth_mestre.md`, `compliance_mestre.md`, `outlook_mestre.md`

#### 5b. Compactação do aprendizados

Para cada operação aprovada:
1. **Mesclar entradas similares**: combinar em entrada única mantendo o "por quê" mais informativo.
2. **Promover para o mestre**: copiar regra para `documento_mestre.md` (seção "Regras permanentes") e remover do `aprendizados.md`.
3. **Remover lições obsoletas**: registrar no changelog antes de apagar (formato: `## YYYY-MM-DD — Limpeza de aprendizados.md` + lista do que foi removido).

#### 5c. Arquivamento de memory/

Para cada arquivo aprovado:
1. Criar `<projeto>/memory/arquivo/` se não existir.
2. Mover arquivo para `<projeto>/memory/arquivo/<nome_original>.md`.
3. Registrar entrada no changelog: `## YYYY-MM-DD — Arquivamento` + lista do que foi movido + razão.

#### 5d. Sub-split de satélite_mestre

Mesma lógica do 5a, mas operando sobre `<tema>_mestre.md` ao invés do mestre principal. Sub-satélite vira `<tema>_<sub_tema>_mestre.md`.

#### 5e. Confirmação por alteração

Antes de cada Edit/Write em arquivo existente, mostrar antes/depois resumido:

```
Antes (documento_mestre.md, linhas 142-204):
[primeiras linhas do bloco]

Depois:
> **Decisões Stripe:** integração com webhook + reconciliação. Ver `stripe_mestre.md`.

Confirmar? (sim / não / ajustar)
```

### Passo 6 — Relatório final

```
Otimização concluída ✓

| Arquivo | Antes | Depois | Δ |
|---|---|---|---|
| documento_mestre.md | 487 linhas | 220 linhas | -55% |
| aprendizados.md | 178 linhas | 142 linhas | -20% |

Novos satélites criados (raiz):
- stripe_mestre.md (62 linhas)
- compliance_mestre.md (55 linhas)
- stripe_webhooks_mestre.md (110 linhas, sub-split)

Arquivados:
- memory/arquivo/feedback_old_thing.md (registrado no changelog)

Status pós-otimização:
- documento_mestre.md: 🟢
- aprendizados.md: 🟢
- changelog.md: imutável (não tocado)

Próxima higiene recomendada: ~30 dias ou quando algum semáforo voltar pro 🔴.
```

Se restou item 🟡/🔴 não aprovado:
> "Ainda há X arquivos acima do limite ideal. Posso revisar numa próxima execução."

## Regras invioláveis

- **Nunca executar sem aprovação explícita** — plano primeiro, ação depois.
- **NUNCA tocar em `changelog.md`** — imutabilidade total. Se precisa registrar arquivamento/limpeza, é via append normal seguindo regra do /codex.
- **Nunca apagar conteúdo direto** — sempre move (changelog, satélite ou `memory/arquivo/`).
- **Nunca criar mais de 3 satélites por execução** — anti-fragmentação.
- **Nunca quebrar links bidirecionais** — todo `<tema>_mestre.md` tem header `> Documento pai: documento_mestre.md`, e o mestre tem ponteiro descendente.
- **Sempre ler o arquivo antes de alterar** — nada de editar sem entender o conteúdo.
- **Sempre mostrar antes/depois** para alterações em arquivos existentes.
- **Idioma:** PT-BR para todos os arquivos de memória.

## Integração com outras skills

- **`/codex`** — wrap de fim de sessão. Faz faxina rápida do memory/ + atualização cirúrgica do mestre + append no changelog. Esta skill faz higiene macro periódica (split, compactação, arquivamento).
- **Bootstrap de novo projeto** — `~/.claude/shared/memory-template/` tem os esqueletos.

## Quando invocar

- `/codex` reportou mestre 🟡 ou 🔴 (>300 ou >500 linhas)
- `aprendizados.md` passou de 150 linhas
- Arquivos em `memory/` há mais de 90 dias acumulando
- Mensal como higiene preventiva
- Antes de uma sessão longa, para limpar o terreno
- Quando o usuário pede explicitamente: "otimizar projeto", "split do mestre", "limpar memory antigo", "compactar aprendizados"
