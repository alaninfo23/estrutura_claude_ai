# Ata – Refinamento de Negócios: Cálculo de IBS e CBS (US 1747321)

**Data:** 10/11/2026  
**Sessões:** Duas gravações (a principal com cerca de 37 minutos e uma complementar com cerca de 4 minutos).

## Participantes

- **Renata Cristina Barbosa De Lima** (Produto, condução)
- **Fabiana Santos Leão** (Liderança técnica)
- **Luciane Taminato Sato Barraviera** (QA)
- **Raiela Quirino Lima** (QA)
- **Jose Alan Dos Santos Rocha** (QA)
- **Matheus Henrique Ramos Da Silva** (Desenvolvimento)
- **Wallyson Neves De Moraes** (Desenvolvimento)

**Pauta:** Apresentação da US de cálculo de IBS e CBS (Cenário 1 – base sem ISS): descrição, critérios de aceite, regras de negócio, dados gravados e sugestões de BDD.

---

## 1. Resumo

A Renata apresentou a história de cálculo de IBS e CBS sobre o total das taxas do pedido menos o ISS. O time tirou dúvidas sobre a base de cálculo, o momento do cálculo, o arredondamento, a Recarga Certa, o balanceador e o uso do valor calculado.

O principal alinhamento foi que, neste momento, **o cálculo serve ao lançamento contábil e não à emissão da nota fiscal**. Os impostos são calculados e arredondados individualmente, sem somar IBS e CBS, e o lançamento contábil também é feito separado por imposto.

---

## 2. Apresentação da história

### 2.1. Objetivo

Como área de emissão de nota fiscal e contábil do VT, calcular o IBS e a CBS sobre o valor total das taxas menos o ISS, para conferir o cálculo fiscal e salvar as informações.

### 2.2. Alíquotas

- **De 01/01/2027 a 31/12/2028:** IBS 0,10% e CBS 11,20%.
- As alíquotas são lidas do parâmetro com vigência, na tela de _Tributário > Alíquotas_, que está em desenvolvimento.
- A data que define a vigência é a **data de geração do pedido**.

### 2.3. Quando o cálculo acontece

- **Regime Legado (todo o ano de 2026):** não calcula e não grava. Nada muda no que existe hoje.
- **Regime Reforma Tributária:** calcula e grava ISS, IBS e CBS de cada pedido gerado.
- **Mudança de regime:** pode ser configurada para acontecer sozinha, pela vigência, em 01/01/2027. Haverá um controle de regime (_feature flag_). O detalhe técnico fica no refinamento técnico.
- **Portais 1.0 e 2.0:** os dois calculam e gravam, e o resultado precisa ser o mesmo nos dois.
- **Recarga Certa:** quando o pedido é recalculado, os valores gravados na geração do pedido são substituídos pelos novos.

### 2.4. Cálculo em 4 passos (2 casas, arredondamento `HALF_UP`)

1. **ISS** = total das taxas × alíquota do ISS (2%).
2. **Base IBS/CBS** = total das taxas − ISS.
3. **IBS** = base × alíquota do IBS.
4. **CBS** = base × alíquota da CBS.

_Nota: O cálculo nunca é feito sobre o valor total do pedido. Ele é feito só sobre o total das taxas, depois de tirar o ISS._

### 2.5. Exemplo apresentado (total das taxas de R$ 100,00)

| Item                         | Conta                | Valor     |
| :--------------------------- | :------------------- | :-------- |
| **ISS**                      | 100 × 2%             | R$ 2,00   |
| **Base**                     | 100 − 2              | R$ 98,00  |
| **IBS**                      | 98 × 0,10% = 0,098   | R$ 0,10   |
| **CBS**                      | 98 × 11,20% = 10,976 | R$ 10,98  |
| **IBS + CBS**                | 0,10 + 10,98         | R$ 11,08  |
| **Simulação do valor final** | 100 + 11,08          | R$ 111,08 |

### 2.6. O que é gravado

