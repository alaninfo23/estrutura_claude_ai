# Refinamento — Aprovar

Voltar: [visão geral](https://dev.azure.com/alelo/ValeTransporte-2.0/_wiki/wikis/ValeTransporte-2.0.wiki/69932/Contexto-Sobre-Aprovar-e-Contestar)

## Em uma frase

O RH clica em Aprovar na Rota Inteligente. O microsserviço vt-roteirizacao valida a rota, substitui o Vale-Transporte do colaborador, marca a roteirização como _Finalizado (F)_ e regenera o PDF.  
_Não chama a SAT._ O serviço de calculation só grava um snapshot de economia; se ele falhar, a aprovação _segue mesmo assim_.

Não confundir com:

- Aceite do colaborador no PortalVT antigo (só cria solicitação para o RH analisar)
- Alterar status genérico (PUT .../status/{status-id}) — só muda a flag, _não_ troca o benefício
- Contestar — pede reanálise à SAT; não efetiva Vale-Transporte

_Escopo da decisão:_ o Portal novo usa só _Aprovar_ e _Contestar_. Rejeitar está fora.

---

## Glossário rápido

| Termo           | Significado                                                                                 |
| --------------- | ------------------------------------------------------------------------------------------- |
| _VT_            | Vale-Transporte do colaborador                                                              |
| _EUP_           | Tabela EMPRESA_USUARIO_PRODUTO — cadastro real do benefício                                 |
| _SAT_           | Fornecedor que sugeriu trajeto e produtos (já consultada antes; não é chamada de novo aqui) |
| _FL_STACOM_RTR_ | Status da codificação de compra na rota (F = compra finalizada)                             |
| _FL_STATUS_RTR_ | Status Alelo da roteirização (F = Finalizado após aprovar)                                  |

---

## Pré-condição

| Stack           | Pronto para aprovar                                                                                            |
| --------------- | -------------------------------------------------------------------------------------------------------------- |
| Microsserviço   | Rota existe para a empresa + status da compra FL_STACOM_RTR = F (Finalizado) + há linhas em ROTA_TRANSP_COMPRA |
| Legado smartweb | Status Alelo CD_STSALE_RTZ = FINALIZADO                                                                        |

Se a compra não estiver finalizada, o microsserviço _não efetiva_ aquele item (só registra log e pula).

---

## Endpoints (completos)

### Aprovar — o que a tela chama (API Connect)

http
POST /apiconnect/routes-services/v1/companies/routes-approval
Content-Type: application/json

[
{
"idEmpresa": 12345,
"idRoteirizacao": 98765
}
]

### Aprovar — no microsserviço vt-roteirizacao

http
POST /routes-approve-services/routes-approval
Content-Type: application/json

Mesmo body. Pode ir mais de um item (aprovação em lote).  
VO Java: RotaAprovacaoVO (idEmpresa, idRoteirizacao).

### Detalhe da rota — antes de aprovar (leitura)

Via API Connect:

http
GET /apiconnect/routes-services/v1/companies/{companyId}/routes/{routeId}/data

No microsserviço:

http
GET /routes-approve-services/companies/{company-id}/routes/{route-id}/data

Devolve RoteirizacaoDetalheVO (nome, endereços, percursos, lista de VT, status) para o modal da tela.

### Endpoint que NÃO é aprovar

Via API Connect (se exposto):

http
PUT /apiconnect/routes-services/v1/companies/{companyId}/routes/{routeId}/status/{statusId}

No microsserviço:

http
PUT /routes-approve-services/companies/{company-id}/routes/{route-id}/status/{status-id}

Só altera FL_STATUS_RTR. _Não_ mexe em EMPRESA_USUARIO_PRODUTO, economia nem PDF. O botão Aprovar da Rota Inteligente _não_ usa esse PUT.

### Calculation (economia, chamada interna)

http
GET {vt-orders-calculation}/benefits/calculation/companies/{companyId}/users/{userId}

Base configurável (ENV_VT_CALCULATION_API_URL). Não é chamada pela tela; o vt-roteirizacao chama por baixo.

---

## Fluxo no microsserviço — etapa por etapa

Caminho: tela Rota Inteligente (vt-frontend) → API Connect → _vt-roteirizacao_.

### Etapa 1 — A tela envia a lista

_Frontend:_ RoteirizacaoService.approveRoutes(lista)  
Dispara o POST completo de aprovação (API Connect) com um ou mais pares empresa + roteirização.

### Etapa 2 — Controller recebe

_Método:_ RoteirizacaoAprovacaoController.aprovaRoteirizacao  
Recebe o body e encaminha para o usecase.

### Etapa 3 — Usecase orquestra

_Método:_ RoteirizacaoAprovacaoUsecaseImpl.aprovarRoteirizacao  
Processa item a item da lista.

### Etapa 4 — Lê a rota e as compras sugeridas

1. RotaTranspRoteirizacaoRepository.findByIdRoteirizacaoAndIdEmpresa — carrega a roteirização.
2. RotaTranspCompraRepository.findByIdRoteirizacao — carrega os produtos de Vale-Transporte sugeridos pela SAT.

### Etapa 5 — Gate de negócio

Só continua se _todas_ forem verdadeiras:

- a rota existe para aquele par empresa + id;
- o status da compra (FL_STACOM_RTR) está _Finalizado (F)_ — a SAT já devolveu a codificação;
- existe lista de compras em ROTA_TRANSP_COMPRA.

Se falhar: o código _só loga_ e _não aprova_ aquele item (não lança exceção para o lote inteiro).

### Etapa 6 — Valida cada benefício sugerido

_Método:_ validacoesSalvarProduto para cada linha de ROTA_TRANSP_COMPRA

Checa:

- produto e quantidade informados;
- produto existe na tabela PRODUTO;
- se o produto for facial, a tarifa precisa ser maior que zero.

Falha → ValidacaoException → resposta NOK para a tela.

### Etapa 7 — Grava economia (opcional)

_Método:_ calculaEconomia → cliente REST VtCalculationRepositoryRestClient.calcularValoresValesPorUsuario

Chama o calculation (endpoint completo acima). Em seguida faz _INSERT_ em ROTA_TRANSP_ROTEIRIZ_ECONOMIA:

| Coluna        | Significado                                                       |
| ------------- | ----------------------------------------------------------------- |
| ID_ROTEIR_RTR | Qual roteirização                                                 |
| ID_EMPRES_EMP | Qual empresa                                                      |
| VL_BENEFI_RTE | Quanto o Vale-Transporte _custava antes_ (retorno do calculation) |
| VL_BENSAT_RTE | Quanto _vai custar_ com os produtos da SAT                        |
| VL_ECONOM_RTE | Diferença (economia estimada)                                     |

Se o calculation falhar: registra log e a aprovação _continua sem_ essa linha.

### Etapa 8 — Troca o Vale-Transporte (efeito de negócio)

1. _DELETE_ em EMPRESA_USUARIO_PRODUTO do colaborador (deleteByIdIdUsuario) — remove o benefício atual.
2. _INSERT_ de uma linha por compra SAT (salvarBeneficiosUsuario), com:
   - ID_USUARI_EUS — colaborador
   - ID_PRODUT_PRO — produto sugerido
   - QT_QUANTI_EUP — quantidade
   - VL_UNITAR_EUP — valor unitário / tarifa
   - QT_BLOCAG_EUP = 1
   - FL_ATIVOS_EUP = 'S' — benefício ativo
   - datas de solicitação / atualização

É neste passo que o cadastro do colaborador muda de fato.

### Etapa 9 — Fecha a roteirização

_Método:_ alterarStatusFlagRoteirizacao(idRoteirizacao, 'F')

sql
UPDATE ROTA_TRANSP_ROTEIRIZACAO
SET FL_STATUS_RTR = 'F', -- Finalizado
DT_CONCLU_RTR = SYSDATE -- data de conclusão
WHERE ID_ROTEIR_RTR = :id;

Neste fluxo _não_ preenche FL_ACEITA_RTR (flag de aceite) nem DT_ACEITA_RTR (data do aceite).

### Etapa 10 — Regenera o PDF

_Método:_ RoteirizacaoRelatorioUsecaseImpl.populaRoteirizacaoDetalhe

Monta o HTML do detalhe, converte em PDF e sobe no S3 com nome no formato {idUsuario}\_{idRoteirizacao}.pdf.

### Etapa 11 — Resposta para a tela

Sucesso:

json
{
"status": "OK",
"mensagem": "",
"response": null
}

Erro de validação de benefício:

json
{
"status": "NOK",
"mensagem": "Benefício ou quantidade não informado.",
"response": null
}

### A SAT é chamada neste fluxo?

_Não._ Nenhuma operação SOAP. A SAT já foi consultada antes (trajeto + compra); aqui só se efetiva o que já está gravado.

---

## Fluxo legado (smartweb) — etapa por etapa

Não usa vt-roteirizacao, não usa calculation e _não chama a SAT_ na aprovação.

### Cadeia de chamadas (importante)

Tela smartweb (JS aceiteRotaDo)
→ AjaxRoteirizacao.aceitaRoteirizacao (só recebe o Ajax / abre transação)
→ SATWebServiceBusiness.aprovaRoteirizacao (regra de negócio — nome engana)
→ RoteirizacaoDAO (UPDATE ROTEIRIZACAO)
→ EmpresaUsuarioProduto (DELETE + INSERT do VT)
→ PortalUsuAcessoDAO (ajusta termo no portal, se houver)

O método aprovaRoteirizacao está dentro da classe SATWebServiceBusiness porque, no legado, essa classe é a fachada de _tudo de roteirização_ (consulta SAT, grava rota, etc.).  
Na aprovação, porém, ele _só fala com banco Oracle_ — _não_ abre SOAP / webservice da SAT.

### Etapa 1 — RH confirma no smartweb

Entrada Ajax (não é REST do microsserviço):

POST ajaxRoteirizacao
action=aceitaRoteirizacao
idroteir={id}

A tela chama só o Ajax. O Ajax delega a regra para SATWebServiceBusiness.aprovaRoteirizacao.

### Etapa 2 — Carrega e valida a rota

RoteirizacaoDAO.buscaById — só segue se CD_STSALE_RTZ = FINALIZADO.  
Se não estiver finalizada → exceção e rollback.

### Etapa 3 — Marca o aceite do RH

sql
UPDATE ROTEIRIZACAO
SET FL_ACEITA_RTZ = 'S'
WHERE ...;

### Etapa 4 — Troca o Vale-Transporte

_DELETE_ + _INSERT_ em EMPRESA_USUARIO_PRODUTO a partir de ROTA_COMPRA — mesmo papel do microsserviço (substituir o benefício atual pelo sugerido).

### Etapa 5 — Ajuste no portal (se o colaborador tem acesso)

- PORTAL_USU_ACESSO.FL_IMPTMO_UAC = 'N' — obriga ver de novo o termo / impressão
- PORTAL_USU_ACESSO.ID_ROTACE_RTZ — vincula a roteirização aceita

Não grava tabela de economia. Não chama SAT.

### Aceite do colaborador no PortalVT — outro fluxo

O método registraParaAprovar cria um registro em _PORTAL_USU_SOLICITACAO_.  
Isso _não_ troca EMPRESA_USUARIO_PRODUTO. O RH ainda precisa rodar aceitaRoteirizacao no smartweb para efetivar.

---

## Para o Portal do Colaborador

Reutilizar a lógica de aprovarRoteirizacao atrás de um endpoint pensado para o colaborador, por exemplo:

http
POST /apiconnect/routes-services/v1/companies/{companyId}/routes/{routeId}/accept

(contrato sugerido — _ainda não existe_)

E complementar a gravação com:

- FL_ACEITA_RTR = 'S' — registrou o aceite
- DT_ACEITA_RTR = agora — quando aceitou

**_Decisão de PO:_ ao Aceitar, a rota vai direto para _Finalizado (F)_ (como o RH faz hoje) ou passa por um status intermediário _Aprovado colaborador (AC)_?**

---

## Código de referência

- Microsserviço: RoteirizacaoAprovacaoUsecaseImpl.aprovarRoteirizacao
- Frontend: vt-frontend/.../roteirizacao.service.ts (approveRoutes)
- Legado: SATWebServiceBusiness.aprovaRoteirizacao
- Visão geral: [Visão Geral](https://dev.azure.com/alelo/ValeTransporte-2.0/_wiki/wikis/ValeTransporte-2.0.wiki/69932/Contexto-Sobre-Aprovar-e-Contestar)
