# Refinamento — Decisão após a roteirização (visão geral)

Atualizado: 18/09/2026  
Objetivo: base técnica para o _Portal do Colaborador (Fase 5)_ — o que reutilizar nos microsserviços depois que a rota da SAT já foi calculada.

---

## Glossário rápido

| Termo           | Significado                                                              |
| --------------- | ------------------------------------------------------------------------ |
| _SAT_           | Fornecedor externo que calcula a rota e os benefícios de Vale-Transporte |
| _VT_            | Vale-Transporte (benefício do colaborador)                               |
| _EUP_           | Tabela EMPRESA_USUARIO_PRODUTO — cadastro real do VT do colaborador      |
| _MS_            | Microsserviço (vt-roteirizacao, vt-roteirizacao-sat, …)                  |
| _RH_            | Usuário empresa na Rota Inteligente / smartweb                           |
| _FL_STATUS_RTR_ | Status Alelo da roteirização (máquina de estados)                        |
| _CD_CONFOR_RTR_ | Id do funcionário/consulta na SAT (chave para contestar)                 |

---

## Escopo

| Entra                                            | Não entra                                                           |
| ------------------------------------------------ | ------------------------------------------------------------------- |
| _Aprovar_ — Aceitar e efetivar o Vale-Transporte | Rejeitar / “Não quero” / Cancelar benefício                         |
| _Contestar_ — Pedir reanálise da rota à SAT      | Login CPF+MFA, listagem, notificação, geração da rota (US #1694122) |

![Diagrama-aprovar-contestar-rejeitar.drawio (1).png](</.attachments/Diagrama-aprovar-contestar-rejeitar.drawio%20(1)-061c6dcf-2b87-4223-8f7c-328802b3b093.png>)

Documentos:

| Doc                                                                                                                                 | Quando ler                                          |
| ----------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| _Este_                                                                                                                              | Decisão, status, comparação e o que falta no Portal |
| [REFINAMENTO-APROVAR](https://dev.azure.com/alelo/ValeTransporte-2.0/_wiki/wikis/ValeTransporte-2.0.wiki/69934/Refinamento-Aprovar) | Passo a passo da efetivação do benefício            |
| [REFINAMENTO-CONTESTAR.md](./REFINAMENTO-CONTESTAR.md)                                                                              | Passo a passo da reanálise na SAT                   |

Formato: explicação _etapa a etapa_, sem diagramas.  
Histórico fora de escopo: [REFINAMENTO-REJEITAR.md](./REFINAMENTO-REJEITAR.md).

---

## Veredito (o que fazer)

1. _Não_ reconstruir a integração com a SAT.
2. _Aprovar_ — reutilizar POST /apiconnect/routes-services/v1/companies/routes-approval (no MS: POST /routes-approve-services/routes-approval). No Portal novo, o “Aceitar” do colaborador deve chamar essa mesma lógica (direto ou via um wrapper fino).
3. _Contestar_ — reutilizar o fluxo que já existe de ponta a ponta: listar motivos, abrir na SAT, batch consultar resposta.
4. _Rejeitar_ — fora do Portal novo; não planejar botão nem regra de negócio.

Há _dois pipelines_ que não se misturam:

Novo: tela Rota Inteligente ──API Connect──► vt-roteirizacao ──► vt-roteirizacao-sat ──SOAP──► SAT
Legado: smartweb / portalvt ─────────────────► smartdb ───────────────────────────────SOAP──► SAT

O Portal 1.0 (smartweb) _não_ chama o microsserviço vt-roteirizacao.

---

## Pré-condição (antes de decidir)

A rota precisa estar _pronta para decisão_. Antes disso, o sistema já executou:

1. _Inserir rota na SAT_ — a SAT devolve um id de consulta; gravamos em CD_CONFOR_RTR.
2. _Consultar trajeto_ — linhas de ônibus/metrô salvas em ROTA_TRANSP_TRAJETO.
3. _Consultar codificação de compra_ — produtos de VT sugeridos salvos em ROTA_TRANSP_COMPRA.
4. _Status de “pronta”:_
   - No microsserviço: status da rota = _Aguardando aprovação (AP)_ e status da compra = _Finalizado (F)_.
   - No legado: status Alelo = _FINALIZADO_.

Sem o id da SAT (CD*CONFOR_RTR), não dá para abrir contestação.  
Sem compra finalizada e lista de produtos, a aprovação \_não efetiva* o benefício.

---

## Ciclo de decisão (negócio)

Com a rota _aguardando aprovação (AP)_ — ou _contestação respondida (CR)_ depois de uma reanálise — restam duas ações:

| Ação        | O que acontece na prática                                                                            | Status resultante         |
| ----------- | ---------------------------------------------------------------------------------------------------- | ------------------------- |
| _Aprovar_   | Substitui o Vale-Transporte atual do colaborador pelo sugerido pela SAT e encerra a roteirização     | _Finalizado (F)_          |
| _Contestar_ | Abre uma reclamação na SAT pedindo nova análise. O benefício do colaborador _não muda_ nesse momento | _Contestação aberta (CA)_ |

Enquanto a rota está em _contestação aberta (CA)_, um job (batch) pergunta periodicamente à SAT se já há resposta:

| O que a SAT devolve         | Novo status                            | O que o usuário faz                                   |
| --------------------------- | -------------------------------------- | ----------------------------------------------------- |
| Nenhum item de contestação  | Volta para _aguardando aprovação (AP)_ | Pode aprovar ou contestar de novo                     |
| Texto de solução preenchido | Vai para _contestação respondida (CR)_ | Lê a resposta e decide de novo (aprovar ou contestar) |
| Ainda sem solução           | Continua em _CA_                       | Espera o próximo ciclo do batch                       |

Limite: _no máximo 3_ aberturas de contestação por roteirização.

---

## Aprovar × Contestar (comparativo)

|                                                      | _Aprovar_                                                                                    | _Contestar_                                                                                                                                                                                                  |
| ---------------------------------------------------- | -------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Quem usa hoje                                        | RH na Rota Inteligente ou no smartweb                                                        | RH na Rota Inteligente                                                                                                                                                                                       |
| Serviços envolvidos                                  | Principalmente vt-roteirizacao                                                               | vt-roteirizacao (lista motivos) + vt-roteirizacao-sat (fala com a SAT)                                                                                                                                       |
| Fala com a SAT?                                      | _Não_                                                                                        | _Sim_ — abre reclamação e depois consulta a resposta                                                                                                                                                         |
| Outro serviço                                        | vt-orders-calculation calcula economia estimada (se falhar, a aprovação _segue_ mesmo assim) | Filas ActiveMQ regeneram o PDF; após resposta também tentam reconsultar trajeto/compra                                                                                                                       |
| Muda o VT do colaborador?                            | _Sim_ — apaga o cadastro atual e grava o sugerido pela SAT                                   | _Não_ na abertura da contestação                                                                                                                                                                             |
| Caminho de status                                    | Aguardando aprovação → Finalizado                                                            | Aguardando / Respondida → Aberta → Respondida ou de volta a Aguardando                                                                                                                                       |
| Chamadas da tela (Rota Inteligente, via API Connect) | POST /apiconnect/routes-services/v1/companies/routes-approval                                | Listar motivos: GET /apiconnect/routes-services/v1/dispute-reasons · Abrir: POST /apiconnect/routes-services/v1/sat/companies/{companyId}/routes/{routeId}/contestations?idReclamacao={id}&codTipo={codTipo} |
| Equivalente no microsserviço                         | POST /routes-approve-services/routes-approval                                                | GET /routes-services/dispute-reasons · POST /routes-services/sat/companies/{company-id}/routes/{route-id}/contestations?idReclamacao={id}&codTipo={codTipo}                                                  |

---

## Status que importam

Campo FL_STATUS_RTR na tabela ROTA_TRANSP_ROTEIRIZACAO:

| Código | Nome                                  | Pode o usuário decidir?            |
| ------ | ------------------------------------- | ---------------------------------- |
| _AP_   | Aguardando aprovação                  | Sim — aprovar ou contestar         |
| _CA_   | Contestação aberta (aguardando a SAT) | Não — só esperar o batch           |
| _CR_   | Contestação respondida                | Sim — aprovar ou contestar de novo |
| _F_    | Finalizado (benefício já efetivado)   | Não — fluxo encerrado              |

A chave da SAT na rota é o campo _CD_CONFOR_RTR_.

Os códigos AC (aprovado colaborador) e RC (reprovado colaborador) existem no enum do código, mas _não_ entram neste escopo.

---

## Quem faz o quê hoje

### Aprovar — stack novo (microsserviços)

1. O RH, na tela _Rota Inteligente_ (vt-frontend), escolhe a(s) roteirização(ões) e clica em Aprovar.
2. O serviço vt-roteirizacao confere se a rota existe, se a _compra_ já está finalizada e se há produtos sugeridos pela SAT.
3. Opcionalmente grava um _snapshot de economia_ (custo antes × custo depois), via vt-orders-calculation. Se esse cálculo falhar, a aprovação _não para_.
4. _Troca o benefício:_ remove o Vale-Transporte atual do colaborador na tabela EMPRESA_USUARIO_PRODUTO e inclui os produtos vindos da SAT.
5. Marca a roteirização como _Finalizado (F)_ e preenche a data de conclusão.
6. Regenera o _PDF_ do detalhe da rota e envia para o S3.

Detalhamento técnico: [REFINAMENTO-APROVAR](https://dev.azure.com/alelo/ValeTransporte-2.0/_wiki/wikis/ValeTransporte-2.0.wiki/69934/Refinamento-Aprovar).

### Aprovar — stack legado (smartweb / PortalVT)

1. O RH, no _smartweb_, confirma a rota (aceitaRoteirizacao).
2. O sistema marca o aceite do RH (FL*ACEITA_RTZ = S) e \_troca o Vale-Transporte* do colaborador (mesma ideia: apaga o atual e grava o sugerido).
3. Se o colaborador tiver acesso ao portal, ajusta flags para ele ver de novo o termo/impressão.

### Contestar — só no stack novo

O legado tem stubs SOAP, mas _não_ há botão Contestar equivalente no portal/smartweb.

1. _Carregar motivos_ — a tela busca a lista no Oracle (ROTA*TRANSP_MOTIVO_CONTES) pelo serviço vt-roteirizacao. Neste passo \_não* há chamada à SAT.
2. _Abrir a contestação_ — o RH escolhe o motivo; o serviço vt-roteirizacao-sat envia SOAP IncluiJustificativa à SAT. Se der certo, a rota fica em _Contestação aberta (CA)_, grava histórico e regenera o PDF.
3. _Aguardar resposta_ — um batch no vt-roteirizacao lista as rotas em CA, publica na fila vt.routes.sat.contestation, e o vt-roteirizacao-sat consulta a SAT (ConsultaJustificativa). Com solução → _Contestação respondida (CR); sem itens → volta a \*\*Aguardando aprovação (AP)_; sem solução ainda → permanece em CA.

Detalhamento técnico: [REFINAMENTO-CONTESTAR.md](./REFINAMENTO-CONTESTAR.md).

---

## Portal do Colaborador (Fase 5)

| Ação do usuário | O que já existe para reutilizar                                          | O que ainda falta                                                                                                  |
| --------------- | ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| _Aceitar_       | A lógica de aprovarRoteirizacao (troca do benefício + status Finalizado) | Endpoint pensado para o colaborador (“accept”) e gravar explicitamente o aceite (FL_ACEITA_RTR / DT_ACEITA_RTR)    |
| _Contestar_     | Fluxo completo: motivos + abrir na SAT + batch de resposta               | Camada BFF/proxy para o Portal, tela de espera enquanto está em contestação aberta, e aviso quando a SAT responder |

### Perguntas abertas (PO)

1. No _Aceitar_ do colaborador: o benefício é efetivado _na hora_ (vira Finalizado, como o RH faz hoje) ou ainda passa por uma aprovação do RH?
2. Se a rota ficar dias em _Contestação aberta_ sem resposta da SAT: mantém, volta para Aguardando aprovação, ou dispara alerta?
3. Quando chegar _Contestação respondida_, a tela mostra o texto da solução da SAT (DS_STAFOR_RTR) antes de o usuário decidir de novo?

---

## Leitura seguinte

1. [REFINAMENTO-APROVAR](https://dev.azure.com/alelo/ValeTransporte-2.0/_wiki/wikis/ValeTransporte-2.0.wiki/69934/Refinamento-Aprovar) — efetivação do Vale-Transporte
2. [REFINAMENTO-CONTESTAR.md](./REFINAMENTO-CONTESTAR.md) — reanálise na SAT
