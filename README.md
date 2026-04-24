# Esbocos Iniciais da Arquitetura

## Grupo 8 - Desafio 6
### Ala da Pediatria (Formularios)

Grupo: mlmsp, nvgb, vox, ymws  
Local e data: Recife, Abril de 2026

---

## 1. Estrategia de Integracao

### Abordagem adotada

- [x] API / servicos (Shared Business Function)
- [ ] Replicacao de dados (Data Replication)
- [x] Integracao orientada a eventos (mensageria)
- [ ] Processo distribuido (orquestracao)

### Como sera a integracao

- Predominantemente assincrona para operacoes nao criticas em tempo real (envio de formularios, atualizacoes de status e sincronizacao de eventos).
- Pontos sincronos apenas quando houver necessidade de validacao imediata no AGHU (exemplo: checagem de existencia de paciente no momento da abertura do formulario).
- Havera intermediarios:
  - API Gateway para controle de acesso, autenticacao e observabilidade.
  - Fila de mensagens para desacoplar o modulo de Formularios do AGHU.
  - Adaptador de integracao para isolar mudancas de contrato do AGHU.

### Justificativa e trade-offs

- Latencia: chamadas sincronas diretas ao AGHU podem aumentar o tempo de resposta em horario de pico. A fila reduz impacto no usuario final em operacoes que aceitam processamento posterior.
- Consistencia: o modelo assincrono aceita consistencia eventual em troca de resiliencia.
- Autonomia: o modulo de Formularios evolui de forma independente, pois o adaptador reduz acoplamento com o AGHU.
- Complexidade: incluir mensageria e retentativas aumenta complexidade operacional, mas melhora tolerancia a falhas.

---

## 2. Decisoes Arquiteturais

### Como lidar com inconsistencias de dados

- Aplicar idempotencia no processamento de eventos para evitar duplicidade.
- Manter rastreabilidade por identificador de correlacao e versao do formulario.
- Executar reconciliacao periodica entre base local e estado no AGHU.
- Definir regras de precedencia:
  - Dados clinicos oficiais no AGHU sao fonte de verdade.
  - Dados de preenchimento operacional no modulo de Formularios sao fonte de verdade ate confirmacao de recebimento no AGHU.

### Como lidar com indisponibilidade do AGHU

- Circuit breaker no adaptador para evitar cascata de falhas.
- Retentativas com backoff exponencial para erros transientes.
- Persistencia local de eventos em fila (store and forward).
- Modo degradado: permitir captura de formularios e enfileirar envio posterior.
- Alertas operacionais para equipe tecnica quando fila ultrapassar limite de atraso.

### Trade-off priorizado (CAP)

Prioridade principal: Disponibilidade e Tolerancia a Particao nas funcionalidades de captura e registro de formularios.

- Em cenarios de falha de rede ou AGHU indisponivel, o sistema permanece operacional localmente.
- A Consistencia forte entre modulo e AGHU nao e priorizada em tempo real para todos os fluxos; adota-se consistencia eventual com mecanismos de reconciliacao.

### Evolucao futura da solucao

- Curto prazo: modulo em arquitetura modular com fronteiras claras.
- Medio prazo: extracao do adaptador de integracao para servico independente.
- Longo prazo: evolucao para microservicos orientados a dominio (Formularios, Notificacoes, Integracao AGHU) com eventos de dominio.
- Possivel adocao de SOA leve para integracao com outros setores hospitalares.

---

## 3. Modelagem seguindo C4

Referencia: https://c4model.com

### Nivel 1 - Contexto

diagramas_c4/nivel1 

### Nivel 2 - Containers

diagramas_c4/nivel2

### Nivel 3 - Componentes

diagramas_c4/nivel3

---

## 4. Fluxo de Integracao

### Fluxo principal

1. Profissional da pediatria preenche e salva formulario no frontend.
2. Backend valida dados obrigatorios e grava no banco local.
3. Backend publica evento de integracao na fila.
4. Worker consome evento e aciona o adaptador do AGHU.
5. Adaptador transforma payload para contrato esperado pelo AGHU.
6. AGHU processa e retorna sucesso, rejeicao de negocio ou erro tecnico.
7. Resultado e registrado no banco com status de sincronizacao.
8. Usuario visualiza status do formulario (pendente, sincronizado ou com erro).

### Em caso de falha

- Falha transiente (timeout/rede): retentar com backoff exponencial.
- Falha persistente: enviar para DLQ e notificar equipe tecnica.
- AGHU indisponivel: manter formularios em estado pendente e seguir capturando localmente.
- Inconsistencia detectada: executar reconciliacao automatica e, se necessario, abrir tarefa manual de correção.

## 5. Protótipo simples “vibe coded"


[Protótipo Simples - V0 (Vibecoded)](https://v0-pediatric-clinical-app.vercel.app/)
