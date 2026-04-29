---
name: codex
description: Use esta skill quando o usuário invocar "/codex", ou pedir para "encerrar a sessão", "fechar", "salvar handoff", "atualizar memória do projeto", "compactar memória", "escrever resumo de sessão", ou qualquer frase indicando que a sessão de trabalho deve ser encerrada. Lê a conversa atual, compacta o handoff.md e aprendizados.md do projeto ativo em digests rolantes concisos, propõe promoção de regras permanentes para o documento_mestre.md, e escreve apenas após aprovação do usuário.
version: 0.3.0
---

# Codex — Rolling Digest

Encerra uma sessão de trabalho do Claude Code compactando a memória do projeto em um digest pequeno, atual e carregando peso. Previne os arquivos de `memory/` de crescerem sem limite, preservando lições que ainda importam, e mantém o `documento_mestre.md` (documento vivo do projeto) atualizado com regras permanentes.

## Filosofia

Memória que só cresce vira ruído. Esta skill aplica **rolling digest**: a cada fim de sessão, a memória é relida, mesclada com o trabalho de hoje e reescrita concisa. Quatro princípios:

1. **Pequeno** — tamanho limitado, cabe no contexto sem queimar tokens.
2. **Atual** — reflete estado mais recente, não histórico. Histórico vive no git.
3. **Carrega peso** — toda linha justifica o lugar evitando um erro futuro ou esclarecendo estado atual.
4. **Promove para o mestre** — regras que viraram permanentes sobem para `documento_mestre.md` e saem do `aprendizados.md` (anti-duplicação).

## Arquivos gerenciados

Relativos à raiz do projeto ativo:

- **`memory/documento_mestre.md`** — documento vivo do projeto (escopo, status, pendências, regras permanentes, decisões-chave). Lido no início; atualizado de forma cirúrgica (status + pendências + promoção de regras). Limite <300 linhas (🟢) / 300-500 (🟡) / >500 (🔴 split).
- **`memory/handoff.md`** — estado da sessão. Sempre sobrescrito.
- **`memory/aprendizados.md`** — lições curadas. Sempre sobrescrito, depois de mesclar antigas + novas. Cap ~150 linhas.
- **`memory/index.md`** — catálogo da árvore. Atualizado se houver criação/arquivamento de satélite na sessão (raro no /codex; mais comum no `/otimizar-projeto`).

A auto-memória global (`~/.claude/projects/.../MEMORY.md`) NÃO é tocada por esta skill — ela tem disciplina própria.

## Workflow

Seguir os passos na ordem quando a skill for acionada.

### Passo 1: Confirmar escopo do projeto

Identificar o projeto ativo a partir do diretório de trabalho atual e do contexto da conversa. Se ambíguo, ou se o diretório não tem pasta `memory/` e criar uma não é obviamente correto, pedir para o usuário confirmar projeto e caminho antes de prosseguir. Não assumir.

### Passo 2: Ler contexto da sessão atual

Escanear a conversa em busca de:
- O que foi entregue (estados finais, testes passando, mudanças deployadas).
- Lições não-óbvias aprendidas — técnicas, de negócio, de processo.
- **Regras permanentes** descobertas ou ratificadas (decisões estruturais, convenções "sempre/nunca").
- Evoluções estruturais (mudanças de stack, decisões de arquitetura, novas integrações).
- O que está em progresso, o que vem a seguir, o que está bloqueado.
- **Pendências resolvidas** (do `documento_mestre.md` que foram fechadas) e **pendências novas**.

### Passo 3: Carregar memória existente

Ler (se existir):
- `<raiz-do-projeto>/memory/documento_mestre.md`
- `<raiz-do-projeto>/memory/handoff.md`
- `<raiz-do-projeto>/memory/aprendizados.md`
- `<raiz-do-projeto>/memory/index.md`

Se algum deles não existir, tratar como vazio. Se `documento_mestre.md` não existir, propor criar usando o template em `~/.claude/shared/memory-template/documento_mestre.md` antes de prosseguir.

### Passo 4: Compor novo handoff.md

`handoff.md` sempre é sobrescrito. Usar exatamente esta estrutura:

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
- Cap ~50 linhas. Se a sessão foi muito densa, condensar para o essencial.

### Passo 5: Compor novo aprendizados.md (digest)

`aprendizados.md` também é sobrescrito, mas apenas depois de mesclar antigos + novos.

**Manter dos aprendizados existentes:**
- Não-óbvios E ainda acionáveis.
- Lições que, se esquecidas, causariam repetição de erro.

**Descartar dos aprendizados existentes:**
- Bug fixes pontuais já refletidos no código.
- Lições que viraram convenção do código (vivem no código agora).
- Qualquer coisa genérica já coberta em arquivos de identidade global (ex: `~/.claude/soul.md`, CLAUDE.md).
- Itens duplicados — mesclar em um.
- **Itens já promovidos para `documento_mestre.md`** (na seção "Regras permanentes").

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

### Passo 5.5: Identificar promoções para o documento_mestre

Examinar a lista final de `aprendizados.md` (mesclada) e identificar entradas que **viraram regra permanente** — itens que:
- O usuário cita repetidamente em sessões diferentes
- Definem comportamento "sempre/nunca" para o projeto
- São referência de decisão estrutural (não gotcha pontual)
- O agente deveria ler **antes** de tocar em código relacionado

