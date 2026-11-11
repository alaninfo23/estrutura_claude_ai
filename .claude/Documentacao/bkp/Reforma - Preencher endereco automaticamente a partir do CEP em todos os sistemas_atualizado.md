# [RT-Cadastro][Cluster 1] Preencher endereço automaticamente a partir do CEP em todos os sistemas [Apontamento V3]

## Descrição

### Descrição geral
Esta história faz parte da Reforma Tributária, no domínio Cadastro e no Cluster 1 – Cadastro do Participante. As Diretrizes de Cadastro definem que o CEP é obrigatório, é a origem da localização e deve seguir esta ordem: CEP → Endereço → Município → UF → Código IBGE.

Como usuário que cadastra ou edita uma empresa ou filial,  
quero informar um CEP válido e receber automaticamente logradouro, bairro, município e UF,  
para manter o endereço completo, correto e padronizado no cadastro.

Ao informar o CEP, o sistema deve consultar exclusivamente a API Places V3 pelo fluxo da Sensedia. O endereço retornado deve aparecer na tela e ser salvo no banco de dados junto ao endereço da empresa ou da filial.

Na regra normal, somente número e complemento são preenchidos manualmente. Como exceção, quando a API reconhecer o CEP como válido, mas não devolver logradouro e/ou bairro, somente o campo que não veio na resposta será liberado para preenchimento manual e continuará obrigatório. Os dados que vierem da API permanecem preenchidos automaticamente e sem edição manual.

O comportamento deve existir no cadastro e na edição:
* **CRM:** Comercial > Clientes, para empresa/contrato e filial.
* **Portal 1.0:** Dados da Empresa e CNPJs Filiais da Empresa > Adicionar filial, incluindo as edições.
* **Portal 2.0:** aba Empresas e Adicionar nova filial, incluindo as edições.

Nesta história, a expressão “todos os sistemas” corresponde aos fluxos do CRM, Portal 1.0 e Portal 2.0 descritos acima.

Esta definição foi feita pelas equipes REGTECH, GBA e ACELERA.

---

## BDD

### Cenário 1 – Preencher o endereço de uma empresa por um CEP válido
* **Dado** que estou cadastrando ou editando uma empresa no CRM, Portal 1.0 ou Portal 2.0
* **Quando** informo um CEP válido e a API devolve o endereço completo
* **Então** o sistema consulta a API Places V3 pela Sensedia
* **E** preenche automaticamente logradouro, bairro, município e UF
* **E** deixa somente número e complemento para preenchimento manual.

### Cenário 2 – Preencher o endereço de uma filial por um CEP válido
* **Dado** que estou cadastrando ou editando uma filial no CRM, Portal 1.0 ou Portal 2.0
* **Quando** informo um CEP válido e a API devolve o endereço completo
* **Então** o sistema preenche automaticamente logradouro, bairro, município e UF
* **E** associa o endereço à filial correta.

### Cenário 3 – Salvar e reapresentar o endereço
* **Dado** que o endereço foi preenchido a partir do CEP e está completo
* **Quando** concluo o cadastro ou a edição
* **Então** CEP, logradouro, bairro, município e UF são salvos no endereço da empresa ou filial
* **E** aparecem corretamente quando o cadastro é aberto novamente.

### Cenário 4 – Alterar o CEP durante uma edição
* **Dado** que estou editando uma empresa ou filial já cadastrada
* **Quando** troco o CEP por outro CEP válido
* **Então** o sistema consulta novamente a API Places V3
* **E** substitui os dados derivados pelos dados do novo CEP
* **E** salva o novo endereço após a confirmação da edição.

### Cenário 5 – CEP válido sem logradouro e/ou bairro
* **Dado** que informei um CEP reconhecido como válido pela API
* **E** a resposta não trouxe logradouro e/ou bairro
* **Quando** o sistema apresenta o endereço
* **Então** libera para preenchimento manual somente o campo que não veio na resposta
* **E** mantém bloqueados os dados que vieram da API
* **E** exige o preenchimento do campo liberado antes de salvar
* **E** salva o endereço completo na empresa ou filial.

### Cenário 6 – CEP inválido, inexistente ou sem retorno
* **Dado** que estou cadastrando ou editando uma empresa ou filial
* **Quando** informo um CEP inválido, inexistente ou sem retorno da API
* **Então** o sistema mostra uma mensagem clara
* **E** não libera o preenchimento manual como exceção
* **E** não permite concluir o cadastro com o endereço obrigatório incompleto.

