# Middleware de Integração — Desligamento Protheus → Oracle Cloud ERP/HCM

Esqueleto funcional de um serviço Node.js que consulta periodicamente uma fila de desligamentos exposta pelo Protheus e replica esse desligamento no Oracle Cloud HCM via API REST com autenticação OAuth2.

Este código é um **ponto de partida**, não uma solução final: os endpoints exatos do Oracle, os nomes de campos e o contrato da API do Protheus precisam ser ajustados ao ambiente real (ver seção 7 do documento de arquitetura).

## Estrutura

```
src/
  config/        configuração centralizada (lê o .env)
  services/
    protheusClient.js     chamadas à API REST do Protheus
    oracleClient.js        autenticação OAuth2 e chamadas à API do Oracle
    mapaMotivos.js          mapeamento de motivo de desligamento Protheus -> Oracle
    desligamentoService.js  orquestra o fluxo completo e a lógica de retry
  routes/        endpoints HTTP do próprio middleware (health check, disparo manual)
  utils/logger.js logger estruturado (winston)
  index.js        ponto de entrada: sobe o servidor e agenda o polling
```

## Como rodar

```bash
npm install
cp .env.example .env
# edite o .env com as credenciais e URLs reais
npm start
```

Em desenvolvimento, `npm run dev` reinicia automaticamente ao salvar um arquivo.

## Configuração (.env)

Veja `.env.example` para a lista completa de variáveis. As principais:

- `PROTHEUS_BASE_URL` / `PROTHEUS_API_KEY` — endpoint REST exposto pelo Protheus e sua credencial.
- `ORACLE_OAUTH_TOKEN_URL`, `ORACLE_CLIENT_ID`, `ORACLE_CLIENT_SECRET` — credenciais OAuth2 do Oracle Identity Cloud.
- `POLLING_CRON` — frequência de verificação da fila (padrão: a cada 5 minutos).
- `MAX_TENTATIVAS` — quantas vezes tentar antes de marcar como erro definitivo e notificar.

## O que ainda precisa ser ajustado antes de produção

1. **Endpoint do Oracle** (`src/services/oracleClient.js`): o caminho `/hcmRestApi/resources/.../workRelationships` e os nomes de campos (`TerminationDate`, `TerminationReasonCode`) são ilustrativos. Confirmar com a documentação REST da versão específica do Oracle HCM Cloud em uso.
2. **Busca por CPF**: a query `NationalIdentifierNumber='...'` no recurso `workers` depende de como o identificador nacional está configurado no ambiente Oracle. Validar com o time responsável.
3. **Contrato da API do Protheus** (`src/services/protheusClient.js`): os endpoints `/api/desligamentos/pendentes` e `/api/desligamentos/confirmar` precisam existir do lado Protheus (via TLPP/Rest Framework), conforme descrito no documento de arquitetura.
4. **Mapeamento de motivos** (`src/services/mapaMotivos.js`): os códigos à direita (`INVOLUNTARY`, `VOLUNTARY` etc.) são placeholders — substituir pelos lookup codes reais configurados no Oracle.
5. **Notificação de falhas** (`src/services/desligamentoService.js`, função `notificarResponsavel`): hoje só loga um aviso; integrar com e-mail/Slack/Teams real.
6. **Persistência do contador de tentativas**: neste esqueleto, o número de tentativas é assumido como vindo do registro da fila do Protheus (campo `tentativas`). Confirmar onde esse contador será mantido (lado Protheus é o mais natural, já que ele controla o status do registro).

## Testando sem o Oracle/Protheus reais

Para testar a lógica isoladamente, é possível subir mocks simples dos dois endpoints (por exemplo, com `json-server` ou um servidor Express de teste) e apontar `PROTHEUS_BASE_URL` e `ORACLE_BASE_URL`/`ORACLE_OAUTH_TOKEN_URL` para eles antes de integrar com os ambientes reais.