- Três linhas por pedido, uma para cada imposto (ISS, IBS e CBS), com pedido, imposto, base, alíquota, valor e data/hora do cálculo.
- No máximo uma linha por imposto em cada pedido, sem duplicar.
- A data e a hora mostram qual percentual valia no cálculo e separam a primeira gravação de um recálculo.
- A tabela é nova (nome sugerido está no refinamento técnico).

---

## 3. Dúvidas levantadas e respostas

| #      | Quem perguntou | Dúvida                                                                              | Resposta                                                                                                                                                                                |
| :----- | :------------- | :---------------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **1**  | Luciane        | O ISS já existe e tem valor? Muda algo?                                             | Sim, o ISS de 2% já é usado hoje e não muda.                                                                                                                                            |
| **2**  | Luciane        | Na Recarga Certa o pedido nasce com valor zero. Em que momento o cálculo é refeito? | No retorno da Recarga Certa: a rotina da RC e a tela do CRM em que se sobe o arquivo de retorno. Só depois dessa rotina o pedido é recalculado. Luciane pediu para detalhar isso na US. |
| **3**  | Jose Alan      | Não entendeu o critério "base do ISS é o total das taxas do pedido".                | O imposto é cobrado só sobre taxas e tarifas. Soma-se o total das taxas e tarifas (valor da NF), tira-se os 2% de ISS, e o resultado é a base.                                          |
| **4**  | Raiela         | No exemplo, o valor de R$ 100,00 é do pedido ou das taxas?                          | Das taxas do pedido. O ISS é calculado sobre as taxas, e os demais impostos sobre a base.                                                                                               |
| **5**  | Luciane        | Onde consultar as taxas do pedido?                                                  | No CRM, na tela de detalhes do pedido, que mostra as taxas e o total de taxas. Também estão no consolidado da nota fiscal.                                                              |
| **6**  | Raiela         | O arredondamento é feito em cada passo ou só no final?                              | Em cada imposto. IBS e CBS não são somados.                                                                                                                                             |
| **7**  | Luciane        | Arredondar cada imposto pode dar diferença considerável no total?                   | Como os impostos não serão somados e o lançamento é separado, a diferença não afeta. Se no futuro for preciso a soma, talvez chame o balanceador novamente.                             |
| **8**  | Raiela         | O que é o balanceador?                                                              | Desenvolvimento existente para corrigir diferença de um centavo. O cálculo novo não usa o balanceador, mas precisa rodar depois dele para pegar as taxas ajustadas.                     |
| **9**  | Wallyson       | A base é total das taxas menos o ISS, ou multiplicado?                              | Menos (subtração).                                                                                                                                                                      |
| **10** | Renata         | A regra de cálculo fica em um lugar só ou uma no microsserviço e outra no legado?   | Em um endpoint do microsserviço de cálculo de pedidos (sem duplicar código), válido para 1.0 e 2.0.                                                                                     |
| **11** | Luciane        | Qual é o nome da tabela? Já existe?                                                 | É nova. O nome sugerido está no refinamento técnico e será incluído na US.                                                                                                              |
| **12** | Luciane        | As alíquotas vão aparecer no demonstrativo da fatura e na nota fiscal?              | Não vamos confrontar esses valores agora. Uma opção de teste é cadastrar a alíquota teste e comparar.                                                                                   |
| **13** | Raiela         | O total das taxas já existe e está correto? Está no escopo mexer nele?              | Já existe e é calculado. Não está no escopo mexer; só vamos usar o valor no cálculo.                                                                                                    |
| **14** | Matheus        | Existe documento oficial do governo com essas regras?                               | Ainda não está na US. Renata vai incluir a fonte.                                                                                                                                       |
| **15** | Matheus        | Há telas/relatórios que vão mostrar esses valores para Contabilidade/Faturamento?   | Existe outra US para relatório. Fabiana levantou os pontos impactados e vai reenviar. Matheus sugeriu que as áreas indiquem as telas mais importantes.                                  |
| **16** | Jose Alan      | Nesta US só se atua no pedido, sem chegar na nota fiscal?                           | Sim. O cálculo serve para o lançamento contábil. Recomenda-se rodar o regressivo da nota para garantir que nada quebrou.                                                                |
| **17** | Luciane        | As novas alíquotas vão estar nos arquivos de envio à Prefeitura?                    | O layout não mudou. Hoje é a própria Prefeitura que calcula.                                                                                                                            |
| **18** | Luciane        | Haverá apresentação às áreas demandantes depois da homologação?                     | Sim. Como o resultado vai para a Contabilidade, é importante ter a validação deles.                                                                                                     |
| **19** | Wallyson       | O ISS não vai estar na tabela de alíquotas? Vamos ter de inserir?                   | Fabiana vai incluir na tabela e tirar o valor fixo do código. Em DEV está 1%; em PRD não está na tabela.                                                                                |
| **20** | Wallyson       | "Todas as taxas" são as que aparecem no detalhe do pedido?                          | Sim. No código há um ponto que já consolida o total das taxas.                                                                                                                          |
| **21** | Wallyson       | Na base é divisão ou subtração?                                                     | Subtração. IBS e CBS calculados sobre a base para evitar imposto sobre imposto.                                                                                                         |
| **22** | Wallyson       | O que é "crédito fora da base"? O imposto abate no crédito?                         | Valor de recarga não entra na base. As tarifas já abatem do crédito hoje, isso não muda.                                                                                                |

