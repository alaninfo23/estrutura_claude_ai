# API de Gestão de Vale Transporte

> **Documentação Técnica de Integração**  
> `vt-orders-management v1` · Atualizado em Setembro/2026

---

## Sumário

1. [O que mudou nesta versão](#1-o-que-mudou-nesta-versão)
2. [Visão Geral](#2-visão-geral)
3. [Autenticação e Ambiente](#3-autenticação-e-ambiente)
4. [Fluxo de Integração](#4-fluxo-de-integração)
5. [Módulos da API](#5-módulos-da-api)
   - [5.1 Empresas](#51-empresas)
   - [5.2 Produtos](#52-produtos)
   - [5.3 Colaboradores](#53-colaboradores)
   - [5.4 Pedidos](#54-pedidos)
   - [5.5 Validações](#55-validações)
   - [5.6 Relatórios](#56-relatórios)
6. [Referência de Endpoints](#6-referência-de-endpoints)
7. [Modelos de Dados](#7-modelos-de-dados)
8. [Códigos de Erro](#8-códigos-de-erro)
9. [Cenários de Teste (Mock)](#9-cenários-de-teste-mock)
10. [Boas Práticas](#10-boas-práticas)
11. [FAQ](#11-faq)

---

## 1. O que mudou nesta versão

Esta versão da documentação reflete o contrato atual do microsserviço `vt-orders-management v1`. As principais mudanças em relação à versão anterior estão listadas abaixo.

> ⚠️ **Breaking change — modelo de responsáveis pelo recebimento.**  
> Os campos achatados `receivingResponsible1`, `receivingResponsible2` e `receivingResponsible3` (e seus subcampos numerados) foram substituídos por uma única lista `receivingResponsibles[]`. Integrações que enviam ou leem os campos numerados precisam ser migradas para o formato de array.

| # | Mudança | Impacto | Onde |
|---|---------|---------|------|
| 1 | **Responsáveis pelo recebimento viraram lista.** Os campos `receivingResponsible1/2/3` (achatados) deram lugar ao array `receivingResponsibles[]`, com os subcampos `name`, `phoneType`, `phone` e `extension`. | Breaking — requer ajuste no request e no parse do response. | 4.1.2 Listar Endereços · 4.1.3 Cadastrar Endereço |
| 2 | **Limite de 3 responsáveis.** A lista `receivingResponsibles[]` está definida no contrato com no máximo 3 itens (`maxItems: 3`). Enviar no máximo 3 contatos por endereço. | Novo — validação no envio. | 4.1.2 · 4.1.3 |
| 3 | **Novo campo `phoneType`.** Cada responsável passa a ter o tipo do telefone de contato, com valores `PBX`, `DIRECT`, `MOBILE`, `FAX`, `PAGER` ou `MESSAGE`. Quando um responsável é informado, os dados de contato se tornam obrigatórios. | Novo — campo adicional por responsável. | 4.1.2 · 4.1.3 |
| 4 | **Listagens são paginadas.** Endpoints de listagem (ex.: `GET /companies/{id}/places` e `GET /companies/{id}/responsibles`) retornam um envelope paginado com `content[]`, `page`, `size`, `totalElements` e `totalPages`. Paginação zero-based (`page=0` é a primeira página). | Ajuste — leitura via `content[]` em vez de array direto. | 4.1.2 · 4.1.4 |

### Exemplo de migração do modelo de responsáveis

**Antes (achatado):**

```json
{
    "receivingResponsible1Name": "João Silva",
    "receivingResponsible1Phone": "11999998888",
    "receivingResponsible2Name": "Maria Souza",
    "receivingResponsible2Phone": "1133334444"
}
```

**Agora (lista, máximo 3 itens):**

```json
{
  "receivingResponsibles": [
    { "name": "João Silva", "phoneType": "MOBILE", "phone": "11999998888", "extension": null },
    { "name": "Maria Souza", "phoneType": "PBX", "phone": "1133334444", "extension": "205" }
  ]
}
```

---

## 2. Visão Geral

A API de Gestão de Vale Transporte permite que empresas clientes realizem de forma programática o gerenciamento completo do ciclo de vida do Vale Transporte, incluindo:

- Consulta e configuração de dados da empresa
- Cadastro e manutenção de colaboradores (beneficiários)
- Criação e acompanhamento de pedidos de VT
- Validação prévia de dados antes da submissão
- Download de boletos e notas fiscais
- Geração de relatórios consolidados

| Item | Valor |
|------|-------|
| Base Path | `/vt-orders-management/v1` |
| Protocolo | HTTPS |
| Formato | JSON (`application/json`) |
| Versionamento | v1 (path-based) |
| Paginação | Zero-based (`page=0` é a primeira página) |

---

## 3. Autenticação e Ambiente

### Ambientes

A URL base será disponibilizada pela equipe de integração durante o processo de onboarding.

### Credenciais

A autenticação é realizada via **Basic Authentication** (`clientId` como usuário e `clientSecret` como senha). As credenciais serão fornecidas pela equipe de integração durante o processo de onboarding e devem ser enviadas em todas as requisições.

### Headers Obrigatórios

```
Authorization: Basic {base64(client_id:client_secret)}
Content-Type: application/json
```

---

## 4. Fluxo de Integração

O fluxo recomendado para uma integração completa:

| # | Etapa | Ação |
|---|-------|------|
| 1 | **Empresa** | Consultar empresa pelo CNPJ — `POST /companies?action=SEARCH` |
| 2 | **Endereços** | Consultar/cadastrar endereços — `GET/POST /companies/{id}/places` |
| 3 | **Contatos** | Consultar responsáveis — `GET /companies/{id}/responsibles` |
| 4 | **Produtos** | Consultar catálogo disponível — `GET /products` |
| 5 | **Regras** | Consultar regras de validação — `GET /companies/{id}/rules-validate` |
| 6 | **Colaboradores** | Cadastrar/atualizar colaboradores — `POST /companies/{id}/beneficiaries` |
| 7 | **Validação** | (Opcional) Pré-validar dados — `POST /companies/{id}/orders-validate` |
| 8 | **Pedido** | Submeter pedido — `POST /companies/{id}/orders` |
| 9 | **Polling** | Acompanhar processamento — `GET /companies/{id}/processing/{uuid}` |
| 10 | **Aprovação** | Aprovar pedido (pós-pago) — `POST /companies/{id}/orders/{id}/approve` |
| 11 | **Documentos** | Boleto / Nota Fiscal — `GET .../payment-slip | .../invoice` |

---

## 5. Módulos da API

### 5.1 Empresas

O módulo de Empresas permite consultar dados cadastrais, endereços e contatos responsáveis.

#### 5.1.1 Consultar Empresa por CNPJ

Retorna os dados da empresa a partir do CNPJ informado.

```
POST /companies?action=SEARCH
```

**Request Body:**

```json
{
    "companyDocumentNumber": "12345678000199"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `companyDocumentNumber` | string | ✓ | CNPJ da empresa (14 caracteres) |

**Response (200):**

```json
{
    "content": [
      {
          "companyId": 12345,
          "companyDocumentNumber": "12345678000199",
          "corporateName": "Empresa Exemplo Ltda",
          "tradeName": "Empresa Exemplo",
          "contractNumber": "CT-001",
          "paymentMethod": "POST_PAID",
          "isHeadquarters": true
      }
    ],
    "page": 0,
    "size": 20,
    "totalElements": 1,
    "totalPages": 1
}
```

**Campos do Response (CompanySummary):**

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `companyId` | integer | Identificador único da empresa (use em todos os endpoints subsequentes) |
| `companyDocumentNumber` | string | CNPJ da empresa |
| `corporateName` | string | Razão social |
| `tradeName` | string | Nome fantasia |
| `stateRegistration` | string | Inscrição estadual |
| `municipalRegistration` | string | Inscrição municipal |
| `businessSegment` | string | Ramo de atividade |
| `contractNumber` | string | Número do contrato |
| `paymentMethod` | string | `PRE_PAID` ou `POST_PAID` |
| `paymentMethodDescription` | string | Descrição do meio de pagamento |
| `isHeadquarters` | boolean | Se é a matriz do grupo |

---

#### 5.1.2 Listar Endereços

Retorna os endereços cadastrados na empresa (cobrança, entrega ou faturamento/entrega).

```
GET /companies/{companyId}/places
```

**Parâmetros de Query (opcionais):**

| Parâmetro | Tipo | Descrição |
|-----------|------|-----------|
| `type` | string | `BILLING`, `DELIVERY` ou `BILLING_AND_DELIVERY` |
| `city` | string | Filtrar por cidade |
| `state` | string | UF (2 caracteres) |
| `zipCode` | string | CEP (8 dígitos, sem formatação) |
| `neighborhood` | string | Filtrar por bairro |
| `search` | string | Busca textual (logradouro, bairro, cidade) |
| `page` | integer | Página (inicia em 0) |
| `size` | integer | Registros por página (1-100, padrão 20) |
| `sort` | string | Ordenação (padrão: `id,asc`) |

**Tipos de Endereço:**

| Tipo | Finalidade |
|------|-----------|
| `BILLING` | Cobrança (boletos, faturas) |
| `DELIVERY` | Entrega (cartões/vales) |
| `BILLING_AND_DELIVERY` | Faturamento e entrega combinados |

**Exemplo de Response (200) — paginado:**

```json
{
    "content": [
      {
          "id": 12351682,
          "companyId": 835244,
          "type": "DELIVERY",
          "street": "Avenida Paulista",
          "number": "1000",
          "complement": "Sala 201",
          "neighborhood": "Bela Vista",
          "zipCode": "01310100",
          "city": "Sao Paulo",
          "state": "SP",
          "referencePoint": "Proximo ao metro Trianon-Masp",
          "receivingResponsibles": [
            {
                "name": "João Silva",
                "phoneType": "MOBILE",
                "phone": "11999998888",
                "extension": null
            }
          ]
      },
      {
          "id": 12451658,
          "companyId": 835244,
          "type": "BILLING",
          "street": "Rua Augusta",
          "number": "500",
          "complement": null,
          "neighborhood": "Consolacao",
          "zipCode": "01305000",
          "city": "Sao Paulo",
          "state": "SP",
          "referencePoint": null,
          "receivingResponsibles": []
      }
    ],
    "page": 0,
    "size": 20,
    "totalElements": 2,
    "totalPages": 1
}
```

**Campos do Response (paginação):**

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `content` | array\<Address\> | Lista de endereços da página atual |
| `page` | integer | Número da página atual (inicia em 0) |
| `size` | integer | Quantidade de itens por página |
| `totalElements` | integer | Total de registros existentes |
| `totalPages` | integer | Total de páginas disponíveis |

**Campos do Response (Address):**

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | integer | Identificador único do endereço (use como `deliveryAddressId` ou `billingAddressId` no pedido) |
| `companyId` | integer | Identificador da empresa |
| `type` | string | `BILLING`, `DELIVERY` ou `BILLING_AND_DELIVERY` |
| `street` | string | Logradouro |
| `number` | string | Número |
| `complement` | string | Complemento |
| `neighborhood` | string | Bairro |
| `zipCode` | string | CEP (8 dígitos) |
| `city` | string | Cidade |
| `state` | string | UF (2 caracteres) |
| `referencePoint` | string | Ponto de referência |
| `receivingResponsibles` | array\<object\> | Lista de contatos responsáveis pelo recebimento no local (no máximo 3) |
| `receivingResponsibles[].name` | string | Nome do responsável pelo recebimento |
| `receivingResponsibles[].phoneType` | string | Tipo do telefone de contato: `PBX`, `DIRECT`, `MOBILE`, `FAX`, `PAGER` ou `MESSAGE` |
| `receivingResponsibles[].phone` | string | Número do telefone de contato com DDD |
| `receivingResponsibles[].extension` | string | Ramal telefônico para contato interno |

> **Limite:** a lista `receivingResponsibles[]` retorna no máximo 3 contatos por endereço.

---

#### 5.1.3 Cadastrar Endereço

```
POST /companies/{companyId}/places
```

**Request Body:**

```json
{
    "type": "DELIVERY",
    "street": "Rua Exemplo",
    "number": "100",
    "complement": "Sala 201",
    "neighborhood": "Centro",
    "zipCode": "01001000",
    "city": "Sao Paulo",
    "state": "SP",
    "referencePoint": "Proximo ao metro",
    "receivingResponsibles": [
        {
            "name": "João Silva",
            "phoneType": "MOBILE",
            "phone": "11999998888",
            "extension": "201"
        }
    ]
}
```

| Campo | Tipo | Obrig. | Max | Descrição |
|-------|------|--------|-----|-----------|
| `type` | string | ✓ | - | `BILLING`, `DELIVERY` ou `BILLING_AND_DELIVERY` |
| `street` | string | ✓ | 50 | Logradouro |
| `number` | string | ✓ | 15 | Número |
| `complement` | string | - | 20 | Complemento |
| `neighborhood` | string | ✓ | 35 | Bairro |
| `zipCode` | string | ✓ | 8 | CEP (apenas números) |
| `city` | string | ✓ | 30 | Cidade |
| `state` | string | ✓ | 2 | UF (sigla) |
| `referencePoint` | string | - | 100 | Ponto de referência |
| `receivingResponsibles` | array\<object\> | - | 3 itens | Lista de contatos responsáveis pelo recebimento |
| `receivingResponsibles[].name` | string | - | 60 | Nome do responsável pelo recebimento |
| `receivingResponsibles[].phoneType` | string | - | - | Tipo do telefone: `PBX`, `DIRECT`, `MOBILE`, `FAX`, `PAGER` ou `MESSAGE` |
| `receivingResponsibles[].phone` | string | - | 20 | Telefone de contato com DDD |
| `receivingResponsibles[].extension` | string | - | 10 | Ramal telefônico para contato interno |

> **Limite:** o contrato define `receivingResponsibles[]` com no máximo 3 itens (`maxItems: 3`); não enviar mais que 3. Ao informar um responsável, preencha os dados de contato (`name`, `phoneType`, `phone`).

---

#### 5.1.4 Listar Contatos Responsáveis

Retorna os contatos cadastrados na empresa. Use este endpoint para obter o `id` do contato responsável ao criar um pedido.

```
GET /companies/{companyId}/responsibles
```

| Parâmetro | Tipo | Descrição |
|-----------|------|-----------|
| `name` | string | Nome (busca parcial, case-insensitive) |
| `documentNumber` | string | CPF (11 dígitos) |
| `email` | string | Email (busca parcial) |
| `department` | string | Departamento |
| `position` | string | Cargo |
| `page` | integer | Página (inicia em 0) |
| `size` | integer | Registros por página (1-100, padrão 20) |

**Exemplo de Response (200) — paginado:**

```json
{
    "content": [
         {
             "id": 45012,
             "name": "Maria Souza",
             "documentNumber": "12345678901",
             "gender": "FEMALE",
             "birthDate": "1988-05-14",
             "email": "maria.souza@empresa.com.br",
             "phoneAreaCode": "11",
             "phoneNumber": "988887777",
             "extension": "305",
             "department": "Recursos Humanos",
             "position": "Analista de Benefícios"
         }
    ],
    "page": 0,
    "size": 20,
    "totalElements": 1,
    "totalPages": 1
}
```

**Campos do Response (Contact):**

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | integer | Identificador único (use como `responsibleContactId` no pedido) |
| `name` | string | Nome do contato |
| `documentNumber` | string | CPF (11 dígitos) |
| `gender` | string | `MALE`, `FEMALE` ou `OTHER` |
| `birthDate` | date | Data de nascimento |
| `email` | string | Email |
| `phoneAreaCode` | string | DDD (2 dígitos) |
| `phoneNumber` | string | Telefone (8 ou 9 dígitos) |
| `extension` | string | Ramal |
| `department` | string | Departamento ou setor |
| `position` | string | Cargo ou função |

---

### 5.2 Produtos

O catálogo de produtos lista os benefícios e linhas de transporte disponíveis para comercialização.

#### 5.2.1 Listar Produtos Disponíveis

```
GET /products
```

| Parâmetro | Tipo | Descrição |
|-----------|------|-----------|
| `type` | string | `CARD` ou `PAPER` |
| `deliveryAddressId` | integer | Filtrar por região do endereço de entrega (recomendado para pedido `PAPER`) |
| `saleCode` | string | Filtrar por código de venda exato (quando já conhecido) |
| `search` | string | Busca por descrição do produto |
| `page` | integer | Página (inicia em 0) |
| `size` | integer | Registros por página (1-100, padrão 20) |

> **Importante:** Utilize o campo `saleCode` ao vincular benefícios aos colaboradores. O `productId` é utilizado apenas como referência para consultas.

**Campos do Response (Product):**

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `productId` | integer | Identificador do produto |
| `saleCode` | string | Código de venda (use este ao cadastrar benefícios) |
| `description` | string | Descrição do produto/linha de transporte |
| `type` | string | `CARD` (cartão eletrônico) ou `PAPER` (vale papel) |
| `active` | boolean | Se está ativo para comercialização |
| `isElectronicCard` | boolean | Se é cartão eletrônico |
| `requiresSalary` | boolean | Se exige salário preenchido |
| `requiresDocumentNumber` | boolean | Se exige CPF |
| `requiresIdNumber` | boolean | Se exige RG |
| `requiresBirthDate` | boolean | Se exige data de nascimento |
| `requiresMotherName` | boolean | Se exige nome da mãe |
| `requiresHomeAddress` | boolean | Se exige endereço residencial |
| `blocksIfBeneficiaryWithoutCard` | boolean | Se bloqueia quando colaborador não possui cartão |
| `minRechargeValue` | number | Valor mínimo de recarga (R$) |
| `maxRechargeValue` | number | Valor máximo de recarga (R$) |
| `minFirstRechargeValue` | number | Valor mínimo para primeira recarga |
| `minUnitPrice` | number | Valor unitário mínimo |
| `maxUnitPrice` | number | Valor unitário máximo |
| `rechargeBillingValue` | number | Valor de cobrança da recarga |

---

### 5.3 Colaboradores

O módulo de Colaboradores permite o cadastro, consulta e atualização de beneficiários de Vale Transporte.

#### 5.3.1 Cadastrar/Atualizar Colaborador (Upsert)

Cadastra ou atualiza um colaborador com execução síncrona.

> **Comportamento (Upsert):**
> - Se o CPF não existe na empresa → cria o colaborador (retorna 201)
> - Se o CPF já existe com a mesma matrícula → atualiza os dados (retorna 200)
> - Se o CPF já existe com matrícula diferente → erro `USU-002` (retorna 409)

```
POST /companies/{companyId}/beneficiaries
```

**Request Body:**

```json
{
    "beneficiary": {
      "registrationNumber": "MAT001",
        "fullName": "Maria Oliveira Santos",
        "beneficiaryDocumentNumber": "12345678901",
        "idNumber": "123456789",
        "idNumberDigit": "X",
        "idNumberState": "SP",
        "birthDate": "1990-05-15",
        "department": "Tecnologia",
        "position": "Analista",
        "workingDaysPerMonth": 22,
        "salary": 5500.00,
        "workAddressId": 1001,
        "homeAddress": {
          "street": "Rua Residencial",
            "number": "50",
            "complement": "Apto 12",
            "neighborhood": "Liberdade",
            "zipCode": "01503010",
            "city": "Sao Paulo",
            "state": "SP",
            "requestRouteSuggestion": false
        },
        "motherName": "Ana Santos",
        "email": "maria.santos@email.com",
        "phoneAreaCode": "11",
        "phoneNumber": "999887766",
        "bankDetails": {
          "pixKeyType": 1,
          "pixKey": "12345678901"
        },
        "benefits": [
            {
                "saleCode": "9825",
                "unitPrice": 4.40,
                "usagePerDay": 2
            }
        ]
    }
}
```

**Campos do Colaborador:**

| Campo | Tipo | Obrig. | Max | Descrição |
|-------|------|--------|-----|-----------|
| `registrationNumber` | string | ✓ | 15 | Matrícula na empresa |
| `fullName` | string | ✓ | 50 | Nome completo |
| `beneficiaryDocumentNumber` | string | ✓ | 11 | CPF (apenas números) |
| `idNumber` | string | ✓ | 9 | Número do RG |
| `idNumberDigit` | string | - | 2 | Dígito do RG |
| `idNumberState` | string | ✓ | 2 | UF de emissão do RG |
| `birthDate` | date | ✓ | - | Data de nascimento (`YYYY-MM-DD`) |
| `department` | string | - | 45 | Departamento ou setor |
| `position` | string | - | 45 | Cargo ou função |
| `workingDaysPerMonth` | integer | ✓ | 31 | Dias trabalhados por mês (1 a 31) |
| `salary` | number | * | - | Salário (R$). Mín: 0.01 |
| `workAddressId` | integer | ✓ | - | ID do endereço de trabalho (obtido em `/places`) |
| `homeAddress` | object | * | - | Endereço residencial |
| `motherName` | string | * | 50 | Nome da mãe |
| `email` | string | ✓ | 60 | Email |
| `phoneAreaCode` | string | ✓ | 2 | DDD |
| `phoneNumber` | string | ✓ | 9 | Telefone (8 ou 9 dígitos) |
| `bankDetails` | object | - | - | Dados bancários/PIX |
| `benefits` | array | ✓ | - | Lista de benefícios (mín. 1 item) |

> \* Campos marcados com `*` podem ser obrigatórios dependendo do produto. Consulte as regras de validação (`GET /companies/{id}/rules-validate`) para verificar quais campos são exigidos por cada produto.

**Campos do Benefício (`benefits[]`):**

| Campo | Tipo | Obrig. | Descrição |
|-------|------|--------|-----------|
| `saleCode` | string | ✓ | Código de venda do produto |
| `unitPrice` | number | ✓ | Valor unitário (R$). Mín: 0.01 |
| `usagePerDay` | integer | ✓ | Utilizações por dia. Mín: 1 |

**Campos do Endereço Residencial (`homeAddress`):**

| Campo | Tipo | Obrig. | Max | Descrição |
|-------|------|--------|-----|-----------|
| `street` | string | ✓ | 50 | Logradouro |
| `number` | string | ✓ | 15 | Número |
| `complement` | string | - | 20 | Complemento |
| `neighborhood` | string | ✓ | 35 | Bairro |
| `zipCode` | string | ✓ | 8 | CEP (apenas números, sem formatação) |
| `city` | string | ✓ | 30 | Cidade |
| `state` | string | ✓ | 2 | UF (sigla de 2 caracteres) |
| `requestRouteSuggestion` | boolean | - | - | Solicitar sugestão de rota (padrão: `false`) |

**Campos dos Dados Bancários (`bankDetails`):**

| Campo | Tipo | Obrig. | Max | Descrição |
|-------|------|--------|-----|-----------|
| `pixKeyType` | integer | ✓ | - | Tipo de chave PIX |
| `pixKey` | string | - | 77 | Valor da chave PIX |
| `bankCode` | string | - | 5 | Código do banco |
| `branchNumber` | string | - | 10 | Número da agência |
| `accountNumber` | string | - | 15 | Número da conta |
| `accountDigit` | string | - | 2 | Dígito da conta |

---

#### 5.3.2 Listar Colaboradores

```
GET /companies/{companyId}/beneficiaries
```

Retorna a lista paginada de colaboradores. Suporta filtros por nome, CPF, matrícula, departamento, cargo, status e busca textual.

| Parâmetro | Tipo | Descrição |
|-----------|------|-----------|
| `beneficiaryDocumentNumber` | string | Filtrar por CPF exato (11 dígitos) |
| `registrationNumber` | string | Filtrar por matrícula exata |
| `name` | string | Filtrar por nome (busca parcial, case-insensitive) |
| `department` | string | Filtrar por departamento (busca parcial) |
| `position` | string | Filtrar por cargo (busca parcial) |
| `workAddressId` | integer | Filtrar por ID do endereço de trabalho |
| `status` | string | `ACTIVE` ou `INACTIVE` |
| `search` | string | Busca textual livre (nome, CPF, matrícula, departamento, cargo) |
| `page` | integer | Página (inicia em 0) |
| `size` | integer | Registros por página (1-200, padrão 50) |
| `sort` | string | Ordenação (padrão: `nome,asc`) |

---

#### 5.3.3 Atualizar Parcialmente (PATCH)

Atualiza apenas os campos enviados no body. Campos omitidos permanecem inalterados.

```
PATCH /companies/{companyId}/beneficiaries/{beneficiaryId}
```

> **Regra de benefícios no PATCH:**
> - Se `benefits` não for enviado → benefícios existentes permanecem inalterados
> - Se `benefits` for enviado (mesmo vazio) → substitui completamente a lista existente

```json
{
    "salary": 6200.00,
    "workingDaysPerMonth": 20
}
```

---

### 5.4 Pedidos

#### 5.4.1 Submeter Pedido

O processamento de pedidos é sempre **assíncrono**. Ao submeter, você recebe um `processingId` (UUID) para acompanhar o status via polling.

```
POST /companies/{companyId}/orders
```

**Request Body:**

```json
{
    "deliveryAddressId": 1001,
    "billingAddressId": 2001,
    "responsibleContactId": 3001,
    "paymentDate": "2026-09-05",
    "usagePeriod": {
      "startDate": "2026-09-01",
         "endDate": "2026-09-30"
    },
    "invoiceEmail1": "financeiro@empresa.com.br",
    "invoiceEmail2": "rh@empresa.com.br",
    "modality": "FULL",
    "scope": "COMPLETE"
}
```

**Campos do Pedido:**

| Campo | Tipo | Obrig. | Descrição |
|-------|------|--------|-----------|
| `deliveryAddressId` | integer | ✓ | ID do endereço de entrega (obtido em `/places`) |
| `billingAddressId` | integer | ✓ | ID do endereço de cobrança |
| `responsibleContactId` | integer | ✓ | ID do contato responsável (obtido em `/responsibles`) |
| `paymentDate` | date | ✓ | Data de pagamento (`YYYY-MM-DD`) |
| `usagePeriod.startDate` | date | ✓ | Início do período de utilização |
| `usagePeriod.endDate` | date | ✓ | Fim do período de utilização |
| `invoiceEmail1` | string | ✓ | Email para envio de fatura (até 60 chars) |
| `invoiceEmail2` | string | - | Email adicional para fatura |
| `modality` | string | ✓ | Modalidade (ver tabela abaixo) |
| `scope` | string | ✓ | `COMPLETE` ou `PARTIAL` |
| `beneficiaryDocumentNumbers` | array | * | CPFs para pedido parcial |
| `products` | array | * | Produtos (para modalidade `PAPER`) |

**Modalidades de Pedido:**

| Modalidade | Descrição |
|------------|-----------|
| `FULL` | Pedido completo com cálculo integral de valores |
| `RECHARGE_WITH_DISCOUNT` | Recarga Certa com desconto aplicado |
| `PAPER` | Vale papel (requer campo `products`) |

**Escopo do Pedido:**

| Escopo | Descrição |
|--------|-----------|
| `COMPLETE` | Processa todos os colaboradores ativos da empresa |
| `PARTIAL` | Processa apenas os CPFs informados em `beneficiaryDocumentNumbers` |

**Campos de `products[]` (obrigatório quando modalidade = `PAPER`):**

| Campo | Tipo | Obrig. | Descrição |
|-------|------|--------|-----------|
| `productId` | integer | ✓ | Identificador do produto (obtido em `GET /products`) |
| `quantity` | integer | ✓ | Quantidade. Mín: 1 |

**Exemplo de Request — Pedido PAPER (vale papel):**

```json
{
    "deliveryAddressId": 1001,
    "billingAddressId": 2001,
    "responsibleContactId": 3001,
    "paymentDate": "2026-09-05",
    "usagePeriod": {
      "startDate": "2026-09-01",
         "endDate": "2026-09-30"
    },
    "modality": "PAPER",
    "products": [
         { "productId": 9825, "quantity": 44 },
         { "productId": 4012, "quantity": 22 }
    ]
}
```

> **Pedido PAPER:** O campo `products` é obrigatório (ao menos 1 item). Os campos `scope` e `beneficiaryDocumentNumbers` são ignorados. Cada `productId` deve existir na região do `deliveryAddressId` informado — consulte previamente com `GET /products?type=PAPER&deliveryAddressId={id}`.

**Response (202 - Aceito para processamento):**

```json
{
    "processingId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "companyId": 12345,
    "status": "PROCESSING",
    "statusDescription": "Pedido aceito para processamento",
    "createdAt": "2026-08-26T10:30:00Z",
    "processingUrl": "/companies/12345/processing/a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

**Campos do Response (OrderAccepted):**

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `processingId` | string (UUID) | UUID de rastreamento — use no polling |
| `companyId` | integer | Identificador da empresa |
| `status` | string | Sempre `PROCESSING` neste momento |
| `statusDescription` | string | Descrição legível do status |
| `createdAt` | datetime | Data/hora de criação |
| `processingUrl` | string | URL relativa para acompanhamento via polling |

---

#### 5.4.2 Acompanhar Processamento (Polling)

```
GET /companies/{companyId}/processing/{processingId}
```

**Response (200):**

```json
{
    "processingId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "orderId": 98765,
    "status": "COMPLETED",
    "statusDescription": "Processamento concluido com sucesso",
    "currentStep": 19,
    "totalSteps": 19,
    "completionPercentage": 100,
    "totalBeneficiariesEvaluated": 150,
    "totalValidBeneficiaries": 148,
    "totalBeneficiariesWithError": 2,
    "startedAt": "2026-08-26T10:30:00Z",
    "finishedAt": "2026-08-26T10:35:22Z"
}
```

**Status de Processamento:**

| Status | Descrição |
|--------|-----------|
| `PROCESSING` | Em andamento (continue o polling) |
| `COMPLETED` | Concluído com sucesso |
| `ERROR` | Erro no processamento (consulte `validationErrors`) |

**Campos do Response (ProcessingStatus):**

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `processingId` | string (UUID) | UUID de rastreamento |
| `orderId` | integer \| null | ID do pedido gerado (preenchido quando `COMPLETED`) |
| `status` | string | `PROCESSING`, `COMPLETED` ou `ERROR` |
| `statusDescription` | string | Descrição legível do status |
| `currentStep` | integer | Etapa atual do processamento (0 a 19) |
| `totalSteps` | integer | Total de etapas |
| `completionPercentage` | integer | Percentual de conclusão (0 a 100) |
| `totalBeneficiariesEvaluated` | integer \| null | Total de beneficiários avaliados |
| `totalValidBeneficiaries` | integer \| null | Total de beneficiários válidos |
| `totalBeneficiariesWithError` | integer \| null | Total com erro de validação |
| `startedAt` | datetime | Data/hora de início do processamento |
| `finishedAt` | datetime \| null | Data/hora de conclusão (null se ainda em andamento) |
| `error` | string \| null | Descrição do erro (quando `status = ERROR`) |
| `validationErrors` | object \| null | Detalhamento dos erros por beneficiário/produto |

> **Recomendação:** Intervalo de polling de 5 a 10 segundos. Não ultrapasse 1 requisição por segundo.

---

#### 5.4.3 Consultar Pedido

```
GET /companies/{companyId}/orders/{orderId}
```

Retorna os dados completos do pedido, incluindo valores, taxas e itens por beneficiário.

**Campos do Response (Order):**

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `orderId` | integer | Identificador do pedido |
| `companyId` | integer | Identificador da empresa |
| `companyDocumentNumber` | string | CNPJ da empresa |
| `companyName` | string | Nome da empresa |
| `modality` | string | `FULL`, `RECHARGE_WITH_DISCOUNT` ou `PAPER` |
| `status` | string | Status atual do pedido (ver ciclo de vida) |
| `createdAt` | datetime | Data/hora de criação |
| `issuedAt` | datetime | Data/hora de emissão |
| `paymentDate` | date | Data de pagamento |
| `deliveryDate` | date | Data de entrega |
| `usagePeriod.startDate` | date | Início do período de utilização |
| `usagePeriod.endDate` | date | Fim do período de utilização |
| `contact.id` | integer | ID do contato responsável |
| `contact.name` | string | Nome do contato responsável |
| `beneficiaryCount` | integer | Quantidade de colaboradores no pedido |
| `financialStatus` | string | Status financeiro |
| `operationalStatus` | string | Status operacional |
| `itemsValue` | number | Valor dos itens (R$) |
| `ratesValue` | number | Valor das taxas (R$) |
| `totalValue` | number | Valor total do pedido (R$) |
| `discountValue` | number | Valor do desconto (R$) |
| `benefitsValue` | number | Valor dos benefícios (R$) |
| `rechargeCostValue` | number | Custo de recarga (R$) |
| `firstCardIssuanceValue` | number | Valor de emissão de 1ª via (R$) |
| `secondCardIssuanceValue` | number | Valor de emissão de 2ª via (R$) |
| `otherCostsValue` | number | Outros custos (R$) |
| `additionalBillingValue` | number | Valor adicional de cobrança (R$) |
| `hasBankTicket` | boolean | Possui boleto disponível para download |
| `hasInvoice` | boolean | Possui nota fiscal disponível para download |
| `rates[]` | array | Lista de taxas aplicadas ao pedido |
| `items[]` | array | Itens do pedido (beneficiários e seus benefícios) |

**Campos de `items[]` (OrderItem):**

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `beneficiaryDocumentNumber` | string | CPF do beneficiário |
| `fullName` | string | Nome completo |
| `benefits[]` | array | Benefícios do colaborador (`productId`, `saleCode`, `description`, `unitPrice`, `quantity`, `cardNumber`) |
| `beneficiaryTotalValue` | number | Valor total do beneficiário (R$) |

---

#### 5.4.4 Listar Pedidos

```
GET /companies/{companyId}/orders
```

| Parâmetro | Tipo | Descrição |
|-----------|------|-----------|
| `status` | string | Filtrar por status |
| `startDate` | date | Data de criação (início) |
| `endDate` | date | Data de criação (fim) |
| `page` | integer | Página |
| `size` | integer | Registros por página |

**Status do Pedido (ciclo de vida):**

| Status | Descrição |
|--------|-----------|
| `PENDING` | Pendente de aprovação (pós-pago) |
| `CONFIRMED` | Aprovado/confirmado |
| `PRODUCTION` | Em produção |
| `SHIPPING` | Em transporte |
| `PARTIALLY_DELIVERED` | Entregue parcialmente |
| `DELIVERED` | Entregue |
| `COMPLETED` | Finalizado |
| `CANCELLED` | Cancelado |
| `ERROR` | Erro no processamento |

**Campos do Response (OrderSummary — listagem):**

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `orderId` | integer | Identificador do pedido |
| `companyId` | integer | Identificador da empresa |
| `modality` | string | Modalidade do pedido |
| `status` | string | Status atual |
| `createdAt` | datetime | Data/hora de criação |
| `paymentDate` | date | Data de pagamento |
| `usagePeriod` | object | Período de utilização (`startDate`, `endDate`) |
| `contact` | object | Contato responsável (`id`, `name`) |
| `financialStatus` | string | Status financeiro |
| `operationalStatus` | string | Status operacional |
| `itemsValue` | number | Valor dos itens (R$) |
| `rateValue` | number | Valor da taxa (R$) |
| `orderValue` | number | Valor total do pedido (R$) |
| `additionsValue` | number | Valor de acréscimos (R$) |
| `discountsValue` | number | Valor de descontos (R$) |
| `hasBankTicket` | boolean | Possui boleto |
| `hasInvoice` | boolean | Possui nota fiscal |

---

#### 5.4.5 Aprovar Pedido (Pós-pago)

Para empresas com método de pagamento pós-pago, os pedidos são criados com status `PENDING` e requerem aprovação explícita.

```
POST /companies/{companyId}/orders/{orderId}/approve
```

**Response:** `204 No Content` (sucesso)

---

#### 5.4.6 Cancelar Pedido

Cancela um pedido já criado, desde que as regras de negócio permitam.

```
POST /companies/{companyId}/orders/{orderId}/cancel
```

> **Regras de cancelamento:** Pedidos somente podem ser cancelados antes do pagamento do boleto (pré-pago) ou antes da liberação para produção (pós-pago). O pedido deve estar em status `PENDING` ou `CONFIRMED`.

**Cenários de erro 409 (cancelamento recusado):**

| Motivo | Descrição |
|--------|-----------|
| `FASE_NAO_PERMITE` | Pedido já está em produção, expedição ou entregue |
| `STATUS_NAO_PERMITE` | Pedido já está cancelado ou rejeitado |
| `PAGAMENTO_EFETIVADO` | Pagamento do boleto já foi compensado (pré-pago) |
| `LIBERADO_PRODUCAO` | Pedido já foi liberado para produção (pós-pago) |

**Response:** `204 No Content` (sucesso)

---

#### 5.4.7 Download de Boleto

```
GET /companies/{companyId}/orders/{orderId}/payment-slip
```

Retorna uma URL pré-assinada (válida por 3 minutos) para download do boleto em PDF.

```json
{
    "type": "BANK_TICKET",
    "format": "PDF",
    "downloadUrl": "https://storage.example.com/boleto.pdf?token=...",
    "generatedAt": "2026-08-26T10:00:00Z",
    "expiresAt": "2026-08-26T11:00:00Z"
}
```

**Campos do Response (DocumentDownload):**

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `type` | string | `BANK_TICKET` (boleto) ou `INVOICE` (nota fiscal) |
| `format` | string | Formato do arquivo (`PDF`) |
| `downloadUrl` | string (URI) | URL pré-assinada para download |
| `generatedAt` | datetime | Data/hora de geração da URL |
| `expiresAt` | datetime | Data/hora de expiração (3 minutos após geração) |

---

#### 5.4.8 Download de Nota Fiscal

```
GET /companies/{companyId}/orders/{orderId}/invoice
```

Retorna a URL para download/visualização da nota fiscal (NFe) em PDF.

---

### 5.5 Validações

#### 5.5.1 Consultar Regras de Validação

Retorna as regras que serão aplicadas na criação do pedido, permitindo pré-validação local.

```
GET /companies/{companyId}/rules-validate
```

**Flags de Obrigatoriedade por Produto:**

| Flag | Quando `true` |
|------|--------------|
| `requiresSalary` | Campo `salary` obrigatório no colaborador |
| `requiresDocumentNumber` | CPF obrigatório |
| `requiresIdNumber` | RG obrigatório |
| `requiresBirthDate` | Data de nascimento obrigatória |
| `requiresMotherName` | Nome da mãe obrigatório |
| `requiresHomeAddress` | Endereço residencial obrigatório |
| `requiresUnitPrice` | Valor unitário obrigatório no benefício |
| `blocksIfBeneficiaryWithoutCard` | Bloqueia se colaborador não possui cartão ativo |

---

#### 5.5.2 Pré-validar Dados dos Colaboradores

Executa validações de forma síncrona sem criar pedido nem calcular valores.

```
POST /companies/{companyId}/orders-validate
```

```json
// Validação completa
{ "scope": "COMPLETE" }

// Validação parcial
{
    "scope": "PARTIAL",
    "beneficiaryDocumentNumbers": ["12345678901", "98765432100"]
}
```

**Categorias de Erro na Validação:**

| Categoria | Descrição |
|-----------|-----------|
| `COMPANY_PREREQUISITE` | Pré-requisito da empresa não atendido |
| `PRODUCT_UNAVAILABLE` | Produto indisponível |
| `SALARY_REQUIRED` | Salário obrigatório |
| `ELECTRONIC_CARD` | Problema com cartão eletrônico |
| `PERSONAL_DATA` | Dados pessoais incompletos |
| `ADDRESS` | Problema com endereço |
| `RETURN` | Regra de devolução violada |
| `VALUE_LIMIT` | Valor fora dos limites permitidos |

---

### 5.6 Relatórios

#### 5.6.1 Solicitar Relatório Consolidado

A geração de relatórios é assíncrona. O endpoint retorna um `reportId` para acompanhamento.

```
POST /companies/{companyId}/reports?orderId={orderId}&type=consolidated
```

| Parâmetro | Tipo | Obrig. | Descrição |
|-----------|------|--------|-----------|
| `orderId` | integer | ✓ | ID do pedido para gerar o relatório |
| `type` | string | ✓ | Tipo do relatório: `consolidated` |

**Campos do Response (ReportRequest — 202):**

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `reportId` | string (UUID) | UUID do relatório — use no polling de status |
| `type` | string | `CONSOLIDATED` |
| `format` | string | Formato do arquivo (`XLS`) |
| `status` | string | Sempre `PROCESSING` neste momento |
| `requestedAt` | datetime | Data/hora da solicitação |

---

#### 5.6.2 Consultar Status do Relatório

```
GET /companies/{companyId}/reports/{reportId}
```

**Campos do Response (ReportStatus):**

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `reportId` | string (UUID) | UUID do relatório |
| `type` | string | `CONSOLIDATED` |
| `format` | string | Formato do arquivo (`XLS`) |
| `status` | string | `PROCESSING`, `COMPLETED` ou `ERROR` |
| `requestedAt` | datetime | Data/hora da solicitação |
| `completedAt` | datetime \| null | Data/hora da conclusão |
| `downloadUrl` | string (URI) \| null | URL para download (preenchido quando `COMPLETED`) |
| `error` | string \| null | Descrição do erro (quando `ERROR`) |

```json
{
    "reportId": "f1e2d3c4-b5a6-7890-1234-567890abcdef",
    "type": "CONSOLIDATED",
    "format": "XLS",
    "status": "COMPLETED",
    "requestedAt": "2026-08-26T10:00:00Z",
    "completedAt": "2026-08-26T10:02:15Z",
    "downloadUrl": "https://storage.example.com/relatorio.xls?token=..."
}
```

---

## 6. Referência de Endpoints

| Método | Endpoint | Descrição |
|--------|----------|-----------|
| `POST` | `/companies?action=SEARCH` | Consultar empresa por CNPJ |
| `GET` | `/companies/{companyId}/places` | Listar endereços |
| `POST` | `/companies/{companyId}/places` | Cadastrar endereço |
| `GET` | `/companies/{companyId}/responsibles` | Listar contatos responsáveis |
| `GET` | `/products` | Listar produtos disponíveis |
| `GET` | `/companies/{companyId}/rules-validate` | Consultar regras de validação |
| `GET` | `/companies/{companyId}/beneficiaries` | Listar colaboradores |
| `POST` | `/companies/{companyId}/beneficiaries` | Cadastrar/atualizar colaborador |
| `PATCH` | `/companies/{companyId}/beneficiaries/{beneficiaryId}` | Atualizar parcialmente |
| `POST` | `/companies/{companyId}/orders-validate` | Pré-validar dados |
| `POST` | `/companies/{companyId}/orders` | Submeter pedido |
| `GET` | `/companies/{companyId}/processing/{processingId}` | Status do processamento |
| `GET` | `/companies/{companyId}/orders` | Listar pedidos |
| `GET` | `/companies/{companyId}/orders/{orderId}` | Consultar pedido |
| `POST` | `/companies/{companyId}/orders/{orderId}/approve` | Aprovar pedido |
| `POST` | `/companies/{companyId}/orders/{orderId}/cancel` | Cancelar pedido |
| `GET` | `/companies/{companyId}/orders/{orderId}/payment-slip` | Download boleto |
| `GET` | `/companies/{companyId}/orders/{orderId}/invoice` | Download nota fiscal |
| `POST` | `/companies/{companyId}/reports` | Solicitar relatório |
| `GET` | `/companies/{companyId}/reports/{reportId}` | Status do relatório |

---

## 7. Modelos de Dados

### Padrão de Paginação

Todas as listagens seguem o mesmo padrão de resposta paginada:

```json
{
    "content": [],
    "page": 0,
    "size": 20,
    "totalElements": 100,
    "totalPages": 5
}
```

| Campo | Descrição |
|-------|-----------|
| `content` | Array com os registros da página |
| `page` | Página atual (inicia em 0) |
| `size` | Registros por página |
| `totalElements` | Total de registros |
| `totalPages` | Total de páginas |

**Parâmetros de paginação:**

| Parâmetro | Padrão | Mínimo | Máximo | Descrição |
|-----------|--------|--------|--------|-----------|
| `page` | 0 | 0 | - | Número da página |
| `size` | 20 | 1 | 100 | Registros por página |

---

## 8. Códigos de Erro

### Erros HTTP Padrão

| Código | Descrição | Ação Recomendada |
|--------|-----------|------------------|
| 400 | Requisição inválida | Verifique o formato do body/parâmetros |
| 404 | Recurso não encontrado | Confirme os IDs utilizados |
| 409 | Conflito (processamento ativo) | Aguarde o processamento anterior finalizar |
| 422 | Erro de validação | Corrija os dados conforme a resposta |
| 500 | Erro interno | Tente novamente; se persistir, entre em contato |
| 502 | Dependência externa indisponível | Tente novamente após alguns minutos |
| 503 | Serviço indisponível | Tente novamente após alguns minutos |

### Formato de Erro Padrão

```json
{
    "timestamp": "2026-08-26T10:00:00Z",
    "status": 400,
    "error": "Bad Request",
    "message": "Descricao do problema",
    "path": "/vt-orders-management/v1/companies/12345/orders"
}
```

### Códigos de Erro de Validação de Colaborador

| Código | Descrição | HTTP |
|--------|-----------|------|
| `USU-001` | CPF inválido (dígito verificador incorreto) | 422 |
| `USU-002` | CPF já cadastrado com matrícula diferente na empresa | 409 |
| `USU-003` | Endereço de trabalho (id) não existe na empresa | 422 |
| `USU-004` | CNPJ informado não cadastrado no sistema | 422 |
| `USU-005` | CNPJ informado não pertence ao grupo econômico da empresa | 422 |
| `USU-006` | Data de nascimento inválida (data futura) | 422 |
| `USU-007` | CEP do endereço residencial inexistente na base de CEPs | 422 |
| `USU-008` | E-mail com formato inválido | 422 |
| `USU-009` | Campo RG não preenchido (quando produto exige) | 422 |
| `USU-010` | Campo Data de Nascimento não preenchido (quando produto exige) | 422 |
| `USU-011` | Colaborador sem nenhum benefício informado | 422 |

### Códigos de Erro de Validação de Benefícios

| Código | Descrição | HTTP |
|--------|-----------|------|
| `BEN-001` | Produto (id) não existe ou não está ativo para comercialização | 422 |
| `BEN-002` | Produto não é entregue na região do endereço de trabalho | 422 |
| `BEN-003` | Produto facial sem valor unitário informado | 422 |
| `BEN-004` | Produto não disponível para a empresa (sem contrato) | 422 |
| `BEN-005` | Quantidade de dias trabalhados não informada | 422 |
| `BEN-006` | Quantidade de uso por dia não informada | 422 |

### Códigos de Erro de Validação de Valores

| Código | Descrição | HTTP |
|--------|-----------|------|
| `VAL-001` | Valor total acima do máximo do produto | 422 |
| `VAL-002` | Valor total abaixo do mínimo do produto | 422 |

### Códigos de Pré-requisito

| Código | Descrição | HTTP |
|--------|-----------|------|
| `IMP-002` | Já existe um processamento ativo para esta empresa | 409 |

---

## 9. Cenários de Teste (Mock)

Os cenários abaixo estão disponíveis no ambiente de virtualização (mock) para testes de integração. Cada endpoint possui valores específicos que controlam qual resposta HTTP será retornada.

> **Como usar:**
> - **PATH (URL):** O valor informado na coluna "Como simular" deve ser colocado no segmento variável da URL.  
>   Ex: para simular 404 em `/companies/{companyId}/orders/{orderId}`, use `orderId = 999999` na URL.
> - **Body:** Inclua o campo indicado com o valor exato no JSON do request body.
> - **Query:** Adicione o query parameter indicado na URL da chamada.
> - Em endpoints com dois segmentos variáveis (ex: `/companies/{companyId}/orders/{orderId}`), o cenário é controlado pelo **último segmento** — o primeiro pode ser qualquer valor válido (ex: `12345`).

### POST /companies (Consultar Empresa)

| HTTP | Como simular |
|------|-------------|
| 200 | Body: `companyDocumentNumber: "61190096000192"` |
| 200 | Body: `companyDocumentNumber: "05800256000954"` |
| 404 | Body: `companyDocumentNumber: "00000000000000"` |
| 500 | Body: `companyDocumentNumber: "61190096000500"` |
| 502 | Body: `companyDocumentNumber: "61190096000502"` |
| 503 | Body: `companyDocumentNumber: "61190096000503"` |

### GET /companies/{companyId}/places

| HTTP | Como simular |
|------|-------------|
| 200 | PATH: `12345` |
| 200 | PATH: `12345` + query: `type=BILLING` |
| 500 | PATH: `12345` + query: `page=0` |
| 502 | PATH: `502` |
| 503 | PATH: `503` |

### POST /companies/{companyId}/places

| HTTP | Como simular |
|------|-------------|
| 201 | PATH: `12345` + Body: `type: "DELIVERY"` |
| 201 | PATH: `12345` + Body: `type: "BILLING"` |
| 400 | PATH: `400` |
| 422 | Body: `zipCode: "123"` |
| 500 | PATH: `500` |
| 502 | PATH: `502` |
| 503 | PATH: `503` |

### GET /companies/{companyId}/responsibles

| HTTP | Como simular |
|------|-------------|
| 200 | PATH: `12345` |
| 200 | PATH: `12345` + query: `department=FINANCEIRO` |
| 404 | PATH: `99999` |
| 500 | PATH: `500` |
| 502 | PATH: `502` |
| 503 | PATH: `503` |

### GET /products

| HTTP | Como simular |
|------|-------------|
| 200 | Query: `type=CARD` |
| 200 | Query: `type=PAPER&deliveryAddressId=1` |
| 500 | Query: `deliveryAddressId=500` |
| 502 | Query: `deliveryAddressId=502` |
| 503 | Query: `type=PAPER` |

### GET /companies/{companyId}/beneficiaries

| HTTP | Como simular |
|------|-------------|
| 200 | PATH: `12345` + query: `status=ACTIVE` |
| 200 | PATH: `12345` + query: `beneficiaryDocumentNumber=12345678901` |
| 500 | PATH: `12345` |
| 502 | PATH: `502` |
| 503 | PATH: `503` |

### POST /companies/{companyId}/beneficiaries

| HTTP | Como simular |
|------|-------------|
| 200 | PATH: `12345` + Body: `street: "Rua das Palmeiras"` |
| 201 | PATH: `12345` + Body: `registrationNumber: "000012345"` |
| 400 | PATH: `400` |
| 409 | PATH: `12345` + Body: `beneficiaryDocumentNumber: "12345678901"` |
| 422 | PATH: `422` |
| 500 | PATH: `500` |
| 502 | PATH: `502` |
| 503 | PATH: `503` |

### PATCH /companies/{companyId}/beneficiaries/{beneficiaryId}

| HTTP | Como simular |
|------|-------------|
| 200 | `beneficiaryId: 98765` |
| 200 | `beneficiaryId: 98766` |
| 404 | `beneficiaryId: 11111` |
| 409 | `beneficiaryId: 409` |
| 422 | `beneficiaryId: 422` |
| 500 | `beneficiaryId: 500` |
| 502 | `beneficiaryId: 502` |
| 503 | `beneficiaryId: 503` |

### GET /companies/{companyId}/rules-validate

| HTTP | Como simular |
|------|-------------|
| 200 | PATH: `12345` |
| 200 | PATH: `67890` |
| 404 | PATH: `99999` |
| 500 | PATH: `500` |
| 502 | PATH: `502` |
| 503 | PATH: `503` |

### POST /companies/{companyId}/orders-validate

| HTTP | Como simular |
|------|-------------|
| 200 | PATH: `12345` |
| 200 | Body: `scope: "PARTIAL"` |
| 404 | PATH: `404` |
| 409 | Body: `scope: "COMPLETE"` |
| 500 | PATH: `500` |
| 502 | PATH: `502` |
| 503 | PATH: `503` |

### POST /companies/{companyId}/orders

| HTTP | Como simular |
|------|-------------|
| 202 | PATH: `12345` |
| 202 | Body: `modality: "PAPER"` |
| 404 | PATH: `404` |
| 422 | Body: `scope: "PARTIAL"` |
| 500 | PATH: `500` |
| 502 | PATH: `502` |
| 503 | PATH: `503` |

### GET /companies/{companyId}/processing/{processingId}

| HTTP | Como simular |
|------|-------------|
| 200 | `processingId: f47ac10b-58cc-4372-a567-0e02b2c3d478` |
| 200 | `processingId: f47ac10b-58cc-4372-a567-0e02b2c3d479` |
| 404 | `processingId: 00000000-0000-0000-0000-000000000000` |
| 500 | `processingId: 500` |
| 502 | `processingId: 502` |
| 503 | `processingId: 503` |

### GET /companies/{companyId}/orders

| HTTP | Como simular |
|------|-------------|
| 200 | PATH: `12345` + query: `status=CONFIRMED` |
| 200 | PATH: `12345` |
| 404 | PATH: `99999` |
| 500 | PATH: `500` |
| 502 | PATH: `502` |
| 503 | PATH: `503` |

### GET /companies/{companyId}/orders/{orderId}

| HTTP | Como simular |
|------|-------------|
| 200 | `orderId: 456789` |
| 200 | `orderId: 456800` |
| 404 | `orderId: 999999` |
| 500 | `orderId: 500` |
| 502 | `orderId: 502` |
| 503 | `orderId: 503` |

### POST /companies/{companyId}/orders/{orderId}/approve

| HTTP | Como simular |
|------|-------------|
| 204 | `orderId: 500001` |
| 204 | `orderId: 500002` |
| 404 | `orderId: 404` |
| 409 | `orderId: 409` |
| 422 | `orderId: 500003` |
| 500 | `orderId: 500` |
| 502 | `orderId: 502` |
| 503 | `orderId: 503` |

### POST /companies/{companyId}/orders/{orderId}/cancel

| HTTP | Como simular |
|------|-------------|
| 204 | `orderId: 456791` |
| 204 | `orderId: 456792` |
| 404 | `orderId: 404` |
| 409 | `orderId: 456789` |
| 500 | `orderId: 500` |
| 502 | `orderId: 502` |
| 503 | `orderId: 503` |

### GET /companies/{companyId}/orders/{orderId}/payment-slip

| HTTP | Como simular |
|------|-------------|
| 200 | `orderId: 456789` |
| 200 | `orderId: 4567890` |
| 404 | `orderId: 456791` |
| 500 | `orderId: 500` |
| 502 | `orderId: 502` |
| 503 | `orderId: 503` |

### GET /companies/{companyId}/orders/{orderId}/invoice

| HTTP | Como simular |
|------|-------------|
| 200 | `orderId: 456789` |
| 200 | `orderId: 456800` |
| 404 | `orderId: 456792` |
| 500 | `orderId: 500` |
| 502 | `orderId: 502` |
| 503 | `orderId: 503` |

### POST /companies/{companyId}/reports

| HTTP | Como simular |
|------|-------------|
| 202 | PATH: `12345` + query: `orderId=456789` |
| 202 | PATH: `12345` + query: `orderId=456800` |
| 404 | PATH: `404` |
| 409 | PATH: `12345` + query: `type=consolidated` |
| 500 | PATH: `500` |
| 502 | PATH: `502` |
| 503 | PATH: `503` |

### GET /companies/{companyId}/reports/{reportId}

| HTTP | Como simular |
|------|-------------|
| 200 | `reportId: b7e8f9a0-1234-5678-abcd-ef0123456788` |
| 200 | `reportId: b7e8f9a0-1234-5678-abcd-ef0123456789` |
| 404 | `reportId: 00000000-0000-0000-0000-000000000000` |
| 500 | `reportId: 500` |
| 502 | `reportId: 502` |
| 503 | `reportId: 503` |

---

## 10. Boas Práticas

### Tratamento de Erros

1. Sempre implemente retry com backoff exponencial para erros `502` e `503`
2. Quando receber `409` (processamento ativo), aguarde e tente novamente
3. Utilize a pré-validação (`orders-validate`) antes de submeter pedidos grandes

### Polling de Processamento

1. Após submeter o pedido (`POST /orders`), aguarde 5 segundos antes do primeiro polling
2. Utilize intervalo de 5 a 10 segundos entre cada chamada
3. O campo `completionPercentage` permite exibir progresso ao usuário
4. Quando `status = COMPLETED`, utilize o `orderId` retornado para consultas subsequentes

### Performance

1. Utilize filtros nos endpoints de listagem para reduzir o volume de dados
2. Ajuste o `size` da paginação conforme sua necessidade (máximo 100)
3. Não ultrapasse 10 requisições por segundo ao API Gateway

### Segurança

1. Nunca exponha `client_id` / `client_secret` em código frontend
2. Utilize HTTPS em todas as comunicações

---

## 11. FAQ

**Posso submeter mais de um pedido por vez para a mesma empresa?**

Não. Apenas um processamento pode estar ativo por empresa. Aguarde a finalização antes de submeter outro.

---

**O que acontece se o polling retornar status ERROR?**

Consulte o campo `validationErrors` na resposta. Ele detalha os erros por colaborador/produto. Corrija os dados e submeta novamente.

---

**Preciso aprovar todo pedido?**

Apenas empresas com método de pagamento pós-pago (`POST_PAID`) geram pedidos com status `PENDING` que requerem aprovação.

---

**As URLs de download do boleto expiram?**

Sim, após 3 minutos. Basta chamar o endpoint novamente para obter uma nova URL válida.

---

**Posso atualizar apenas um campo do colaborador?**

Sim, utilize o método `PATCH /companies/{id}/beneficiaries/{id}` enviando apenas os campos que deseja alterar.

---

**Como sei quais campos são obrigatórios para cada produto?**

Consulte o endpoint `GET /companies/{id}/rules-validate`. As flags `requires*` indicam a obrigatoriedade por produto.

---

**O que é o campo `requestRouteSuggestion` no endereço residencial?**

Quando `true`, indica que o sistema deve sugerir uma rota de transporte público do endereço residencial até o local de trabalho.

---

*Alelo — Gestão de Vale Transporte*  
*Documento gerado em Agosto/2026 — Atualizado em Setembro/2026 — Versão 1.1*  
*v1.1: seção 4.1 (Empresas) alinhada ao contrato vt-empresas — responsáveis pelo recebimento agora em array*