### Cenário 7 – Interpretar o contrato da API Places V3
* **Dado** que a Sensedia responde com `zipCode`, `uf`, `cityName`, `ibgeCode` e, quando disponíveis, `streetName` e `district`
* **Quando** o sistema recebe a resposta
* **Então** usa `uf` como UF, `cityName` como município, `streetName` como logradouro e `district` como bairro
* **E** não espera os nomes antigos `state` e `city`.

### Cenário 8 – Usar somente o fluxo oficial
* **Dado** que o sistema precisa consultar um CEP
* **Quando** realiza a chamada
* **Então** utiliza o endpoint V3 exposto pela Sensedia
* **E** não chama diretamente o endpoint interno `ibge-codes`.

---

## Critérios de aceite

1. Ao informar um CEP válido no cadastro ou na edição, o sistema preenche automaticamente logradouro, bairro, município e UF, conforme os dados disponíveis na resposta da API.
2. O preenchimento automático funciona para empresa/contrato e filial no CRM, Portal 1.0 e Portal 2.0. Nesta história, “todos os sistemas” corresponde somente a esses três canais e aos fluxos indicados.
3. O CRM contempla Comercial > Clientes, no cadastro e na edição de empresa/contrato e filial.
4. O Portal 1.0 contempla Dados da Empresa e CNPJs Filiais da Empresa > Adicionar filial, incluindo as edições.
5. O Portal 2.0 contempla a aba Empresas e Adicionar nova filial, incluindo as edições.
6. A consulta utiliza exclusivamente a API Places V3 via Sensedia. Não é permitida chamada direta ao endpoint interno `ibge-codes`.
7. O contrato esperado da V3 contém `zipCode`, `uf`, `cityName`, `ibgeCode` e, quando disponíveis, `streetName` e `district`.
8. O mapeamento usa `uf` para UF, `cityName` para município, `streetName` para logradouro e `district` para bairro.
9. Os consumidores não dependem mais dos nomes antigos `state` e `city`. A adequação do `vt-empresas`, registrada no Débito Técnico 1738746, deve estar disponível para a validação integrada desse consumidor.
10. CEP, logradouro, bairro, município e UF são obrigatórios e ficam salvos no endereço da empresa ou filial correta.
11. Ao reabrir o cadastro, o endereço salvo é exibido corretamente.
12. Ao trocar o CEP por outro válido, os campos derivados são atualizados com o novo retorno da API e persistidos após salvar.
13. Na regra normal, somente número e complemento ficam disponíveis para digitação manual. Os dados retornados pela API não aceitam alteração manual.
14. Quando um CEP válido não retornar `streetName` e/ou `district`, somente o campo correspondente ausente (logradouro e/ou bairro) é liberado para preenchimento manual.
15. O logradouro e/ou bairro liberado pela exceção continua obrigatório. O endereço só pode ser salvo depois de ficar completo.
16. Município e UF continuam derivados da API e não são liberados para digitação manual por essa exceção.
17. CEP inválido, inexistente ou sem retorno da API mostra uma mensagem clara, não ativa a exceção de preenchimento manual e não permite concluir o cadastro.
18. O QA valida a regra normal e a exceção em cadastro e edição, para empresa e filial, nos três canais, sem alterar o comportamento já homologado além do apontamento e do novo contrato da V3.

---

## Descrição de negócio

A Reforma Tributária aumenta a importância dos dados cadastrais usados nas operações sujeitas ao IBS e à CBS. Conforme o documento "01_Domínio Cadastro_Reforma Tributária", a localização ajuda a determinar onde ocorre a incidência tributária e a repartição do IBS. Endereços incorretos podem causar inconsistências territoriais e problemas nas obrigações.

As "Diretrizes_Cadastro_Reforma_Tributaria_Alelo" classificam esta necessidade no Cluster 1 – Cadastro do Participante e estabelecem a hierarquia CEP → Endereço → Município → UF → Código IBGE. O CEP é a informação inicial. A partir dele, o sistema recebe e preenche os demais dados de localização, evitando digitação livre.

### Definição desta história

| Item | Regra |
| :--- | :--- |
| **Aplicável a** | Cadastro e edição de empresa/contrato e filial no CRM, Portal 1.0 e Portal 2.0 |
| **Origem** | CEP informado no cadastro |
| **Integração** | API Places V3, exclusivamente via Sensedia |
| **Preenchimento automático** | Logradouro e bairro, quando disponíveis; município e UF derivados da API |
| **Preenchimento manual normal** | Número e complemento |
| **Exceção manual** | Para CEP válido sem logradouro e/ou bairro, liberar somente o campo ausente e exigir seu preenchimento |
| **Persistência** | No registro de endereço da empresa ou da filial |
| **Obrigatoriedade** | CEP e endereço completo são obrigatórios para concluir o cadastro ou a edição |