---

## 4. O que foi combinado

1. **Base de cálculo:** total das taxas do pedido − ISS (2%), por subtração. O crédito do cliente não entra.
2. **Alíquotas (01/01/2027 a 31/12/2028):** IBS 0,10% e CBS 11,20%, lidas do parâmetro com vigência baseada na geração do pedido.
3. **Arredondamento:** 2 casas, `HALF_UP`, aplicado em cada imposto separadamente.
4. **Tratamento IBS/CBS:** não são somados. Cada imposto é calculado, arredondado e lançado contabilmente de forma individual.
5. **Momento do cálculo:** na geração do pedido, depois do balanceador já existente.
6. **Entrada do cálculo:** o total das taxas já balanceado. Não será alterado.
7. **Gravação:** três linhas por pedido (ISS, IBS e CBS), em tabela nova, com base, alíquota, valor e data/hora.
8. **Recarga Certa:** recalcula no retorno da RC e substitui as linhas gravadas.
9. **Regimes:** 2026 (Legado) não calcula/grava. Reforma Tributária calcula/grava. Virada pode ser automática em 01/01/2027.
10. **Portais 1.0 e 2.0:** mesmo resultado via microsserviço.
11. **Contabilização (OPOPS):** usa os valores gravados, sem recalcular.
12. **Finalidade:** lançamento contábil (emissão da nota não muda nesta US).
13. **ISS:** passa a ser lido da tabela de alíquotas.
14. **Apresentação:** resultado será validado com a Contabilidade após homologação.
15. **Regressivo:** rodar testes regressivos de emissão de nota.

---

## 5. Pontos pendentes

