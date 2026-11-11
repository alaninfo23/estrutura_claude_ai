# Regra: Atualizar worklog ao concluir atividade significativa

## O que é uma atividade significativa

- Análise de User Story ou Issue (via Azure DevOps ou contexto colado)
- Geração de cenários BDD / planilha de test cases (.xlsx)
- Geração de collection Postman (.json)
- Análise de API ou Swagger
- Elaboração ou atualização de documentação
- Investigação de problema ou mapeamento de erros
- Configuração do ambiente ou das skills

**Não registre**: perguntas simples, pequenas correções, dúvidas pontuais sem artefato produzido.

## Obrigação imperativa

Ao concluir qualquer atividade significativa acima, a **última ação obrigatória** antes de encerrar a resposta é escrever no worklog. Não pergunte ao usuário. Não postergue. Escreva.

Arquivo: `/home/josealan/Claude/.claude/docs/worklog.md`
Permissão já concedida em `settings.local.json` — nenhum prompt será exibido.

## Formato de entrada (entradas mais recentes no topo)

```
## AAAA-MM-DD — <título curto da atividade>

**Atividade:** <Análise de US | BDD | Postman | Documentação | Investigação | Configuração>
**Escopo:** <nome da US, issue, endpoint ou componente>
**Resultado:** <o que foi produzido ou concluído>
**Pendências:** <próximos passos, se houver — omita a linha se não houver>

---
```

## Restrições

- Máximo 5–6 linhas por entrada
- Nunca registrar credenciais, tokens, senhas, dados pessoais ou sensíveis
- Não inventar informações — registre apenas o que foi efetivamente feito
