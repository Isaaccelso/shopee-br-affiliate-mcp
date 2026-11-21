# Shopee Affiliate MCP privado no Dokploy

Este guia é para o fork [Isaaccelso/shopee-br-affiliate-mcp](https://github.com/Isaaccelso/shopee-br-affiliate-mcp), usando o transporte **Streamable HTTP + OAuth**. A instalação atende um uso pessoal; não é um sistema multiusuário com login por e-mail.

**O que fica protegido:** qualquer ferramenta em `/mcp` exige um token OAuth válido. Para emitir esse token, a autorização interativa exige uma chave privada do proprietário **além** das credenciais da Open API de Afiliados da Shopee. A Shopee não autentica visitantes do seu servidor MCP.

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

**Atenção:** trocar o hash do proprietário invalida conexões existentes. Trocar `MCP_CREDENTIALS_KEY` sem migrar o banco impede abrir o estado criptografado. Guarde backups do volume e da chave Fernet separadamente.

## 3. Criar o serviço no Dokploy

1. **Create Application** → GitHub → `Isaaccelso/shopee-br-affiliate-mcp` → branch `main`.
2. Selecione **Dockerfile**, caminho `/Dockerfile`, com **build target `remote`** (crucial: o estágio padrão do Dockerfile é `stdio` e não inicia uma API HTTP).
3. Porta **interna 8765**. Configure o domínio `https://shopee-mcp.seudominio.com.br` no proxy do Dokploy e HTTPS.
4. Não publique a porta 8765 diretamente no host (nada de `0.0.0.0:8765:8765`). Somente o proxy da rede Docker deve encaminhar tráfego até o container.
5. Configure **um volume persistente** em `/app/data`, gravável pelo UID 10001. Configure 1 réplica e 1 worker.
6. Adicione as variáveis abaixo como **runtime/secrets**, não como build args. Ajuste o domínio e os valores gerados:

```dotenv
MCP_PUBLIC_URL=https://shopee-mcp.seudominio.com.br
MCP_OWNER_KEY_SHA256=COLE_O_HASH_SHA256_AQUI
MCP_CREDENTIALS_KEY=COLE_A_CHAVE_FERNET_AQUI
MCP_STATE_DB=/app/data/oauth.sqlite3
SHOPEE_VERIFIED_PROFILE=/app/examples/profile.official.json
MCP_ALLOW_LOOPBACK_CALLBACKS=false
MCP_OAUTH_REDIRECT_URIS=[]
```

**Não coloque `SHOPEE_APP_ID` e `SHOPEE_APP_SECRET` no runtime `oauth-http`**. Nesse transporte as credenciais são informadas uma vez no formulário de autorização e ficam criptografadas no banco. No modo `stdio --mode live`, as variáveis `SHOPEE_APP_ID` e `SHOPEE_APP_SECRET` são necessárias ao processo local.

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

- **Não existe `X-API-Key` obrigatório** nem API key estática para `/mcp`.
- `MCP_OWNER_KEY_SHA256` **não é** um header: é o hash de uma chave privada que você digita no formulário `/connect`.
- Não coloque App ID, App Secret ou chave do proprietário no `Authorization` do cliente MCP.
- O Bearer token só é entregue **depois** da autorização e expira/renova pelo fluxo OAuth. O cliente faz a renovação.
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

Endpoint: `https://open-api.affiliate.shopee.com.br/graphql`.

A assinatura é:
```text
SHA256( APP_ID + TIMESTAMP_EM_SEGUNDOS + CORPO_JSON_EXATAMENTE_COMO_ENVIADO + APP_SECRET )
```

O timestamp e o JSON assinado mudam a cada requisição. **Não copie um header de exemplo como valor fixo**. O código `SignedClient`, em `src/shopee_mcp/core.py`, calcula e envia a assinatura para você.

## 6. Conectar no ChatGPT

1. Nas configurações de plugins/conectores MCP, quando a opção estiver disponível, cadastre `https://shopee-mcp.seudominio.com.br/mcp` e escolha OAuth.
2. O cliente abre o formulário de conexão. Verifique o domínio e o endereço de redirecionamento exibidos.
3. Informe **CHAVE_PRIVADA** (gerada no passo 2), **App ID** e **App Secret** da Shopee.
4. Confirme a autorização. O MCP valida as credenciais com uma pesquisa de produto antes de emitir token.
5. Teste `affiliate_status` e depois `search_offers` com uma palavra-chave.

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

- **Chave privada do proprietário:** gere outra, atualize `MCP_OWNER_KEY_SHA256` e reconecte os clientes; as conexões antigas são revogadas quando o estado é reaberto com a política nova.
- **Token MCP:** revogue a conexão/tokens e reconecte; proteja dispositivos e histórico dos clientes.
- **App Secret da Shopee:** rotacione pelo painel oficial e autorize novamente; não basta trocar o hash do proprietário.
- **Chave Fernet:** não troque sem plano de migração/rotação dos dados criptografados. Se comprometida junto com o banco, considere as credenciais expostas e rotacione na origem.

## Referências do projeto

- [Segurança e ameaça](../SECURITY.md)
- [Modo privado e chave do proprietário](PRIVATE.md)
- [Configuração do transporte remoto](REMOTE.md)
- [Contrato/assinatura da API Shopee](OFFICIAL-CONTRACT.md)
- [Open API da Shopee para Afiliados](https://affiliate.shopee.com.br/open_api)

> O transporte remoto ainda é experimental e não substitui uma auditoria de segurança. As ferramentas em modo live dependem das permissões aprovadas para a conta Shopee.
