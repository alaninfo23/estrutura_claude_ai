# Mocks da API Vale Transporte - Roteirização (MVP)

> Documento com exemplos de request/response para cada endpoint da API de Roteirização.  
> Para cada endpoint: **2 exemplos de sucesso** + **1 exemplo de erro**.

---

## Sumário

1. [GET /companies/{companyId}/beneficiaries](#1-get-companiescompanyidbeneficiaries)
2. [POST /companies/{companyId}/routing-requests](#2-post-companiescompanyidrouting-requests)
3. [GET /companies/{companyId}/routing-requests/{routingRequestId}](#3-get-companiescompanyidrouting-requestsroutingrequestid)
4. [GET /companies/{companyId}/routes/{routeId}](#4-get-companiescompanyidroutesrouteid)

---

## 1. GET /companies/{companyId}/beneficiaries

**Descrição:** Listar usuários pendentes de roteirização.

### Sucesso 1 — Listar todos os colaboradores (elegíveis e não elegíveis)

**Request:**

```http
GET /vt/v1/companies/12345/beneficiaries?page=0&size=20&sort=name,asc
X-Company-Document-Root: 61190096
```

**Response:** `200 OK`

```json
{
  "content": [
    {
      "beneficiaryId": 98765,
      "companyId": 12345,
      "beneficiaryDocumentNumber": "09304730783",
      "fullName": "JOSIMAR FREIRE FERREIRA",
      "registrationNumber": "000000213",
      "department": "OPERAÇÕES",
      "position": "MANIPULADOR DE ALIMENTOS",
      "homeAddress": {
        "street": "RUA ANA CASEMIRO",
        "number": "27",
        "complement": "CASA",
        "neighborhood": "BENFICA",
        "city": "RIO DE JANEIRO",
        "state": "RJ",
        "zipCode": "20930190",
        "referencePoint": "Em frente à padaria São Jorge"
      },
      "workAddress": {
        "street": "RUA SIQUEIRA CAMPOS",
        "number": "74",
        "complement": null,
        "neighborhood": "COPACABANA",
        "city": "RIO DE JANEIRO",
        "state": "RJ",
        "zipCode": "22440033",
        "referencePoint": "Próximo à estação de metrô Siqueira Campos"
      },
      "workAddressId": 1,
      "eligible": true,
      "ineligibilityReason": null,
      "lastRouteStatus": null,
      "lastRouteDate": null
    },
    {
      "beneficiaryId": 98766,
      "companyId": 12345,
      "beneficiaryDocumentNumber": "45678901234",
      "fullName": "MARIA APARECIDA DOS SANTOS",
      "registrationNumber": "000000450",
      "department": "ADMINISTRATIVO",
      "position": "AUXILIAR ADMINISTRATIVO",
      "homeAddress": null,
      "workAddress": {
        "street": "RUA SIQUEIRA CAMPOS",
        "number": "74",
        "complement": null,
        "neighborhood": "COPACABANA",
        "city": "RIO DE JANEIRO",
        "state": "RJ",
        "zipCode": "22440033",
        "referencePoint": "Próximo à estação de metrô Siqueira Campos"
      },
      "workAddressId": 1,
      "eligible": false,
      "ineligibilityReason": "MISSING_HOME_ADDRESS",
      "lastRouteStatus": null,
      "lastRouteDate": null
    },
    {
      "beneficiaryId": 98767,
      "companyId": 12345,
      "beneficiaryDocumentNumber": "78901234567",
      "fullName": "PEDRO HENRIQUE COSTA SILVA",
      "registrationNumber": "000000789",
      "department": "LOGÍSTICA",
      "position": "MOTORISTA",
      "homeAddress": {
        "street": "RUA DAS LARANJEIRAS",
        "number": "150",
        "complement": "APTO 501",
        "neighborhood": "LARANJEIRAS",
        "city": "RIO DE JANEIRO",
        "state": "RJ",
        "zipCode": "22240003",
        "referencePoint": null
      },
      "workAddress": {
        "street": "AV PRESIDENTE VARGAS",
        "number": "3131",
        "complement": "GALPÃO 2",
        "neighborhood": "CIDADE NOVA",
        "city": "RIO DE JANEIRO",
        "state": "RJ",
        "zipCode": "20210030",
        "referencePoint": "Ao lado do Terminal Américo Fontenelle"
      },
      "workAddressId": 2,
      "eligible": true,
      "ineligibilityReason": null,
      "lastRouteStatus": "REJECTED",
      "lastRouteDate": "2026-07-01T10:30:00-03:00"
    }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 3,
  "totalPages": 1,
  "totalEligible": 2
}
```

---

### Sucesso 2 — Filtrar apenas colaboradores elegíveis, nunca roteirizados

**Request:**

```http
GET /vt/v1/companies/12345/beneficiaries?eligible=true&lastRouteStatus=NEVER&page=0&size=20&sort=lastRouteDate,asc
X-Company-Document-Root: 61190096
```

**Response:** `200 OK`

```json
{
  "content": [
    {
      "beneficiaryId": 98765,
      "companyId": 12345,
      "beneficiaryDocumentNumber": "09304730783",
      "fullName": "JOSIMAR FREIRE FERREIRA",
      "registrationNumber": "000000213",
      "department": "OPERAÇÕES",
      "position": "MANIPULADOR DE ALIMENTOS",
      "homeAddress": {
        "street": "RUA ANA CASEMIRO",
        "number": "27",
        "complement": "CASA",
        "neighborhood": "BENFICA",
        "city": "RIO DE JANEIRO",
        "state": "RJ",
        "zipCode": "20930190",
        "referencePoint": "Em frente à padaria São Jorge"
      },
      "workAddress": {
        "street": "RUA SIQUEIRA CAMPOS",
        "number": "74",
        "complement": null,
        "neighborhood": "COPACABANA",
        "city": "RIO DE JANEIRO",
        "state": "RJ",
        "zipCode": "22440033",
        "referencePoint": "Próximo à estação de metrô Siqueira Campos"
      },
      "workAddressId": 1,
      "eligible": true,
      "ineligibilityReason": null,
      "lastRouteStatus": null,
      "lastRouteDate": null
    },
    {
      "beneficiaryId": 98770,
      "companyId": 12345,
      "beneficiaryDocumentNumber": "11223344556",
      "fullName": "LUCAS MARTINS OLIVEIRA",
      "registrationNumber": "000001002",
      "department": "PRODUÇÃO",
      "position": "OPERADOR DE MÁQUINAS",
      "homeAddress": {
        "street": "RUA BARÃO DE MESQUITA",
        "number": "400",
        "complement": "BL 3 APTO 201",
        "neighborhood": "TIJUCA",
        "city": "RIO DE JANEIRO",
        "state": "RJ",
        "zipCode": "20540004",
        "referencePoint": "Próximo ao Shopping Tijuca"
      },
      "workAddress": {
        "street": "RUA SIQUEIRA CAMPOS",
        "number": "74",
        "complement": null,
        "neighborhood": "COPACABANA",
        "city": "RIO DE JANEIRO",
        "state": "RJ",
        "zipCode": "22440033",
        "referencePoint": "Próximo à estação de metrô Siqueira Campos"
      },
      "workAddressId": 1,
      "eligible": true,
      "ineligibilityReason": null,
      "lastRouteStatus": null,
      "lastRouteDate": null
    }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 2,
  "totalPages": 1,
  "totalEligible": 2
}
```

---

### Erro — Empresa não encontrada

**Request:**

```http
GET /vt/v1/companies/99999/beneficiaries?page=0&size=20
X-Company-Document-Root: 61190096
```

**Response:** `404 Not Found`

```json
{
  "timestamp": "2026-08-19T10:00:00Z",
  "status": 404,
  "error": "Not Found",
  "message": "Empresa não encontrada: 99999",
  "path": "/vt/v1/companies/99999/beneficiaries",
  "errors": null
}
```

---

## 2. POST /companies/{companyId}/routing-requests

**Descrição:** Solicitar roteirização para um ou mais usuários.

### Sucesso 1 — Solicitar roteirização para 3 colaboradores

**Request:**

```http
POST /vt/v1/companies/12345/routing-requests
Content-Type: application/json
X-Company-Document-Root: 61190096
X-Logged-User-Id: 54321
```

```json
{
  "beneficiaryIds": [98765, 98767, 98770]
}
```

**Response:** `202 Accepted`

Header: `Location: /vt/v1/companies/12345/routing-requests/7890`

```json
{
  "routingRequestId": 7890,
  "beneficiaryCount": 3
}
```

---

### Sucesso 2 — Solicitar roteirização para 1 colaborador

**Request:**

```http
POST /vt/v1/companies/12345/routing-requests
Content-Type: application/json
X-Company-Document-Root: 61190096
X-Logged-User-Id: 54321
```

```json
{
  "beneficiaryIds": [98765]
}
```

**Response:** `202 Accepted`

Header: `Location: /vt/v1/companies/12345/routing-requests/7891`

```json
{
  "routingRequestId": 7891,
  "beneficiaryCount": 1
}
```

---

### Erro — Validação: colaboradores sem endereço válido e rota em processamento

**Request:**

```http
POST /vt/v1/companies/12345/routing-requests
Content-Type: application/json
X-Company-Document-Root: 61190096
X-Logged-User-Id: 54321
```

```json
{
  "beneficiaryIds": [98766, 98768, 98769]
}
```

**Response:** `422 Unprocessable Entity`

```json
{
  "timestamp": "2026-08-19T10:30:00Z",
  "status": 422,
  "error": "Unprocessable Entity",
  "message": "Validação falhou para 3 beneficiários",
  "path": "/vt/v1/companies/12345/routing-requests",
  "errors": [
    {
      "field": "beneficiaryIds[0]",
      "message": "Beneficiário 98766 não possui endereço residencial cadastrado",
      "code": "ROT-001"
    },
    {
      "field": "beneficiaryIds[1]",
      "message": "Beneficiário 98768 já possui roteirização em processamento",
      "code": "ROT-003"
    },
    {
      "field": "beneficiaryIds[2]",
      "message": "Beneficiário 98769 não pertence à empresa 12345",
      "code": "ROT-004"
    }
  ]
}
```

---

## 3. GET /companies/{companyId}/routing-requests/{routingRequestId}

**Descrição:** Consultar status do lote de roteirização.

### Sucesso 1 — Lote em processamento (parcialmente concluído)

**Request:**

```http
GET /vt/v1/companies/12345/routing-requests/7890
X-Company-Document-Root: 61190096
```

**Response:** `200 OK`

Header: `Retry-After: 10`

```json
{
  "routingRequestId": 7890,
  "status": "PROCESSING",
  "totalCount": 3,
  "processedCount": 2,
  "errorCount": 0,
  "pendingCount": 1,
  "completionPercentage": 66.7,
  "requestedAt": "2026-08-19T10:30:00-03:00",
  "completedAt": null,
  "elapsedTime": "2 min 30s",
  "beneficiaries": [
    {
      "beneficiaryId": 98765,
      "beneficiaryDocumentNumber": "09304730783",
      "fullName": "JOSIMAR FREIRE FERREIRA",
      "routeId": 5500,
      "status": "COMPLETED",
      "currentStep": "FINISHED",
      "updatedAt": "2026-08-19T10:31:45-03:00",
      "attempts": 3,
      "error": null,
      "errorDetail": null
    },
    {
      "beneficiaryId": 98767,
      "beneficiaryDocumentNumber": "78901234567",
      "fullName": "PEDRO HENRIQUE COSTA SILVA",
      "routeId": 5501,
      "status": "COMPLETED",
      "currentStep": "FINISHED",
      "updatedAt": "2026-08-19T10:32:10-03:00",
      "attempts": 2,
      "error": null,
      "errorDetail": null
    },
    {
      "beneficiaryId": 98770,
      "beneficiaryDocumentNumber": "11223344556",
      "fullName": "LUCAS MARTINS OLIVEIRA",
      "routeId": 5502,
      "status": "PROCESSING",
      "currentStep": "QUERY_ROUTE",
      "updatedAt": "2026-08-19T10:32:30-03:00",
      "attempts": 1,
      "error": null,
      "errorDetail": null
    }
  ]
}
```

---

### Sucesso 2 — Lote concluído com erros parciais

**Request:**

```http
GET /vt/v1/companies/12345/routing-requests/7890
X-Company-Document-Root: 61190096
```

**Response:** `200 OK`

```json
{
  "routingRequestId": 7890,
  "status": "COMPLETED_WITH_ERRORS",
  "totalCount": 3,
  "processedCount": 3,
  "errorCount": 1,
  "pendingCount": 0,
  "completionPercentage": 100.0,
  "requestedAt": "2026-08-19T10:30:00-03:00",
  "completedAt": "2026-08-19T10:35:22-03:00",
  "elapsedTime": "5 min 22s",
  "beneficiaries": [
    {
      "beneficiaryId": 98765,
      "beneficiaryDocumentNumber": "09304730783",
      "fullName": "JOSIMAR FREIRE FERREIRA",
      "routeId": 5500,
      "status": "COMPLETED",
      "currentStep": "FINISHED",
      "updatedAt": "2026-08-19T10:31:45-03:00",
      "attempts": 3,
      "error": null,
      "errorDetail": null
    },
    {
      "beneficiaryId": 98767,
      "beneficiaryDocumentNumber": "78901234567",
      "fullName": "PEDRO HENRIQUE COSTA SILVA",
      "routeId": 5501,
      "status": "COMPLETED",
      "currentStep": "FINISHED",
      "updatedAt": "2026-08-19T10:32:10-03:00",
      "attempts": 2,
      "error": null,
      "errorDetail": null
    },
    {
      "beneficiaryId": 98770,
      "beneficiaryDocumentNumber": "11223344556",
      "fullName": "LUCAS MARTINS OLIVEIRA",
      "routeId": 5502,
      "status": "ERROR",
      "currentStep": "QUERY_ROUTE",
      "updatedAt": "2026-08-19T10:35:22-03:00",
      "attempts": 5,
      "error": "Erro no cálculo da rota",
      "errorDetail": "Distancia menos de 1000 metros"
    }
  ]
}
```

---

### Erro — Solicitação não encontrada

**Request:**

```http
GET /vt/v1/companies/12345/routing-requests/99999
X-Company-Document-Root: 61190096
```

**Response:** `404 Not Found`

```json
{
  "timestamp": "2026-08-19T11:00:00Z",
  "status": 404,
  "error": "Not Found",
  "message": "Solicitação de roteirização 99999 não encontrada para a empresa 12345",
  "path": "/vt/v1/companies/12345/routing-requests/99999",
  "errors": null
}
```

---

## 4. GET /companies/{companyId}/routes/{routeId}

**Descrição:** Consultar detalhe da rota processada.

### Sucesso 1 — Rota completa com trajetos ida e volta (múltiplas linhas)

**Request:**

```http
GET /vt/v1/companies/12345/routes/5500
X-Company-Document-Root: 61190096
```

**Response:** `200 OK`

```json
{
  "routeId": 5500,
  "companyId": 12345,
  "routingRequestId": 7890,
  "beneficiaryId": 98765,
  "beneficiaryDocumentNumber": "09304730783",
  "fullName": "JOSIMAR FREIRE FERREIRA",
  "status": "COMPLETED",
  "requestedAt": "2026-08-19T10:30:00-03:00",
  "completedAt": "2026-08-19T10:31:45-03:00",
  "workAddress": {
    "street": "RUA SIQUEIRA CAMPOS",
    "number": "74",
    "complement": null,
    "neighborhood": "COPACABANA",
    "city": "RIO DE JANEIRO",
    "state": "RJ",
    "zipCode": "22440033",
    "referencePoint": "Próximo à estação de metrô Siqueira Campos"
  },
  "homeAddress": {
    "street": "RUA ANA CASEMIRO",
    "number": "27",
    "complement": "CASA",
    "neighborhood": "BENFICA",
    "city": "RIO DE JANEIRO",
    "state": "RJ",
    "zipCode": "20930190",
    "referencePoint": "Em frente à padaria São Jorge"
  },
  "outboundRouteLegs": [
    {
      "sequence": 1,
      "direction": "OUTBOUND",
      "lineCode": "397",
      "lineDescription": "PENHA X COPACABANA (VIA ATERRO DO FLAMENGO)",
      "instruction": "Pegar a linha 397 PENHA X COPACABANA (VIA ATERRO DO FLAMENGO)",
      "instructionDetail": "andar 320 metros até a RUA CONSELHEIRO OLEGÁRIO e embarcar, desembarcar na RUA SIQUEIRA CAMPOS e andar 80 metros",
      "boardingLocation": "RUA CONSELHEIRO OLEGÁRIO",
      "alightingLocation": "RUA SIQUEIRA CAMPOS",
      "boardingWalkMeters": 320,
      "alightingWalkMeters": 80,
      "estimatedTime": "Tempo estimado : 42.50 min.",
      "travelTimeMinutes": 42.5
    }
  ],
  "returnRouteLegs": [
    {
      "sequence": 1,
      "direction": "RETURN",
      "lineCode": "397",
      "lineDescription": "COPACABANA X PENHA (VIA ATERRO DO FLAMENGO)",
      "instruction": "Pegar a linha 397 COPACABANA X PENHA (VIA ATERRO DO FLAMENGO)",
      "instructionDetail": "andar 80 metros até a RUA SIQUEIRA CAMPOS e embarcar, desembarcar na RUA CONSELHEIRO OLEGÁRIO e andar 320 metros",
      "boardingLocation": "RUA SIQUEIRA CAMPOS",
      "alightingLocation": "RUA CONSELHEIRO OLEGÁRIO",
      "boardingWalkMeters": 80,
      "alightingWalkMeters": 320,
      "estimatedTime": "Tempo estimado : 45.10 min.",
      "travelTimeMinutes": 45.1
    }
  ],
  "totalOutboundTimeMinutes": 42.5,
  "totalReturnTimeMinutes": 45.1,
  "totalWalkingDistanceKm": 0.8,
  "suggestedBenefits": [
    {
      "productId": 283,
      "saleCode": "000283",
      "description": "FETRANSPOR",
      "quantity": 2,
      "unitPrice": 4.05,
      "subtotal": 8.1
    }
  ]
}
```

---

### Sucesso 2 — Rota com múltiplas linhas (baldeação) ida e volta

**Request:**

```http
GET /vt/v1/companies/12345/routes/5501
X-Company-Document-Root: 61190096
```

**Response:** `200 OK`

```json
{
  "routeId": 5501,
  "companyId": 12345,
  "routingRequestId": 7890,
  "beneficiaryId": 98767,
  "beneficiaryDocumentNumber": "78901234567",
  "fullName": "PEDRO HENRIQUE COSTA SILVA",
  "status": "COMPLETED",
  "requestedAt": "2026-08-19T10:30:00-03:00",
  "completedAt": "2026-08-19T10:32:10-03:00",
  "workAddress": {
    "street": "AV PRESIDENTE VARGAS",
    "number": "3131",
    "complement": "GALPÃO 2",
    "neighborhood": "CIDADE NOVA",
    "city": "RIO DE JANEIRO",
    "state": "RJ",
    "zipCode": "20210030",
    "referencePoint": "Ao lado do Terminal Américo Fontenelle"
  },
  "homeAddress": {
    "street": "RUA DAS LARANJEIRAS",
    "number": "150",
    "complement": "APTO 501",
    "neighborhood": "LARANJEIRAS",
    "city": "RIO DE JANEIRO",
    "state": "RJ",
    "zipCode": "22240003",
    "referencePoint": null
  },
  "outboundRouteLegs": [
    {
      "sequence": 1,
      "direction": "OUTBOUND",
      "lineCode": "METRO-L1",
      "lineDescription": "METRÔ LINHA 1 (LARANJEIRAS → CIDADE NOVA)",
      "instruction": "Pegar o Metrô Linha 1 sentido Uruguai",
      "instructionDetail": "andar 450 metros até a estação LARGO DO MACHADO e embarcar, desembarcar na estação CIDADE NOVA e andar 200 metros",
      "boardingLocation": "ESTAÇÃO LARGO DO MACHADO",
      "alightingLocation": "ESTAÇÃO CIDADE NOVA",
      "boardingWalkMeters": 450,
      "alightingWalkMeters": 200,
      "estimatedTime": "Tempo estimado : 12.00 min.",
      "travelTimeMinutes": 12.0
    },
    {
      "sequence": 2,
      "direction": "OUTBOUND",
      "lineCode": "475",
      "lineDescription": "COSME VELHO X CIDADE NOVA",
      "instruction": "Pegar a linha 475 COSME VELHO X CIDADE NOVA",
      "instructionDetail": "andar 150 metros e embarcar na AV PRESIDENTE VARGAS, desembarcar na AV PRESIDENTE VARGAS 3131 e andar 50 metros",
      "boardingLocation": "AV PRESIDENTE VARGAS",
      "alightingLocation": "AV PRESIDENTE VARGAS 3131",
      "boardingWalkMeters": 150,
      "alightingWalkMeters": 50,
      "estimatedTime": "Tempo estimado : 8.30 min.",
      "travelTimeMinutes": 8.3
    }
  ],
  "returnRouteLegs": [
    {
      "sequence": 1,
      "direction": "RETURN",
      "lineCode": "METRO-L1",
      "lineDescription": "METRÔ LINHA 1 (CIDADE NOVA → LARANJEIRAS)",
      "instruction": "Pegar o Metrô Linha 1 sentido General Osório",
      "instructionDetail": "andar 200 metros até a estação CIDADE NOVA e embarcar, desembarcar na estação LARGO DO MACHADO e andar 450 metros",
      "boardingLocation": "ESTAÇÃO CIDADE NOVA",
      "alightingLocation": "ESTAÇÃO LARGO DO MACHADO",
      "boardingWalkMeters": 200,
      "alightingWalkMeters": 450,
      "estimatedTime": "Tempo estimado : 12.00 min.",
      "travelTimeMinutes": 12.0
    }
  ],
  "totalOutboundTimeMinutes": 20.3,
  "totalReturnTimeMinutes": 12.0,
  "totalWalkingDistanceKm": 1.55,
  "suggestedBenefits": [
    {
      "productId": 9825,
      "saleCode": "009825",
      "description": "METRO LINHA 1 AZUL",
      "quantity": 2,
      "unitPrice": 6.9,
      "subtotal": 13.8
    },
    {
      "productId": 283,
      "saleCode": "000283",
      "description": "FETRANSPOR",
      "quantity": 1,
      "unitPrice": 4.05,
      "subtotal": 4.05
    }
  ]
}
```

---

### Erro — Rota não encontrada

**Request:**

```http
GET /vt/v1/companies/12345/routes/99999
X-Company-Document-Root: 61190096
```

**Response:** `404 Not Found`

```json
{
  "timestamp": "2026-08-19T12:00:00Z",
  "status": 404,
  "error": "Not Found",
  "message": "Rota 99999 não encontrada para a empresa 12345",
  "path": "/vt/v1/companies/12345/routes/99999",
  "errors": null
}
```

---

## Cenários de Erro Adicionais (aplicáveis a todos os endpoints)

### 401 — Credenciais inválidas

```json
{
  "timestamp": "2026-08-19T10:00:00Z",
  "status": 401,
  "error": "Unauthorized",
  "message": "Credenciais inválidas ou ausentes. Verifique o clientId/clientSecret.",
  "path": "/vt/v1/companies/12345/beneficiaries",
  "errors": null
}
```

### 403 — Empresa não pertence ao grupo econômico do clientId

```json
{
  "timestamp": "2026-08-19T10:00:00Z",
  "status": 403,
  "error": "Forbidden",
  "message": "Empresa 12345 não pertence à raiz de CNPJ vinculada ao clientId (61190096)",
  "path": "/vt/v1/companies/12345/beneficiaries",
  "errors": null
}
```

### 429 — Rate limit excedido (POST /routing-requests)

Header: `Retry-After: 60`

```json
{
  "timestamp": "2026-08-19T10:00:00Z",
  "status": 429,
  "error": "Too Many Requests",
  "message": "Limite de requisições excedido. Aguarde 60 segundos antes de tentar novamente.",
  "path": "/vt/v1/companies/12345/routing-requests",
  "errors": null
}
```

### 500 — Erro interno

```json
{
  "timestamp": "2026-08-19T10:00:00Z",
  "status": 500,
  "error": "Internal Server Error",
  "message": "Erro interno não tratado. Tente novamente mais tarde.",
  "path": "/vt/v1/companies/12345/routing-requests",
  "errors": null
}
```

### 502 — Bad Gateway

```json
{
  "timestamp": "2026-08-19T10:00:00Z",
  "status": 502,
  "error": "Bad Gateway",
  "message": "Dependência externa indisponível. O processamento da roteirização não pôde ser concluído.",
  "path": "/vt/v1/companies/12345/routing-requests",
  "errors": null
}
```

### 503 — Serviço indisponível

Header: `Retry-After: 30`

```json
{
  "timestamp": "2026-08-19T10:00:00Z",
  "status": 503,
  "error": "Service Unavailable",
  "message": "Serviço temporariamente indisponível. Tente novamente após o período indicado no header Retry-After.",
  "path": "/vt/v1/companies/12345/routing-requests",
  "errors": null
}
```

---

## Notas Gerais

### Headers comuns em todas as requisições

```http
Content-Type: application/json
Accept: application/json
X-Company-Document-Root: {raiz_cnpj_8_digitos}
```

### Headers específicos do POST /routing-requests

```http
X-Logged-User-Id: {id_operador}
```

### Base URLs

| Ambiente    | URL                                  |
| ----------- | ------------------------------------ |
| Produção    | `https://api.alelo.com.br/vt/v1`     |
| Homologação | `https://api-hml.alelo.com.br/vt/v1` |

---

## Fluxo Completo de Exemplo

```
1. Listar colaboradores elegíveis:
   GET /companies/12345/beneficiaries?eligible=true
   → Identifica IDs: [98765, 98767, 98770]

2. Solicitar roteirização:
   POST /companies/12345/routing-requests
   Body: { "beneficiaryIds": [98765, 98767, 98770] }
   → 202 { "routingRequestId": 7890, "beneficiaryCount": 3 }

3. Polling do status (a cada Retry-After segundos):
   GET /companies/12345/routing-requests/7890
   → 200 { "status": "PROCESSING", "completionPercentage": 66.7, ... }
   → 200 { "status": "COMPLETED_WITH_ERRORS", "completionPercentage": 100, ... }

4. Consultar rotas individuais:
   GET /companies/12345/routes/5500  → Rota completa de JOSIMAR
   GET /companies/12345/routes/5501  → Rota completa de PEDRO
   GET /companies/12345/routes/5502  → Erro: "Distancia menos de 1000 metros"
```
