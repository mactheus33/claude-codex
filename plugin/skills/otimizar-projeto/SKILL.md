---
name: otimizar-projeto
description: Use esta skill quando o usuário invocar "/otimizar-projeto", ou pedir para "otimizar memória do projeto", "limpar memory", "compactar documento_mestre", "split do mestre", "organizar memory", "higienizar memória do projeto". Escaneia memory/, propõe plano de organização (split, compactação, promoção, arquivamento) e executa apenas com aprovação do usuário.
version: 0.1.0
---

# Otimizar Projeto — Higiene da memória técnica do projeto

Faz higiene da pasta `memory/` de um projeto técnico do claude-identity. Adapta o conceito de `otimizar-os` (MatheusOS) para projeto com git: escaneia documento_mestre + satélites + arquivos de sessão, identifica o que está crescendo demais ou ficou obsoleto, e propõe um plano de reorganização. Executa só com aprovação.

## Filosofia

`/codex` mantém o digest curto a cada sessão (handoff + aprendizados). Esta skill faz a manutenção de **médio prazo**: split do mestre quando ele cresce, arquivamento de conteúdo obsoleto por idade, promoção de regras que estavam dispersas, e atualização do `index.md`.

Quatro princípios:

1. **Plano antes de execução.** Sempre apresentar o que será feito numerado e categorizado, com aprovação por item ou em bloco.
2. **Nunca apagar.** Conteúdo obsoleto vai para `memory/arquivo/`, não pro lixo. Git preserva histórico.
3. **Limites numéricos como gatilho.** Não otimizar por intuição — usar os semáforos da convenção.
4. **Anti-fragmentação.** Máximo 3 satélites novos por execução. Split agressivo gera ruído de navegação.

## Arquivos gerenciados

Relativos à raiz do projeto ativo:

- **`memory/documento_mestre.md`** — split em satélites quando passa de 300 linhas
- **`memory/<PROJ>_<AREA>_<TEMA>.md`** — satélites do mestre (criados aqui ou herdados)
- **`memory/aprendizados.md`** — promoção de regras permanentes para o mestre, compactação por idade
- **`memory/handoff.md`** — apenas leitura (mantido pelo `/codex`)
- **`memory/index.md`** — atualizado a cada criação/arquivamento de satélite
- **`memory/arquivo/`** — destino do conteúdo obsoleto (criado sob demanda)

A skill **não** toca em arquivos de identidade (`~/.claude/`) nem em CLAUDE.md do projeto.

## Workflow

### Passo 0 — Confirmar projeto

Identificar o projeto ativo a partir do diretório de trabalho. Se ambíguo, pedir confirmação. Se `<projeto>/memory/` não existir, parar e sugerir bootstrap via template em `~/.claude/shared/memory-template/`.

### Passo 1 — Scan e relatório de saúde

Escanear todos os arquivos `.md` em `<projeto>/memory/` (incluindo `arquivo/` se existir). Para cada arquivo coletar: nome, número de linhas, tamanho em KB, data da última modificação.

Apresentar tabela com semáforos:

```
Relatório de saúde — <projeto>/memory/
Data: DD/MM/AAAA

| Arquivo | Linhas | Tamanho | Última mod | Status |
|---------|--------|---------|------------|--------|
| documento_mestre.md | 487 | 28KB | hoje | 🔴 grande |
| aprendizados.md | 220 | 14KB | hoje | 🟡 atenção |
| handoff.md | 45 | 4KB | hoje | 🟢 ok |
| index.md | 62 | 5KB | DD/MM | 🟡 atenção |
| FULLCRED_TECH_API.md | 180 | 12KB | DD/MM | 🟢 ok |
| feedback_old_thing.md | 32 | 1KB | há 100 dias | 🔴 antigo |
```

**Regras de semáforo:**

| Arquivo | 🟢 OK | 🟡 Atenção | 🔴 Ação |
|---|---|---|---|
| `documento_mestre.md` | <300 linhas | 300-500 | >500 → split obrigatório |
| Satélite `<PROJ>_<AREA>_<TEMA>.md` | <25KB | 25-40KB | >40KB → sub-split |
| `aprendizados.md` | <150 linhas | 150-200 | >200 → compactar |
| `handoff.md` | <50 linhas | 50-80 | >80 |
| `index.md` | <50 linhas | 50-100 | >100 (árvore fragmentada demais) |
| Outros `.md` | <20KB | 20-35KB | >35KB |
| Idade sem modificação | <60 dias | 60-90 dias | >90 dias |

### Passo 2 — Análise profunda

