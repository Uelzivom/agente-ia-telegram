# Agente de IA privado no Telegram, com cérebro FreeLLMAPI — na VPS Oracle

Guia passo a passo para instalar um agente de IA só seu no Telegram, rodando 24/7 na VPS gratuita da Oracle: o agente **Hermes** (Nous Research), com o cérebro **FreeLLMAPI** — um roteador que junta os níveis gratuitos de vários provedores de IA atrás de um único endpoint compatível com OpenAI.

No final você fala com seu agente no Telegram, ele responde com modelos gratuitos (Gemini, Groq e outros), com memória persistente — e nada disso fica exposto na internet.

> **Conferido na VPS em 24/11/2026:** as versões, redes, portas e volumes abaixo são os reais da instalação (os 9 containers estavam `Up` há 12 h, após o reboot de 23/11). Um detalhe corrigido na revisão: os três serviços **não** compartilham uma rede Docker — cada um tem a sua (ver a arquitetura).
>
> Pré-requisitos: a VPS do [projeto 1](../vps-oracle-gratis) com Docker, e a VPN do [projeto 2](../projeto-02-vpn-wireguard) funcionando. Você também vai precisar de um bot no Telegram (grátis, via @BotFather) e de chaves gratuitas de pelo menos um provedor (ex.: Google AI Studio, Groq).

---

## A arquitetura (real)

```
Telegram (você)
   │  (polling — o agente busca as mensagens; nada exposto)
   ▼
┌──────────────────────┐   base_url http://freellmapi-   ┌──────────────────────┐
│ Hermes gateway       │ ──────────────────────────────► │ FreeLLMAPI :3001     │
│ nousresearch/        │   freellmapi-1:3001/v1          │ roteador de modelos  │
│ hermes-agent:latest  │   provider: custom, model: auto │ 5 provedores grátis  │
│ sem portas publicadas│                                 │ chaves criptografadas│
│ rede ai-net          │   ◄── a rede ai-net é          │ redes ai-net         │
│ 172.21.0.3           │       compartilhada: é nela     │ 172.21.0.2 e         │
└──────────────────────┘       que o nome do container   │ freellmapi_default   │
                           resolve (172.21.0.2)      │ 172.22.0.2         │
                                                      └──────────────────────┘
        ▲                                                          ▲
        │                                                   172.19.0.1:3001
        │                                          (painel publicado no endereço
                                              do host na rede da VPN — só via túnel)
```

| Serviço | Imagem | Rede Docker | Portas publicadas |
|---|---|---|---|
| `hermes` | `nousresearch/hermes-agent:latest` | `ai-net` (`172.21.0.0/16`, IP `172.21.0.3`) + `bridge` | nenhuma |
| `freellmapi-freellmapi-1` | `ghcr.io/tashfeenahmed/freellmapi:latest` | `ai-net` (IP `172.21.0.2`) + `freellmapi_default` (`172.22.0.0/16`, IP `172.22.0.2`) | `127.0.0.1:3001` e `172.19.0.1:3001` |
| `wg-easy` (projeto 2) | `ghcr.io/wg-easy/wg-easy:13` | `wg-easy-oracle_default` (`172.19.0.0/16`, IP `172.19.0.2`) | `51820/udp`, `51821/tcp` |

O FreeLLMAPI está ligado a **duas redes ao mesmo tempo**: a rede própria do seu compose (`freellmapi_default`) e a `ai-net`, que ele compartilha com o agente. É por isso que o nome `freellmapi-freellmapi-1` resolve de dentro do agente (para `172.21.0.2`): o DNS embutido do Docker resolve nomes de containers dentro da mesma rede, e a `ai-net` é a rede em comum.

## ⭐ O destaque: o FreeLLMAPI só existe dentro da sua VPN

O FreeLLMAPI, por padrão, só tem dois jeitos de ser acessado: só no `localhost` do servidor, ou aberto para a rede local com `HOST_BIND=0.0.0.0` — ou seja, para quem alcançar o servidor.

Aqui ele está de um terceiro jeito, que não existe no padrão: **a porta 3001 é publicada no `172.19.0.1`, o endereço do host na rede Docker da VPN (`wg-easy-oracle_default`)**. Na prática:

- **Sem VPN, o painel não existe.** O IP público da VPS não responde na porta 3001 — não há nada para atacar.
- **Com a VPN conectada, o painel abre** no celular ou no computador, de qualquer lugar: o tráfego vem pelo túnel até o wg-easy e dali ao `172.19.0.1:3001`.
- As chaves dos provedores ficam **criptografadas em repouso** no FreeLLMAPI, com uma chave de criptografia gerada na instalação.

