# Nginx Proxy Manager — Homelab Stack

> Stack Docker para proxy reverso com painel web, otimizada para rodar em hardware modesto de homelab.

---

## O que é isso

Este repositório contém a configuração completa para subir o **Nginx Proxy Manager (NPM)** via Docker Compose em um servidor doméstico. O NPM é um proxy reverso com interface web que permite expor serviços internos da rede para a internet — ou apenas organizar o tráfego interno — sem precisar escrever configurações Nginx manualmente. Ele também cuida automaticamente da emissão e renovação de certificados SSL via **Let's Encrypt**.

A stack foi construída para um homelab rodando em hardware modesto com recursos de CPU, RAM e armazenamento limitados. Cada decisão de configuração leva esse cenário em conta.

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
               │ rede interna (proxy_net)
               ▼
┌──────────────────────────────┐
│          MariaDB             │  ← banco de dados do NPM
└──────────────────────────────┘
```

O NPM recebe todo o tráfego externo e decide para onde encaminhar cada requisição com base no domínio (host-based routing). O MariaDB armazena as configurações de hosts, certificados e usuários. Os dois containers se comunicam por uma rede bridge isolada (`proxy_net`), sem expor o banco para fora do host.

---

## Engenharia das decisões

### Limites de recursos (`deploy.resources`)

O Docker, por padrão, não impõe nenhum limite de CPU ou memória nos containers — eles podem consumir tudo que o host tiver disponível. Em um servidor com RAM limitada compartilhada com o sistema operacional, isso é um problema.

Os limites definidos no compose reservam espaço para o SO e evitam que um pico de tráfego derrube o servidor inteiro:

| Container | CPU limit | RAM limit |
|-----------|-----------|-----------|
| NPM | 1,5 cores | 256 MB |
| MariaDB | 0,5 core | 512 MB |

### MariaDB LTS ao invés de `latest`

A tag `mariadb:lts` garante uma versão com suporte longo e comportamento previsível. A tag `latest` pode trazer mudanças de versão major sem aviso, quebrando a inicialização do banco ou exigindo migrações manuais.

### Otimizações para HDD (`mariadb-custom.cnf`)

O MariaDB por padrão assume que está rodando em SSD e tem RAM abundante. As configurações abaixo corrigem esse comportamento para hardware com armazenamento mecânico:

**`innodb_buffer_pool_size = 96M`**
O buffer pool é o cache principal do InnoDB. O NPM gera um banco pequeno (apenas configurações e metadados), então 96 MB é mais que suficiente e libera RAM para o sistema.

**`innodb_io_capacity = 100` / `innodb_io_capacity_max = 200`**
Controlam quantas operações de I/O por segundo o InnoDB executa em background. O padrão (200/2000) é calibrado para SSD. Em HDD isso causa rajadas de escrita que bloqueiam o disco. Reduzindo para 100/200 as escritas são distribuídas de forma mais suave.

**`innodb_flush_log_at_trx_commit = 2`**
Com o valor padrão `1`, o MariaDB sincroniza o log de transações com o disco a cada commit — operação cara em HDD. O valor `2` sincroniza apenas uma vez por segundo. Para dados de configuração de proxy em homelab, essa troca é perfeitamente aceitável.

**`innodb_flush_method = O_DIRECT`**
Faz o InnoDB escrever direto no disco ignorando o cache de página do kernel para dados. Evita double buffering, economizando RAM.

**`innodb_use_native_aio = 0`**
O AIO nativo do Linux pode ser instável em algumas combinações de kernel + ext4 + HDD. Desativá-lo usa o fallback de threads simuladas, que é mais estável nesse cenário.

### Healthcheck no banco

O NPM só inicia depois que o MariaDB está realmente pronto para aceitar conexões (`condition: service_healthy`). Sem isso, o NPM tenta conectar durante a inicialização do banco, falha, e o container entra em loop de restart.

### Rede isolada (`proxy_net`)

O banco de dados não expõe nenhuma porta para o host — só é acessível pelo NPM dentro da rede bridge `proxy_net`. Mesmo que alguém acesse o servidor na rede local, não consegue conectar diretamente ao MariaDB.

### Variáveis de ambiente via `.env`

As senhas ficam em um arquivo `.env` separado do `compose.yaml`. Isso permite versionar o compose no Git sem expor credenciais. O `.gitignore` já está configurado para ignorar o `.env` e os volumes gerados em runtime.

---

## Estrutura do repositório

```
.
├── compose.yaml          # definição dos serviços
├── .env                  # credenciais (não versionar)
├── .env.example          # modelo sem valores reais (pode versionar)
├── .gitignore
├── mariadb-custom.cnf    # tunning do MariaDB para HDD + pouca RAM
├── data/                 # volume do NPM (gerado em runtime)
├── letsencrypt/          # certificados SSL (gerado em runtime)
└── mysql/                # dados do MariaDB (gerado em runtime)
```

---

## Como usar

**1. Clone ou copie os arquivos para o servidor**

**2. Configure as credenciais**

```bash
DB_NAME=npm
DB_USER=npm
DB_PASSWORD=Sua_Senha_Forte_Aqui
DB_ROOT_PASSWORD=Sua_Senha_Forte_Aqui
```

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

## Requisitos

- Docker Engine 24+
- Docker Compose v2 (plugin, não o `docker-compose` legado)
- Portas 80, 443 e 81 livres no host

---

## Notas de segurança

- O arquivo `.env` **nunca** deve ser commitado no Git.
- A porta `:81` do painel admin deve ser bloqueada no firewall para acesso externo — use apenas na rede local ou via VPN.
- Os certificados Let's Encrypt ficam no volume `./letsencrypt` e são renovados automaticamente pelo NPM.

---

*Homelab pessoal — hardware modesto, configuração honesta.*