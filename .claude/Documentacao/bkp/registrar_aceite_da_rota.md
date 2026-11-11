# 2 - [BE] Registrar aceite da rota

## História
**Como** sistema,  
**quero** registrar o aceite da rota com data, hora, identidade e versão do termo,  
**para** manter evidência legal e atualizar a jornada.

---

## Contexto
Após revisar a rota e manifestar concordância no termo, o colaborador confirma o aceite. O registro precisa ser íntegro, auditável e resistente a repetições.

---

## Objetivo
Persistir uma única evidência de aceite, alterar o estado da rota para **Aceita** e iniciar a disponibilização da carta.

---

## Escopo
- Validar sessão, vínculo, rota e estado elegível.
- Persistir timestamp, identificação do usuário e versão do termo.
- Garantir idempotência.
- Atualizar estado para **Aceita**.
- Disparar geração/disponibilização da carta.
- Registrar trilha de auditoria e observabilidade.

---

## Fora do Escopo
- Renderizar o termo ou a confirmação.
- Gerar o conteúdo PDF da carta.
- Reverter aceite já concluído.

---

## Dependências
- Rota válida e aguardando aceite.
- Versão vigente do termo e identidade autenticada.
- Capacidade de carta da US 1718650 e fluxo da US 1718652.

---

## Cenários BDD

### Cenário 1 — Aceite válido
**Dado** um colaborador autenticado com rota aguardando aceite e termo vigente  
**Quando** confirmar o aceite  
**Então** a evidência é persistida, a rota passa para Aceita e a carta é solicitada.

### Cenário 2 — Repetição idempotente
**Dado** que a mesma rota já foi aceita pelo colaborador  
**Quando** a confirmação for repetida  
**Então** nenhum aceite duplicado é criado e o resultado permanece consistente.

### Cenário 3 — Estado incompatível
**Dado** uma rota que não está elegível para aceite  
**Quando** a confirmação for solicitada  
**Então** a operação é rejeitada sem alterar a rota.

### Cenário 4 — Rota de outro vínculo
**Dado** uma rota não pertencente ao contexto autenticado  
**Quando** houver tentativa de aceite  
**Então** o acesso é negado e nenhum dado é persistido.

---

## Critérios de Aceite
1. O aceite exige sessão válida e rota pertencente ao colaborador e à empresa selecionada.
2. A operação só é permitida quando a rota está em estado elegível para aceite.
3. A evidência persiste data/hora, identificação do usuário e versão do termo aceito.
4. Após a confirmação bem-sucedida, o estado da rota passa para **Aceita**.
5. Repetições da mesma confirmação não criam registros duplicados nem repetem efeitos.
6. Solicitações concorrentes mantêm uma única evidência válida e resultado consistente.
7. O sucesso dispara a geração ou disponibilização da carta de confirmação de forma rastreável.
8. Rejeições e falhas não deixam atualização parcial e são observáveis sem exposição de dados sensíveis.

---

## Regras de Negócio
- Somente o colaborador autorizado pode aceitar sua rota no vínculo selecionado.
- O aceite registra data/hora, identidade e versão do termo apresentada.
- Após sucesso, a rota passa para **Aceita**.
- Repetições da mesma confirmação não criam evidências duplicadas nem repetem efeitos.
- Rota inexistente, de outro vínculo ou em estado incompatível não pode ser aceita.
- O aceite bem-sucedido inicia a geração ou disponibilização da carta de confirmação.