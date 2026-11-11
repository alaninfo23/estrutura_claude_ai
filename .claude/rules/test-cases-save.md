# Regra: Salvar cenários de teste gerados

Sempre que gerar cenários de teste BDD para uma User Story:

1. Siga o processo definido em `skills/gerar-cenarios-bdd/SKILL.MD` para gerar os cenários
2. Salve o resultado como planilha `.xlsx` em:

```
/Users/luciane.barraviera/Alelo/Claude/docs/test_cases/<nome-da-us>.xlsx
```

## Formato da planilha

Use o modelo `/Users/luciane.barraviera/Alelo/Claude/docs/modelo_testplan.xlsx` como base:

| Título do Caso de Teste | Procedimento do Teste | Resultado Esperado |
| ----------------------- | --------------------- | ------------------ |

- Cada linha = um caso de teste
- Sem markdown, apenas a planilha
- **A linha 1 é sempre o cabeçalho das colunas (`#` | `Título do Caso de Teste` | `Procedimento do Teste` | `Resultado Esperado`), com fundo azul (`00B0F0`), fonte branca e bold — nunca o nome ou título da US**

## Nomenclatura do arquivo

- Lowercase, espaços substituídos por `-`, sem acentos ou caracteres especiais
- Exemplo: "Mapeamento do código de erro do bloco 9" → `mapeamento-do-codigo-de-erro-do-bloco-9.xlsx`

## Validação obrigatória de cenários BDD

Todo cenário gerado deve conter obrigatoriamente as três palavras-chave BDD:

- **Dado que** — no campo Procedimento do Teste (pré-condição/contexto)
- **Quando** — no campo Procedimento do Teste (ação executada)
- **Então** — no campo Resultado Esperado (resultado esperado)

Cenários sem qualquer uma dessas três palavras-chave **não devem ser gravados**. O script de geração (`docs/test_cases/gen_bdd.py`) implementa essa validação na função `validate_scenarios()` e aborta antes de salvar o arquivo caso encontre violação.

## Quando aplicar

- Quando a skill `gerar-cenarios-bdd` for invocada
- Quando o usuário pedir para gerar cenários/casos de teste de uma US
