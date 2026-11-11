# Spike #1722540 - Fluxo Aprovar (V2)

![contestar-v2.drawio.png](/.attachments/contestar-v2.drawio-67e0f150-da38-469b-b1ab-133d0e2095dc.png)

---

## 1. Endpoint

| Campo         | Valor                                                       |
| ------------- | ----------------------------------------------------------- |
| Metodo / Path | `POST /v2/companies/{companyId}/routes/{routeId}/approvals` |
| MS            | `vt-roteirizacao`                                           |
| Sync/Async    | **Sincrono** (sem SAT)                                      |
| HTTP sucesso  | `200 OK` _(ou `201` - aberto #11)_                          |
| Controller    | `RouteApprovalController`                                   |
| UseCase entry | `ApproveRouteUsecase`                                       |

---

## 2. Orquestracao (SRP)

```
RouteApprovalController
  -> ApproveRouteUsecase.execute(companyId, routeId, loggedUserId)
       1. ValidateRouteForApprovalUsecase
       2. CalculateRouteEconomyUsecase          // falha NAO aborta
       3. ApplyApprovedBenefitsUsecase
       4. FinalizeApprovedRouteUsecase
       5. PublishApprovalDocumentUsecase
       <- ApproveRouteResponse
```

| Camada      | Classe                                        | Responsabilidade unica                                 |
| ----------- | --------------------------------------------- | ------------------------------------------------------ |
| **UseCase** | `ApproveRouteUsecase`                         | Orquestrar steps e montar response                     |
| **UseCase** | `ValidateRouteForApprovalUsecase`             | Empresa, rota, ownership, status, compra, produtos     |
| **UseCase** | `CalculateRouteEconomyUsecase`                | Consultar calculation + gravar economia (best-effort)  |
| **UseCase** | `ApplyApprovedBenefitsUsecase`                | Remover vinculos antigos + gravar beneficios da compra |
| **UseCase** | `FinalizeApprovedRouteUsecase`                | Status final + data conclusao + historico              |
| **UseCase** | `PublishApprovalDocumentUsecase`              | Disparar geracao PDF / fila documentacao               |
| **Gateway** | `CompanyGateway` / `RoutingGateway`           | READ empresa/rota                                      |
| **Gateway** | `RoutePurchaseDataGateway` / `ProductGateway` | READ compra/itens/produto                              |
| **Gateway** | `OrdersCalculationGateway`                    | Feign vt-orders-calculation                            |
| **Gateway** | `EconomyGateway`                              | UPDATE economia (opcional)                             |
| **Gateway** | `EmployeeBenefitGateway`                      | DELETE + INSERT EmpresaUsuarioProduto                  |
| **Gateway** | `HistoryGateway`                              | INSERT historico                                       |
| **Gateway** | `DocumentQueueGateway`                        | PUBLISH fila docs (async)                              |

**Regra:** UseCase orquestra. Gateway acessa DB/Feign/MQ. Steps **nao** se chamam entre si. **SEM SAT**.

Pacote UseCase: `usecase/approval/`  
Pacote Gateway: `gateway/`

---

## 3. Entrada (payload do endpoint)

```http
POST /v2/companies/{companyId}/routes/{routeId}/approvals
X-Logged-User-Id: 88051342
Content-Type: application/json
```

**Body:** vazio (1 rota por request).

| Campo              | Onde   | Tipo         | Obrigatorio | Validacao               |
| ------------------ | ------ | ------------ | ----------- | ----------------------- |
| `companyId`        | path   | `BigInteger` | **sim**     | `@Positive`, `@Digits`  |
| `routeId`          | path   | `BigInteger` | **sim**     | `@Positive`, `@Digits`  |
| `X-Logged-User-Id` | header | `BigInteger` | **sim**     | `@NotNull`, `@Positive` |

V1 aceita lista; bulk fora desta entrega (aberto #9).

---

## 4. Regras (por step)

### ValidateRouteForApprovalUsecase

1. Empresa existe (`CompanyGateway`).
2. Rota existe e pertence a empresa (`Forbidden` se divergir).
3. Compra finalizada (`flagStatusCompra == F`).
4. Itens: `idProduto > 0`, `qtdeBeneficio > 0`, produto existe, facial => `valorTarifa > 0`.
5. Status elegivel: tipicamente `AP` (e `CR`? - aberto #2).

### CalculateRouteEconomyUsecase

- Chama `OrdersCalculationGateway` -> `vt-orders-calculation`.
- **Nao bloqueante:** falha so loga; aprovacao segue; `economy` pode ser `null`.
- Se OK: `EconomyGateway` UPDATE economia.

### ApplyApprovedBenefitsUsecase

- `EmployeeBenefitGateway`: DELETE vinculos `EmpresaUsuarioProduto` do usuario.
- INSERT beneficios a partir das compras SAT **ja no banco** (nao SOAP).
- DELETE + INSERT atomico.

### FinalizeApprovedRouteUsecase

- `RoutingGateway`: status **`AP -> F`** (FINALIZADO) - igual V1; nao usa `AC`.
- Preenche `dataConclusaoRoteirizacao` (`DT_CONCLU_RTR`).
- `HistoryGateway`: historico (aberto #3).

### PublishApprovalDocumentUsecase

- `DocumentQueueGateway`: gera PDF / publica fila (async).
- Sem chamada SAT.

---

## 5. Response sucesso

```json
{
  "routeId": 7654321,
  "companyId": 834552,
  "status": "F",
  "completedAt": "2026-09-22T15:00:00",
  "economy": {
    "previousBenefitsAmount": 450.0,
    "suggestedBenefitsAmount": 380.0,
    "savedAmount": 70.0
  }
}
```

---

## 6. Erros tipados

Regra V2: `errors[]` **apenas** para parametro obrigatorio / Bean Validation. Negocio = so `message`.

| HTTP | Situacao                                       | Excecao / origem     | `errors[]`?                   |
| ---- | ---------------------------------------------- | -------------------- | ----------------------------- |
| 400  | path/header obrigatorios ausentes ou invalidos | Bean Validation      | **sim** (`field` + `message`) |
| 403  | rota de outra empresa                          | `ForbiddenException` | nao                           |
| 404  | empresa/rota nao encontrada                    | `NotFoundException`  | nao                           |
| 409  | status invalido / compra pendente              | `ConflictException`  | nao                           |
| 500  | erro inesperado                                | generico             | nao                           |

Exemplo **400** (parametro):

```json
{
  "timestamp": "2026-09-22T15:01:00.123",
  "status": 400,
  "error": "Bad Request",
  "message": "Parametros invalidos para aprovacao da rota",
  "errors": [
    {
      "field": "X-Logged-User-Id",
      "message": "Header X-Logged-User-Id e obrigatorio para auditoria"
    },
    {
      "field": "routeId",
      "message": "Identificador da rota (routeId) deve ser um numero positivo"
    }
  ]
}
```

Catalogo de mensagens 400 (aprovacao):

| field              | message                                                          |
| ------------------ | ---------------------------------------------------------------- |
| `X-Logged-User-Id` | Header X-Logged-User-Id e obrigatorio para auditoria             |
| `companyId`        | Identificador da empresa (companyId) deve ser um numero positivo |
| `routeId`          | Identificador da rota (routeId) deve ser um numero positivo      |

Exemplo **403** (negocio - sem `errors`):

```json
{
  "timestamp": "2026-09-22T15:01:00.123",
  "status": 403,
  "error": "Forbidden",
  "message": "Rota 7654321 nao pertence a empresa 834552"
}
```

Exemplo **409** (negocio - sem `errors`):

```json
{
  "timestamp": "2026-09-22T15:01:00.123",
  "status": 409,
  "error": "Conflict",
  "message": "Rota 7654321 nao esta elegivel para aprovacao (status CA)"
}
```

> Mensagens de `errors[].message` em portugues, especificas por campo (evitar defaults do Bean Validation como `must not be null`).

---

## 7. Efeitos colaterais

| Efeito     | Detalhe                                                            |
| ---------- | ------------------------------------------------------------------ |
| Banco      | status `F`, `DT_CONCLU`, produtos colaborador, economia (opcional) |
| Historico  | aberto #3                                                          |
| PDF/S3     | geracao documento (async)                                          |
| SAT        | **nenhuma**                                                        |
| E-mail/SMS | mapear ponto; nao implementar                                      |

---

## 8. Quem chama quem

```
Portal
  -> RouteApprovalController
       -> ApproveRouteUsecase (ENTRY)
            -> 1. ValidateRouteForApprovalUsecase
                 -> CompanyGateway / RoutingGateway / RoutePurchaseDataGateway / ProductGateway
            -> 2. CalculateRouteEconomyUsecase
                 -> OrdersCalculationGateway -> vt-orders-calculation
                 -> EconomyGateway (UPDATE opcional)
            -> 3. ApplyApprovedBenefitsUsecase
                 -> EmployeeBenefitGateway (DELETE + INSERT)
            -> 4. FinalizeApprovedRouteUsecase
                 -> RoutingGateway (UPDATE F) + HistoryGateway
            -> 5. PublishApprovalDocumentUsecase
                 -> DocumentQueueGateway (async)
       <- ApproveRouteResponse (HTTP 200)
```

---

## 9. Exemplo

```http
POST /v2/companies/834552/routes/7654321/approvals
X-Logged-User-Id: 88051342
Content-Type: application/json
```

```http
HTTP/1.1 200 OK
{
  "routeId": 7654321,
  "companyId": 834552,
  "status": "F",
  "completedAt": "2026-09-22T15:00:00",
  "economy": {
    "previousBenefitsAmount": 450.00,
    "suggestedBenefitsAmount": 380.00,
    "savedAmount": 70.00
  }
}
```

---

## 10. Artefatos de implementacao (aprovacao)

```
controllers/RouteApprovalController.java
controllers/swagger/annotations/ApproveRouteApiResponse.java

usecase/approval/ApproveRouteUsecase.java + Impl
usecase/approval/ValidateRouteForApprovalUsecase.java + Impl
usecase/approval/CalculateRouteEconomyUsecase.java + Impl
usecase/approval/ApplyApprovedBenefitsUsecase.java + Impl
usecase/approval/FinalizeApprovedRouteUsecase.java + Impl
usecase/approval/PublishApprovalDocumentUsecase.java + Impl

gateway/CompanyGateway.java
gateway/RoutingGateway.java
gateway/RoutePurchaseDataGateway.java
gateway/ProductGateway.java
gateway/OrdersCalculationGateway.java
gateway/EconomyGateway.java
gateway/EmployeeBenefitGateway.java
gateway/HistoryGateway.java
gateway/DocumentQueueGateway.java

dtos/v2/approval/response/ApproveRouteResponse.java

tests/usecase/approval/*
tests/components/ApproveRouteComponentTest.java
```

---

## 11. Pontos abertos deste fluxo

| #   | Tema                                   |
| --- | -------------------------------------- |
| 2   | Status elegiveis (`AP` / `CR`)         |
| 3   | Historico na aprovacao                 |
| 9   | Bulk approve                           |
| 11  | HTTP `200` vs `201`                    |
| 16  | Prefixo `/v2` na solicitacao existente |

> Status pos-aprovacao fechado: **`AP -> F`**. Cenario `AP -> AC -> F` removido (nao existe no codigo).
