# [Reforma Tributária] Retornar Endereço (Logradouro/Bairro) por CEP via Correios e Base (`api-places`)

## Contexto

Atualmente, o endpoint `GET /ibge-codes` retorna apenas `zipCode`, `state`, `city` e `ibgeCode` (`IbgeCodeResponse`).

Quando a busca é realizada por **CEP**, tanto a API dos Correios quanto a base local de dados possuem informações de **logradouro** e **bairro**, porém esses dados são ignorados no fluxo atual:

- **`CorreiosResponseParser`**: Lê apenas `cep`, `uf`, `localidade` e `codigoIbge` (descarta `logradouro` e `bairro`).
- **`DatabasePlaceResolver`**: Utiliza `LocalityRepository.findLocalityByZipRange` (que consulta apenas o município) e não realiza a consulta em `PlaceRepository.findByZipCode` (tabela `TCEP_LOGRADOURO`, que possui os campos de logradouro `NM_LOGDR` / `DS_TPO_LOGDR` e bairro `District` / `TCEP_BAIRRO`).

---

## Objetivo

Ao consultar o endpoint `GET /ibge-codes` enviando um CEP, retornar também o logradouro (`streetName`) e o bairro (`district`), desde que a fonte resolvedora (Correios **OU** Base Local) contenha essas informações.

---

## Escopo e Regras de Negócio

- **Manutenção da API:** Manter o mesmo endpoint (`/ibge-codes`) e a mesma cadeia de _resolvers_.
- **Opcionalidade de Campos:** Os campos adicionais (`streetName` e `district`) devem ser preenchidos **apenas** quando houver `zipCode` na requisição.
- **Provedor Correios:** Expandir o _parser_ e o _resolver_ para extrair e propagar os dados de logradouro e bairro contidos no _payload_.
- **Provedor Base Local (`DatabasePlaceResolver`):**
  1. Antes de executar o _fallback_ via `findLocalityByZipRange` (busca por faixa/apenas IBGE), tentar realizar o match exato via `PlaceRepository.findByZipCode`.
  2. Se houver o match exato, obter `streetName`, `district` e `ibgeCode` (via localidade/cidade).
  3. Caso não exista logradouro exato associado ao CEP, manter o _fallback_ por faixa de CEP atual (retornando a resposta sem os campos extras).
- **Provedor IBGE (`IbgePlaceResolver`):** Permanece inalterado (atende apenas consultas por `city` + `state`, sem CEP e sem campos de endereço).
- **Serialização de JSON:** Campos ausentes/nulos **não devem** ser exibidos no JSON da resposta (garantir anotação `@JsonInclude(NON_NULL)`).
- **Gerenciamento de Cache:** A camada de _cache_ deve preservar e retornar os novos campos extras.

> **Observação:** Confirmado que atualmente apenas o serviço **VT** está utilizando a versão `v3` do `places`.

---

## Alterações Previstas (Referência de Código)

1. **`IbgeCodeResponse`**
   - Adicionar os campos `streetName` e `district` (opcionais, anotados com `@JsonInclude(NON_NULL)`).
   - Atualizar documentação via anotações do OpenAPI/Swagger (`@Schema`).
2. **`CorreiosResponseParser.CorreiosResult`**
   - Mapear os campos `streetName` e `district` a partir do _payload_ de resposta dos Correios.
3. **`CorreiosPlaceResolver`**
   - Propagar os novos campos para o objeto de resposta `IbgeCodeResponse`.
4. **`DatabasePlaceResolver`**
   - Injetar a dependência do `PlaceRepository`.
   - Tentar o _match_ exato por CEP na entidade `Place`.
   - Em caso de falha, manter o comportamento de _fallback_ com `findLocalityByZipRange`.
   - Montar o campo `streetName` concatenando `streetType` + `streetName` (seguindo a mesma regra de negócio já aplicada no `PlaceVO`).
   - Extrair o campo `district` através de `Place.getDistrict()`.
5. **Documentação (Swagger / OpenAPI)**
   - Atualizar as anotações `IbgeCodeSearchApiResponse` e `@Operation` adicionando exemplos do _payload_ com o endereço completo.

---

## Fora de Escopo

- Buscar informações em outras fontes de dados como `GrandPlace`, `OperationalUnit` ou `PostalAddress` dentro do endpoint `/ibge-codes`.

---