Para cada arquivo 🔴 ou 🟡, analisar conteúdo e classificar cada bloco em 4 tipos:

| Tipo | Critério | Ação |
|---|---|---|
| **Permanente** | Regras de negócio, padrões "sempre/nunca", configurações ativas. Identificar por palavras-chave: "nunca", "sempre", "regra", "padrão", "obrigatório" | Fica onde está (no mestre ou no satélite atual) |
| **Promovível** | Regra usada toda sessão, hoje vivendo em `aprendizados.md` ou em satélite mas que deveria estar no mestre | Sobe para `documento_mestre.md` (seção "Regras permanentes") |
| **Temporário** | Decisões pontuais, bug fixes, status updates datados. Identificar por: data específica + resultado pontual | Compactar conforme idade (ver tabela abaixo) |
| **Ultrapassado** | Status que não vale mais ("aguardando X" quando X já aconteceu), pesquisa concluída, decisão substituída por outra | Mover para `arquivo/` |

**Critério de idade para conteúdo temporário:**

| Idade | Ação |
|---|---|
| <30 dias | Não mexer — conteúdo ativo |
| 30-60 dias | Condensar (3 parágrafos viram 3 linhas) |
| 60-90 dias | Condensar obrigatoriamente + mover detalhes para `arquivo/` |
| >90 dias | Manter apenas 1 linha resumo + ref ao arquivo |

**Identificar candidatos a split do mestre:**
- Seções com 50+ linhas sobre um único tema (decisões por área, especificações de módulo, mapeamentos de API, histórico comercial, etc.)
- Cada candidato vira proposta de satélite `<PROJ>_<AREA>_<TEMA>.md`

### Passo 3 — Plano de ação

Apresentar lista numerada agrupada por tipo, com aprovação do usuário antes de qualquer escrita:

```
Plano de organização — <projeto>/memory/

SPLIT DO MESTRE (X ações)
1. [documento_mestre.md] Seção "Decisões de Tech Stack" (62 linhas) → criar FULLCRED_TECH_DECISOES.md
2. [documento_mestre.md] Seção "Regras de Compliance" (55 linhas) → criar FULLCRED_COMPLIANCE_REGRAS.md

PROMOVER PARA O MESTRE (X ações)
3. [aprendizados.md] "CRM internal-only" (regra usada toda sessão) → seção "Regras permanentes > Segurança"
4. [feedback_old_pattern.md] Promover regra → mestre, manter satélite como referência histórica

CONDENSAR / ARQUIVAR (X ações)
5. [aprendizados.md] Entradas de Janeiro (60-90d) → condensar 3 → 1 linha + manter
6. [project_old_decision.md] >90 dias, decisão substituída → mover para arquivo/

ATUALIZAR INDEX (auto)
7. [index.md] Refletir 2 satélites novos + 1 arquivamento

MANTER (sem ação) — X arquivos
- handoff.md: dentro do limite
- aprendizados.md (após item 3 e 5): dentro do limite
- demais satélites recentes
```

Perguntar:
> "Posso executar o plano completo? Ou prefere aprovar item por item?"

**NUNCA executar sem resposta.**

### Passo 4 — Execução

Aceitar respostas tipo "tudo sim", "só 1, 3 e 5", "não quero arquivar nada". Executar apenas itens aprovados, nesta ordem:

#### 4a. Split do mestre

Para cada split aprovado:
1. Ler a seção original do mestre
2. Criar arquivo `<projeto>/memory/<PROJ>_<AREA>_<TEMA>.md` com cabeçalho:
   ```markdown
   > Documento pai: documento_mestre.md
   > Criado em: DD/MM/AAAA | Extraído de: documento_mestre.md

   # <Título do tema>

   <conteúdo completo da seção>
   ```
3. No `documento_mestre.md`, substituir a seção original por ponteiro de 2-3 linhas:
   ```markdown
   > **<TEMA>:** <resumo de 1-2 linhas>. Documento completo: `<PROJ>_<AREA>_<TEMA>.md`
   ```
4. Atualizar `index.md` adicionando entrada do novo satélite

**Convenção de nomes:**
- `<PROJ>` = prefixo do projeto em maiúsculas (FULLCRED, VIABOX, BWPSITE, CODEX)
- `<AREA>` = TECH / BIZ / INFRA / PRODUTO / DECISOES / INTEGRACAO / COMERCIAL / COMPLIANCE
- `<TEMA>` = assunto específico

#### 4b. Promoção para o mestre

