# Conexões — 0.6.0

Use stdio para um cliente no mesmo computador ou VPS. O cliente executa `.venv/bin/python run.py --mode live`, com credenciais e perfil no ambiente do processo. Modelos em `examples/mcp.live.json` e `examples/hermes.live.yaml`; instruções em [VPS.md](VPS.md).

O perfil é configuração do administrador. Ferramentas nunca recebem segredo, endpoint, query GraphQL livre ou caminho de arquivo. O rótulo da conta é local e não comprova titularidade. A geração de links confere `expected_account_reference` antes de enviar.

HTTP privado: `--transport private-http --port 8765`, somente loopback, rota `/mcp/`, `MCP_LOCAL_ACCESS_TOKEN` ASCII com pelo menos 32 caracteres e allowlist de Host/Origin. Esse transporte ainda apresenta avisos de encerramento em teste ASGI e deve ser tratado como experimental.

ChatGPT e outros clientes remotos podem usar `oauth-http` atrás de HTTPS com OAuth ou uma chave estática Bearer. No modo API key, configure `MCP_API_KEY`, `SHOPEE_APP_ID` e `SHOPEE_APP_SECRET` como secrets de runtime; a chave protege `/mcp` e as credenciais Shopee assinam as chamadas à API. No modo OAuth, as credenciais ficam criptografadas no volume persistente. Veja [REMOTE.md](REMOTE.md) e [DOKPLOY-PRIVADO.md](DOKPLOY-PRIVADO.md). O token de `private-http` continua restrito ao loopback.

Múltiplos consumidores da mesma conta precisam coordenar a quota Shopee e a política dos relatórios. Os controles deste processo não sincronizam máquinas ou processos diferentes.
