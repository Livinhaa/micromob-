# MicroMob — Monitoramento e alertas em tempo real de micromobilidade

Aplicação web full-stack que monitora uma frota de **bikes e patinetes elétricos compartilhados**: recebe telemetria
por um **broker de mensagens (RabbitMQ)**, persiste com **Prisma/PostgreSQL**, transmite ao dashboard por **WebSocket
(Socket.IO)** e dispara **push no celular via ntfy** quando ocorre um evento crítico (queda/colisão, bateria crítica,
saída da área de operação).

Trabalho Final de Programação Multiplataforma — Prof. Dr. Alexandre Galvani.

## Arquitetura

```
Simulador / POST /api/simulate
        │ publica (mensagem JSON validada)
        ▼
   RabbitMQ  ──(fila durável vehicle.events; DLQ vehicle.events.dlq)
        │ consome (ack manual, prefetch 20)
        ▼
   API Node/Fastify ── Prisma ──▶ PostgreSQL (Vehicle, Event, NotificationLog, ...)
        │                │
        │ Socket.IO      └─ se CRITICAL ──▶ ntfy (HTTP, retry exponencial) ──▶ celular
        ▼
   Dashboard React (mobile-first)
```

| Camada | Tecnologia |
|---|---|
| Backend | Node 22, TypeScript, Fastify, Socket.IO, amqplib, Zod |
| ORM / banco | Prisma 6 + PostgreSQL 16 |
| Broker | RabbitMQ 3.13 |
| Push | ntfy (público ou auto-hospedado) |
| Frontend | React 19 + Vite + CSS (Grid/Flexbox), socket.io-client |
| Infra | Docker Compose |

## Como executar (Docker Compose)

Pré-requisitos: Docker e Docker Compose.

```bash
cp .env.example .env
# edite .env: defina NTFY_TOPIC com um nome único e JWT_SECRET
docker compose up --build
```

- Dashboard: http://localhost:8080
- API: http://localhost:3000/api/health
- Painel do RabbitMQ: http://localhost:15672 (guest / guest)
- Login de demonstração (criado pelo seed): `operador@micromob.dev` / `micromob123`

**Receber o push no celular:** instale o app **ntfy** (Android/iOS) e assine o mesmo tópico definido em `NTFY_TOPIC`
(servidor `https://ntfy.sh`). Para usar uma instância própria: `NTFY_BASE_URL=http://ntfy:80` e
`docker compose --profile selfhosted up --build` (o celular precisa alcançar o servidor, ex.: `http://IP-DA-MAQUINA:8081`).

## Variáveis de ambiente

| Variável | Padrão | Descrição |
|---|---|---|
| `DATABASE_URL` | (compose) | Conexão PostgreSQL |
| `RABBITMQ_URL` | `amqp://guest:guest@rabbitmq:5672` | Conexão do broker |
| `JWT_SECRET` | — (obrigatória, ≥ 16 chars) | Assinatura dos tokens (REST e handshake WebSocket) |
| `NTFY_BASE_URL` | `https://ntfy.sh` | Servidor ntfy |
| `NTFY_TOPIC` | `micromob-alertas-CHANGE-ME` | Tópico que recebe os alertas |
| `NTFY_TOKEN` | vazio | Token Bearer, se o servidor ntfy exigir |
| `PUBLIC_URL` | `http://localhost:8080` | Base do link "Click" da notificação |
| `GEOFENCE_LAT` / `GEOFENCE_LNG` / `GEOFENCE_RADIUS_KM` | Bauru-SP / 5 km | Área de operação |
| `PORT`, `CORS_ORIGIN` | `3000`, `*` | Servidor HTTP |
| `SIM_TICK_MS` | `2000` | Intervalo do simulador |

## Modelo de dados (`backend/prisma/schema.prisma`)

`User` → `NotificationTopic` (tópico ntfy) · `Vehicle` → `Event` → `NotificationLog`.

- **Idempotência:** `Event.externalId` é `@unique`; `NotificationLog` tem `@@unique([eventId, topicId])`.
- **Índices:** `Event(receivedAt)`, `Event(severity, receivedAt)`, `Event(vehicleId, receivedAt)`, `NotificationLog(status, createdAt)`.
- **Sem N+1:** as listagens usam `include`/`select` em uma única chamada (ver `GET /api/events`).
- **Migrações versionadas:** `backend/prisma/migrations/` (aplicadas por `prisma migrate deploy` ao subir a API).
- **Seed:** `backend/prisma/seed.ts` cria 1 usuário, 1 tópico ntfy e 20 veículos (idempotente).