### Regra da exceção
* A exceção só se aplica quando a API reconhece o CEP como válido, mas não devolve `streetName` e/ou `district`.
* Se faltar apenas `streetName`, somente logradouro é liberado.
* Se faltar apenas `district`, somente bairro é liberado.
* Se faltarem os dois, logradouro e bairro são liberados.
* Os campos recebidos da API permanecem bloqueados para alteração manual.
* Município e UF não fazem parte dessa exceção e permanecem derivados da API.
* CEP inválido, inexistente ou sem retorno não ativa a exceção.

### Contrato esperado da API Places V3

```json
{
  "zipCode": "01311941",
  "uf": "SP",
  "cityName": "São Paulo",
  "ibgeCode": "3550308",
  "streetName": "Avenida Paulista",
  "district": "Bela Vista"
}
```

Para manter a padronização entre as versões da API, a resposta usa `uf` e `cityName`. A V3 retorna `streetName` e `district` quando essas informações estiverem disponíveis na fonte. O exemplo de endpoint no ambiente DEV é:

`https://apidev.homologacaoalelo.com.br/alelo/sb/places/v3/cities?zipCode=01311941`

### Onde ocorre

| Canal | Empresa/contrato | Filial |
| :--- | :--- | :--- |
| **CRM** | Comercial > Clientes (cadastro e edição) | Cadastro e edição de filial |
| **Portal 1.0** | Dados da Empresa | CNPJs Filiais da Empresa > Adicionar filial, incluindo edição |
| **Portal 2.0** | Aba Empresas | Adicionar nova filial, incluindo edição |

### Separação de responsabilidades entre as histórias
* **Enabler 1717392:** faz a API Places retornar logradouro (`streetName`) e bairro (`district`) por CEP quando a fonte tiver os dados, usando Correios ou base conforme a lógica interna do serviço.
* **US 1725106 (esta história):** faz CRM e Portais apontarem para a API Places V3 via Sensedia, preencherem o endereço na tela, aplicarem a exceção de preenchimento manual e salvarem o endereço da empresa/filial.
* **US 1725112:** persiste o `ibgeCode` no banco sem mostrar ou permitir edição em tela. Embora a V3 devolva esse dado, sua persistência invisível não deve ser testada como entrega desta US.
* **Débito Técnico 1738746:** ajusta o DTO/mapeamento do `vt-empresas` para os nomes `uf` e `cityName` e é necessário para a validação integrada desse consumidor.

### Observações
* As aplicações devem utilizar somente o fluxo da Sensedia, garantindo rastreabilidade e suporte.
* Não realizar chamada direta ao endpoint interno `ibge-codes`.
* Nesta história, “todos os sistemas” significa os fluxos descritos do CRM, Portal 1.0 e Portal 2.0.
* A data de release da V3 deve seguir o alinhamento técnico próprio; uma data planejada não altera os critérios funcionais desta história.
* Esta atualização complementa o conteúdo já existente e não muda o restante do escopo que está em desenvolvimento/teste.

---

## DoR – Definition of Ready

* Contrato de resposta da API Places V3 alinhado: `zipCode`, `uf`, `cityName`, `ibgeCode` e, quando disponíveis, `streetName` e `district`.
* Endpoint V3 via Sensedia identificado para o ambiente de testes.
* Fluxos e telas de empresa/contrato e filial identificados no CRM, Portal 1.0 e Portal 2.0.
* Exceção para CEP válido sem logradouro e/ou bairro definida pelo negócio.
* Responsabilidades do Enabler 1717392, desta US, da US 1725112 e do Débito Técnico 1738746 separadas.

---

## DoD – Definition of Done

* CRM, Portal 1.0 e Portal 2.0 consultam a API Places V3 exclusivamente via Sensedia nos fluxos desta história.
* Logradouro, bairro, município e UF são preenchidos automaticamente a partir de CEP válido, conforme os dados disponíveis.
* Para CEP válido sem logradouro e/ou bairro, somente o campo ausente é liberado, preenchido obrigatoriamente e salvo.
* O novo contrato com `uf` e `cityName` é interpretado corretamente pelos consumidores, com a adequação do Débito Técnico 1738746 disponível na validação integrada do `vt-empresas`.
* O endereço é salvo e reapresentado corretamente para empresa e filial.
* CEP inválido, inexistente ou sem retorno não ativa a exceção e não permite salvar endereço obrigatório incompleto.
* Regra normal e exceção validadas pelo QA nos três canais, em cadastro e edição de empresa e filial.
* O funcionamento já homologado fora deste complemento permanece sem regressão.