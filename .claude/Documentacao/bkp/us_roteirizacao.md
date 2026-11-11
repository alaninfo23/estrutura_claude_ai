# [API Bradesco] 4 - VT-Roteirização | Solicitação e Acompanhamento de Roteirização de Colaboradores

## 📝 Descrição de Negócio

Viabilizar que o Bradesco execute integralmente o processo de roteirização de Vale-Transporte através de APIs, permitindo identificar colaboradores aptos, solicitar o cálculo automatizado de trajetos, acompanhar o processamento e consumir as rotas sugeridas para suportar a concessão correta do benefício aos colaboradores.

---

## 🎯 Objetivo

Como sistema integrador do Bradesco,  
Quero identificar colaboradores elegíveis para roteirização, solicitar o processamento de suas rotas e acompanhar a execução do processamento,  
Para que seja possível calcular automaticamente os trajetos de Vale-Transporte dos colaboradores, disponibilizando posteriormente as informações necessárias para aprovação, concessão do benefício e geração dos pedidos.

---

## 🌐 Contexto

Após o cadastro dos colaboradores e seus respectivos benefícios, é necessário realizar o processo de roteirização para identificar os trajetos utilizados pelo colaborador entre residência e local de trabalho.

O microsserviço **VT-Roteirização** será responsável por disponibilizar APIs para:

- Identificação dos colaboradores pendentes de roteirização
- Solicitação da roteirização
- Acompanhamento do processamento
- Consulta detalhada das rotas calculadas

O processamento ocorre de forma **assíncrona** e envolve integrações com motores responsáveis pelo cálculo dos trajetos e identificação dos produtos de transporte aplicáveis.

---

## 📌 Escopo

Evoluir o microsserviço **VT-Roteirização** para disponibilizar os seguintes recursos:

### 1. Listagem de Colaboradores Pendentes de Roteirização

`GET /companies/{companyId}/beneficiaries`

Permitir identificar colaboradores ativos que necessitam de roteirização.  
_Diferente do modelo legado, a API deverá retornar todos os colaboradores ativos da empresa, informando sua elegibilidade para roteirização._

#### Dados Retornados

- **Dados do Colaborador:**
  - Id do colaborador
  - Nome
  - CPF
  - Matrícula
  - Departamento
  - Endereço de trabalho
- **Dados de Elegibilidade:**
  - `eligible`
  - `ineligibilityReason`

#### Regras de Elegibilidade

| `eligible` | Descrição                              |
| :--------- | :------------------------------------- |
| `true`     | Colaborador apto para roteirização     |
| `false`    | Colaborador não apto para roteirização |

#### Motivos de Inelegibilidade (`ineligibilityReason`)

| Código                 | Descrição                             |
| :--------------------- | :------------------------------------ |
| `MISSING_HOME_ADDRESS` | Endereço residencial não cadastrado   |
| `MISSING_WORK_ADDRESS` | Endereço de trabalho não cadastrado   |
| `ROUTE_IN_PROGRESS`    | Possui roteirização em processamento  |
| `ROUTE_ACTIVE`         | Possui rota ativa aprovada/finalizada |

#### Filtros Disponíveis

- Nome
- CPF
- Departamento
- Endereço de trabalho
- Elegibilidade
- Status da última roteirização

_A API deverá suportar Paginação e Ordenação._

---

### 2. Solicitação de Roteirização

`POST /companies/{companyId}/routing-requests`

Permitir solicitar o processamento de roteirização para um ou mais colaboradores.  
O processamento deverá ocorrer de forma **assíncrona**.

#### Retorno Esperado

- **HTTP 202** (Accepted)
- `routingRequestId` (O identificador gerado permitirá o acompanhamento posterior do processamento).

#### Identificação do Operador

O usuário responsável pela solicitação será recebido através do header:

- `X-Logged-User-Id` _(injetado automaticamente pelo API Gateway Sensedia)_

As informações deverão ser utilizadas para:

- Auditoria
- Histórico operacional
- Rastreabilidade

#### Pipeline de Processamento

A solicitação deverá executar as seguintes etapas:

1. **INS:** Envio dos dados para cálculo das rotas (`CPF`, `Endereço residencial`, `Endereço de trabalho`).
2. **CON:** Consulta dos trajetos calculados (`Ida`, `Volta`).
3. **COM:** Consulta da codificação de compra (`Produtos`, `Tarifas`, `Modalidades`).
4. **END:** Atualização do status para **Aguardando Aprovação**.
5. **GER:** Geração dos documentos associados à rota.

> ⏱️ **Tempo Estimado:** O processamento poderá variar entre **30 segundos até 6 minutos por colaborador**, dependendo da complexidade da rota calculada.

#### Validações

A requisição deverá retornar **HTTP 422** quando:

- Existirem colaboradores de outra empresa.
- Houver colaborador sem endereço residencial válido.
- Houver colaborador sem endereço de trabalho válido.
- Existir roteirização em processamento para algum colaborador selecionado.

#### Cancelamento Automático

Lotes que permanecerem sem processamento por **mais de 24 horas** deverão ser cancelados automaticamente.

---

### 3. Consulta de Status do Lote

`GET /companies/{companyId}/routing-requests/{routingRequestId}`

Permitir o acompanhamento do processamento assíncrono.

#### Status Possíveis

