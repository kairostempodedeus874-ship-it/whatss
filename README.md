# Servidor de WhatsApp — Kairós (v5)

Este é o "motor" que mantém o WhatsApp da empresa conectado 24 horas por dia.
Ele recebe as mensagens dos clientes, responde pela IA, envia as respostas dos
atendentes e dispara as cobranças e confirmações de pagamento.

> ⚠️ **Ele precisa ficar ligado o tempo todo e precisa de um disco próprio.**
> Hospedagens que "dormem" por inatividade ou que apagam o disco a cada deploy
> (Vercel, Netlify, Cloudflare, funções serverless) **não servem** — a sessão do
> WhatsApp morre e você precisa escanear o QR Code de novo toda hora.

---

## 1. Qual hospedagem usar

| Plataforma | Custo mensal | Dorme? | Disco | Veredito |
|---|---|---|---|---|
| **Fly.io** (recomendado) | ~US$ 3,84 (≈ R$ 21) | Não | Sim (US$ 0,15/GB) | Mais barato com disco e máquina sempre ligada |
| Render (plano Starter) | ~US$ 7,25 (≈ R$ 39) | Não | Sim (US$ 0,25/GB) | Mais simples de configurar pela interface |
| Railway | ~US$ 5 + uso | Não (no plano pago) | Sim | Foi onde deu instabilidade — não é obrigatório voltar |
| VPS (Hetzner/Contabo) | €4–5 (≈ R$ 25–30) | Não | Sim (disco inteiro) | Mais estável e barato no longo prazo, mas você cuida do servidor |
| Oracle Cloud Free | R$ 0 | Não | Sim | Existe plano gratuito, mas o cadastro é chato e a máquina é lenta |

**Escolha do projeto: Fly.io.** É o menor custo que realmente cumpre os três
requisitos de produção (processo sempre vivo, disco persistente, reinício
automático em caso de falha).

Custos que podem aparecer além da mensalidade:
- Volume maior que 1 GB: US$ 0,15 por GB extra (1 GB é mais que suficiente; a sessão ocupa poucos MB).
- Tráfego de saída: há uma franquia mensal gratuita; o volume deste serviço é ínfimo.
- IP dedicado: US$ 2/mês — **não é necessário**.

---

## 2. Publicar no Fly.io (passo a passo)

Pré-requisito: cartão de crédito internacional cadastrado na conta Fly.io.

```bash
# 1. Instale a ferramenta (uma vez só)
#    Windows (PowerShell):  iwr https://fly.io/install.ps1 -useb | iex
#    Mac/Linux:             curl -L https://fly.io/install.sh | sh

# 2. Entre na sua conta
fly auth login

# 3. Na pasta deste projeto, crie a aplicação (NÃO faça deploy ainda)
#    Troque SEU-APP pelo nome que você quer (tem que ser igual ao do fly.toml)
fly launch --no-deploy --name SEU-APP --region gru

# 4. Crie o disco da sessão (1 GB é suficiente)
fly volumes create wpp_auth --size 1 --region gru

# 5. Cadastre as chaves (nunca coloque segredo dentro do código)
fly secrets set WPP_API_KEY="sua_chave" WPP_AGENT_SECRET="sua_chave_do_agente"
fly secrets set SUPABASE_URL="https://SEU-BACKEND" SUPABASE_ANON_KEY="sua_chave_publica"
# opcional, mas recomendado:
fly secrets set SUPABASE_SERVICE_KEY="sua_chave_de_servico"

# 6. Publique
fly deploy
```

No fim, o Fly mostra o endereço, algo como:

```
https://SEU-APP.fly.dev
```

Cole esse endereço na aba **Conexão** do painel e clique em **Conectar WhatsApp**.

### Sem terminal (só pelo navegador)
1. Suba esta pasta em um repositório no GitHub.
2. Em fly.io → **New App** → **Deploy from GitHub** → escolha o repositório.
3. Em **Volumes** → **Create volume** → nome `wpp_auth`, região `gru`, 1 GB.
   Monte em `/data`.
4. Em **Secrets**, cadastre as variáveis da tabela abaixo.
5. Em **Settings**, garanta que **auto stop machines** está **desligado**.

---

## 3. Publicar no Render (alternativa mais simples)

1. Suba esta pasta em um repositório no GitHub.
2. Em render.com → **New** → **Blueprint** → escolha o repositório.
   O arquivo `render.yaml` já configura plano, disco e verificação de saúde.
3. Preencha as variáveis marcadas como secretas.
4. Confirme que o plano é **Starter** e que o disco está em `/var/data`.

---

## 4. Publicar em VPS próprio (mais barato no longo prazo)

```bash
# na VPS, com Docker instalado
docker compose up -d --build
```

O `docker-compose.yml` já cria o volume e reinicia o serviço sozinho.
Para o HTTPS, coloque um proxy (Caddy ou Nginx) na frente apontando para a porta 3001.

---

## 5. Variáveis de ambiente

| Variável | Obrigatória | O que é |
|---|---|---|
| `PORT` | não | Porta HTTP. A hospedagem injeta sozinha (padrão 3001). |
| `WPP_API_KEY` | **sim** | Senha que o painel usa para falar com este servidor. |
| `WPP_AGENT_SECRET` | **sim** | A mesma chave configurada na função `wpp-agente` do Backend. |
| `SUPABASE_URL` | **sim** | Endereço do Backend (Readdy). |
| `SUPABASE_ANON_KEY` | **sim** | Chave pública do Backend. |
| `SUPABASE_SERVICE_KEY` | recomendada | Chave de serviço do Backend. Sem ela o servidor usa a pública e perde acesso a partes do banco. |
| `WPP_AUTH_DIR` | **sim** | Pasta da sessão. **Precisa estar dentro do disco/volume.** |
| `WPP_PUBLIC_URL` | não | URL pública do serviço. É detectada automaticamente no Fly/Render/Railway. |
| `TZ` | não | `America/Sao_Paulo` para os horários saírem certos. |