Para cada candidato a promoção, preparar:
1. **Para qual seção do mestre vai** — "Regras permanentes" (com sub-seção como "Operacional", "Segurança", "Frontend", "Compliance" etc.) OU "Decisões-chave" se for decisão estrutural.
2. **Versão condensada da regra** — 1-2 linhas, formato "regra direta. Por quê: razão" (idêntico ao formato do mestre).
3. **Confirmar que será removida do `aprendizados.md`** após promoção (anti-duplicação).

### Passo 5.7: Atualizar documento_mestre.md (cirúrgico)

Não sobrescrever o mestre. Aplicar mudanças cirúrgicas:
- **Header:** atualizar "Última atualização" para a data de hoje.
- **Status atual:** atualizar com 1-3 frases sobre o estado pós-sessão.
- **Pendências abertas:** marcar com `~~[x]~~ → ver changelog` as que foram resolvidas; adicionar novas com `[ ]`.
- **Próximos passos:** atualizar conforme handoff.md.
- **Regras permanentes:** adicionar as promoções identificadas no Passo 5.5.
- **Decisões-chave:** adicionar decisões estruturais novas da sessão.
- **Roadmap:** marcar items concluídos, adicionar novos se aplicável.
- **Árvore de arquivos:** se algum satélite foi criado/arquivado, atualizar (raro nesta skill — `/otimizar-projeto` é a skill primária pra isso).

**Health check:** após as edições, verificar tamanho do mestre.

| Linhas | Ação |
|---|---|
| <300 | 🟢 Tudo certo |
| 300-500 | 🟡 Sugerir rodar `/otimizar-projeto` na próxima sessão |
| >500 | 🔴 **Sugerir rodar `/otimizar-projeto` agora** após o /codex completar — split de seções com 50+ linhas para satélites `<PROJ>_<AREA>_<TEMA>.md` |

### Passo 6: Mostrar drafts, pedir aprovação

Antes de gravar, exibir ao usuário:

1. O `handoff.md` proposto (conteúdo completo).
2. O `aprendizados.md` proposto (conteúdo completo).
3. As mudanças cirúrgicas no `documento_mestre.md` (formato diff: linhas adicionadas/removidas/modificadas, com seção).
4. Resumo do diff: "Mantidos N itens dos aprendizados existentes, descartados M, adicionados K novos. **L promoções para documento_mestre.md.** Mestre agora com X linhas (🟢/🟡/🔴)."

Perguntar explicitamente: "Aprova para gravar?"

### Passo 7: Gravar apenas após aprovação

- Aprovação: gravar os 3 arquivos com a tool Write/Edit (handoff e aprendizados sobrescritos; mestre editado de forma cirúrgica).
- Redirecionamento: revisar com base no feedback do usuário, mostrar de novo. Não gravar até aprovação explícita.
- Rejeição: parar. Não gravar parcialmente.

Se o mestre passou para 🔴 (>500 linhas), terminar com a recomendação:
> "documento_mestre.md está em [N] linhas (🔴). Recomendo rodar `/otimizar-projeto` agora para split em satélites antes de continuar."

## Regras

- **Um projeto por invocação.** Nunca gravar em memória de múltiplos projetos numa execução.
- **Nunca anexar** ao handoff ou aprendizados. Sempre ler → mesclar → sobrescrever.
- **Documento mestre é editado, não sobrescrito.** Mudanças cirúrgicas preservam o que já está consolidado.
- **Sem fabricação.** Se a sessão não produziu aprendizado, deixar o aprendizados.md intacto. Vazio é melhor que inventado. Mesma regra para promoções: só promover regras realmente ratificadas.
- **Anti-duplicação.** Toda regra promovida para o mestre é REMOVIDA do aprendizados.md no mesmo turno.
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
- Bypass de file size no pentest de upload. Fixed via max upload sizing. Por quê: atacante conseguia burlar o check de MIME spoofando a extensão num arquivo que excedia checks de content-type só depois do estágio de descompressão.

## Infra
- Hetzner CX33 chega a 85% CPU em rebuilds Docker. Por quê: instância sizeada para steady-state, não burst.

## Negócio
- Conversão da landing page +40% quando o campo telefone vira opcional. Por quê: leads de real estate resistem a dar telefone de cara.
```

### Edição cirúrgica do documento_mestre.md (exemplo de diff proposto)

```diff
 # viabox — Documento Mestre

-> Última atualização: 14/04/2026
+> Última atualização: 17/04/2026
 > Status: MVP em produção

 ## Status atual
-MVP rodando, próximo marco é deploy em staging.
+MVP rodando, e2e tests 15/15 passando. Próximo marco: deploy em staging
+pós-revisão de segurança externa.

 ## Pendências abertas
-- [ ] Validar e2e tests
-- [ ] Agendar revisão de segurança
+- ~~[x] Validar e2e tests~~ → 15/15 passaram, ver handoff 17/04
+- [ ] Agendar revisão de segurança com auditor externo

 ## Regras permanentes
+
+### Segurança
+- Validar tamanho de upload em servidor, não confiar em content-type. Por quê: bypass via spoofing de extensão em arquivo grande.
```
