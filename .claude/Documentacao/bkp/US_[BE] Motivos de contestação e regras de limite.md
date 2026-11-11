# [BE] Motivos de contestação e regras de limite

## Descrição

### História
> **Como** sistema do Portal do Portador,  
> **quero** disponibilizar os motivos de contestação e controlar o limite por período,  
> **para** orientar o colaborador e impedir aberturas acima da regra permitida.

### Contexto
A jornada de contestação deve apresentar um catálogo único de nove motivos e informar, antes da abertura, se o colaborador ainda pode contestar a rota da empresa selecionada.

### Objetivo
Centralizar as regras de elegibilidade, limite e saldo de contestações, garantindo comportamento consistente para consulta e abertura.

---

## Escopo e Limites

### Escopo
- Disponibilização dos nove motivos previstos.
- Apuração do limite de três contestações no período aplicável.
- Retorno da elegibilidade, quantidade utilizada e quantidade restante.
- Bloqueio de nova abertura ao atingir o limite.
- Segurança, rastreabilidade e observabilidade da regra.

### Fora do Escopo
- Registro do payload da contestação.
- Análise ou resposta da contestação.
- Construção das telas e formulários.

### Dependências
- Identificação do colaborador e contexto da empresa/rota.
- Histórico de contestações do período.
- Integração com a abertura de contestação da US 1718655.

---

## BDD (Behavior-Driven Development)

### Cenário 1 — Catálogo completo
* **Dado** um colaborador elegível para consultar motivos
* **Quando** os motivos forem solicitados
* **Então** os nove motivos previstos serão retornados sem duplicidade.

### Cenário 2 — Saldo disponível
* **Dado** que o colaborador possui uma contestação no período
* **Quando** a elegibilidade for consultada
* **Então** o limite será três, o total utilizado será um e o saldo restante será dois.

### Cenário 3 — Limite atingido
* **Dado** que o colaborador já possui três contestações no período
* **Quando** tentar iniciar uma nova contestação
* **Então** a abertura será bloqueada e o saldo retornado será zero.

### Cenário 4 — Isolamento de contexto
* **Dado** um colaborador vinculado a mais de uma empresa
* **Quando** consultar o limite no contexto da empresa selecionada
* **Então** somente as contestações desse contexto e período serão consideradas.

---

## Critérios de Aceite

1. **Catálogo de Motivos:** Disponibilizar exatamente os nove motivos definidos para a jornada, sem omissão ou duplicidade.
2. **Identificação dos Motivos:** Identificar cada motivo de forma estável e apresentar descrição compreensível para consumo do Portal.
3. **Apuração do Limite:** Apurar o limite de três contestações no período aplicável ao colaborador e à empresa selecionada.
4. **Retorno de Saldos:** Retornar quantidade utilizada, limite total e quantidade restante, sem valores negativos.
5. **Permissão de Continuidade:** Permitir continuidade quando a quantidade utilizada for menor que três.
6. **Bloqueio Reativo:** Bloquear nova abertura quando a quantidade utilizada for igual ou superior a três.
7. **Consistência das Regras:** Usar na consulta e na abertura a mesma regra de elegibilidade e saldo.
8. **Isolamento de Dados:** Manter a apuração isolada por colaborador, empresa e período, sem exposição cruzada de dados.
9. **Concorrência:** Garantir consistência do limite diante de tentativas simultâneas de abertura.
10. **Observabilidade e Rastreabilidade:** Registrar sucesso, bloqueio e falha para observabilidade, sem dados sensíveis nos registros.

---

## Regras de Negócio

* **Catálogo Fixo:** O catálogo contém exatamente os 9 motivos abaixo:
  1. Valor da tarifa desatualizado
  2. Tempo de integração maior que o permitido
  3. Tempo de percurso incorreto
  4. Local de embarque distante
  5. Local de desembarque distante
  6. Linha não está mais em circulação
  7. Linha não circula no horário de trabalho
  8. Itinerário incorreto
  9. Unificar cartão de transporte
* **Limite do Período:** O limite é de três contestações por colaborador, empresa e período aplicável.
* **Bloqueio Estrito:** Ao atingir o limite, nenhuma nova contestação pode ser iniciada.
* **Consistência do Saldo:** O saldo restante deve refletir a mesma apuração usada na abertura.
* **Privacidade:** A regra não deve expor dados de outros colaboradores ou empresas.