# Limitados Painel

> Painel web completo para gerenciamento de servidores Counter-Strike 2, com suporte a múltiplos modos de jogo, controle de plugins, gravação de demos CSTV e muito mais.

Repositório oficial:
https://github.com/davilouback/limitadospainel

---

## Índice

- [Funcionalidades](#funcionalidades)
- [Arquitetura](#arquitetura)
- [Stack](#stack)
- [Instalação Local](#instalação-local)
- [Deploy em VPS](#deploy-em-vps)
- [Configuração do Agente CS2](#configuração-do-agente-cs2)
- [Configurando o CS2](#configurando-o-cs2)
- [SourceMod e Metamod](#sourcemod-e-metamod)
- [Sistema de Modos de Jogo](#sistema-de-modos-de-jogo)
- [CSTV e Demos](#cstv-e-demos)
- [Docker](#docker)
- [Variáveis de Ambiente](#variáveis-de-ambiente)
- [Referência de Portas e Firewall](#referência-de-portas-e-firewall)
- [Guia para IA / Codex](#guia-para-ia--codex)
- [Troubleshooting](#troubleshooting)

---

## Funcionalidades

| Módulo | O que faz |
|--------|-----------|
| **Dashboard** | Visão geral: servidores online, players ativos, timeline de atividades |
| **Servidor** | Start / Stop / Restart / Update via agente na VPS |
| **Modos de Jogo** | Troca dinâmica: move plugins, aplica configs, muda game_type/game_mode, reinicia |
| **Players** | Lista em tempo real, Kick / Ban / Mute via RCON |
| **Admins** | Gerenciar admins SourceMod via RCON (`sm_addadmin`, `sm_removeadmin`) |
| **Mapas** | Troca de mapa via RCON, suporte a mapas workshop |
| **Plugins** | Lista de plugins SourceMod ativos via `sm plugins list` |
| **CSTV / Demos** | Gravar, pausar, parar, listar, baixar, renomear e excluir demos |
| **Console** | RCON direto com painel de comandos rápidos (~35 botões) |
| **Logs** | Logs do servidor em tempo real com polling |
| **Usuários** | Gerenciar usuários do painel (admin/operador) |

---

## Arquitetura

```txt
┌─────────────────────────────────────────────────────────────┐
│                     Limitados Painel                        │
│                                                             │
│  ┌──────────────────┐     ┌─────────────────────────────┐  │
│  │  Frontend         │────▶│  API Backend (Express)      │  │
│  │  React + Vite     │     │  JWT Auth + PostgreSQL      │  │
│  │  Tailwind CSS     │     │  Proxy para o Agente        │  │
│  └──────────────────┘     └────────────┬────────────────┘  │
└────────────────────────────────────────┼────────────────────┘
                                          │ HTTP (Bearer token)
                          ┌───────────────▼───────────────────┐
                          │       VPS do Servidor CS2          │
                          │                                    │
                          │  ┌─────────────────────────────┐  │
                          │  │  cs2_agent.py (Python)       │  │
                          │  │  • Controla arquivos/plugins │  │
                          │  │  • RCON para o CS2          │  │
                          │  │  • Gerencia demos           │  │
                          │  └──────────────┬──────────────┘  │
                          │                 │ RCON TCP         │
                          │  ┌─────────────▼──────────────┐   │
                          │  │  CS2 Server + SourceMod     │   │
                          │  │  + Metamod + Plugins        │   │
                          │  └────────────────────────────┘   │
                          └────────────────────────────────────┘
Princípio fundamental: o backend do painel nunca executa comandos localmente. Tudo é encaminhado ao agente Python rodando na VPS via HTTP, com autenticação por Bearer token por servidor.

Stack
Camada	Tecnologia
Frontend	React 19, Vite, Tailwind CSS, TanStack Query, wouter
Backend	Node.js 20+, Express 5, JWT, bcrypt
Banco de dados	PostgreSQL + Drizzle ORM
Validação	Zod v4, drizzle-zod
Agente VPS	Python 3.8+ (sem dependências externas, opcional: psutil)
Monorepo	pnpm workspaces
Instalação Local
Pré-requisitos
Node.js 20+
pnpm (npm install -g pnpm)
PostgreSQL 14+
Passo a passo
# 1. Clonar o repositório
git clone https://github.com/davilouback/limitadospainel.git
cd limitadospainel

# 2. Criar arquivo .env
cp .env.example .env
# Edite .env com suas configurações

# 3. Instalar dependências
pnpm install

# 4. Criar tabelas no banco de dados
pnpm --filter @workspace/db run push

# 5. Iniciar o backend (terminal 1)
pnpm --filter @workspace/api-server run dev

# 6. Iniciar o frontend (terminal 2)
pnpm --filter @workspace/cs2-panel run dev

O painel estará disponível em http://localhost:19623.

Login padrão: admin / admin

Deploy em VPS
Opção 1 — Script automático (recomendado)
# Na VPS (Ubuntu 22.04 / Debian 11+), como root:
wget https://raw.githubusercontent.com/davilouback/limitadospainel/main/scripts/vps-install.sh
chmod +x vps-install.sh
REPO_URL=https://github.com/davilouback/limitadospainel.git ./vps-install.sh
Opção 2 — Manual
# 1. Instalar Node.js 20
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo bash -
sudo apt-get install -y nodejs
sudo npm install -g pnpm

# 2. Instalar PostgreSQL
sudo apt-get install -y postgresql
sudo -u postgres psql -c "CREATE USER limitados WITH PASSWORD 'SENHA';"
sudo -u postgres psql -c "CREATE DATABASE limitados_painel OWNER limitados;"

# 3. Clonar e instalar
git clone https://github.com/davilouback/limitadospainel.git /opt/limitados-painel
cd /opt/limitados-painel
cp .env.example .env
# Edite .env com DATABASE_URL e SESSION_SECRET
pnpm install
pnpm --filter @workspace/db run push
pnpm run build
CSTV e Demos
Comandos CS2 usados
tv_enable 1
tv_autorecord 1
tv_delay 30
tv_record nome_demo
tv_stoprecord
tv_status
Funcionalidades do painel
Iniciar/Pausar/Retomar/Parar gravação
Status ao vivo com timer e tamanho
Lista de demos com busca e ordenação
Download direto pelo navegador
Renomear e excluir demos
Limite de armazenamento configurável
Auto-delete de demos antigas
Referência de Portas e Firewall
Porta	Protocolo	Serviço
80/443	TCP	Painel Web
8080	TCP	API Backend
27015	UDP+TCP	CS2 Server
27020	UDP	CSTV
7777	TCP	Agente CS2
Login Padrão
Usuário	Senha	Role
admin	admin	Administrador
operator	admin	Operador

⚠️ Mude as senhas padrão imediatamente após o primeiro acesso.

Licença

Projeto privado. Todos os direitos reservados.
