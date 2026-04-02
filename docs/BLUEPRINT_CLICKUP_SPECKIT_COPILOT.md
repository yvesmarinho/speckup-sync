# Blueprint: ClickUp (Desktop) ⇄ Speckit Workflow ⇄ Copilot ⇄ Sincronismo via API (Local)

## Objetivo

Usar o **ClickUp (cliente desktop)** como o ambiente principal de gestão de projetos e tarefas (substituindo bloco de notas), mantendo o desenvolvimento **local** (sem CI obrigatório), e integrando:

- **ClickUp**: UI/gestão/visual
- **Speckit**: workflow completo (spec, plano, execução, validação, outcome)
- **Copilot**: assistência para gerar/atualizar specs, resumos, checklists e textos de PR
- **cu-sync (CLI local)**: sincronizar dados do Speckit para o ClickUp via **API**, de forma determinística e idempotente

O propósito é ter um loop fechado e rastreável:

> ClickUp (task) → Speckit (workflow e artefatos) → Copilot (assistência) → Speckit (outcome) → `cu-sync push` → ClickUp (comentários/status/campos)

---

## Princípios de Arquitetura

1. **ClickUp é a fonte de verdade para status e planejamento**  
   Você acompanha e ajusta no ClickUp Desktop.

2. **Speckit é a fonte de verdade para a execução técnica e artefatos**  
   Specs, planos, decisões e outcome ficam versionados no repo (quando fizer sentido).

3. **Automação local e explícita**  
   Você dispara a sincronização quando quiser (pré-PR, pós-PR, “fim do dia”, etc.).

4. **Idempotência e baixo ruído**  
   Nunca spammar comentários duplicados: o sincronizador detecta se já publicou aquele outcome.

5. **Contrato de dados**  
   Tudo o que for sincronizável deve ter um formato estruturado: JSON/YAML com campos obrigatórios.

---

## Componentes

### 1) ClickUp Desktop
- Criação e acompanhamento das tarefas.
- Ajustes manuais de prioridade, datas, dependências, etc.
- Visual: List/Board/Gantt conforme sua preferência.

### 2) Repositório local do projeto (Git opcional, mas recomendado)
Estruturas fixas para:
- Contexto do ClickUp para o projeto/task atual
- Artefatos do Speckit
- Outcome (resultado consolidado para sincronizar)

### 3) Speckit Workflow
- Você já utiliza o workflow completo.
- O blueprint assume que ao final do ciclo há um “resultado” consolidável (outcome).

### 4) Copilot (rotinas e prompts)
- Rotinas repetíveis (playbook)
- Ajuda a preencher/atualizar spec/outcome
- Ajuda a gerar resumos (changelog, PR description, notas técnicas)

### 5) `cu-sync` (novo projeto/CLI)
- Lê `.clickup/context.json` + `.speckit/outcome.json`
- Faz chamadas para a API do ClickUp:
  - Comentários
  - Atualização de status
  - Atualização de campos customizados
- Implementa idempotência (assinatura no comentário)

---

## Layout de arquivos no repositório

Recomendado:

```
<repo>/
  .clickup/
    context.json
  .speckit/
    outcome.json
    (outros artefatos do workflow)
  docs/
    (documentação e specs, se você versionar)
```

### `.clickup/context.json` (obrigatório)
Arquivo simples que define o **contexto do ClickUp** associado ao repo e/ou ao ciclo atual.

Exemplo:

```json
{
  "clickup": {
    "team_id": "SEU_TEAM_ID",
    "space_id": "SEU_SPACE_ID",
    "list_id": "SEU_LIST_ID",
    "task_id": "SEU_TASK_ID"
  },
  "project": {
    "name": "meu-projeto",
    "repo_path": "."
  }
}
```

> Observação: você pode começar só com `task_id` e evoluir depois.

---

## Contrato do Outcome (artefato do Speckit para sincronização)

### `.speckit/outcome.json` (obrigatório para `push`)
Este é o arquivo que o Speckit (ou você, assistido pelo Copilot) deve preencher ao final do workflow.

Exemplo:

```json
{
  "task_id": "abc123",
  "status": "IN_REVIEW",
  "summary": "Implementado OAuth callback + testes básicos. Ajustado timeout e logs.",
  "details": [
    "Criado endpoint /auth/callback",
    "Adicionados testes unitários para o fluxo de token",
    "Atualizada documentação em docs/oauth.md"
  ],
  "checklist_done": [
    "Implementar endpoint /auth/callback",
    "Adicionar testes unitários",
    "Atualizar documentação"
  ],
  "links": [
    { "type": "commit", "value": "local:HEAD" },
    { "type": "doc", "value": "docs/oauth.md" }
  ],
  "custom_fields": {
    "version": "0.3.0",
    "module": "auth"
  }
}
```

Campos recomendados:
- `task_id` (**obrigatório**): o ID do ClickUp Task (ou inferido de `.clickup/context.json`)
- `status` (opcional): status desejado no ClickUp (ex.: `IN_REVIEW`, `DONE`)
- `summary` (**obrigatório**): resumo curto para comentário no ClickUp
- `details` (opcional): lista de bullets para enriquecer o comentário
- `checklist_done` (opcional): itens concluídos
- `links` (opcional): commits locais, docs, etc.
- `custom_fields` (opcional): map para preencher campos customizados do ClickUp

---

## Idempotência: evitar spam no ClickUp

### Estratégia de assinatura
Ao publicar um comentário no ClickUp, `cu-sync` deve inserir uma assinatura estável, por exemplo:

- `<!-- cu-sync: outcome_sha=<sha256> -->`

Onde `outcome_sha` é um hash do conteúdo de `.speckit/outcome.json` normalizado.

Regras:
1. Se existir comentário com a mesma assinatura, **não postar** novamente.
2. Se existir comentário com assinatura anterior para a mesma task, opcionalmente:
   - editar o comentário (se a API suportar update)  
   - ou postar um novo comentário apenas se houver mudanças reais (hash diferente)

> Se a API não permitir editar comentário facilmente, apenas “não duplicar” já resolve 90% do problema.

---

## Comandos do `cu-sync` (CLI local)

### Comandos mínimos (MVP)
1. `cu-sync init`
   - cria `.clickup/context.json` (template)
   - cria `.speckit/outcome.json` (template)
   - cria config do usuário em `~/.config/cu-sync/config.json`

2. `cu-sync set-task <task_id>`
   - atualiza `.clickup/context.json` com o task atual

3. `cu-sync push [--mode pre-pr|done|progress]`
   - lê `.clickup/context.json`
   - lê `.speckit/outcome.json`
   - valida parâmetros e tipagem
   - gera assinatura hash
   - publica no ClickUp:
     - comentário com resumo + detalhes + links
     - atualiza status (se fornecido)
     - atualiza custom fields (se fornecido)

4. `cu-sync pull` (opcional, mas útil)
   - baixa a tarefa do ClickUp (título/descrição/status)
   - atualiza um arquivo local (ex.: `.speckit/task_snapshot.json`)

### Config de token (usuário)
Arquivo: `~/.config/cu-sync/config.json`

```json
{
  "clickup": {
    "token": "SEU_PERSONAL_API_TOKEN"
  },
  "defaults": {
    "team_id": "",
    "space_id": "",
    "list_id": ""
  }
}
```

Segurança:
- permissões do arquivo: `chmod 600 ~/.config/cu-sync/config.json`
- alternativa: variável de ambiente `CLICKUP_TOKEN`

---

## Integração com Copilot (Playbooks)

A ideia é transformar o “quando atualizar ClickUp” em rotinas repetíveis.

### Playbook A: Antes de PR (ou antes de publicar mudanças)
1. Rodar o Speckit workflow até ter o resultado consolidado.
2. Pedir ao Copilot para gerar/atualizar `.speckit/outcome.json`:
   - resumo curto
   - bullets técnicos
   - versão/módulo (se você usa)
3. Rodar:
   - `cu-sync push --mode pre-pr`
4. Opcional: pedir ao Copilot para gerar o texto do PR com base no outcome.

