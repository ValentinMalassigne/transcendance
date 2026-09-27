# Transcendance

A full-stack, multiplayer Pong web platform. Users sign up, chat with each other in real time, create or join Pong rooms with a shareable code, and play live matches where the game state is simulated server-side and streamed to every player over WebSockets.

Built with a **React + Vite** frontend, a **Django + Channels** backend, and orchestrated with **Docker Compose** across six services, with strict security requirements baked in: HTTPS only (TLS 1.3), HashiCorp Vault-managed secrets, JWT authentication, a ModSecurity WAF in front of Nginx, and sanitized user input.

## Table of contents

- [Features](#features)
- [Tech stack](#tech-stack)
- [Architecture](#architecture)
- [Project structure](#project-structure)
- [Getting started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Configuration](#configuration)
  - [Build and run](#build-and-run)
  - [Make targets](#make-targets)
- [Services](#services)
- [REST API](#rest-api)
- [WebSocket protocol](#websocket-protocol)
  - [Pong](#pong)
  - [Chat](#chat)
- [Security](#security)
- [Internationalization](#internationalization)

## Features

**Gameplay**
- Create a Pong room and get a unique 6-character room code to share
- Join an existing room by code, or spectate
- Supports 2-player matches and solo play against a bot
- Server-authoritative game loop: the ball physics run on the backend at a fixed tick rate, so every connected client sees the same state
- Pause and restart a live game
- Player stats (score, aces, rank) stored on the user profile

**Social**
- Real-time chat over WebSockets, including private one-to-one conversations
- User profiles with avatar and bio
- Social page to browse and interact with other players
- Floating chat widget available on any page while logged in

**Accounts**
- Sign up / log in with JWT authentication (short-lived access token + refresh token)
- Profile and settings pages (username, password, account deletion)
- Protected routes: everything except login, signup and the 404 page requires a valid session

**Platform**
- Full HTTPS deployment with self-signed certificates generated on first build
- Secrets (database credentials, Django secret key) stored in HashiCorp Vault, never hardcoded
- ModSecurity web application firewall compiled into the Nginx image
- English / French / Spanish UI with a language switcher
- Cookie consent banner for GDPR compliance

## Tech stack

| Layer | Technologies |
| --- | --- |
| Frontend | React 18, TypeScript, Vite, React Router, TanStack Query, Axios, i18next, DOMPurify, react-hot-toast |
| Backend | Python 3.12, Django 5, Django REST Framework, SimpleJWT, Django Channels, Daphne (ASGI) |
| Realtime | WebSockets (Django Channels), Redis (provisioned channel layer) |
| Database | PostgreSQL |
| Secrets | HashiCorp Vault 1.16 (AppRole auth, TLS) |
| Reverse proxy | Nginx 1.22 + ModSecurity (compiled from source) |
| Orchestration | Docker Compose, Make |

## Architecture

```
                      https://localhost:8000
                              |
                        +-----------+
                        |   nginx   |  TLS 1.3 + ModSecurity WAF
                        +-----------+
                        /     |     \
              static SPA   /api/  /pong-api/   /ws/  /admin/
              (React build)      |                |
                            +-----------+    +-----------+
                            |  daphne   |<---|  redis    |
                            | (django) |    | (channel  |
                            +-----------+    |   layer)  |
                                  |         +-----------+
                        +-----------+
                        |    db     |  PostgreSQL
                        +-----------+

                        +-----------+        +------------+
                        |   vault   |<------| vault-init |
                        +-----------+        +------------+
                           (secrets, unsealed and provisioned at startup)
```

Nginx terminates TLS and serves the built React bundle as static files. API calls are proxied to the Django application, which runs under Daphne so HTTP and WebSocket traffic share the same ASGI process. Redis is provisioned alongside as the channel layer backend. Vault runs in parallel, and the `vault-init` one-shot container initializes, unseals it, and provisions AppRole-based policies for the backend and frontend.

## Project structure

```
transcendance/
├── Makefile                  # build / run / clean entry points
├── docker-compose.yml        # 6 services: db, django, nginx, vault, vault-init, redis
├── .env                      # required environment file (not committed)
├── scripts/
│   ├── generate_cert.sh      # self-signed CA + TLS certificate for nginx and vault
│   └── cleaner.sh            # docker prune helper for `make fclean`
├── ssl/                      # generated certificates and private keys
├── volume/                   # bind-mounted data (postgres, vault)
└── images/
    ├── backend/              # Django project
    │   ├── Dockerfile
    │   ├── init.sh           # sources vault-provided env vars, then runs CMD
    │   ├── manage.py
    │   ├── source/           # settings, urls, asgi/wsgi, vault client helper
    │   ├── user/             # custom AppUser model, registration, JWT views
    │   ├── pong/             # rooms, game loop, websocket consumer, REST views
    │   └── chat/             # chat websocket consumer
    ├── build/                # frontend + nginx image
    │   ├── Dockerfile        # node build stage, then nginx with ModSecurity
    │   ├── frontend/         # React + Vite + TypeScript app
    │   │   └── src/
    │   │       ├── pages/        # home, login, signup, profile, settings, social, pong...
    │   │       ├── components/   # Navbar, Chat, PongGame, avatar, ProtectedRoute
    │   │       ├── context.tsx  # auth context
    │   │       └── translation/ # en / fr / es message catalogs
    │   └── nginx/            # nginx.conf, ModSecurity rules, entrypoint
    └── vault/                # vault config, startup and provisioning scripts
```

## Getting started

### Prerequisites

- Docker and Docker Compose
- GNU Make
- OpenSSL (used by the certificate script)

### Configuration

The build refuses to start without a `.env` file at the repository root. Create one with the following variables:

```dotenv
# Backend
SECRET_KEY=<django secret key>

# PostgreSQL
POSTGRES_DB=<database name>
POSTGRES_USER=<database user>
POSTGRES_PASSWORD=<database password>
DATABASE_HOST_NAME=db
DATABASE_PORT=5432

# Frontend
VITE_API_URL=https://localhost:8000

# Vault
VAULT_ADDR=<vault address, e.g. https://vault:8200>
VAULT_INIT_ADDR=<address used by init containers>
MY_VAULT_TOKEN=<app token minted into vault>
VAULT_CACERT=<path to the CA certificate>
```

The first `make build` generates the CA and TLS certificates into `ssl/` automatically via `scripts/generate_cert.sh`.

### Build and run

```bash
make            # or `make all`: generates certs, builds images, starts the stack
```

Once the stack is up, open:

```
https://localhost:8000
```

The certificate is self-signed, so your browser will warn you on first visit — accept it or import `ssl/certs/ca.crt` as a trusted CA.

### Make targets

| Target | Effect |
| --- | --- |
| `make` / `make all` | Generate certificates if missing, build all images, start the stack, and report when the site is up |
| `make build` | Run setup, generate certificates, build the Docker images |
| `make up` | Start the stack detached |
| `make stop` | Stop the containers without removing them |
| `make clean` | Stop and remove containers and volumes (`docker compose down -v`) |
| `make fclean` | Full cleanup: `clean`, prune helper script, Docker system and volume prune |
| `make logs` | Dump all compose logs to a `.logs` file |
| `make setup` | Create `volume/db_data` and verify `.env` exists |

## Services

| Service | Image | Role |
| --- | --- | --- |
| `nginx` | Custom (nginx 1.22 + ModSecurity) | TLS termination, static SPA, reverse proxy for API and WebSockets, WAF |
| `django` | Custom (python 3.12 + daphne) | REST API, JWT auth, WebSocket consumers, game loop |
| `db` | `postgres` | Application database |
| `redis` | `redis` | Provisioned as the channel layer backend for Django Channels |
| `vault` | `hashicorp/vault:1.16` | Secret storage (database credentials, Django secret key) |
| `vault-init` | `hashicorp/vault:1.16` | One-shot: initialize, unseal, mint tokens, write AppRole policies |

## REST API

All endpoints are served under `https://localhost:8000` through Nginx.

| Method | Path | Description |
| --- | --- | --- |
| `POST` | `/api/user/register` | Create a new account |
| `POST` | `/api/token/` | Obtain a JWT access/refresh pair (login) |
| `POST` | `/api/token/refresh/` | Refresh an access token |
| `GET` | `/api/` | API health check (used by the container healthcheck) |
| `POST` | `/pong-api/create-room` | Create a Pong room, returns its code and settings |
| `POST` | `/pong-api/join-room` | Look up a room by its 6-character code |
| — | `/admin/` | Django admin |

Authentication uses SimpleJWT: a 30-minute access token and a 1-day refresh token, with `IsAuthenticated` as the default permission for DRF views.

## WebSocket protocol

Both real-time channels are routed through Nginx at `/ws/` (HTTP/1.1 upgrade) to the same Daphne process, and use per-room channel layer groups. Note: the Redis service and `channels-redis` are provisioned in the stack, but `settings.py` currently configures the in-memory channel layer, so websocket groups are per-process.

### Pong

- Endpoint: `ws/pong/<room_code>/`
- On connect, the consumer registers the player in the channel group and starts/joins the server-side game loop
- Clients send paddle updates (`update_paddle`), pause and restart commands; the server broadcasts `game_state` frames containing ball position and direction, both paddle positions, and a timestamp
- The simulation runs at a fixed tick rate on a 600x400 map (20px ball, 100px paddles); when the room has fewer than 2 human players, the loop drives the right paddle as a bot
- Room state (paddle positions, pause/restart flags, players) is persisted in the `PongRoom` model so the loop survives reconnects

### Chat

- Endpoint: `ws/chat/<room_name>/`
- Supports public rooms and private two-user conversations (`create_private_chat`)
- Messages are broadcast to the room group and rendered on the frontend

## Security

- **HTTPS everywhere**: the site is only served over TLS 1.3 with a self-signed certificate; the same PKI secures Vault's API
- **Secrets management**: no credentials in the codebase — Django pulls its database credentials and secret key from Vault at startup, using AppRole policies scoped to `secret/data/django/*`
- **JWT authentication**: stateless tokens with short expiry, no server-side session for the API
- **WAF**: ModSecurity is compiled into the Nginx image and enabled on the server block, filtering malicious requests before they reach Django
- **Input sanitization**: user-supplied content rendered on the frontend is sanitized with DOMPurify (e.g., the social page) and validated with DRF serializers on the backend
- **Hardened headers**: `server_tokens off`, forwarded headers set on every proxy location

## Internationalization

The UI ships with English, French, and Spanish translations (`src/translation/{en,fr,es}.json`) wired through i18next. The language can be switched from the settings page via the `LanguageSwitcher` component. Users can also manage cookie preferences from the settings page.
