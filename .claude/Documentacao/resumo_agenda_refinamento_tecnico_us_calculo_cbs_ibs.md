# 📌 Refinamento Técnico | Cálculo CBS/IBS na Geração do Pedido

## Objetivo

Implementar o cálculo de CBS e IBS no momento da geração do pedido, utilizando o parâmetro de regime tributário (Reforma Tributária ou Automático) como gatilho de execução.

## Principais decisões

- ✅ O cálculo deixa de ocorrer em etapas posteriores e passa a ser executado na geração do pedido.
- ✅ Será criada a tabela `PEDIDO_IMPOSTO` para armazenar:
  - Imposto (ISS/CBS/IBS)
  - Valor base utilizado
  - Alíquota aplicada
  - Valor calculado
  - Data/Hora do cálculo

  _Mantendo histórico completo do cálculo realizado em cada pedido._

- ✅ O cálculo ficará centralizado no microsserviço **VT Orders Calculation**, consumido pelo Portal 1.0 e Portal 2.0.
- ✅ Será adicionada uma _flag_ na tabela de alíquotas para identificar quais impostos participam do cálculo da geração do pedido. Essa informação deverá ser herdada automaticamente em novas vigências.
- ✅ Não haverá exibição desses valores em tela neste momento.

## Pontos de atenção

- ⚠️ **Recarga Certa:** continuará permitindo recálculo após retorno da SAT, recriando os registros de impostos calculados.
- ⚠️ **Dependência do Time de Preços:** Caso as tarifas passem a incorporar CBS/IBS, a base de cálculo utilizada hoje poderá precisar ser revisada (valor líquido x cálculo reverso).
- ⚠️ Foi sugerido adicionar um diagrama técnico ao refinamento para facilitar o entendimento dos fluxos e integrações.

## Impactos

- Criação da tabela `PEDIDO_IMPOSTO`.
- Alteração da tabela de alíquotas.
- Novo _endpoint_ de cálculo no **VT Orders Calculation**.
- Ajustes no Portal 1.0 e Portal 2.0.
- Ajuste da contabilização para utilizar `PEDIDO_IMPOSTO` em vez das informações atualmente consideradas no fluxo anterior.

## Próximos passos

1. Criar estrutura de banco (tabela, sequência e ajustes em alíquota).
2. Implementar _endpoint_ de cálculo no **VT Orders Calculation**.
3. Integrar Portal 1.0 e Portal 2.0.
4. Persistir os dados de cálculo em `PEDIDO_IMPOSTO`.
5. Ajustar contabilização para consumir a nova estrutura.
6. Executar testes dos cenários de geração de pedido, Recarga Certa e contabilização.

## Priorização acordada

1. 1️⃣ Cálculo CBS/IBS
2. 2️⃣ Lançamento contábil
3. 3️⃣ Logs/Auditoria (posteriormente)

```eof

O arquivo está pronto! Você pode usar a opção de copiar no canto superior direito do bloco de código para levar esse conteúdo direto para o seu editor Markdown, Notion, Jira ou qualquer outra ferramenta. Se precisar de mais algum ajuste na formatação, é só avisar!
```
