# Shopee Affiliate MCP no Dokploy

Este guia é para o fork [Isaaccelso/shopee-br-affiliate-mcp](https://github.com/Isaaccelso/shopee-br-affiliate-mcp), usando Streamable HTTP com OAuth ou Bearer API key. A instalação atende um uso pessoal; não é um sistema multiusuário com login por e-mail.

**Modo API key:** `/mcp` aceita `Authorization: Bearer <MCP_API_KEY>` e usa a conta única definida por `SHOPEE_APP_ID`/`SHOPEE_APP_SECRET`. A chave MCP autentica o cliente no seu serviço; as credenciais Shopee assinam chamadas à API oficial.

**Modo OAuth:** a autorização interativa exige uma chave privada do proprietário além das credenciais de afiliado. O servidor guarda credenciais e tokens criptografados no volume persistente. A Shopee não autentica visitantes do seu servidor MCP.

**O que continua público:** `/health` e os metadados/registro OAuth (`/.well-known/*`, `/register`) precisam estar disponíveis para o fluxo de conexão. Registrá-lo como cliente OAuth **não** autoriza usar ferramentas. Não publique sua chave privada, chaves de criptografia, App Secret, tokens nem o banco SQLite.

## 1. Pré-requisitos

- VPS Linux com Dokploy, DNS de um subdomínio exclusivo e HTTPS válido.
- Credenciais **App ID / App Secret** da [Open API de Afiliados Shopee](https://affiliate.shopee.com.br/open_api), após aprovação. Não são credenciais da plataforma de vendedores.
- Repositório `Isaaccelso/shopee-br-affiliate-mcp` disponível para o Dokploy.
- Uma única réplica/worker, com volume persistente. O banco OAuth não suporta várias réplicas compartilhando estado.
- Cliente MCP compatível com OAuth (ex.: ChatGPT) para concluir a autorização.

## 2. Gere os dois segredos no seu terminal (nunca no Git)

**Chave privada do proprietário**: mantenha o valor apresentado em `CHAVE_PRIVADA` somente no seu gerenciador de senhas. No Dokploy você informa apenas o hash.

```bash
python3 - <<'PY'
import hashlib, secrets
chave = secrets.token_urlsafe(48)
print("CHAVE_PRIVADA (guardar fora do Dokploy):", chave)
print("MCP_OWNER_KEY_SHA256=", hashlib.sha256(chave.encode()).hexdigest(), sep="")
PY
```

**Chave Fernet**: salva como `MCP_CREDENTIALS_KEY` nas variáveis seguras do Dokploy e deve ser mantida entre deploys.

```bash
python3 - <<'PY'
import base64, secrets
print(base64.urlsafe_b64encode(secrets.token_bytes(32)).decode())
PY
```

**API key do Agentes Chat**: gere um token separado e salve como secret `MCP_API_KEY` no Dokploy. Nunca use o App Secret Shopee como chave do MCP.

```bash
python3 - <<'PY'
import secrets
print(secrets.token_urlsafe(48))
PY
```

**Atenção:** trocar o hash do proprietário invalida conexões existentes. Trocar `MCP_CREDENTIALS_KEY` sem migrar o banco impede abrir o estado criptografado. Guarde backups do volume e da chave Fernet separadamente.

## 3. Criar o serviço no Dokploy

1. **Create Application** → GitHub → `Isaaccelso/shopee-br-affiliate-mcp` → branch `main`.
2. Selecione **Dockerfile**, caminho `/Dockerfile`, com **build target `remote`** (crucial: o estágio padrão do Dockerfile é `stdio` e não inicia uma API HTTP).
3. Porta **interna 8765**. Configure o domínio `https://shopee-mcp.seudominio.com.br` no proxy do Dokploy e HTTPS.
4. Não publique a porta 8765 diretamente no host (nada de `0.0.0.0:8765:8765`). Somente o proxy da rede Docker deve encaminhar tráfego até o container.
5. Configure **um volume persistente** em `/app/data`, gravável pelo UID 10001. Configure 1 réplica e 1 worker.
6. Para a conexão por API key do Agentes Chat, adicione apenas estas variáveis como **runtime/secrets**, não como build args:

```dotenv
MCP_PUBLIC_URL=https://shopee-mcp.seudominio.com.br
MCP_API_KEY=CHAVE_ALEATORIA_DE_PELO_MENOS_32_CARACTERES
SHOPEE_APP_ID=APP_ID_DA_OPEN_API
SHOPEE_APP_SECRET=APP_SECRET_DA_OPEN_API
```

Esse modo não usa `MCP_OWNER_KEY_SHA256`, `MCP_CREDENTIALS_KEY` nem o volume OAuth. Se optar pelo OAuth, configure essas variáveis e o volume conforme o restante deste guia; sem `MCP_API_KEY`, o servidor inicia OAuth.

Se sua tela do Dokploy não disponibilizar a escolha do build target, use um deploy Docker Compose personalizado com `build.target: remote`, conectando o serviço ao proxy do Dokploy pela rede interna. Não use o estágio `stdio` esperando que ele abra uma porta.

## 4. Domínio, proxy e headers HTTP

**URL pública do servidor** (somente origem, sem barra final ou `/mcp`):
`https://shopee-mcp.seudominio.com.br`

**URL para cadastrar no cliente MCP:**
`https://shopee-mcp.seudominio.com.br/mcp`

O proxy HTTPS precisa encaminhar todos os caminhos, incluindo `/mcp`, `/connect`, `/authorize`, `/register`, `/token`, `/revoke` e `/.well-known/*`, preservando o header **Host** público. Não registre corpos de requisição nem valores de `Authorization` em logs. Desabilite cache no proxy e aplique rate limiting por IP, sobretudo em `/register`, `/authorize` e `/connect`.

O app inclui controles `Cache-Control: no-store`, `Referrer-Policy: no-referrer`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Content-Security-Policy`, `Permissions-Policy` e, em HTTPS, `Strict-Transport-Security`. Isso **não** substitui firewall nem revisão de permissões do Dokploy.

**Não coloque um Basic Auth, Cloudflare Access ou um header estático obrigatório na frente de todas as rotas sem configurar uma exceção compatível com OAuth:** clientes como ChatGPT precisam acessar os endpoints públicos de descoberta e autorização para se conectar. O próprio `/mcp` já exige OAuth.

## 5. Qual autenticação e quais headers usar?

Existem **duas APIs diferentes**. Não misture os formatos de autenticação.

### Cliente → seu MCP (OAuth)

O cliente MCP recebe um access token depois do fluxo de autorização e o manda **automaticamente** nas chamadas HTTP. Exemplo dos headers de uma chamada autenticada:

```http
POST /mcp HTTP/1.1
Host: shopee-mcp.seudominio.com.br
Authorization: Bearer <TOKEN_OAUTH_DO_MCP>
Content-Type: application/json
Accept: application/json, text/event-stream
MCP-Protocol-Version: 2025-06-18
```

- Não há header `X-API-Key`; no modo estático use o header `Authorization: Bearer`.
- `MCP_API_KEY` é a chave estática do modo API key; o cliente a envia como Bearer. Gere-a aleatoriamente e não a reutilize em outros serviços.
- `MCP_OWNER_KEY_SHA256` é usado somente pelo fluxo OAuth legado, não pelo modo API key.
- App ID e App Secret Shopee nunca são enviados pelo cliente MCP em cada chamada; ficam no ambiente protegido do servidor.
- OAuth continua disponível como modo alternativo, com tokens rotativos.
- `/mcp` usa transporte **Streamable HTTP stateless**; não é um SSE ou um endpoint REST comum de ofertas.

### Seu MCP → API oficial Shopee (assinatura SHA256)

O próprio projeto monta os headers. **Você não configura manualmente esses headers no Dokploy**.

```http
POST /graphql HTTP/1.1
Host: open-api.affiliate.shopee.com.br
Authorization: SHA256 Credential=<APP_ID>, Timestamp=<UNIX_SECONDS>, Signature=<SHA256_HEX>
Content-Type: application/json
Accept: application/json
```

Para o cliente Agentes Chat no modo API key, a chamada MCP usa:

```http
Authorization: Bearer <MCP_API_KEY>
```

Endpoint: `https://open-api.affiliate.shopee.com.br/graphql`.

A assinatura é:
```text
SHA256( APP_ID + TIMESTAMP_EM_SEGUNDOS + CORPO_JSON_EXATAMENTE_COMO_ENVIADO + APP_SECRET )
```

O timestamp e o JSON assinado mudam a cada requisição. **Não copie um header de exemplo como valor fixo**. O código `SignedClient`, em `src/shopee_mcp/core.py`, calcula e envia a assinatura para você.

## 6. Conectar no ChatGPT

1. Cadastre `https://shopee-mcp.seudominio.com.br/mcp` como servidor MCP.
2. Escolha **Chave de API** e informe `MCP_API_KEY`; não adicione um header manual.
3. O Agentes Chat valida e sincroniza as ferramentas usando `Authorization: Bearer ...`.
4. Confirme que a conexão ficou ativa e que as ferramentas foram sincronizadas.

Para OAuth, siga o formulário com a chave privada e as credenciais da Shopee conforme [REMOTE.md](REMOTE.md).

Por padrão, callbacks locais (`localhost` / `127.0.0.1`) ficam **bloqueados**. Para conectar Hermes/Codex com retorno ao próprio computador, cadastre **somente** a URI exata necessária em `MCP_OAUTH_REDIRECT_URIS` ou habilite `MCP_ALLOW_LOOPBACK_CALLBACKS=true` conscientemente. Nunca libere URLs externas genéricas ou curingas.

## 7. Teste de segurança após o deploy

Execute apenas requisições **sem segredos**:

```bash
# Saúde: 200 (não comprova autenticação)
curl -i https://shopee-mcp.seudominio.com.br/health

# Metadados OAuth: 200 (precisam ser públicos para descoberta)
curl -i https://shopee-mcp.seudominio.com.br/.well-known/oauth-authorization-server

# Acesso SEM Bearer ao MCP: TEM de responder 401
curl -i -X POST https://shopee-mcp.seudominio.com.br/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{}'
```

Se `/mcp` retornar 200 sem token, **não utilize o deploy**. Verifique se selecionou o estágio `remote`, se a URL realmente aponta para seu container e se nenhuma camada de proxy remove a proteção.

O endpoint `/register` possui limite local de 10 tentativas por minuto e cota de clientes; as tentativas de autorização com chave privada também têm limite local. Para proteção real contra abuso ou indisponibilidade, use regras de rate limiting/WAF no proxy **por IP**, pois os limites do processo são globais e não substituem proteção de borda.

## 8. Checklist de proteção e operação

- [ ] Acesso à API de Afiliados aprovado pela Shopee
- [ ] HTTPS válido, origem `MCP_PUBLIC_URL` correta
- [ ] Build com target `remote`, somente **uma réplica**
- [ ] Segredos configurados apenas no runtime, sem commits no Git
- [ ] Chave privada aleatória e hash SHA256 corretamente gerados
- [ ] Fernet permanente + volume `/app/data` persistente
- [ ] Callbacks localhost desabilitados, salvo necessidade explícita
- [ ] Porta 8765 não exposta diretamente na internet
- [ ] Logs e cache de credenciais desabilitados no proxy
- [ ] `POST /mcp` sem token devolve **401**
- [ ] Revise permissões de acesso à VPS/Dokploy e faça backup criptografado

## 9. Se uma chave vazar

- **Chave privada do proprietário:** gere outra, atualize `MCP_OWNER_KEY_SHA256` e reconecte os clientes OAuth; as conexões antigas são revogadas quando o estado é reaberto com a política nova.
- **MCP_API_KEY:** gere outro token, atualize o secret no Dokploy e reconecte os clientes que usam a chave estática.
- **Token OAuth MCP:** revogue a conexão/tokens e reconecte; proteja dispositivos e histórico dos clientes.
- **App Secret da Shopee:** rotacione pelo painel oficial e autorize novamente; não basta trocar o hash do proprietário.
- **Chave Fernet:** não troque sem plano de migração/rotação dos dados criptografados. Se comprometida junto com o banco, considere as credenciais expostas e rotacione na origem.

## Referências do projeto

- [Segurança e ameaça](../SECURITY.md)
- [Modo privado e chave do proprietário](PRIVATE.md)
- [Configuração do transporte remoto](REMOTE.md)
- [Contrato/assinatura da API Shopee](OFFICIAL-CONTRACT.md)
- [Open API da Shopee para Afiliados](https://affiliate.shopee.com.br/open_api)

> O transporte remoto ainda é experimental e não substitui uma auditoria de segurança. As ferramentas em modo live dependem das permissões aprovadas para a conta Shopee.