Ou seja: o cérebro da IA é privado por desenho, não por senha na porta. A VPN que você instalou no projeto 2 vira a única porta de entrada para gerenciar a IA.

---

## Passo 1 — Criar o bot no Telegram

1. No Telegram, fale com o **@BotFather** e envie `/newbot`.
2. Escolha um nome de exibição (ex.: "Da Vinci Agent") e um username terminado em `bot`.
3. Copie o **token** que o BotFather mostra. Guarde — ele é a senha do bot.
4. Descubra o seu **Telegram user ID** (ex.: com o bot @userinfobot). Ele será a allowlist: só você fala com o agente.

---

## Passo 2 — Instalar o FreeLLMAPI (o cérebro)

O FreeLLMAPI é o projeto [tashfeenahmed/freellmapi](https://github.com/tashfeenahmed/freellmapi) (imagem `ghcr.io/tashfeenahmed/freellmapi:latest`). Aqui ele foi instalado a partir de um clone completo do repositório:

```bash
git clone https://github.com/tashfeenahmed/freellmapi.git
cd freellmapi
docker compose up -d
```

(O projeto também oferece um instalador de uma linha, `curl -fsSL https://freellmapi.co/install.sh | bash`, que cria `~/freellmapi` e gera a chave de criptografia.)

O `docker-compose.yml` do projeto publica a porta como `"${HOST_BIND:-127.0.0.1}:${PORT:-3001}:3001"`, usa o `.env` para a chave de criptografia, persiste os dados no volume `freellmapi-data` e tem healthcheck em `/api/ping`. Existe ainda um serviço `watchtower` (perfil `autoupdate`, opcional) para auto-atualização — **não está em execução** nesta instalação.

Depois de subir:

1. Abra o painel (veja o passo 5 para o acesso via VPN) e crie sua conta.
2. Na página **Keys**, adicione as chaves dos provedores gratuitos que você usa (ex.: Google AI Studio, Groq, NVIDIA NIM, OpenRouter, Z.ai).
3. Ordene a **Fallback Chain** (a ordem de tentativa quando um provedor limita).
4. Copie a **unified API key** do topo da página Keys — é ela que o agente vai usar. Trate como senha.

Para atualizar depois: `docker compose pull && docker compose up -d` na pasta do projeto — o `.env` (e a chave de criptografia) é preservado.

---

## Passo 3 — Instalar o agente Hermes

O agente é o [Hermes Agent, da Nous Research](https://github.com/NousResearch/hermes-agent) (imagem `nousresearch/hermes-agent:latest`). Ele roda como um gateway persistente: conecta no Telegram, usa o modelo como motor e guarda memória persistente.

Crie o diretório de dados e rode o assistente de configuração (ele é interativo — rode no terminal, não em background):

```bash
mkdir -p ~/.hermes
docker run -it --rm \
  -v ~/.hermes:/opt/data \
  nousresearch/hermes-agent setup
```

O assistente pergunta as chaves e escreve tudo em `~/.hermes/.env`. Depois, suba o gateway em background, **sem publicar nenhuma porta**:

```bash
docker run -d \
  --name hermes \
  --restart unless-stopped \
  -v ~/.hermes:/opt/data \
  nousresearch/hermes-agent gateway run
```

Tudo o que o agente é vive em `~/.hermes` (montado em `/opt/data` no container): `.env` (segredos), `config.yaml` (configuração), `SOUL.md` (personalidade), `sessions/`, `memories/`, `skills/`, `cron/` e `logs/`. A imagem é stateless — atualizar é só puxar a imagem nova, os dados ficam.

> Dicas de segurança: rode o container com um usuário dedicado sem sudo (aqui o usuário `hermes`, com os dados em `/home/hermes/.hermes`), e não publique portas do agente. O Telegram fala com ele por polling — ele busca as mensagens, ninguém conecta nele. O token do Telegram não aparece nas variáveis de ambiente do container: a configuração vem do volume `/opt/data`.

---

## Passo 4 — Apontar o agente para o FreeLLMAPI

No `config.yaml` do Hermes (em `~/.hermes`), o modelo vira um provedor `custom` apontando para o FreeLLMAPI:

```yaml
model:
  default: auto
  provider: custom
  base_url: http://freellmapi-freellmapi-1:3001/v1
```

A `api_key` é a unified API key copiada no passo 2 (ela vai para o `.env` do Hermes, nunca para o guia nem para o GitHub).

Reinicie o gateway e confira o log:

```bash
docker restart hermes
docker logs hermes --tail 30
```

O log deve mostrar `[Telegram] Connected to Telegram (polling mode)`, sem erros de provedor. Teste real: mande uma mensagem ao bot — a resposta vem do modelo `auto` do roteador (aqui, Gemini respondeu em ~2 s e um modelo da Groq em ~3 s, com chamadas de ferramenta funcionando).

---

## Passo 5 — Acessar o painel do FreeLLMAPI pela VPN (o destaque)

1. Conecte o WireGuard no celular ou no computador (projeto 2).
2. Abra no navegador: `http://172.19.0.1:3001`.
3. Gerencie chaves, ordem de fallback e uso por provedor — tudo por dentro do túnel.

Sem a VPN conectada, esse endereço não responde. É assim que o cérebro fica privado.

## Personalidade e nome

- A personalidade do agente é a primeira linha do `SOUL.md` (ex.: "You are Da Vinci Agent, built by Nous Research.").
- O nome de exibição no Telegram pode ser mudado depois pela API do bot (`setMyName`) sem mexer no BotFather — o username do bot não muda.
- Depois de editar `SOUL.md` ou `config.yaml`: `docker restart hermes`.

### Mensagens do sistema ("Hermes is shutting down"...)

As mensagens automáticas do agente (desligamento, restart, "estou trabalhando") não vêm do `SOUL.md` nem do `config.yaml` — são textos fixos nos arquivos de tradução dentro da imagem, em `/opt/hermes/locales/*.yaml`. Não há opção de configuração para trocá-las. A solução usada aqui: copiar essa pasta para fora do container, trocar "Hermes" por "Da Vinci" nos textos e montar a cópia corrigida por cima, só leitura:

```bash
docker cp hermes:/opt/hermes/locales /home/hermes/locales
sed -i 's/Hermes/Da Vinci/g' /home/hermes/locales/*.yaml
```

Depois recrie o container com os mesmos parâmetros de antes, acrescentando o mount `-v /home/hermes/locales:/opt/hermes/locales:ro` (o comando exato da recriação está guardado em `/home/hermes/run-hermes.sh` na VPS, para reutilizar nas próximas atualizações). O banner "Hermes Gateway Starting..." que aparece nos logs do container continua igual — ele é fixo no código do gateway e não é enviado ao Telegram.

## Backup

Copie `~/.hermes` (dados do agente) e `~/freellmapi` (dados do FreeLLMAPI, incluindo o `.env` com a chave de criptografia) para fora da VPS. Sem o `.env` do FreeLLMAPI, as chaves dos provedores guardadas não podem ser lidas de volta.

## Solução de problemas

- **Gateway travado na tela inicial de configuração:** o assistente de setup é interativo — rode o `setup` com `-it` no terminal, não em background. O container de setup some sozinho depois (`--rm`).
- **Primeira tentativa de SSH cai com `Connection closed`:** comportamento conhecido do Cloud Shell — tente de novo, a segunda entra.
- **Bot não responde:** confira `docker logs hermes` — token do Telegram e allowlist (seu user ID) são os culpados mais comuns. Se o log mostrar avisos repetidos de falha ao alcançar `api.telegram.org` (o agente tentando e repetindo), é a conectividade com o Telegram no momento — o container segue tentando sozinho; teste o bot novamente depois de alguns minutos.
- **Painel do FreeLLMAPI não abre:** confira se a VPN está conectada e se o container está `Up` (`docker ps`).

---

## Próximos passos

- Dar uma primeira missão ao agente (ex.: briefing matinal ou monitoramento da VPS).
- Publicar os guias dos projetos 1 e 2, e os sites, mantendo o perfil alimentado.

---

## Confirmado na VPS em 24/11/2026

- [x] `hermes`: imagem `nousresearch/hermes-agent:latest`, sem portas publicadas, bind `/home/hermes/.hermes → /opt/data`, rede `ai-net` (`172.21.0.3`).
- [x] `freellmapi-freellmapi-1`: imagem `ghcr.io/tashfeenahmed/freellmapi:latest`, saudável, portas `127.0.0.1:3001` e `172.19.0.1:3001`, rede `freellmapi_default` (`172.22.0.2`); `~/freellmapi` é um clone completo do repositório; watchtower não está em execução.
- [x] Três redes Docker no conjunto, com a `ai-net` compartilhada entre agente e FreeLLMAPI (é nela que o nome `freellmapi-freellmapi-1` resolve, para `172.21.0.2` — confirmado com `docker exec hermes getent hosts`).
- [x] Logs do FreeLLMAPI saudáveis (4 chaves verificadas, catálogo sincronizado).
- [x] `config.yaml` do agente confirmado: `provider: "custom"`, `default: "auto"`, `base_url: "http://freellmapi-freellmapi-1:3001/v1"`.

## Licença

MIT — veja `LICENSE`.