## Eventos e regras

| Evento | Regra | Severidade | Push ntfy |
|---|---|---|---|
| `BATTERY_LOW` | bateria cruza < 20% | WARNING | não |
| `BATTERY_CRITICAL` | bateria cruza ≤ 5% | **CRITICAL** | sim (prioridade 4) |
| `GEOFENCE_EXIT` | sai do raio da área | **CRITICAL** | sim (prioridade 4) |
| `CRASH_DETECTED` | acelerômetro acima do limite | **CRITICAL** | sim (prioridade 5) |
| `RETURNED_OUTSIDE_STATION` | devolução fora da estação | WARNING | não |
| `TELEMETRY` | leitura periódica | INFO | não (atualiza o estado do veículo e o dashboard; não gera linha em `Event`) |

Leituras `TELEMETRY` só geram evento quando **cruzam** um limite, evitando alertas repetidos.

## Rotas

| Método | Rota | Auth | Descrição |
|---|---|---|---|
| GET | `/api/health` | não | Verificação de saúde |
| POST | `/api/auth/login` | não | `{email, password}` → `{token, user}` |
| GET | `/api/vehicles` | JWT | Frota e estado atual |
| GET | `/api/events?severity=&limit=` | JWT | Eventos com veículo e status de entrega |
| GET | `/api/notifications?limit=` | JWT | Histórico de disparos ntfy |
| GET | `/api/stats` | JWT | Métricas agregadas (24 h) |
| GET | `/api/weather` | JWT | Clima da área (API externa Open-Meteo, cache 60 s) |
| POST | `/api/simulate` | JWT | `{type, vehicleCode?}` publica um evento de teste **no broker** |

**WebSocket (Socket.IO):** conexão com `auth: { token }` no handshake (JWT; sem token válido → `unauthorized`).
Eventos do servidor: `hello`, `telemetry`, `event`, `notification`. Heartbeat: ping a cada 10 s, timeout de 5 s.

## Como os requisitos são atendidos

- **Persistência confiável:** todo evento é gravado com `receivedAt`/`latencyMs`; todo push gera `NotificationLog` (`PENDING → SENT | FAILED`, tentativas, erro, `sentAt`).
- **Cliente HTTP tipado + Zod + retry:** `src/http.ts` (timeout, retry exponencial com jitter para erro de rede/429/5xx; 4xx não repete). Usado no ntfy e no clima.
- **Consumidor do broker:** `src/broker.ts` (ack manual; mensagem inválida vai direto à DLQ; erro transitório reenfileira uma vez).
- **Idempotência:** checagem prévia por `externalId`, constraint `@unique` (captura `P2002` em corrida) e `NotificationLog` único por evento+tópico — a reentrega da mesma mensagem não duplica evento nem push.
- **ntfy:** cabeçalhos `Title`, `Priority`, `Tags`, `Click`, `Actions` (abrir mapa) e `Attach` (quando a mensagem traz `photoUrl`).
- **Frontend reativo:** `src/useDashboard.ts` — sem polling para eventos; reconexão automática com backoff exponencial (1 s → 15 s, com jitter), indicador de estado, ressincronização após reconectar, toasts/banners acessíveis.
- **Latência:** o dashboard mostra, para cada evento, o tempo entre a publicação no broker e a renderização (alvo ≤ 500 ms).
- **Responsividade/acessibilidade:** mobile-first; breakpoints em 768, 1024 e 1600 px; alvos de toque ≥ 44 px; contraste alto nos alertas; `role="alert"/"status"`, `aria-live`, foco visível, `prefers-reduced-motion`.

## Testes

```bash
cd backend && npm install && npm test
```

Cobrem: backoff exponencial, `withRetry`, validação Zod da mensagem, regras de alerta por cruzamento de limite
(determinismo do `externalId`) e o dispatcher ntfy contra um servidor HTTP simulado (cabeçalhos, retry em 5xx, sem retry em 4xx).

## Desenvolvimento local (sem Docker para a aplicação)

```bash
docker compose up postgres rabbitmq -d
cd backend && cp ../.env.example .env   # ajuste DATABASE_URL=postgresql://micromob:micromob@localhost:5432/micromob e RABBITMQ_URL=amqp://guest:guest@localhost:5672
npx prisma migrate deploy && npm run seed
npm run dev            # API + consumidor + WebSocket
npm run simulator      # em outro terminal
cd ../frontend && npm install && npm run dev   # http://localhost:5173
```