Para cada promoção aprovada:
1. Ler a regra na origem (aprendizados.md ou satélite)
2. Adicionar no `documento_mestre.md` (seção "Regras permanentes" ou "Decisões-chave"), formato condensado: `**<Regra>:** <descrição>. Por quê: <razão>.`
3. Remover da origem (se vier do `aprendizados.md`)
4. Se vier de satélite, manter o satélite como referência histórica mas atualizar com nota: `> Promovido para documento_mestre.md em DD/MM`

#### 4c. Condensar conteúdo antigo

Para cada bloco aprovado:
1. Ler a seção original
2. Reescrever em formato compacto:
   - Decisões: `**[DATA] Decisão:** <resultado em 1 frase>. Detalhes em arquivo/<arquivo>.md.`
   - Processos concluídos: `**<TEMA>:** <resultado final>. Ver arquivo/.`
   - Status desatualizados: remover (já estão em git log)
3. Verificar que referências cruzadas não quebraram

#### 4d. Arquivar obsoletos

Para cada arquivamento aprovado:
1. Criar pasta `<projeto>/memory/arquivo/` se não existir
2. Mover arquivo para `<projeto>/memory/arquivo/<nome_original>.md`
3. Remover entrada do `index.md`

#### 4e. Atualizar index.md

Após todas as operações:
1. Recalcular total de arquivos
2. Adicionar entradas dos novos satélites com link e descrição
3. Remover entradas dos arquivos arquivados
4. Atualizar data no cabeçalho

#### 4f. Confirmação por alteração

Antes de cada Edit/Write em arquivo existente, mostrar antes/depois resumido:

```
Antes (memory/documento_mestre.md, linhas 142-204):
[primeiras linhas do bloco]

Depois:
> **Decisões de Tech Stack:** stack consolidado pós-MVP. Detalhes em FULLCRED_TECH_DECISOES.md

Confirmar? (sim / não / ajustar)
```

### Passo 5 — Relatório final

```
Otimização concluída ✓

| Arquivo | Antes | Depois | Δ |
|---------|-------|--------|---|
| documento_mestre.md | 487 linhas | 220 linhas | -55% |
| aprendizados.md | 220 linhas | 145 linhas | -34% |

Novos satélites criados:
- FULLCRED_TECH_DECISOES.md (62 linhas)
- FULLCRED_COMPLIANCE_REGRAS.md (55 linhas)

Promovidos para o mestre: 2 regras
Arquivados: 1 arquivo (project_old_decision.md → arquivo/)

Status pós-otimização:
- documento_mestre.md: 🟢 ok
- aprendizados.md: 🟢 ok
- index.md: atualizado, 41 arquivos

Próxima higiene recomendada: ~30 dias ou quando algum semáforo voltar pro 🔴.
```

Se restou item 🟡/🔴 não aprovado:
> "Ainda há X arquivos acima do limite ideal. Posso revisar numa próxima execução."

## Regras

- **Nunca executar sem aprovação explícita** — plano primeiro, ação depois.
- **Nunca apagar conteúdo** — apenas mover (arquivo/, satélite ou git log).
- **Nunca resumir regras de negócio fundamentais** — independente de idade.
- **Nunca criar mais de 3 satélites por execução** — evita fragmentação.
- **Nunca quebrar links bidirecionais** — todo satélite tem header `> Documento pai: documento_mestre.md`, e o mestre tem ponteiro descendente.
- **Sempre ler o arquivo antes de alterar** — nada de editar sem entender o conteúdo.
- **Sempre mostrar antes/depois** para alterações em arquivos existentes.
- **Sempre atualizar `index.md`** quando criar/mover satélite — não esperar fim da execução.
- **Idioma:** PT-BR para todos os arquivos de memória, exceto identifiers técnicos (paths, comandos, código).

## Integração com outras skills

- **`/codex`** — wrap de fim de sessão. Faz promoção pontual de regra do dia para o mestre. Esta skill (`/otimizar-projeto`) faz higiene macro de médio/longo prazo.
- **Bootstrap de novo projeto** — `~/.claude/shared/memory-template/` tem os esqueletos. Se memory/ não existe, sugerir bootstrap antes de rodar esta skill.

## Quando invocar

- `/codex` reportou mestre 🟡 ou 🔴 (>300 ou >500 linhas)
- Mensal como higiene preventiva
- Antes de uma sessão longa, para limpar o terreno
- Quando arquivos `aprendizados.md` ou `index.md` ficaram densos
- Quando o usuário pede explicitamente: "otimizar projeto", "limpar memory", "split do mestre", "organizar memória"
