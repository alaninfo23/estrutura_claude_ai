# Evolucao do vt-roteirizacao e vt-roteirizacao-sat

**Spike:** https://dev.azure.com/alelo/ValeTransporte-2.0/_workitems/edit/1688004
**Swagger de referencia:** [`api-vt-roteirizacao-minimo-swagger 7 1.yaml` (MVP)](https://elopar.sharepoint.com/:u:/r/sites/ValeTransporte4/Shared%20Documents/Marte/Refinamentos/Spike%20-%20%5B1688004%5D%20%5BAPI%20Bradesco%5D%5BRoteiriza%C3%A7%C3%A3o%5D%20An%C3%A1lise%20da%20roteiriza%C3%A7%C3%A3o%20atual%20feita%20no%20VT%20e%20refinamento%20da%20API%20pela%20qual%20o%20banco%20se%20integrar%C3%A1/api-roteirizacao-bradesco/docs/materiais_arquitetura/api-vt-roteirizacao-minimo-swagger%207%201.yaml?d=w3166abbf466c4c39b9dd76bd0d341c85&csf=1&web=1&e=AXMD6b)
**Consulta de dúvidas:**: https://elopar.sharepoint.com/:t:/r/sites/ValeTransporte4/Shared%20Documents/Marte/Refinamentos/Spike%20-%20%5B1688004%5D%20%5BAPI%20Bradesco%5D%5BRoteiriza%C3%A7%C3%A3o%5D%20An%C3%A1lise%20da%20roteiriza%C3%A7%C3%A3o%20atual%20feita%20no%20VT%20e%20refinamento%20da%20API%20pela%20qual%20o%20banco%20se%20integrar%C3%A1/D%C3%BAvidas%20sobre%20contrato%20API%20roteirizacao.md?d=w72e3624955204b8190aa0f360f197aa7&csf=1&web=1&e=cpGzBO

![Fluxo - Roteirização (vt-roteirizacao e vt-roteirizacao-sat).drawio.png](</.attachments/Fluxo%20-%20Roteirização%20(vt-roteirizacao%20e%20vt-roteirizacao-sat).drawio-537c7763-8c58-4469-859a-17f625cfb827.png>)

---

## Contexto

Os servicos `vt-roteirizacao` e `vt-roteirizacao-sat` serao evoluidos para suportar
uma API generica de roteirizacao, pronta para ser consumida por qualquer cliente
(web, mobile, B2B).

O objetivo desta documentacao e detalhar as adaptacoes necessarias nos 4 endpoints
do MVP, servindo de base para o refinamento e desenvolvimento.

---

## Fluxo geral

```
Cliente -> Sensedia (gateway) -> vt-roteirizacao -> MQ -> vt-roteirizacao-sat -> SAT (SOAP)
```

O Sensedia gerencia a autenticacao e injeta o header `X-Logged-User-Id` com o ID
do usuario logado. O `vt-roteirizacao` usa esse valor apenas para auditoria,
gravando-o em `ID_USUCRI_RTR`. A empresa e identificada pelo `companyId` no path
de cada endpoint.

---

## Origem dos registros

Pontos importantes sobre o ciclo de vida dos registros envolvidos no fluxo de roteirizacao:

- O registro em `ROTA_TRANSP_ROTEIRIZACAO` **nao e criado no `vt-roteirizacao`**. Ele e criado no `vt-roteirizacao-sat`, dentro do consumer da fila `mq.vt.routes.sat.add`, em `RoteirizacaoSatUsecaseImpl.inseriRotaSat()`, com status inicial `PR` (Processando).
- Logo apos a insercao, um registro de historico tambem e gravado em `ROTA_TRANSP_ROTEIRIZACAO_HIST`.
- O `vt-roteirizacao` e responsavel por obter o numero do lote via `SEQ_ROTA_TRANSP_LOTE` (NEXTVAL) e enviar cada colaborador individualmente para a fila. O identificador do lote (`ID_LOTROT_RTR`) e o mesmo para todos os colaboradores de um mesmo request.
- Os status disponiveis para acompanhamento estao em `FlagStatusRoteirizacaoEnum` no `vt-roteirizacao`:

| Codigo | Constante                  | Descricao                             |
| ------ | -------------------------- | ------------------------------------- |
| `PR`   | `PROCESSANDO_ROTA`         | Processando rota                      |
| `ET`   | `ERRO_TECNICO`             | Erro tecnico (excecao)                |
| `E`    | `ERRO`                     | Erro de negocio                       |
| `AP`   | `AGUARDANDO_APROVACAO`     | Aguardando aprovacao (rota calculada) |
| `CA`   | `CONTESTACAO_ABERTA`       | Contestacao aberta                    |
| `CR`   | `CONTESTACAO_RESPONDIDA`   | Contestacao respondida                |
| `AC`   | `APROVADO_COLABORADOR`     | Aprovado pelo colaborador             |
| `RC`   | `REPROVADO_COLABORADOR`    | Reprovado pelo colaborador            |
| `PEN`  | `PENDENTE_APROVACAO_FINAL` | Pendente aprovacao final              |
| `F`    | `FINALIZADO`               | Finalizado                            |

---

## Decisoes de implementacao

| Topico                 | Decisao                                                                                                                                                                                                                                                     |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Logs                   | Utilizar a dependencia `audit-logger` (referencia: `vt-orders-calculation`)                                                                                                                                                                                 |
| Arquitetura            | Manter fila MQ e os dois microsservicos sem mudanca estrutural                                                                                                                                                                                              |
| `cdEmpresa` no SAT     | Configurado via variavel de ambiente `ENV_SAT_CD_EMPRESA`, lida em `application.yml` como `sat.cdEmpresa` e injetada com `@Value("${sat.cdEmpresa}")` em `RoteirizacaoSatUsecaseImpl`                                                                       |
| Nova classe controller | `RoteirizacaoV2Controller` para os 4 endpoints MVP (paths `/v2/companies`), mantendo o legado `RoteirizacaoController` (paths `/routes-services`) e `RoteirizacaoAprovacaoController` (paths `/routes-approve-services`) funcionando normalmente para o CRM |
| Teste de performance   | Testes de performance serao realizados para verificar o comportamento do endpoint ao roteirizar multiplos usuarios. Sem limite pre-definido de beneficiarios por lote.                                                                                      |

---

## Mapeamento: endpoints legados para MVP

| Funcionalidade         | Controller legado                 | Path legado                                                    | Controller MVP             | Path novo (MVP)                                            | Acao  |
| ---------------------- | --------------------------------- | -------------------------------------------------------------- | -------------------------- | ---------------------------------------------------------- | ----- |
| Listar colaboradores   | `RoteirizacaoController`          | `GET /routes-services/companies/{id}/users`                    | `RoteirizacaoV2Controller` | `GET /v2/companies/{companyId}/beneficiaries`              | CRIAR |
| Solicitar roteirizacao | `RoteirizacaoController`          | `POST /routes-services/companies/{id}/insert`                  | `RoteirizacaoV2Controller` | `POST /v2/companies/{companyId}/routing-requests`          | CRIAR |
| Status do lote         | `RoteirizacaoController`          | `GET /routes-services/batch/{id}/status`                       | `RoteirizacaoV2Controller` | `GET /v2/companies/{companyId}/routing-requests/{batchId}` | CRIAR |
| Detalhe da rota        | `RoteirizacaoAprovacaoController` | `GET /routes-approve-services/companies/{id}/routes/{id}/data` | `RoteirizacaoV2Controller` | `GET /v2/companies/{companyId}/routes/{routeId}`           | CRIAR |

**Estrategia de implementacao:**

- **Legado:** endpoints do CRM continuam em `RoteirizacaoController` e `RoteirizacaoAprovacaoController` (paths `/routes-services` e `/routes-approve-services`)
- **MVP Bradesco:** novos endpoints em `RoteirizacaoV2Controller` (path `/v2/companies`)
- **Reutilizacao:** ambos os controllers utilizam os mesmos usecases e repositories, evitando duplicacao de codigo
- **Compatibilidade:** sem impacto no fluxo legado do CRM. Os dois controllers coexistem na mesma aplicacao

---

## Parametros de entrada comuns

Parametros presentes em todos os endpoints que requerem tratamento centralizado.

**DE-PARA: parametros de entrada x banco de dados**

| Parametro          | Tipo   | Tabela / Coluna                                                    | Validacao                                                                                                                                        |
| ------------------ | ------ | ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `X-Logged-User-Id` | Header | `ROTA_TRANSP_ROTEIRIZACAO.ID_USUCRI_RTR`                           | **Nao validar na aplicacao**. Apenas propagar para auditoria. O Sensedia e responsavel pela autenticacao e o header e injetado com um ID valido. |
| `companyId`        | Path   | `ROTA_TRANSP_ROTEIRIZACAO.ID_EMPRES_EMP` → `EMPRESA.ID_EMPRES_EMP` | **Validar existencia** antes de processar: checar se a empresa existe.                                                                           |

> **Recomendacao de implementacao:** criar uma **anotacao customizada** (ex: `@ValidateRoutingParams`) aplicada apenas nos metodos do controller que necessitam validacao de `companyId` e `X-Logged-User-Id`.

---

## Endpoint 1: Listar colaboradores

**`GET /companies/{companyId}/beneficiaries`**

Lista os colaboradores da empresa com indicacao de quem esta apto para roteirizacao.

**Principais mudancas em relacao ao legado:**

| Campo novo                          | Origem                                                         |
| ----------------------------------- | -------------------------------------------------------------- |
| `beneficiaryId`                     | `idUsuario` (renomear)                                         |
| `fullName`                          | `nomeUsuario` (renomear)                                       |
| `registrationNumber`                | `codMatricula` (renomear)                                      |
| `beneficiaryDocumentNumber`         | `cpf` (renomear)                                               |
| `department`                        | join com `EmpresaDepartamentoEntity`                           |
| `position`                          | join com `EMPRESA_CARGO` (adicionar)                           |
| `homeAddress`                       | via `EMPRESA_USUARIO.ID_ENDRES_END` → `ENDERECO.ID_ENDERE_END` |
| `workAddress`                       | via `EMPRESA_USUARIO.ID_ENDTRA_END` → `ENDERECO.ID_ENDERE_END` |
| `eligible` + `ineligibilityReason`  | logica nova (ver abaixo)                                       |
| `lastRouteStatus` + `lastRouteDate` | subquery em `ROTA_TRANSP_ROTEIRIZACAO`                         |
| `totalEligible`                     | count separado, independente da paginacao                      |

**Remover:** envelope `{ status, mensagem, response }` e campo `isMatriz`.

**Novos filtros:** `eligible`, `lastRouteStatus`, `workAddressId`.

**Logica de elegibilidade:**

Um colaborador e elegivel (`eligible = true`) quando:

- `EMPRESA_USUARIO.ID_USUARI_EUS` está preenchido (PK)
- `EMPRESA_USUARIO.ID_ENDRES_END` está preenchido (endereco residencial)
- `EMPRESA_USUARIO.ID_ENDTRA_END` está preenchido (endereco de trabalho)
- Não possui processo aberto em `ROTA_TRANSP_ROTEIRIZACAO` com status: `PR`, `AP`, `CA`, `CR`, `AC`, `PEN`

| `ineligibilityReason`  | Condicao                                                         |
| ---------------------- | ---------------------------------------------------------------- |
| `MISSING_HOME_ADDRESS` | sem endereco residencial                                         |
| `MISSING_WORK_ADDRESS` | sem endereco de trabalho                                         |
| `ROUTE_IN_PROGRESS`    | processo aberto com status `PR`, `AP`, `CA`, `CR`, `AC` ou `PEN` |
| `ROUTE_ACTIVE`         | rota finalizada (status `F`)                                     |

Colaboradores inativos (`FL_ATIVOS_EUS != 'S'`) sao filtrados na consulta e nao aparecem na lista.

**Logica de elegibilidade — detalhada:**

| Situacao                                                          | `eligible` | `ineligibilityReason`  | Observacao                             |
| ----------------------------------------------------------------- | ---------- | ---------------------- | -------------------------------------- |
| Sem `ID_ENDRES_END`                                               | `false`    | `MISSING_HOME_ADDRESS` | —                                      |
| Sem `ID_ENDTRA_END`                                               | `false`    | `MISSING_WORK_ADDRESS` | —                                      |
| `FL_STATUS_RTR IN ('PR', 'AP', 'CA', 'CR', 'AC', 'PEN')`          | `false`    | `ROUTE_IN_PROGRESS`    | Processo aberto (bloqueado)            |
| `FL_STATUS_RTR = F`                                               | `false`    | `ROUTE_ACTIVE`         | Rota finalizada/aprovada               |
| Tem os dois enderecos + sem processo aberto                       | `true`     | `null`                 | Pronto para roteirizar                 |
| Tem os dois enderecos + status `E`, `ET` ou `RC` (erro/reprovado) | `true`     | `null`                 | Pode re-roteirizar apos falha anterior |

**DE-PARA: campos do response x banco de dados**

| Campo do response            | Tabela / Coluna                          | Observacao                                                                                                                                     |
| ---------------------------- | ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `beneficiaryId`              | `EMPRESA_USUARIO.ID_USUARI_EUS`          | PK da tabela                                                                                                                                   |
| `companyId`                  | `EMPRESA.ID_EMPRES_EMP`                  | Via JOIN com `EMPRESA_USUARIO.ID_EMPRES_EMP`                                                                                                   |
| `beneficiaryDocumentNumber`  | `EMPRESA_USUARIO.CD_CODCPF_EUS`          | Mascarar nos logs                                                                                                                              |
| `fullName`                   | `EMPRESA_USUARIO.NM_USUARI_EUS`          |                                                                                                                                                |
| `registrationNumber`         | `EMPRESA_USUARIO.CD_USUARI_EUS`          | Matricula do colaborador                                                                                                                       |
| `department`                 | `EMPRESA_DEPARTAMENTO.DS_DEPART_EDP`     | Via `EMPRESA_USUARIO.ID_DEPART_EDP`                                                                                                            |
| `position`                   | `EMPRESA_CARGO.DS_CARGOS_ECG`            | Via `EMPRESA_USUARIO.ID_CARGOS_ECG` — JOIN nao mapeado atualmente                                                                              |
| `homeAddress.street`         | `ENDERECO.DS_LOGRAD_END`                 | Via `EMPRESA_USUARIO.ID_ENDRES_END` → `ENDERECO.ID_ENDERE_END`                                                                                 |
| `homeAddress.number`         | `ENDERECO.CD_NUMERO_END`                 |                                                                                                                                                |
| `homeAddress.complement`     | `ENDERECO.CD_COMPLE_END`                 |                                                                                                                                                |
| `homeAddress.neighborhood`   | `ENDERECO.CD_BAIRRO_END`                 |                                                                                                                                                |
| `homeAddress.city`           | `ENDERECO.CD_CIDADE_END`                 |                                                                                                                                                |
| `homeAddress.state`          | `ENDERECO.CD_ESTADO_EST`                 |                                                                                                                                                |
| `homeAddress.zipCode`        | `ENDERECO.CD_NUMCEP_CEP`                 |                                                                                                                                                |
| `homeAddress.referencePoint` | `ENDERECO.DS_REFERE_END`                 |                                                                                                                                                |
| `workAddress.*`              | mesmos campos de `homeAddress`           | Via `EMPRESA_USUARIO.ID_ENDTRA_END` → `ENDERECO.ID_ENDERE_END`                                                                                 |
| `workAddressId`              | `EMPRESA_USUARIO.ID_ENDTRA_END`          |                                                                                                                                                |
| `lastRouteStatus`            | `ROTA_TRANSP_ROTEIRIZACAO.FL_STATUS_RTR` | Ultimo registro por `ID_USUARI_EUS` + `ID_EMPRES_EMP`. Mapeamento: `RC` → `REJECTED`; `E` → `ERROR`; `ET` → `TECHNICAL_ERROR`; demais → `null` |
| `lastRouteDate`              | `ROTA_TRANSP_ROTEIRIZACAO.DT_ATUALI_RTR` | Ultimo registro por `ID_USUARI_EUS` + `ID_EMPRES_EMP`. `null` se nunca roteirizado                                                             |
| `totalEligible`              | —                                        | Contagem dos itens com `eligible = true` na lista. Calcular na camada Java apos montar o `content`, antes de retornar o response               |

> **Atencao:** a query atual `listarUsuarioRoteirizacao()` filtra com `NOT EXISTS` e retorna apenas `codMatricula`, `nomeUsuario`, `cpf` e `status`. Para o MVP sera necessario refatorar: remover o `NOT EXISTS`, adicionar JOINs com `ENDERECO`, `EMPRESA_DEPARTAMENTO`, `EMPRESA_CARGO` e subquery em `ROTA_TRANSP_ROTEIRIZACAO` para os campos de status e data.

**Query para `totalEligible`:**

```sql
SELECT COUNT(*) FROM EMPRESA_USUARIO eu
WHERE eu.ID_EMPRES_EMP = ?
  AND eu.ID_ENDRES_END IS NOT NULL
  AND eu.ID_ENDTRA_END IS NOT NULL
  AND NOT EXISTS (
    SELECT 1 FROM ROTA_TRANSP_ROTEIRIZACAO
    WHERE ID_USUARI_EUS = eu.ID_USUARI_EUS AND FL_STATUS_RTR = 'PR'
  )
```

**Exemplo de response:**

```json
{
  "content": [
    {
      "beneficiaryId": 88051342,
      "companyId": 835181,
      "beneficiaryDocumentNumber": "35657149807",
      "fullName": "MATHEUS HENRIQUE RAMOS DA SILVA",
      "registrationNumber": "00002",
      "department": "TECNOLOGIA",
      "position": "DESENVOLVEDOR",
      "homeAddress": {
        "street": "RUA DOUTOR PEDRO ZIMMERMANN",
        "number": "15",
        "complement": null,
        "neighborhood": "ITOUPAVAZINHA",
        "city": "BLUMENAU",
        "state": "SC",
        "zipCode": "89066000",
        "referencePoint": null
      },
      "workAddress": {
        "street": "RUA THEODORO PEREIRA",
        "number": "10",
        "complement": null,
        "neighborhood": "VILA NOVA",
        "city": "BLUMENAU",
        "state": "SC",
        "zipCode": "89035020",
        "referencePoint": null
      },
      "workAddressId": 1,
      "eligible": true,
      "ineligibilityReason": null,
      "lastRouteStatus": null,
      "lastRouteDate": null
    },
    {
      "beneficiaryId": 88051283,
      "companyId": 835181,
      "beneficiaryDocumentNumber": "85437339011",
      "fullName": "CARLOS ALBERTO SILVA",
      "registrationNumber": "00001",
      "department": "OPERACOES",
      "position": "ANALISTA",
      "homeAddress": null,
      "workAddress": null,
      "workAddressId": null,
      "eligible": false,
      "ineligibilityReason": "MISSING_HOME_ADDRESS",
      "lastRouteStatus": null,
      "lastRouteDate": null
    }
  ],
  "page": 0,
  "size": 10,
  "totalElements": 2,
  "totalPages": 1,
  "totalEligible": 1
}
```

---

## Endpoint 2: Solicitar roteirizacao

**`POST /companies/{companyId}/routing-requests`**

Dispara o processamento assincrono de roteirizacao para os colaboradores selecionados.

**Principais mudancas em relacao ao legado:**

| Aspecto                | Antes                                       | Depois                                                                                     |
| ---------------------- | ------------------------------------------- | ------------------------------------------------------------------------------------------ |
| Request body           | `[{idEmpresa, idUsuario, idUsuarioLogado}]` | `{beneficiaryIds: [long]}`                                                                 |
| `idEmpresa`            | no body                                     | vem do path                                                                                |
| `idUsuarioLogado`      | no body                                     | vem do header `X-Logged-User-Id`                                                           |
| HTTP status de retorno | 200                                         | **202 Accepted**                                                                           |
| Header `Location`      | ausente                                     | URL para acompanhar o lote                                                                 |
| Validacoes             | nenhuma antes de enfileirar                 | enderecos, pertencimento, sem processo aberto (status `PR`, `AP`, `CA`, `CR`, `AC`, `PEN`) |
| Response               | `{idLote, quantidadeLote}`                  | `{batchId, beneficiaryCount}`                                                              |

O controller deve ler o header `X-Logged-User-Id` via `@RequestHeader` e propaga-lo para o usecase e a mensagem MQ. O header **nao e validado** — apenas propagado para auditoria.

**DE-PARA: campos do request/response x banco de dados**

| Campo              | Direcao  | Origem no banco                          | Observacao                                                                                                   |
| ------------------ | -------- | ---------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `beneficiaryIds[]` | request  | `EMPRESA_USUARIO.ID_USUARI_EUS`          | Array de IDs dos colaboradores a roteirizar. Todos devem ser validados conforme elegibilidade do Endpoint 1. |
| `X-Logged-User-Id` | header   | `ROTA_TRANSP_ROTEIRIZACAO.ID_USUCRI_RTR` | Propagado apenas para auditoria — nao validar na aplicacao. O Sensedia e responsavel pela autenticacao.      |
| `batchId`          | response | `ROTA_TRANSP_ROTEIRIZACAO.ID_LOTROT_RTR` | ID do lote criado. Usar para consultar o status via `GET /routing-requests/{batchId}`.                       |
| `beneficiaryCount` | response | —                                        | **Quantidade de colaboradores elegíveis que foram enfileirados.** Nao inclui IDs rejeitados nas validacoes.  |

> **Validacoes (mesma logica do Endpoint 1 - elegibilidade):** aplicar as mesmas validacoes do endpoint de listagem de beneficiarios. Se qualquer colaborador falhar, retornar 422 com detalhe do erro e nao enfileirar nenhum (falha atomica).
>
> **Colaborador elegível para enfileiramento quando:**
>
> - `EMPRESA_USUARIO.ID_USUARI_EUS` preenchido
> - `EMPRESA_USUARIO.ID_ENDRES_END` preenchido (endereco residencial)
> - `EMPRESA_USUARIO.ID_ENDTRA_END` preenchido (endereco de trabalho)
> - Nao possui processo aberto: `FL_STATUS_RTR NOT IN ('PR', 'AP', 'CA', 'CR', 'AC', 'PEN')`
>
> **Rejeitaŕ com 422 se:**
>
> - Sem endereco residencial ou de trabalho
> - Nao pertence a empresa
> - Ja possui processo aberto em `ROTA_TRANSP_ROTEIRIZACAO` com status `PR`, `AP`, `CA`, `CR`, `AC` ou `PEN`
> - Colaborador inativo
> - Empresa sem contrato ativo de roteirizacao
> - Empresa nao ativa no SAT
> - Empresa inadimplente
>
> O `beneficiaryCount` retornado sera a quantidade de IDs que **passaram** nestas validacoes e foram enfileirados.
>
> **Observacao sobre limite de beneficiarios:** nao ha limite pre-definido de colaboradores por lote. Testes de performance serao realizados para verificar o comportamento do endpoint ao roteirizar multiplos usuarios e subsidiar futuras decisoes sobre limites, se necessario.

**Sobre o pipeline assincrono (INS → CON → COM → END → GER):**

O `vt-roteirizacao` envia cada colaborador para filas MQ. O `vt-roteirizacao-sat` consome de cada fila e processa de forma assíncrona, transitando pelo pipeline:

| Etapa (enum)             | Codigo MQ | Fila MQ                         | Descricao                                                      |
| ------------------------ | --------- | ------------------------------- | -------------------------------------------------------------- |
| `INSERIR`                | `INS`     | `mq.vt.routes.sat.add`          | Envia dados ao SAT (`InseriRota`)                              |
| `CONSULTAR`              | `CON`     | `mq.vt.routes.sat.consultation` | Consulta trajetos calculados (`ConsultaRoteirizacao`)          |
| `COMPRAR`                | `COM`     | `mq.vt.routes.sat.purchase`     | Consulta codificacao de compra (`ConsultaCodificacaoCompraV2`) |
| `FINALIZAR_ROTEIRIZACAO` | `END`     | `mq.vt.routes.sat.end`          | Marca como aguardando aprovacao (status `AP`)                  |
| `GERAR_DOC`              | `GER`     | `mq.vt.routes.docs.generate`    | Gera documentos da rota                                        |

> **Observacao importante:** cada fila MQ tem um listener em `vt-roteirizacao-sat` que consome as mensagens e executa a etapa correspondente. A ordem de processamento e respeitada: uma message nao avanca para a proxima fila ate que a etapa atual seja concluida com sucesso. Se uma etapa falhar, a message permanece na fila (com redelivery) ou, com um `ErrorHandler` configurado (ver task 6.2), gravam status ERRO + mensagem em `DS_ERRMSG_RTR`.

**Exemplo de request:**

```json
POST /companies/835181/routing-requests
X-Logged-User-Id: <id-do-usuario-logado>

{
  "beneficiaryIds": [88051342, 88051283]
}
```

**Exemplo de response (HTTP 202):**

```json
HTTP/1.1 202 Accepted
Location: /v2/companies/835181/routing-requests/101

{
  "batchId": 101,
  "beneficiaryCount": 2
}
```

**Exemplo de response de erro (HTTP 422):**

```json
{
  "timestamp": "2026-08-17T15:00:00-03:00",
  "status": 422,
  "error": "Unprocessable Entity",
  "message": "Validacao falhou para 1 colaborador(es)",
  "errors": [
    {
      "field": "beneficiaryIds[1]",
      "message": "Colaborador 88051283 sem endereco residencial cadastrado",
      "code": "ROT-013"
    }
  ]
}
```

---

## Endpoint 3: Status do lote

**`GET /companies/{companyId}/routing-requests/{batchId}`**

Retorna o progresso do processamento assincrono, com detalhamento por colaborador.
O endpoint legado retornava apenas `{loteProcessado: true/false}`.

**Como consultar os dados:**

O `batchId` recebido no path param corresponde ao campo `ROTA_TRANSP_ROTEIRIZACAO.ID_LOTROT_RTR` (ID do lote).

A partir dele:

1. Buscar todos os registros em `ROTA_TRANSP_ROTEIRIZACAO` com `ID_LOTROT_RTR = batchId`
2. Calcular totalizadores (`processedCount`, `errorCount`, `pendingCount`, `status`, `completionPercentage`)
3. Montar array `beneficiaries` com JOIN: `ROTA_TRANSP_ROTEIRIZACAO` (JOIN) → `EMPRESA_USUARIO` para obter dados do colaborador

**DE-PARA: campos do response x banco de dados**

| Campo                                     | Origem no banco                          | Observacao                                                                                                                                                |
| ----------------------------------------- | ---------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `status`                                  | derivado de `FL_STATUS_RTR`              | Ver tabela de status do lote abaixo                                                                                                                       |
| `totalCount`                              | —                                        | Contagem dos registros do lote. Retorna 0 se vazio.                                                                                                       |
| `processedCount`                          | —                                        | Contagem de registros com status final (nao `PR`).                                                                                                        |
| `errorCount`                              | —                                        | Contagem de registros com `FL_STATUS_RTR IN ('E', 'ET')`.                                                                                                 |
| `pendingCount`                            | —                                        | Contagem de registros com `FL_STATUS_RTR = 'PR'`.                                                                                                         |
| `completionPercentage`                    | —                                        | **Percentual de conclusão (0-100).** Formula: `((totalCount - errorCount) / totalCount) * 100`. Exemplo: 10 itens, 3 com erro = (10-3)/10\*100 = **70.0** |
| `requestedAt`                             | `ROTA_TRANSP_ROTEIRIZACAO.DT_INICDF_RTR` | Data de inicio da roteirizacao (setada no primeiro inseriRotaSat do lote).                                                                                |
| `completedAt`                             | `ROTA_TRANSP_ROTEIRIZACAO.DT_FINCDF_RTR` | Data de conclusao da roteirizacao. `null` enquanto em processamento.                                                                                      |
| `elapsedTime`                             | —                                        | `completedAt - requestedAt` formatado (ex: "1 min 30s").                                                                                                  |
| `beneficiaries.beneficiaryId`             | `ROTA_TRANSP_ROTEIRIZACAO.CD_CONFOR_RTR` | ID do funcionario no SAT.                                                                                                                                 |
| `beneficiaries.beneficiaryDocumentNumber` | `EMPRESA_USUARIO.CD_CODCPF_EUS`          | CPF do beneficiario. Mascarar nos logs.                                                                                                                   |
| `beneficiaries.fullName`                  | `EMPRESA_USUARIO.NM_USUARI_EUS`          | Nome completo.                                                                                                                                            |
| `beneficiaries.routeId`                   | `ROTA_TRANSP_ROTEIRIZACAO.ID_ROTEIR_RTR` | ID da roteirizacao (PK). Usar para chamar GET `/routes/{routeId}`.                                                                                        |
| `beneficiaries.status`                    | `ROTA_TRANSP_ROTEIRIZACAO.FL_STATUS_RTR` | Mapeado para enum `RoutingStatus` (ver tabela abaixo).                                                                                                    |
| `beneficiaries.updatedAt`                 | `ROTA_TRANSP_ROTEIRIZACAO.DT_ATUALI_RTR` | Timestamp de ultima atualizacao. Gerenciado pelo Hibernate.                                                                                               |
| `beneficiaries.errorDetail`               | `ROTA_TRANSP_ROTEIRIZACAO.DS_STAFOR_RTR` | Mensagem de erro amigavel. Expor apenas quando `FL_STATUS_RTR IN ('E', 'ET')`.                                                                            |

**Status do lote derivado de `FL_STATUS_RTR`:**

| Condicao                                              | `status` do lote        |
| ----------------------------------------------------- | ----------------------- |
| Existe ao menos 1 registro com `FL_STATUS_RTR = 'PR'` | `PROCESSING`            |
| Nenhum `PR`, todos em (`AP`, `F`, `CR`, `AC`, `PEN`)  | `COMPLETED`             |
| Nenhum `PR`, mistura de sucesso e erro (`E` ou `ET`)  | `COMPLETED_WITH_ERRORS` |
| Todos com `FL_STATUS_RTR IN ('E', 'ET')`              | `ERROR`                 |

**Tempo de processamento:**

- Varia de 30 segundos a 6 minutos por colaborador, dependendo da complexidade da rota.

**Exemplo de response (em processamento):**

```json
HTTP/1.1 200 OK

{
  "batchId": 142,
  "status": "PROCESSING",
  "totalCount": 2,
  "processedCount": 1,
  "errorCount": 0,
  "pendingCount": 1,
  "completionPercentage": 50.0,
  "requestedAt": "2026-08-17T15:00:00-03:00",
  "completedAt": null,
  "elapsedTime": "1 min 12s",
  "beneficiaries": [
    {
      "beneficiaryId": 88051342,
      "beneficiaryDocumentNumber": "35657149807",
      "fullName": "MATHEUS HENRIQUE RAMOS DA SILVA",
      "routeId": 142,
      "status": "COMPLETED",
      "updatedAt": "2026-08-17T15:01:10-03:00",
      "errorDetail": null
    },
    {
      "beneficiaryId": 88051283,
      "beneficiaryDocumentNumber": "85437339011",
      "fullName": "CARLOS ALBERTO SILVA",
      "routeId": 143,
      "status": "PROCESSING",
      "updatedAt": "2026-08-17T15:01:05-03:00",
      "errorDetail": null
    }
  ]
}
```

---

## Endpoint 4: Detalhe da rota

**`GET /companies/{companyId}/routes/{routeId}`**

Retorna os detalhes completos de uma roteirizacao processada: percursos, linhas de transporte e beneficios sugeridos.

**Principais mudancas em relacao ao legado:**

| Campo novo                                   | Origem                                                                                     |
| -------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `routeId`                                    | `idRoteirizacao` (renomear)                                                                |
| `companyId`                                  | adicionar ao response                                                                      |
| `batchId`                                    | `idLote` (adicionar ao response)                                                           |
| `beneficiaryId`                              | `idUsuario` (renomear)                                                                     |
| `beneficiaryDocumentNumber`                  | adicionar ao response                                                                      |
| `status`                                     | mapear status interno para enum em ingles                                                  |
| `homeAddress` / `workAddress`                | mesma origem do Endpoint 1: `EMPRESA_USUARIO.ID_ENDRES_END` / `ID_ENDTRA_END` → `ENDERECO` |
| `outboundRouteLegs`                          | Trajetos com `FL_SENTID_RTT = 'I'` (ida)                                                   |
| `returnRouteLegs`                            | Trajetos com `FL_SENTID_RTT = 'V'` (volta)                                                 |
| `sequence` em cada leg                       | `NU_SEQUEN_RTT` (ordem no percurso)                                                        |
| `direction` em cada leg                      | `I`/`V` → `OUTBOUND`/`RETURN`                                                              |
| `instruction` em cada leg                    | String montada: `"Pegar a linha " + CD_CODLIN_RTT + " " + DS_LINHA_RTT`                    |
| `instructionDetail` em cada leg              | String calculada com distancias de caminhada (ver wiki-calculos-endpoints-mvp.md)          |
| `boardingLocation` / `alightingLocation`     | Locais de embarque/desembarque (NM_LOEEMB_RTT / NM_LOEDES_RTT)                             |
| `boardingWalkMeters` / `alightingWalkMeters` | converter de km para metros (`km * 1000`, arredondado HALF_UP)                             |
| `estimatedTime` em cada leg                  | String: `"Tempo estimado : {tempo} min."` (null-safe)                                      |
| `travelTimeMinutes` em cada leg              | `NU_TEMPVI_RTT` como Double                                                                |
| `totalOutboundTimeMinutes`                   | **Ler de `ROTA_TRANSP_ROTEIRIZACAO.NU_IDATEM_RTR`** — NAO calcular por agregacao dos legs  |
| `totalReturnTimeMinutes`                     | **Ler de `ROTA_TRANSP_ROTEIRIZACAO.NU_VOLTEM_RTR`** — NAO calcular por agregacao dos legs  |
| `totalWalkingDistanceKm`                     | Soma de `(QT_EMBDIS_RTT + QT_DESDIS_RTT)` de todos os trajetos, em **quilometros**         |

**Observacoes importantes sobre totalizadores:**

- Os tres totalizadores retornam `null` quando `status != COMPLETED`.
- **`totalOutboundTimeMinutes` e `totalReturnTimeMinutes` sao calculados em runtime** pela agregacao de `travelTimeMinutes` de cada leg (usando Stream do Java), nao pelo banco.
- `totalWalkingDistanceKm` e calculado em runtime pela soma de distancias de embarque + desembarque de todos os trajetos, convertido para **quilometros** (nao em metros).

**Mapeamento de status de `FL_STATUS_RTR` para response:**

| FL_STATUS_RTR | Status no response | Descricao                                     |
| ------------- | ------------------ | --------------------------------------------- |
| `PR`          | `PROCESSING`       | Processando rota                              |
| `AP`          | `COMPLETED`        | Aguardando aprovacao (rota calculada)         |
| `F`           | `COMPLETED`        | Finalizado                                    |
| `CR`          | `COMPLETED`        | Contestacao respondida                        |
| `ET`          | `TECHNICAL_ERROR`  | Erro tecnico no SAT                           |
| `E`           | `ERROR`            | Erro de negocio (ex: endereco nao encontrado) |

**DE-PARA: campos do path param e response x banco de dados**

| Campo                                      | Origem no banco                                                                                      | Observacao                                                                                                  |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `companyId`                                | `EMPRESA.ID_EMPRES_EMP`                                                                              | Via `ROTA_TRANSP_ROTEIRIZACAO.ID_EMPRES_EMP`.                                                               |
| `batchId` (response)                       | `ROTA_TRANSP_ROTEIRIZACAO.ID_LOTROT_RTR`                                                             | ID do lote que originou esta rota.                                                                          |
| `beneficiaryId`                            | `EMPRESA_USUARIO.ID_USUARI_EUS`                                                                      | ID do beneficiario/colaborador.                                                                             |
| `beneficiaryDocumentNumber`                | `EMPRESA_USUARIO.CD_CODCPF_EUS`                                                                      | CPF. Mascarar nos logs.                                                                                     |
| `fullName`                                 | `EMPRESA_USUARIO.NM_USUARI_EUS`                                                                      | Nome completo.                                                                                              |
| `status`                                   | `ROTA_TRANSP_ROTEIRIZACAO.FL_STATUS_RTR`                                                             | Mapeado conforme Endpoint 3: `PR`→`PROCESSING`, `AP/F/CR`→`COMPLETED`, `E`→`ERROR`, `ET`→`TECHNICAL_ERROR`. |
| `requestedAt`                              | `ROTA_TRANSP_ROTEIRIZACAO.DT_CRIACA_RTR`                                                             | Data de criacao do registro.                                                                                |
| `completedAt`                              | `ROTA_TRANSP_ROTEIRIZACAO.DT_CONCLU_RTR`                                                             | Data de conclusao. Preenchida quando status final. `null` se ainda em processamento.                        |
| `homeAddress.*`                            | `ENDERECO.*` (via `EMPRESA_USUARIO.ID_ENDRES_END`)                                                   | Mesmos campos do Endpoint 1.                                                                                |
| `workAddress.*`                            | `ENDERECO.*` (via `EMPRESA_USUARIO.ID_ENDTRA_END`)                                                   | Mesmos campos do Endpoint 1.                                                                                |
| `outboundRouteLegs[*].sequence`            | `ROTA_TRANSP_TRAJETO.NU_SEQUEN_RTT` (filtro `FL_SENTID_RTT = 'I'`)                                   | Ordem do trajeto na ida.                                                                                    |
| `outboundRouteLegs[*].direction`           | Derivado (filtro `FL_SENTID_RTT = 'I'`)                                                              | Fixo: `OUTBOUND` quando `FL_SENTID_RTT = 'I'`.                                                              |
| `outboundRouteLegs[*].lineCode`            | `ROTA_TRANSP_TRAJETO.CD_CODLIN_RTT`                                                                  | Codigo da linha de transporte.                                                                              |
| `outboundRouteLegs[*].lineDescription`     | `ROTA_TRANSP_TRAJETO.DS_LINHA_RTT`                                                                   | Descricao da linha.                                                                                         |
| `outboundRouteLegs[*].instruction`         | Derivado: `"Pegar a linha " + CD_CODLIN_RTT + " " + DS_LINHA_RTT`                                    | String montada em `RoteirizacaoRelatorioUsecaseImpl.formataPercurso()`.                                     |
| `outboundRouteLegs[*].instructionDetail`   | Derivado: concatenacao de `QT_EMBDIS_RTT`, `DS_EMBARQ_RTT`, `DS_DESEMB_RTT`, `QT_DESDIS_RTT`         | Implementado em `RoteirizacaoRelatorioUsecaseImpl.formataPercurso()`.                                       |
| `outboundRouteLegs[*].boardingLocation`    | `ROTA_TRANSP_TRAJETO.DS_EMBARQ_RTT`                                                                  | Local de embarque.                                                                                          |
| `outboundRouteLegs[*].alightingLocation`   | `ROTA_TRANSP_TRAJETO.DS_DESEMB_RTT`                                                                  | Local de desembarque.                                                                                       |
| `outboundRouteLegs[*].boardingWalkMeters`  | `ROTA_TRANSP_TRAJETO.QT_EMBDIS_RTT * 1000` (arredondado HALF_UP)                                     | Distancia em metros (km para m).                                                                            |
| `outboundRouteLegs[*].alightingWalkMeters` | `ROTA_TRANSP_TRAJETO.QT_DESDIS_RTT * 1000` (arredondado HALF_UP)                                     | Distancia em metros (km para m).                                                                            |
| `outboundRouteLegs[*].estimatedTime`       | Derivado: `"Tempo estimado : " + NU_TEMPO_RTT + " min."` (null-safe)                                 | String formatada.                                                                                           |
| `outboundRouteLegs[*].travelTimeMinutes`   | `ROTA_TRANSP_TRAJETO.NU_TEMPO_RTT`                                                                   | Tempo de viagem em minutos.                                                                                 |
| `returnRouteLegs[*].sequence`              | `ROTA_TRANSP_TRAJETO.NU_SEQUEN_RTT` (filtro `FL_SENTID_RTT = 'V'`)                                   | Ordem do trajeto na volta.                                                                                  |
| `returnRouteLegs[*].direction`             | Derivado (filtro `FL_SENTID_RTT = 'V'`)                                                              | Fixo: `RETURN` quando `FL_SENTID_RTT = 'V'`.                                                                |
| `returnRouteLegs[*].lineCode`              | `ROTA_TRANSP_TRAJETO.CD_CODLIN_RTT`                                                                  | Codigo da linha de transporte.                                                                              |
| `returnRouteLegs[*].lineDescription`       | `ROTA_TRANSP_TRAJETO.DS_LINHA_RTT`                                                                   | Descricao da linha.                                                                                         |
| `returnRouteLegs[*].instruction`           | Derivado: `"Pegar a linha " + CD_CODLIN_RTT + " " + DS_LINHA_RTT`                                    | String montada em `RoteirizacaoRelatorioUsecaseImpl.formataPercurso()`.                                     |
| `returnRouteLegs[*].instructionDetail`     | Derivado: concatenacao de `QT_EMBDIS_RTT`, `DS_EMBARQ_RTT`, `DS_DESEMB_RTT`, `QT_DESDIS_RTT`         | Implementado em `RoteirizacaoRelatorioUsecaseImpl.formataPercurso()`.                                       |
| `returnRouteLegs[*].boardingLocation`      | `ROTA_TRANSP_TRAJETO.DS_EMBARQ_RTT`                                                                  | Local de embarque.                                                                                          |
| `returnRouteLegs[*].alightingLocation`     | `ROTA_TRANSP_TRAJETO.DS_DESEMB_RTT`                                                                  | Local de desembarque.                                                                                       |
| `returnRouteLegs[*].boardingWalkMeters`    | `ROTA_TRANSP_TRAJETO.QT_EMBDIS_RTT * 1000` (arredondado HALF_UP)                                     | Distancia em metros (km para m).                                                                            |
| `returnRouteLegs[*].alightingWalkMeters`   | `ROTA_TRANSP_TRAJETO.QT_DESDIS_RTT * 1000` (arredondado HALF_UP)                                     | Distancia em metros (km para m).                                                                            |
| `returnRouteLegs[*].estimatedTime`         | Derivado: `"Tempo estimado : " + NU_TEMPO_RTT + " min."` (null-safe)                                 | String formatada.                                                                                           |
| `returnRouteLegs[*].travelTimeMinutes`     | `ROTA_TRANSP_TRAJETO.NU_TEMPO_RTT`                                                                   | Tempo de viagem em minutos.                                                                                 |
| `totalOutboundTimeMinutes`                 | Agregacao: `SUM(outboundRouteLegs[*].travelTimeMinutes)` usando Stream do Java                       | Soma dos tempos de viagem ida (calculado em runtime). Retorna `null` se status != `COMPLETED`.              |
| `totalReturnTimeMinutes`                   | Agregacao: `SUM(returnRouteLegs[*].travelTimeMinutes)` usando Stream do Java                         | Soma dos tempos de viagem volta (calculado em runtime). Retorna `null` se status != `COMPLETED`.            |
| `totalWalkingDistanceKm`                   | Agregacao: `SUM(QT_EMBDIS_RTT + QT_DESDIS_RTT)` de todos os trajetos ida + volta, convertido para km | Soma de todas as distancias a pe (ida + volta, em quilometros). Retorna `null` se status != `COMPLETED`.    |
| `suggestedBenefits[*].productId`           | `PRODUTO.ID_PRODUT_PRO`                                                                              | ID do produto. `null` se produto nao cadastrado.                                                            |
| `suggestedBenefits[*].saleCode`            | `PRODUTO.CD_CODVEN_PRO`                                                                              | Codigo de venda do produto.                                                                                 |
| `suggestedBenefits[*].description`         | `PRODUTO.DS_DESCRI_PRO`                                                                              | Descricao do produto/linha.                                                                                 |
| `suggestedBenefits[*].unitPrice`           | `ROTA_TRANSP_COMPRA.VL_TARIFA_RTC`                                                                   | Valor unitario da tarifa.                                                                                   |
| `suggestedBenefits[*].quantity`            | `ROTA_TRANSP_COMPRA.QT_QUANTI_RTC`                                                                   | Quantidade de beneficios.                                                                                   |
| `suggestedBenefits[*].subtotal`            | `ROTA_TRANSP_COMPRA.VL_TARIFA_RTC * ROTA_TRANSP_COMPRA.QT_QUANTI_RTC` (BigDecimal.multiply)          | Subtotal calculado.                                                                                         |

**Exemplo de response (status COMPLETED):**

```json
{
  "companyId": 835181,
  "batchId": 141,
  "beneficiaryId": 88051342,
  "beneficiaryDocumentNumber": "35657149807",
  "fullName": "MATHEUS HENRIQUE RAMOS DA SILVA",
  "status": "COMPLETED",
  "requestedAt": "2026-08-17T15:00:00-03:00",
  "completedAt": "2026-08-17T15:01:10-03:00",
  "homeAddress": {
    "street": "RUA DOUTOR PEDRO ZIMMERMANN",
    "number": "15",
    "complement": null,
    "neighborhood": "ITOUPAVAZINHA",
    "city": "BLUMENAU",
    "state": "SC",
    "zipCode": "89066000",
    "referencePoint": null
  },
  "workAddress": {
    "street": "RUA THEODORO PEREIRA",
    "number": "10",
    "complement": null,
    "neighborhood": "VILA NOVA",
    "city": "BLUMENAU",
    "state": "SC",
    "zipCode": "89035020",
    "referencePoint": null
  },
  "outboundRouteLegs": [
    {
      "sequence": 1,
      "direction": "OUTBOUND",
      "lineCode": "233-10",
      "lineDescription": "BLUMENAU / ITOUPAVA CENTRAL",
      "instruction": "Pegar a linha 233-10 BLUMENAU / ITOUPAVA CENTRAL",
      "instructionDetail": "andar 50 metros e embarcar na R DOUTOR PEDRO ZIMMERMANN, desembarcar na R THEODORO PEREIRA e andar 80 metros",
      "boardingLocation": "R DOUTOR PEDRO ZIMMERMANN",
      "boardingWalkMeters": 50,
      "alightingLocation": "R THEODORO PEREIRA",
      "alightingWalkMeters": 80,
      "estimatedTime": "Tempo estimado : 18.50 min.",
      "travelTimeMinutes": 18.5
    }
  ],
  "returnRouteLegs": [
    {
      "sequence": 1,
      "direction": "RETURN",
      "lineCode": "233-10",
      "lineDescription": "ITOUPAVA CENTRAL / BLUMENAU",
      "instruction": "Pegar a linha 233-10 ITOUPAVA CENTRAL / BLUMENAU",
      "instructionDetail": "andar 80 metros e embarcar na R THEODORO PEREIRA, desembarcar na R DOUTOR PEDRO ZIMMERMANN e andar 50 metros",
      "boardingLocation": "R THEODORO PEREIRA",
      "boardingWalkMeters": 80,
      "alightingLocation": "R DOUTOR PEDRO ZIMMERMANN",
      "alightingWalkMeters": 50,
      "estimatedTime": "Tempo estimado : 20.00 min.",
      "travelTimeMinutes": 20.0
    }
  ],
  "totalOutboundTimeMinutes": 18.5,
  "totalReturnTimeMinutes": 20.0,
  "totalWalkingDistanceKm": 0.26,
  "suggestedBenefits": [
    {
      "productId": 9825,
      "saleCode": "009825",
      "description": "SANTA CATARINA INTERMUNICIPAL",
      "quantity": 2,
      "unitPrice": 5.49,
      "subtotal": 10.98
    }
  ]
}
```

**Observacoes importantes:**

- Quando `status = PROCESSING`: retornar apenas campos de identificacao e status, sem trajetos nem totalizadores.
- Quando `status IN ('ERROR', 'TECHNICAL_ERROR')`: incluir campo `errorDetail` com descricao amigavel da falha.
- `totalOutboundTimeMinutes`, `totalReturnTimeMinutes` e `totalWalkingDistanceKm` retornam `null` quando status != `COMPLETED`.
- **Filtros de trajetos:** `outboundRouteLegs` agrega trajetos com `FL_SENTID_RTT = 'I'` (ida). `returnRouteLegs` agrega trajetos com `FL_SENTID_RTT = 'V'` (volta).

**Calculos e derivacoes de campos:**

| Campo                                   | Formula / Metodo                                                                                                                                                                                                                                                     | Implementacao                                                                                                                                           | Exemplo                                                                                                          |
| --------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `instruction` (por leg)                 | `"Pegar a linha " + CD_CODLIN_RTT + " " + DS_LINHA_RTT`                                                                                                                                                                                                              | `RoteirizacaoRelatorioUsecaseImpl.formataPercurso()` (vt-roteirizacao)                                                                                  | `"Pegar a linha 233-10 BLUMENAU / ITOUPAVA CENTRAL"`                                                             |
| `instructionDetail` (por leg)           | Concatenacao condicional de distancias + embarque/desembarque                                                                                                                                                                                                        | `RoteirizacaoRelatorioUsecaseImpl.formataPercurso()` (vt-roteirizacao). Componentes: `QT_EMBDIS_RTT`, `DS_EMBARQ_RTT`, `DS_DESEMB_RTT`, `QT_DESDIS_RTT` | `"andar 50 metros e embarcar na R DOUTOR PEDRO ZIMMERMANN, desembarcar na R THEODORO PEREIRA e andar 80 metros"` |
| `estimatedTime` (por leg)               | `"Tempo estimado : " + NU_TEMPO_RTT + " min."` (null-safe)                                                                                                                                                                                                           | Simples concatenacao de string                                                                                                                          | `"Tempo estimado : 18.50 min."`                                                                                  |
| `boardingWalkMeters` (por leg)          | `QT_EMBDIS_RTT * 1000`, arredondado HALF_UP, convertido para int                                                                                                                                                                                                     | Conversao km→m                                                                                                                                          | `50` (de 0.050 km)                                                                                               |
| `alightingWalkMeters` (por leg)         | `QT_DESDIS_RTT * 1000`, arredondado HALF_UP, convertido para int                                                                                                                                                                                                     | Conversao km→m                                                                                                                                          | `80` (de 0.080 km)                                                                                               |
| `totalOutboundTimeMinutes` (nivel rota) | **Agregacao em runtime:** `outboundRouteLegs.stream().mapToDouble(leg -> leg.getTravelTimeMinutes()).sum()`                                                                                                                                                          | Soma dos `travelTimeMinutes` de TODOS os legs de ida — nao buscar do banco                                                                              | `62.5`                                                                                                           |
| `totalReturnTimeMinutes` (nivel rota)   | **Agregacao em runtime:** `returnRouteLegs.stream().mapToDouble(leg -> leg.getTravelTimeMinutes()).sum()`                                                                                                                                                            | Soma dos `travelTimeMinutes` de TODOS os legs de volta — nao buscar do banco                                                                            | `58.3`                                                                                                           |
| `totalWalkingDistanceKm` (nivel rota)   | **Agregacao em runtime:** `(outboundRouteLegs.stream().mapToDouble(leg -> leg.getBoardingWalkMeters() + leg.getAlightingWalkMeters()).sum() + returnRouteLegs.stream().mapToDouble(leg -> leg.getBoardingWalkMeters() + leg.getAlightingWalkMeters()).sum()) / 1000` | Soma de distancias de caminhada (ida + volta) de TODOS os trajetos, convertida de metros para km. **Não é soma de travelTimeMinutes**                   | `1.46`                                                                                                           |
| `outboundRouteLegs` / `returnRouteLegs` | Agrupamento e formatacao de trajetos                                                                                                                                                                                                                                 | `RoteirizacaoRelatorioUsecaseImpl.gerarRoteirizacaoDetalhe()` (vt-roteirizacao)                                                                         | Array estruturado com 1+ legs                                                                                    |
| `suggestedBenefits`                     | Calculo de produtos VT sugeridos                                                                                                                                                                                                                                     | `RoteirizacaoRelatorioUsecaseImpl.gerarValeTransporte()` (vt-roteirizacao). Join: `ROTA_TRANSP_COMPRA` → `PRODUTO`                                      | Array com productId, saleCode, description, quantity, unitPrice, subtotal                                        |
| `subtotal` (por beneficio)              | `VL_TARIFA_RTC * QT_QUANTI_RTC` (BigDecimal.multiply)                                                                                                                                                                                                                | Operacao `multiply()` de BigDecimal com os campos de `ROTA_TRANSP_COMPRA`                                                                               | `10.98` (ex: 5.49 \* 2)                                                                                          |

---

## Regras de negocio

### Endpoint 1: Listar colaboradores

| Codigo  | Regra                                                 | HTTP             |
| ------- | ----------------------------------------------------- | ---------------- |
| ROT-020 | Colaborador nao pertence a empresa                    | 400              |
| ROT-013 | Sem endereco residencial                              | `eligible=false` |
| ROT-014 | Sem endereco de trabalho                              | `eligible=false` |
| ROT-015 | Processo aberto (`PR`, `AP`, `CA`, `CR`, `AC`, `PEN`) | `eligible=false` |
| ROT-021 | Rota finalizada/aprovada (status `F`)                 | `eligible=false` |

### Endpoint 2: Solicitar roteirizacao

| Codigo  | Regra                                                        | HTTP | Mensagem                                                       |
| ------- | ------------------------------------------------------------ | ---- | -------------------------------------------------------------- |
| ROT-019 | Lista de beneficiaryIds vazia                                | 400  | "Deve informar ao menos um colaborador"                        |
| ROT-020 | Colaborador nao pertence a empresa                           | 400  | "Colaborador {id} nao encontrado na empresa"                   |
| ROT-013 | Sem endereco residencial                                     | 422  | "Colaborador {cpf} sem endereco residencial cadastrado"        |
| ROT-014 | Sem endereco de trabalho                                     | 422  | "Colaborador {cpf} sem endereco de trabalho cadastrado"        |
| ROT-015 | Processo aberto (status `PR`, `AP`, `CA`, `CR`, `AC`, `PEN`) | 422  | "Colaborador {cpf} ja possui processo aberto em roteirizacao"  |
| ROT-016 | Colaborador inativo                                          | 422  | "Colaborador {cpf} inativo na empresa"                         |
| ROT-017 | CEP invalido                                                 | 422  | "CEP invalido no endereco de {tipo} do colaborador {cpf}"      |
| ROT-010 | Empresa sem contrato ativo                                   | 422  | "Empresa nao possui contrato de roteirizacao ativo"            |
| ROT-011 | Empresa nao ativa no SAT                                     | 422  | "CNPJ da empresa nao esta ativo no fornecedor SAT"             |
| ROT-018 | Empresa inadimplente                                         | 422  | "Empresa bloqueada para novas roteirizacoes por inadimplencia" |

- Validar todos os colaboradores antes de enfileirar qualquer um (falha atomica)
- Lotes nao processados em 24h sao cancelados automaticamente
- Sem limite pre-definido de colaboradores por lote. Testes de performance subsidiarao futuras decisoes sobre limites, se necessario.

### Endpoint 3: Status do lote

| Regra                       | HTTP | Mensagem                                          |
| --------------------------- | ---- | ------------------------------------------------- |
| Lote nao encontrado         | 404  | "Solicitacao de roteirizacao {id} nao encontrada" |
| Lote nao pertence a empresa | 403  | "Lote nao pertence a empresa informada"           |

### Endpoint 4: Detalhe da rota

| Regra                       | HTTP | Mensagem                                        |
| --------------------------- | ---- | ----------------------------------------------- |
| Roteirizacao nao encontrada | 404  | "Roteirizacao {id} nao encontrada"              |
| Rota nao pertence a empresa | 403  | "Roteirizacao nao pertence a empresa informada" |

- Quando `status = PROCESSING`: retornar apenas campos de identificacao e status, sem trajetos
- Quando `status = ERROR` ou `TECHNICAL_ERROR`: incluir `errorMessage` com descricao amigavel

---

## Alteracoes de banco necessarias

### Tabela ROTA_TRANSP_LOTE _(opcional — para otimizacao futura)_

Tabela sugerida para consolidar dados do lote e evitar agregacoes em runtime no Endpoint 3. Nao e obrigatoria para o MVP, mas facilitaria manutencao futura se implementada no `vt-roteirizacao-sat`.

---

## Confirmacoes do refinamento

As dúvidas levantadas durante o refinamento foram consolidadas no arquivo `.claude/docs/roteirizacao/Dúvidas sobre contrato API roteirizacao.md` com as respostas marcadas com `R:`.

**Principais confirmacoes:**

| Ponto                             | Confirmacao                                                                                               |
| --------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `X-Logged-User-Id`                | Nao validar na aplicacao. Apenas propagar para auditoria. Sensedia e responsavel.                         |
| `registrationNumber`              | Campo `EMPRESA_USUARIO.CD_USUARI_EUS`.                                                                    |
| Elegibilidade                     | Colaboradores com status `E`/`ET` sao elegíveis para re-roteirizar.                                       |
| `lastRouteStatus`                 | Retornar apenas `RC`, `E`, `ET`. Demais retornam `null`.                                                  |
| `batchId` (Endpoint 2 response)   | Refere-se a `ROTA_TRANSP_ROTEIRIZACAO.ID_LOTROT_RTR` (ID do lote criado).                                 |
| `batchId` (Endpoint 3 path param) | Refere-se a `ROTA_TRANSP_ROTEIRIZACAO.ID_LOTROT_RTR` (ID do lote).                                        |
| `beneficiaryCount`                | Contar IDs elegíveis enfileirados no `vt-roteirizacao` apos validacoes.                                   |
| Campo `attempts` (Endpoint 3)     | **Removido** — nao sera armazenado.                                                                       |
| `completionPercentage`            | Percentual de conclusão (0-100). Formula: `((totalCount - errorCount) / totalCount) * 100`. Exemplo: 70.0 |
| `elapsedTime`                     | Formato legivel (ex: "1 min 30s").                                                                        |
| `batchId` (Endpoint 4 response)   | Refere-se a `ROTA_TRANSP_ROTEIRIZACAO.ID_LOTROT_RTR` (ID do lote correspondente).                         |
| Status mapeamento (Endpoint 4)    | `PR`→`PROCESSING`, `AP/F/CR`→`COMPLETED`, `E`→`ERROR`, `ET`→`TECHNICAL_ERROR`.                            |
| `requestedAt` (Endpoint 4)        | Campo `ROTA_TRANSP_ROTEIRIZACAO.DT_CRIACA_RTR` (data de criacao).                                         |
| `completedAt` (Endpoint 4)        | Campo `ROTA_TRANSP_ROTEIRIZACAO.DT_CONCLU_RTR` (data de conclusao).                                       |