### Playbook B: Encerramento (tarefa concluída)
1. Atualizar `.speckit/outcome.json` para:
   - `status: DONE`
   - incluir “riscos”, “trade-offs”, “pendências” (se aplicável)
2. Rodar:
   - `cu-sync push --mode done`

### Playbook C: Progresso (fim do dia)
1. `summary` com o que foi feito + próximo passo.
2. `status` opcional (“DOING”).
3. `cu-sync push --mode progress`

---

## Convenções recomendadas (para rastreabilidade)

Mesmo sem CI, vale padronizar:

### 1) Identificador da Task
Usar sempre um marcador no texto:
- `CU-abc123` (ou somente o `abc123` conforme seu padrão)

Onde aplicar:
- branch name (se usar git)
- mensagens de commit
- outcome
- docs

### 2) Estrutura de statuses
No ClickUp, tenha um fluxo simples e consistente, por exemplo:

- `TODO`
- `DOING`
- `IN_REVIEW`
- `DONE`

O `cu-sync` só precisa mapear strings do outcome para esses statuses.

### 3) Custom fields (opcional)
Campos úteis para dev:
- `module` (auth, billing, ui)
- `version` (semver)
- `risk` (low/med/high)
- `scope` (feature/bug/chore)

---

## Requisitos do projeto `cu-sync` (conforme regras do usuário)

### Regras funcionais
- Toda função/classe:
  - deve validar parâmetros (vazio e tipagem)
  - deve estar envolvida em `try/except`
  - em erro, retorna `False`

### Docstrings
- Padrão: **reStructuredText**
- Incluir **doctest**
- Exemplo de docstring:

```python
def example(x):
    """
    Exemplo.

    :param x: inteiro
    :type x: int
    :return: True se OK, False em erro
    :rtype: bool

    >>> example(1)
    True
    >>> example("a")
    False
    """
```

### CLI
- `main()` converte `False` em exit code `1`
- Mensagens de erro claras (stdout/stderr conforme preferir)

---

## Endpoints típicos (alto nível)

O `cu-sync` normalmente precisará:
- Obter/validar task (GET)
- Atualizar task (PUT) para status/campos
- Criar comentário (POST)

> Nota: os detalhes exatos de endpoints e payloads devem seguir a documentação oficial da API do ClickUp e o modelo da sua conta (status, custom fields).

---

## Roadmap (incremental e seguro)

### Fase 1 (MVP, 1–2 sessões)
- `cu-sync push`:
  - valida context + outcome
  - posta comentário com `summary` + `details`
  - idempotência por hash
- Sem mexer em status/campos ainda (opcional)

### Fase 2
- Atualizar status via API
- Preencher custom fields via outcome

### Fase 3
- `pull` para snapshot local
- Melhorar edição no `vi`:
  - `cu-sync edit-outcome` abre `$EDITOR` e valida JSON ao salvar

### Fase 4 (opcional)
- Sincronizar checklist
- Anexos (spec, relatórios)
- Cache local e logs estruturados

---

## Checklist de Qualidade (para não virar “bagunça”)

- [ ] Toda task relevante tem `task_id` no `.clickup/context.json`
- [ ] Todo ciclo Speckit termina com `.speckit/outcome.json` válido
- [ ] `cu-sync push` é idempotente
- [ ] Comentários no ClickUp são curtos e “de alto valor”
- [ ] Status no ClickUp reflete a realidade (sem automação agressiva)

---

## Resultado esperado

Com esse blueprint, você:
- abandona bloco de notas e mantém gestão no ClickUp
- mantém execução técnica no Speckit (versionável)
- usa Copilot para acelerar a parte textual/estrutural (spec/resumo/checklist)
- usa um sincronizador local (API) para manter ClickUp atualizado sem esforço manual repetitivo

---

## Próximos passos sugeridos

1. Criar os diretórios e templates:
   - `.clickup/context.json`
   - `.speckit/outcome.json`

2. Definir seu fluxo de statuses no ClickUp (nomes exatos).

3. Começar a implementar `cu-sync` (MVP: comentar outcome).

4. Criar seu “playbook” pessoal (pre-PR / done / progress) e usar o Copilot para manter consistência.

---