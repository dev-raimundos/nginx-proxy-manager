# Nginx Proxy Manager — Homelab Stack

> Stack Docker para proxy reverso com painel web, conectada a um banco PostgreSQL externo.

---

## O que é isso

Este repositório contém a configuração para subir o **Nginx Proxy Manager (NPM)** via Docker Compose. O NPM é um proxy reverso com interface web que permite expor serviços internos da rede para a internet — ou apenas organizar o tráfego interno — sem precisar escrever configurações Nginx manualmente. Ele também cuida automaticamente da emissão e renovação de certificados SSL via **Let's Encrypt**.

O banco de dados não é gerenciado por esta stack: o NPM se conecta a uma instância **PostgreSQL já existente no servidor**.

---

## Arquitetura

```
Internet
   │
   │  :80 (HTTP)
   │  :443 (HTTPS)
   ▼
┌──────────────────────────────┐
│     nginx-proxy-manager      │  ← proxy reverso + painel web (:81)
│                              │
│  Let's Encrypt (automático)  │
└──────────────┬───────────────┘
               │ host.docker.internal
               ▼
┌──────────────────────────────┐
│     PostgreSQL (no host)     │  ← banco de dados externo
└──────────────────────────────┘
```

O NPM recebe todo o tráfego externo e decide para onde encaminhar cada requisição com base no domínio (host-based routing). O PostgreSQL que roda diretamente no servidor armazena as configurações de hosts, certificados e usuários.

---

## Pré-requisitos

- Docker Engine 24+
- Docker Compose v2 (plugin, não o `docker-compose` legado)
- Portas 80, 443 e 81 livres no host
- Instância PostgreSQL acessível no servidor com um banco e usuário criados para o NPM

---

## Como usar

**1. Clone ou copie os arquivos para o servidor**

**2. Crie o arquivo `.env` a partir do exemplo**

```bash
cp .env.example .env
```

Edite o `.env` com as credenciais reais:

```env
DB_HOST=host.docker.internal   # conecta ao host; ou use o IP/hostname do servidor
DB_PORT=5432
DB_USER=npm
DB_PASSWORD=Sua_Senha_Forte_Aqui
DB_NAME=npm
```

> `host.docker.internal` é adicionado automaticamente pelo `extra_hosts` no compose e resolve para o IP do host em servidores Linux com Docker Engine.

**3. Suba a stack**

```bash
docker compose up -d
```

**4. Acesse o painel**

Abra `http://IP-DO-SERVIDOR:81` no navegador.

Credenciais padrão do NPM:
- E-mail: `admin@example.com`
- Senha: `changeme`

> Troque a senha imediatamente após o primeiro login.

**5. Acompanhe os logs**

```bash
docker compose logs -f
```

---

## Estrutura do repositório

```
.
├── compose.yaml          # definição do serviço
├── .env                  # credenciais (não versionar)
├── .env.example          # modelo sem valores reais (pode versionar)
├── .gitignore
├── data/                 # volume do NPM (gerado em runtime)
└── letsencrypt/          # certificados SSL (gerado em runtime)
```

---

## Variáveis de ambiente

| Variável | Descrição | Padrão |
|----------|-----------|--------|
| `DB_HOST` | Host do PostgreSQL | — |
| `DB_PORT` | Porta do PostgreSQL | `5432` |
| `DB_USER` | Usuário do banco | — |
| `DB_PASSWORD` | Senha do banco | — |
| `DB_NAME` | Nome do banco | — |

---

## Notas de segurança

- O arquivo `.env` **nunca** deve ser commitado no Git.
- A porta `:81` do painel admin deve ser bloqueada no firewall para acesso externo — use apenas na rede local ou via VPN.
- Os certificados Let's Encrypt ficam no volume `./letsencrypt` e são renovados automaticamente pelo NPM.
- O PostgreSQL não precisa ter a porta exposta externamente — o container acessa via `host.docker.internal`, que é resolvido internamente pelo Docker.

---

*Homelab pessoal — sem compromisso, sem frescura.*
