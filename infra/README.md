# Infraestrutura Resultz CRM — um Docker por cliente

Arquitetura de hospedagem do Resultz CRM (fork do Twenty) com isolamento total
por cliente: cada colégio roda sua própria pilha Docker (app, worker, Postgres,
Redis), sem multi-tenancy compartilhado.

## Visão geral

```
Internet
   |
Cloudflare (DNS, SSL, WAF, Tunnel, R2, Turnstile, Access)
   |
   |  Cloudflare Tunnel (saída apenas — nenhuma porta aberta no VPS)
   v
VPS São Paulo (Hostinger KVM) rodando Coolify (painel web)
   ├── Stack "crm"       -> crm.resultz.com.br       (Twenty padrão, ambiente de referência)
   ├── Stack "colegio-a" -> colegioa.resultz.com.br  (Postgres + Redis próprios)
   ├── Stack "colegio-b" -> colegiob.resultz.com.br  (Postgres + Redis próprios)
   └── ...
```

- **Isolamento**: cada cliente tem banco, Redis, chave de criptografia, bucket
  R2 e tunnel Cloudflare próprios. Um vazamento em um cliente não alcança outro.
- **Sem portas abertas**: o `cloudflared` dentro de cada stack cria conexão de
  saída até a Cloudflare; o firewall do VPS fica fechado para entrada.
- **Setup rápido**: novo cliente = duplicar a stack no Coolify, colar o
  `docker-compose.yml` deste diretório e preencher o `.env`.

## Papel de cada serviço

| Serviço | Função | Custo |
|---|---|---|
| Cloudflare DNS/SSL/WAF | Domínio resultz.com.br, certificados, proteção DDoS | Grátis |
| Cloudflare Tunnel | Ingresso seguro sem expor o VPS | Grátis |
| Cloudflare R2 | Anexos do CRM (S3) e backups diários do Postgres | 10 GB grátis |
| Cloudflare Turnstile | Captcha na tela de login | Grátis |
| Cloudflare Access | Proteger o painel do Coolify com login por e-mail | Grátis (até 50 usuários) |
| Hostinger VPS KVM (São Paulo) | Roda os containers | ~R$ 30–60/mês |
| Coolify | Painel web para gerenciar as stacks (sem linha de comando) | Grátis (self-hosted) |
| Resend ou Brevo | SMTP para convites e redefinição de senha | Grátis no volume inicial |

## Arquivos

- `cliente-template/docker-compose.yml` — pilha completa de um cliente
  (server, worker, Postgres 16, Redis, cloudflared, backup opcional para R2).
- `cliente-template/.env.example` — todas as variáveis comentadas em português.
- `.github/workflows/build-resultz-image.yml` — gera a imagem do fork pelo
  painel do GitHub (aba Actions > Run workflow) e publica em
  `ghcr.io/lucasddmoraes/resultzcrm`.

## Fases

1. **Fase 1 — referência**: subir o Twenty padrão (`twentycrm/twenty:latest`)
   em `crm.resultz.com.br` para explorar o produto e mapear os ajustes para
   colégios.
2. **Fase 2 — fork**: desenvolver as customizações neste repositório, gerar a
   imagem pelo GitHub Actions e trocar `TWENTY_IMAGE` nas stacks.
3. **Fase 3 — escala**: um novo colégio = nova stack no Coolify (10–15 min).
   Quando o VPS encher (~3 clientes por 8 GB de RAM), criar outro VPS com
   Coolify ou plugar novos servidores no mesmo painel.

## Regras de segurança

- `ENCRYPTION_KEY` é única por cliente e nunca muda depois de criada.
- `PG_DATABASE_PASSWORD` forte e única por cliente, sem caracteres especiais.
- Depois de criar o workspace do cliente, definir `IS_SIGN_UP_DISABLED=true`.
- Backups: ativar o profile `backup` (variável `COMPOSE_PROFILES=backup`) com
  bucket R2 dedicado; testar restauração antes de entregar ao cliente.
- Painel do Coolify sempre atrás de Cloudflare Access (nunca IP:porta aberto).