| #     | Pendência                              | Detalhe                                                                                            | Quem resolve                        |
| :---- | :------------------------------------- | :------------------------------------------------------------------------------------------------- | :---------------------------------- |
| **1** | Cadastro do ISS na tabela de alíquotas | Em DEV o ISS está com 1%. Cadastrar 2% em todos os ambientes, com vigência correta.                | Fabiana                             |
| **2** | Nome da tabela nova                    | Incluir na US o nome definido no refinamento técnico.                                              | Fabiana                             |
| **3** | Detalhar momento do recálculo da RC    | Deixar claro na US que ocorre no retorno (rotina da RC e tela do CRM).                             | Renata (apoio Fabiana)              |
| **4** | Fonte oficial das regras               | Incluir na US a referência oficial do governo.                                                     | Renata                              |
| **5** | Telas e relatórios impactados          | Reenviar levantamento e validar com as áreas quais são prioritários.                               | Fabiana (envio), Renata (validação) |
| **6** | Confronto com nota da Prefeitura       | Não será feito agora. Avaliar no futuro com alíquota teste de 2026.                                | Renata e Fabiana                    |
| **7** | Layout da Prefeitura                   | A Prefeitura não sinalizou se o layout vai mudar.                                                  | Renata / Tributário                 |
| **8** | Critério sobre PIS/COFINS              | Decidir se mantém ou retira o critério "PIS/COFINS não é deduzido" para não confundir.             | Renata                              |
| **9** | Diferença por arredondamento           | Confirmar se o valor da nota da Prefeitura vai seguir a mesma lógica de arredondamento individual. | Tributário                          |

---

## 6. Próximos passos

1. A Fabiana cadastra o ISS na tabela de alíquotas, retira o valor fixo do código e informa o nome da tabela nova na US.
2. A Renata atualiza a US com o detalhe da Recarga Certa, a fonte oficial e a decisão sobre o critério de PIS/COFINS.
3. A Fabiana reenvia o levantamento de telas e relatórios impactados. A Renata valida as prioridades com a Contabilidade e o Faturamento.
4. O time de QA prepara os cenários de teste conforme o BDD da US, incluindo o regressivo da emissão de nota.
5. Depois da homologação, o time apresenta o resultado à Contabilidade para validação.
6. O Wallyson e o time de desenvolvimento releem a US e registram novas dúvidas no chat.

---

## 7. Observação para a próxima revisão da US

Durante a explicação do arredondamento, a regra foi descrita de forma confusa em um trecho. Vale reforçar na US: se a terceira casa for 5 ou mais, a segunda casa sobe; se for menos de 5, fica como está (ex.: `0,098` vira `0,10`; `10,976` vira `10,98`; `0,005` vira `0,01`; `0,004` vira `0,00`).

---

## 8. Tabela de tarefas e responsáveis

| #      | Tarefa                                                                                         | Responsável                 | Status   |
| :----- | :--------------------------------------------------------------------------------------------- | :-------------------------- | :------- |
| **1**  | Cadastrar o ISS (2%) na tabela de alíquotas em DEV, HML e PRD, corrigindo o valor de 1% em DEV | Fabiana Santos Leão         | Pendente |
| **2**  | Retirar do código o valor fixo do ISS e passar a ler da tabela de alíquotas                    | Fabiana / Desenvolvimento   | Pendente |
| **3**  | Informar na US o nome da tabela nova                                                           | Fabiana Santos Leão         | Pendente |
| **4**  | Detalhar na US o momento do recálculo da Recarga Certa (retorno da RC)                         | Renata Cristina             | Pendente |
| **5**  | Incluir na US a fonte oficial das regras e alíquotas                                           | Renata Cristina             | Pendente |
| **6**  | Decidir se mantém ou retira o critério sobre PIS/COFINS                                        | Renata Cristina             | Pendente |
| **7**  | Reenviar o levantamento de telas e relatórios impactados                                       | Fabiana Santos Leão         | Pendente |
| **8**  | Validar com a Contabilidade e o Faturamento as telas e relatórios prioritários                 | Renata Cristina             | Pendente |
| **9**  | Preparar os cenários de teste do BDD, incluindo o regressivo da emissão de nota                | Luciane, Raiela e Jose Alan | Pendente |
| **10** | Reler a US e registrar dúvidas no chat                                                         | Wallyson / Matheus          | Pendente |
| **11** | Apresentar o resultado à Contabilidade depois da homologação                                   | Renata e Fabiana            | Pendente |
| **12** | Acompanhar a definição de layout da Prefeitura                                                 | Renata / Tributário         | Pendente |
| **13** | Confirmar se a nota da Prefeitura segue o arredondamento por imposto                           | Tributário                  | Pendente |