| Status                  | Descrição                                  |
| :---------------------- | :----------------------------------------- |
| `PROCESSING`            | Rotas em processamento                     |
| `COMPLETED`             | Todas as rotas processadas com sucesso     |
| `COMPLETED_WITH_ERRORS` | Processamento concluído com erros parciais |
| `ERROR`                 | Falha geral do processamento               |

#### Dados Retornados

- `routingRequestId`
- Data da solicitação
- Quantidade de colaboradores
- Quantidade processada
- Quantidade com erro
- Status atual
- Mensagens de erro (quando aplicável)

---

### 4. Consulta de Detalhes da Rota

`GET /companies/{companyId}/routes/{routeId}`

Permitir consultar o resultado detalhado da roteirização processada.

#### Informações Retornadas

- **Dados Gerais:** Identificador da rota, Data de geração, Status da rota
- **Percursos:** Trajeto de ida, Trajeto de volta
- **Transporte:** Linhas utilizadas, Modal de transporte, Integrações
- **Benefício Sugerido:** Produtos recomendados, Valor estimado de compra, Quantidade sugerida
- **Informações Financeiras:** Economia calculada, Comparativo de opções
- **Contestação:** Status da contestação, Histórico da contestação

---

## 🚫 Fora do Escopo

1. Aprovação de roteirizações
2. Contestação de rotas
3. Compra de Vale-Transporte
4. Emissão de cartões
5. Processamento financeiro
6. Gestão cadastral de empresas
7. Gestão cadastral de colaboradores
8. Parametrização de produtos
9. Geração de pedidos de compra

---

## 📐 Regras de Negócio e Critérios de Aceite

- **RN01:** A API deverá retornar todos os colaboradores ativos da empresa, independentemente de estarem aptos ou não para roteirização.
- **RN02:** A elegibilidade deverá ser apresentada através do atributo `eligible`.
- **RN03:** Quando o colaborador não estiver apto, deverá ser informado o motivo através do atributo `ineligibilityReason`.
- **RN04:** Somente colaboradores elegíveis poderão ser enviados para processamento de roteirização.
- **RN05:** A solicitação deverá ocorrer de forma assíncrona.
- **RN06:** O endpoint de solicitação deverá retornar HTTP 202 e o identificador do lote.
- **RN07:** O sistema deverá registrar o operador responsável através do header `X-Logged-User-Id`.
- **RN08:** Não deverá ser permitido iniciar nova roteirização para colaboradores que possuam processamento em andamento.
- **RN09:** Todos os colaboradores enviados no lote deverão pertencer à empresa informada.
- **RN10:** A consulta de status deverá permitir o acompanhamento integral do processamento.
- **RN11:** A consulta de rota deverá disponibilizar o detalhamento completo do cálculo realizado.
- **RN12:** Lotes não processados dentro de 24 horas deverão ser cancelados automaticamente.

### Regra de Exibição dos Trajetos

Para o endpoint de busca de trajetos, deverá ser considerado o retorno da **SAT** conforme recebido, sem tratamento ou alteração dos dados:

- Caso a SAT retorne informações com espaços, o valor deverá ser mantido conforme recebido.
- Quando não houver determinada informação e a SAT retornar `“SEM DENOMINACAO”`, o mesmo valor deverá ser considerado.
- Essa regra deverá ser aplicada tanto na exibição no **Portal** quanto na geração do arquivo **PDF**.
- Neste momento, **não será realizada normalização ou tratamento desses dados**.
- Caso seja identificada posteriormente alguma necessidade de ajuste na exibição, o comportamento deverá ser avaliado junto à SAT.

---

## 🥒 BDD (Behavior-Driven Development)

```gherkin
Cenário 1 - Consultar colaboradores elegíveis para roteirização
  Dado que existam colaboradores ativos cadastrados na empresa
  Quando a API for consultada
  Então deverá ser retornada a lista de colaboradores
  E cada colaborador deverá possuir o indicador de elegibilidade

Cenário 2 - Colaborador sem endereço residencial
  Dado um colaborador sem endereço residencial cadastrado
  Quando a API for consultada
  Então o colaborador deverá ser retornado
  E o campo eligible deverá ser igual a false
  E o motivo deverá ser MISSING_HOME_ADDRESS

Cenário 3 - Solicitar roteirização com sucesso
  Dado colaboradores elegíveis para roteirização
  Quando a solicitação for realizada
  Então deverá ser criado um lote de processamento
  E deverá ser retornado HTTP 202
  E deverá ser gerado um routingRequestId

Cenário 4 - Colaborador com rota em processamento
  Dado um colaborador com roteirização em andamento
  Quando for realizada nova solicitação
  Então a requisição deverá ser rejeitada
  E deverá ser retornado HTTP 422

Cenário 5 - Consultar status do lote
  Dado um lote previamente criado
  Quando o endpoint de status for acionado
  Então deverá ser retornado o status atual do processamento

Cenário 6 - Processamento concluído
  Dado um lote processado com sucesso
  Quando o status for consultado
  Então deverá ser retornado o status COMPLETED

Cenário 7 - Processamento concluído com erros
  Dado um lote processado parcialmente
  Quando o status for consultado
  Então deverá ser retornado o status COMPLETED_WITH_ERRORS
  E as ocorrências deverão ser apresentadas

Cenário 8 - Consultar detalhe da rota
  Dado uma roteirização concluída
  Quando a rota for consultada
  Então deverão ser apresentados os percursos de ida e volta
  E as linhas sugeridas
  E os produtos recomendados
  E os valores calculados para concessão do benefício
```
