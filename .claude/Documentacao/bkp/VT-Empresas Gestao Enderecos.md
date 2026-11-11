# [API Bradesco] 1 - VT-Empresas | Gestão de Endereços

## Objetivo

**Como** sistema integrador do Bradesco,
**Quero** consultar e cadastrar endereços corporativos através do microsserviço VT-Empresas,
**Para que** os processos de entrega, faturamento e operação dos benefícios possam ser executados corretamente.

## Contexto

Os endereços corporativos são utilizados em diversos fluxos do Vale-Transporte, incluindo:

- Entrega de cartões
- Entrega de vales
- Faturamento
- Definição de endereço de trabalho para colaboradores

O microsserviço VT-Empresas deverá disponibilizar APIs para consulta e inclusão desses endereços.

---

## Escopo

### Consulta de Endereços

Endpoint: `GET /companies/{companyId}/places`

**Tipos de endereço suportados:**

- `BILLING`
- `DELIVERY`
- `BILLING_AND_DELIVERY`

**Filtros disponíveis:**

- Tipo
- Bairro
- Cidade
- Estado
- CEP
- Busca textual _(case-insensitive e busca parcial nos campos: logradouro, bairro, cidade, local de referência, nome dos responsáveis pelo recebimento)_

**Funcionalidades adicionais:**

- Paginação
- Ordenação

### Cadastro de Endereços

Endpoint: `POST /companies/{companyId}/places`

Permite a inclusão de novos endereços corporativos.

**Dados Retornados:**

- Logradouro
- Número
- Complemento
- Bairro
- Cidade
- Estado
- CEP
- Tipo do Endereço
- Local de Referência

### Endereços de Entrega

Para endereços do tipo `BILLING`, `DELIVERY` ou `BILLING_AND_DELIVERY` deverão ser suportados:

- Local de Referência
- Até 3 responsáveis pelo recebimento

**Dados dos responsáveis:**

- Nome
- Telefone
- Ramal

---

## Cenários de Teste

**Cenário 1 - Consultar Endereços**

- **Dado** uma empresa com endereços cadastrados
- **Quando** a consulta for realizada
- **Então** os endereços deverão ser retornados

**Cenário 2 - Consultar Endereços com Filtros**

- **Dado** uma empresa com múltiplos endereços
- **Quando** filtros forem aplicados
- **Então** apenas os registros compatíveis deverão ser retornados

**Cenário 3 - Cadastrar Endereço**

- **Dado** uma empresa válida
- **Quando** um endereço for cadastrado
- **Então** o endereço deverá ser persistido com sucesso

**Cenário 4 - Endereço de Entrega**

- **Dado** um endereço do tipo DELIVERY
- **Quando** ele for cadastrado
- **Então** deverá permitir a inclusão de até 3 responsáveis para recebimento

---

## Regras de Negócio (RN)

- **RN01:** Permitir consulta dos endereços corporativos da empresa.
- **RN02:** Permitir filtros por tipo, cidade, estado, CEP e texto livre.
- **RN03:** Permitir paginação e ordenação.
- **RN04:** Permitir inclusão de novos endereços corporativos.
- **RN05:** Endereços de entrega deverão suportar até 3 responsáveis para recebimento.
- **RN06:** Todos os endereços deverão possuir uma classificação de utilização.

---

## Validação da API de Endereços

Deverá ser implementada validação exclusivamente na API de Endereços para garantir a consistência dos dados do responsável.

1. Caso seja informado **telefone**, o preenchimento do **DDD** e do **nome do responsável** deverá ser **obrigatório**.
2. A API não deverá permitir o envio do telefone sem DDD.
3. A API não deverá permitir o envio do telefone sem o nome do responsável.
4. O nome do responsável poderá ser informado isoladamente, sem a obrigatoriedade de preenchimento dos demais campos.
5. **Atenção:** Não haverá alteração ou inclusão dessa validação nas telas do sistema, sendo aplicada somente na API.

> **Observação:** os cenários de teste deverão contemplar essas novas validações da API.

```eof

Aqui está o arquivo Markdown pronto! Estruturei com as formatações corretas para negritos, listas e blocos de código (para as rotas da API). Você pode copiá-lo facilmente usando o botão de "copiar" no canto superior direito do bloco, ou salvar com a extensão `.md` em seu computador. Precisa de mais alguma edição no texto?
```
