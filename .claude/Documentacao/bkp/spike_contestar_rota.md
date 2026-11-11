# Refinamento — Contestar

Voltar: [visão geral](https://dev.azure.com/alelo/ValeTransporte-2.0/_wiki/wikis/ValeTransporte-2.0.wiki/69932/Contexto-Sobre-Aprovar-e-Contestar)

## Em uma frase

O RH escolhe um motivo de contestação. O microsserviço vt-roteirizacao-sat pede reanálise à SAT. A rota fica em _Contestação aberta (CA). Depois um batch consulta a resposta: com solução vira **Contestação respondida (CR); sem itens volta para **Aguardando aprovação (AP)_.

Os motivos do combo vêm do _Oracle_ (vt-roteirizacao). Abrir e consultar falam com a _SAT_ (vt-roteirizacao-sat). Máximo _3_ contestações por roteirização.

Não confundir com Aprovar (aprovar efetiva o Vale-Transporte; contestar _não_ troca o benefício na abertura).  
_Fora de escopo:_ Rejeitar / “Não quero” — o Portal novo usa só _Aprovar_ e _Contestar_.

---

## Glossário rápido

| Termo           | Significado                                                          |
| --------------- | -------------------------------------------------------------------- |
| _SAT_           | Fornecedor externo que recalcula / responde a reclamação             |
| _VT_            | Vale-Transporte (não muda na abertura da contestação)                |
| _CD_CONFOR_RTR_ | Id do funcionário/consulta na SAT — chave obrigatória para contestar |
| _FL_STATUS_RTR_ | Status Alelo da roteirização (AP, CA, CR, …)                         |
| _ID_CONTES_RMC_ | Motivo de contestação gravado na rota                                |
| _DS_STAFOR_RTR_ | Texto de status/solução vindo do fornecedor (SAT)                    |

---

## Quem faz o quê

| Precisa de…        | Serviço                                  | O que faz                                                               |
| ------------------ | ---------------------------------------- | ----------------------------------------------------------------------- |
| Motivos do combo   | _vt-roteirizacao_                        | Lê a tabela Oracle ROTA_TRANSP_MOTIVO_CONTES (cache). _Não_ chama a SAT |
| Abrir contestação  | _vt-roteirizacao-sat_                    | Envia SOAP IncluiJustificativa à SAT                                    |
| Consultar resposta | _vt-roteirizacao-sat_ (via batch + fila) | Envia SOAP ConsultaJustificativa à SAT                                  |
| Status da rota     | Oracle                                   | Campo FL_STATUS_RTR (CA / CR / AP)                                      |

No legado existem stubs SOAP, mas smartweb/portalvt _não_ orquestram botão Contestar equivalente.

---

## Pré-condição

Antes de abrir contestação:

- rota em _Aguardando aprovação (AP)_ ou _Contestação respondida (CR)_ (pode decidir de novo);
- _CD_CONFOR_RTR_ preenchido (id devolvido pela SAT no InseriRota);
- menos de _3_ aberturas já registradas no histórico.

---

## Endpoints (completos)

### Listar motivos — o que a tela chama (API Connect)

http
GET /apiconnect/routes-services/v1/dispute-reasons

### Listar motivos — no microsserviço vt-roteirizacao

http
GET /routes-services/dispute-reasons

### Abrir contestação — o que a tela chama (API Connect)

http
POST /apiconnect/routes-services/v1/sat/companies/{companyId}/routes/{routeId}/contestations?idReclamacao={id}&codTipo={codTipo}

### Abrir contestação — no microsserviço vt-roteirizacao-sat

http
POST /routes-services/sat/companies/{company-id}/routes/{route-id}/contestations?idReclamacao={id}&codTipo={codTipo}

Sem body. Query params obrigatórios: idReclamacao, codTipo.  
O controller envia valor = "" fixo para o SOAP.

### Histórico de contestações (consulta)

Via API Connect (se exposto no mesmo padrão):

http
GET /apiconnect/routes-services/v1/companies/{companyId}/routes/{routeId}/contestation-details

No microsserviço vt-roteirizacao:

http
GET /routes-services/companies/{company-id}/routes/{route-id}/contestation-details

### Batch que verifica resposta da SAT

No microsserviço vt-roteirizacao (job / trigger interno — não é a tela do RH):

http
POST /routes-contestation-batch-flow/execute

### Motivos “ao vivo” na SAT (existe, a Rota Inteligente não usa)

http
GET /routes-services/sat/justifications

No vt-roteirizacao-sat: SOAP ConsultaListaJustificativaPadrao + ConsultaCamposJustificativa.  
A tela RI usa o _cache Oracle_, não este endpoint.

---

## 0) Listar motivos — etapa por etapa

### Etapa 1 — A tela carrega o combo

_Frontend:_ RoteirizacaoService.getJustifications  
Chama o GET completo de dispute-reasons (API Connect).

### Etapa 2 — Lê o cache no Oracle

_Método:_ listaJustificativas → RotaTranspMotContRepository.findAll  
Tabela: ROTA_TRANSP_MOTIVO_CONTES.

_Neste passo não há chamada à SAT._

### Etapa 3 — Resposta do combo

json
{
"status": "OK",
"mensagem": "",
"response": [
{
"idReclamacao": 5,
"descReclamacao": "Linha sugerida inadequada",
"codTipo": 1,
"tipo": "...",
"descTipo": "..."
}
]
}

| Campo          | Uso                                                  |
| -------------- | ---------------------------------------------------- |
| idReclamacao   | Código do motivo na SAT (vai no query param do POST) |
| descReclamacao | Texto exibido no select                              |
| codTipo        | Tipo do campo auxiliar exigido pela SAT              |

---

## A) Abrir contestação — etapa por etapa

Serviço: _vt-roteirizacao-sat_

### Cadeia de chamadas

Tela Rota Inteligente (addContestation)
→ API Connect
→ Controller abrirContestacao
→ RoteirizacaoSatUsecaseImpl.abrirContestacaoSat
→ SatSoapUsecaseImpl.abrirContestacao (SOAP IncluiJustificativa)
→ RotaTranspRoteirizacaoRepository.save (UPDATE status CA)
→ criarHistoricoRotaTranspRoteirizacao (INSERT hist)
→ fila vt.routes.docs.generate (regenera PDF)

### Etapa 1 — A tela envia a contestação

_Frontend:_ addContestation(companyId, routeId, contestationId, typeCode)  
Dispara o POST completo (API Connect) com idReclamacao e codTipo.

### Etapa 2 — Controller / usecase

_Métodos:_ controller abrirContestacao → RoteirizacaoSatUsecaseImpl.abrirContestacaoSat.

### Etapa 3 — Valida limite de 3

Conta registros em ROTA_TRANSP_ROTEIRIZACAO_HIST (findQtdeRoteirizacoesByIdRoteirizacao).  
Se já houver 3 ou mais → retorna false (limite atingido).

### Etapa 4 — Carrega a rota

findById — exige rota presente e _CD_CONFOR_RTR_ preenchido.  
Sem o id da SAT → não abre a contestação.

### Etapa 5 — SOAP IncluiJustificativa

_Método:_ SatSoapUsecaseImpl.abrirContestacao

Envia para a SAT:

- idfuncionario = valor de CD_CONFOR_RTR
- idReclamacaoPadrao = query idReclamacao
- codTipo = query codTipo
- valor = "" (sempre vazio no controller atual)
- cpf / matrícula vazios
- header de autenticação SAT

Retorno: string com id da reclamação aberta. Se vier em branco → _não_ atualiza o status da rota.

### Etapa 6 — Atualiza a rota e grava histórico

sql
UPDATE ROTA_TRANSP_ROTEIRIZACAO
SET FL_STATUS_RTR = 'CA', -- Contestação aberta
ID_CONTES_RMC = :idMotivoContestacao,
DT_ATUALI_RTR = SYSDATE
WHERE ID_ROTEIR_RTR = :id;

O método atualizarRot também faz _INSERT_ em ROTA_TRANSP_ROTEIRIZACAO_HIST com FL_TIPLOG_TRH = 'A' (atualização).  
Com status CA, o histórico grava DS_STAFOR_RTR vazio.

### Etapa 7 — Publica fila de documento

Fila ActiveMQ: _vt.routes.docs.generate_  
Payload: só o idRoteirizacao.  
O vt-roteirizacao regenera o PDF do detalhe e sobe no S3.

### Etapa 8 — Resposta para a tela

Sucesso:

json
{
"status": "OK",
"mensagem": "",
"response": true
}

Falha (limite, sem CD_CONFOR_RTR, ou SOAP vazio):

json
{
"status": "NOK",
"mensagem": "...",
"response": false
}

HTTP pode ser 400 quando response é false.

A partir daqui a tela aguarda a SAT — status _Contestação aberta (CA)_.

---

## B) Verificar resposta (batch) — etapa por etapa

### Cadeia de chamadas

POST /routes-contestation-batch-flow/execute (vt-roteirizacao)
→ lista rotas com FL_STATUS_RTR = 'CA'
→ publica na fila vt.routes.sat.contestation
→ FluxoRoteirizacaoMQListeners.verificaContestacao (vt-roteirizacao-sat)
→ RoteirizacaoSatUsecaseImpl.verificarContestacao
→ SatSoapUsecaseImpl.verificaContestacao (SOAP ConsultaJustificativa)
→ atualiza status (AP / CR / mantém CA)

### Etapa 1 — Batch dispara

Projeto _vt-roteirizacao_:

http
POST /routes-contestation-batch-flow/execute

Seleciona rotas com _Contestação aberta (CA)_ e publica na fila:

vt.routes.sat.contestation

Payload da fila: id da roteirização (JSON string).

### Etapa 2 — Listener no vt-roteirizacao-sat

FluxoRoteirizacaoMQListeners.verificaContestacao → verificarContestacao.  
Só processa se o status ainda for _CA_ e existir CD_CONFOR_RTR.

### Etapa 3 — SOAP ConsultaJustificativa

Envia idfuncionario = CD_CONFOR_RTR (mesma chave usada na abertura).

### Etapa 4 — Três saídas possíveis

_4a — Sem itens de justificativa_

sql
UPDATE ROTA_TRANSP_ROTEIRIZACAO
SET FL_STATUS_RTR = 'AP', -- volta a Aguardando aprovação
DS_STAFOR_RTR = 'Não houve contestação na SAT.'
WHERE ID_ROTEIR_RTR = :id;

Grava histórico. O usuário pode aprovar ou contestar de novo.

_4b — Há reclamação, mas a solução ainda está vazia_

Mantém _Contestação aberta (CA)_. Não grava alteração. Espera o próximo ciclo do batch.

_4c — Há solução preenchida_

sql
UPDATE ROTA_TRANSP_ROTEIRIZACAO
SET FL_STATUS_RTR = 'CR', -- Contestação respondida
DS_STAFOR_RTR = :textoSolucaoSat
WHERE ID_ROTEIR_RTR = :id;

Grava histórico (DS_STAFOR_RTR _não_ é zerado — isso só acontece quando o status é CA).

Em seguida publica _em paralelo_ três mensagens ActiveMQ:

| Fila                       | Payload                               | Objetivo                   |
| -------------------------- | ------------------------------------- | -------------------------- |
| vt.routes.docs.generate    | idRoteirizacao                        | Regenerar PDF              |
| vt.routes.sat.consultation | { "idRoteirizacao", "codFornecedor" } | Reconsultar trajeto na SAT |
| vt.routes.sat.purchase     | { "idRoteirizacao", "idTrajeto" }     | Reconsultar compra na SAT  |

codFornecedor e idTrajeto usam o mesmo valor de _CD_CONFOR_RTR_.

### Etapa 5 — O que as filas CONSULTAR e COMPRAR fazem (e o caveat pós-CR)

| Fila                       | Papel quando processa                                                                 |
| -------------------------- | ------------------------------------------------------------------------------------- |
| vt.routes.sat.consultation | SOAP de trajeto → grava ROTA_TRANSP_TRAJETO → marca status do trajeto como finalizado |
| vt.routes.sat.purchase     | SOAP de compra → grava ROTA_TRANSP_COMPRA → marca status da compra como finalizado    |

No fluxo _inicial_ da rota (status _Processando (PR)), quando as duas terminam com sucesso, a fila vt.routes.sat.end pode subir a rota para \*\*Aguardando aprovação (AP)_.

_Caveat importante:_ os listeners de CONSULTAR e COMPRAR só processam se FL_STATUS_RTR = 'PR'. Depois da contestação o status é _CR, então essas duas filas **quase não reprocessam. Quem age de fato no pós-CR é a fila de **gerar documento_. O reenvio parece herdado do fluxo inicial.

---

## Legado

No smartdb existem stubs IncluiJustificativa / ConsultaJustificativa, mas _não_ há botão Contestar no portal/smartweb equivalente ao stack novo. Contestação ponta a ponta existe só nos microsserviços.

---

## Para o Portal do Colaborador

Reutilizar o mesmo desenho da Rota Inteligente (via BFF), com endpoints completos:

1. Listar motivos:  
   GET /apiconnect/routes-services/v1/dispute-reasons
2. Abrir contestação:  
   POST /apiconnect/routes-services/v1/sat/companies/{companyId}/routes/{routeId}/contestations?idReclamacao={id}&codTipo={codTipo}
3. Enquanto estiver em _Contestação aberta (CA)_: polling ou  
   GET /apiconnect/routes-services/v1/companies/{companyId}/routes/{routeId}/contestation-details
4. Quando virar _Contestação respondida (CR): notificar e reabrir a decisão (Aceitar_ ou _Contestar_ de novo)

Limite de 3 já está no microsserviço.  
Gap: motivos com campo tipo “lista” podem exigir SOAP extra; hoje o valor vai sempre vazio.

---

## Código de referência

- Motivos: vt-roteirizacao → listaJustificativas
- Abrir: RoteirizacaoSatUsecaseImpl.abrirContestacaoSat
- SOAP: SatSoapUsecaseImpl.abrirContestacao / verificaContestacao
- Batch: POST /routes-contestation-batch-flow/execute + fila vt.routes.sat.contestation
- Frontend: getJustifications + addContestation em roteirizacao.service.ts
- Visão geral: [Visão Geral](https://dev.azure.com/alelo/ValeTransporte-2.0/_wiki/wikis/ValeTransporte-2.0.wiki/69932/Contexto-Sobre-Aprovar-e-Contestar)
