# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## O que é este repositório

Workspace de configuração do Claude Code para o contexto de QA — contém skills, regras e documentos de referência para análise de User Stories e geração de artefatos de teste a partir do Azure DevOps.

## Estrutura

```
.claude/
├── rules/
│   └── test-cases-save.md              # Regra ativa: sempre salvar test cases como .xlsx
├── skills/
│   ├── analise-e-resumo-qa/
│   │   └── SKILL.MD                    # Skill 1: analisa US, docs e Swagger — sem gerar cenários
│   ├── gerar-cenarios-bdd/
│   │   └── SKILL.MD                    # Skill 2: gera cenários BDD em .xlsx
│   └── gerar-collection-postman/
│       └── SKILL.MD                    # Skill 3: gera collection Postman v2.1 em .json
└── docs/
    ├── modelo_testplan.xlsx            # Planilha-modelo (colunas e estilo visual)
    ├── worklog.md                      # Registro automático de atividades significativas
    ├── test_cases/                     # Saída dos test cases BDD (um .xlsx por US)
    └── postman/                        # Saída das collections Postman (um .json por US)
```

## Pipeline de QA — Fluxos Suportados

As skills são **modulares e independentes**. Execute na ordem que fizer sentido para o contexto:

### Fluxo Padrão (análise → BDD → Postman)
```
Skill 1: analise-e-resumo-qa  ➔  Skill 2: gerar-cenarios-bdd  ➔  Skill 3: gerar-collection-postman
```

### Fluxo Direto de API (análise → Postman, sem BDD)
```
Skill 1: analise-e-resumo-qa  ➔  Skill 3: gerar-collection-postman
```

### Fluxo com Cenários Manuais (collection a partir de cenários prontos)
```
[cenários colados no prompt]  ➔  Skill 3: gerar-collection-postman
```

## Skills — Resumo

| Skill | Invocação | Entrega |
|-------|-----------|---------|
| `analise-e-resumo-qa` | "analisa a US X" / "me dá um resumo da US X" | Resumo analítico: CAs, endpoints, cenários a cobrir, riscos |
| `gerar-cenarios-bdd` | "gera os cenários BDD" / "escreve os test cases" | Planilha `.xlsx` com cenários no formato Dado/Quando/Então |
| `gerar-collection-postman` | "gera a collection Postman" | Arquivo `.json` Postman v2.1 com requests, scripts e variáveis |

## Regras de Salvamento

- **BDD**: `.claude/docs/test_cases/<nome-da-us>.xlsx` — regras em `.claude/rules/test-cases-save.md`
  - Gerado com `openpyxl`, estilo de `.claude/docs/modelo_testplan.xlsx`
  - Cabeçalho azul (`00B0F0`), fonte branca e bold
  - Colunas: `#` | `Título do Caso de Teste` | `Procedimento do Teste` | `Resultado Esperado`
- **Postman**: `.claude/docs/postman/<nome-da-us>.postman_collection.json`
- Nomenclatura: lowercase, hifenizado, sem acentos ou caracteres especiais

## Azure DevOps

- Organização: `alelo` | Projeto: `ValeTransporte-2.0`
- Squad: `Squad Marte`
- Ferramenta: `mcp__azure-devops__wit_work_item` (já com permissão em `settings.local.json`)

## Worklog Automático

Regra completa em `.claude/rules/worklog-update.md` — **leia e siga obrigatoriamente**.

O worklog fica em `.claude/docs/worklog.md`. A permissão de escrita já está em `settings.local.json`.

**Ao concluir qualquer atividade significativa, a última ação DEVE ser escrever no worklog — sem perguntar ao usuário.**

### Quando consultar

Ao iniciar uma sessão em que o usuário mencione uma US, issue, API, endpoint ou atividade de QA com possível contexto anterior, leia o worklog antes de responder.