Nunca escreva essas chaves em arquivos versionados. Use os "Secrets" da plataforma.

---

## 6. Como saber se está tudo bem

- `SUA_URL/api/health` → deve responder `{"ok":true,...,"db":"ok"}`.
- Na aba **Conexão** do painel, o diagnóstico mostra versão, tempo no ar,
  sessão salva, contagem de mensagens e a data da última cobrança.
- Configure um monitor gratuito (UptimeRobot, BetterStack) apontando para
  `/api/health` a cada 5 minutos, com alerta por e-mail.

O que o servidor já garante sozinho:
- **Sem mensagem duplicada**: cada parcela é "reservada" no banco antes do envio.
- **Cobrança não se perde**: se o serviço estiver fora do ar às 11h, ele envia
  assim que voltar (e só uma vez por dia, mesmo reiniciando várias vezes).
- **Reconexão automática** com espera progressiva.
- **Fila do atendente humano** entregue em até 5 segundos.
- **Desligamento limpo**: em deploy/reinício ele fecha a sessão antes de morrer,
  evitando arquivo de credencial corrompido.

---

## 7. Backup e volta atrás

**Backup da sessão (opcional, mas útil):**
```bash
fly ssh console -C "tar -czf /tmp/sessao.tgz -C /data wpp-auth"
fly sftp get /tmp/sessao.tgz ./sessao-backup.tgz
```
Se perder a sessão, basta escanear o QR Code novamente — nada de dados de
clientes é perdido, porque tudo fica no Backend, não neste servidor.

**Voltar para a hospedagem anterior:** este servidor não depende de nenhuma
plataforma específica. Basta publicar esta mesma pasta no host antigo, apontar
o volume para a pasta da sessão e colar a URL nova na aba **Conexão**.
Nada precisa ser alterado no código.

**Nunca apague o serviço antigo antes de o novo responder** em `/api/health`
e de o botão **Enviar teste** chegar no seu celular.

---

## 8. Problemas comuns

| Sintoma | Causa provável | Solução |
|---|---|---|
| `Application not found` | URL errada ou serviço inexistente | Copie a URL atual em Settings/Domains |
| Conecta e cai em minutos | Sem volume (sessão não persiste) | Crie o volume e monte em `WPP_AUTH_DIR` |
| QR não aparece | Serviço ainda subindo | Espere o deploy terminar e clique em Conectar |
| Conectado mas não responde | Falta a chave da IA no Backend | Cadastre `OPENAI_API_KEY` no Backend |
| Cobrança não saiu hoje | Fora do horário ou sem conexão | Veja `last_cobranca` no diagnóstico |

---

## 9. "Deploy concluído, mas a URL não responde" (Request failed)

O Fly mostra **Build image ✓**, **Deploy application ✓** e depois fica repetindo
`Checking https://SEU-APP.fly.dev ... Latest Error: Request failed`. Isso quer
dizer que o pacote foi construído e enviado, mas **nenhuma máquina está
atendendo** naquele endereço. Quase sempre é um destes quatro:

**1. O nome do app no `fly.toml` não é o nome do app no Fly.**
Se a URL é `masterrrr.fly.dev`, a primeira linha do `fly.toml` tem que ser
`app = "masterrrr"`. Nome diferente = o deploy vai para outro lugar e a URL
consultada continua vazia. Confira também em **Settings → App name**.

**2. O volume `wpp_auth` não existe.**
O `fly.toml` monta o volume em `/data`. Sem esse volume a máquina não sobe.
Vá em **Volumes** e confirme: nome `wpp_auth`, região `gru`, 1 GB.

**3. A máquina está parada ou reiniciando em loop.**
Em **Machines**, a máquina precisa estar `started` e com a saúde `passing`.
Se estiver `stopped` ou reiniciando a cada minuto, veja o motivo em **Logs** —
as três mensagens mais comuns são:
- `Error: volume ... not found` → falta o volume (item 2).
- `Killed` / `Out of memory` → aumente `memory = "1gb"` no `fly.toml`.
- `Cannot find module` → algum arquivo não foi enviado ao GitHub; suba a pasta
  inteira, com `package.json` e `Dockerfile` na raiz do repositório.

**4. O deploy foi feito, mas a máquina nunca foi criada.**
No painel do app, vá em **Machines → Add machine** (região `gru`) e aguarde
o status ficar `started`. Se preferir o caminho com terminal, rode `fly deploy`
dentro da pasta do projeto — ele cria a máquina, aplica os secrets e confere
o volume de uma vez.

Depois de qualquer uma dessas correções, abra `https://SEU-APP.fly.dev/api/health`.
Tem que aparecer `{"ok":true,...,"db":"ok"}`. Só então volte ao painel, cole a URL
na aba **Conexão** e clique em **Conectar WhatsApp**.

> Observação técnica: a partir da versão 5.1.0 o `/api/health` responde na hora,
> sem consultar o banco. Antes, uma resposta lenta do banco podia estourar o
> tempo limite da checagem de saúde do Fly, fazendo a máquina reiniciar em loop —
> o que também aparecia como `Request failed`.