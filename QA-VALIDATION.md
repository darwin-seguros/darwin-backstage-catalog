# QA Validation — Automations Platform (Platônico + Infra Backoffice)

> **Escopo:** Validação de tudo implementado na sessão de deploy das automações Darwin.
> **Ambiente:** `shared` (único ambiente de produção — `us-east-1`, AWS account `958549588201`)
> **Data da sessão:** 2026-03-27

---

## 1. Repositórios

| Repositório | Descrição | Branch principal |
|---|---|---|
| `darwin-seguros/darwin-bot` | Bot Platônico (Node.js + Bot Framework SDK v4) | `main` |
| `darwin-seguros/darwin-infra-backoffice` | Portal Infra Backoffice (React + Vite + MSAL) | `main` |
| `darwin-seguros/darwin-backstage` | Developer Portal Backstage | `main` |
| `darwin-seguros/helm-apps` | GitOps — charts Helm para ArgoCD | `main` |
| `darwin-seguros/darwin-backstage-catalog` | Software Catalog centralizado | `main` |

---

## 2. Infraestrutura — Cluster EKS `shared`

### 2.1 Pods em execução

```bash
kubectl get pods -n darwin-bot
kubectl get pods -n darwin-infra-backoffice
```

**Esperado:** 2/2 pods Running em cada namespace, sem restarts recentes.

### 2.2 ArgoCD Applications

```bash
kubectl get applications -n argocd | grep -E "darwin-bot|darwin-infra-backoffice"
```

**Esperado:**
```
darwin-bot                Synced    Healthy
darwin-infra-backoffice   Synced    Healthy
```

### 2.3 ExternalSecret (darwin-bot)

```bash
kubectl get externalsecret -n darwin-bot
kubectl describe externalsecret darwin-bot-secrets -n darwin-bot
```

**Esperado:** `SecretSynced` — secret sincronizado do AWS Secrets Manager (`darwin-bot/shared`).

### 2.4 EKS Access Entry (GitHub Actions)

```bash
aws eks list-access-entries --cluster-name shared --region us-east-1 | grep sts-GitHub-Repo-Access
```

**Esperado:** Role `arn:aws:iam::958549588201:role/sts-GitHub-Repo-Access` listada com política `AmazonEKSClusterAdminPolicy`.

---

## 3. darwin-bot — Platônico

### 3.1 Health check

```bash
curl https://bot.shared.cloud.darwinseguros.com/api/health
```

**Esperado:**
```json
{"status":"ok","service":"darwin-bot","name":"Platônico"}
```

### 3.2 Endpoint de instalação do bot

```bash
curl -H "Authorization: Bearer <token>" \
  https://bot.shared.cloud.darwinseguros.com/api/bot/install-info
```

**Esperado:** JSON com `teamsAppId` e/ou `deepLink`.

### 3.3 Endpoint interno (Backstage → bot)

```bash
curl -X POST https://bot.shared.cloud.darwinseguros.com/api/internal/messages/send \
  -H "Authorization: Bearer <BOT_INTERNAL_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"targets":["<target-id>"],"text":"Teste QA Platônico"}'
```

**BOT_INTERNAL_TOKEN:** recuperar do AWS Secrets Manager `darwin-bot/shared` → chave `BOT_INTERNAL_TOKEN`.

**Esperado:** `{"results":{"<target-id>":{"success":true}}}` (ou `503` se token não configurado).

### 3.4 Listagem de targets

```bash
curl -H "Authorization: Bearer <azure-token>" \
  https://bot.shared.cloud.darwinseguros.com/api/targets
```

**Esperado:** `{"data":[...]}` com channels/grupos registrados.

### 3.5 Histórico de mensagens

```bash
curl -H "Authorization: Bearer <azure-token>" \
  "https://bot.shared.cloud.darwinseguros.com/api/messages/history?limit=5"
```

**Esperado:** `{"data":[...],"total":<n>,"limit":5,"offset":0}`

---

## 4. darwin-infra-backoffice

### 4.1 Portal acessível

Abrir no browser: `https://infra-backoffice.shared.cloud.darwinseguros.com`

**Esperado:** Tela de login MSAL (Azure AD) carregada corretamente. Sem erro 502/504.

### 4.2 Container rodando nginx-unprivileged (porta 8080)

```bash
kubectl get pods -n darwin-infra-backoffice -o jsonpath='{.items[0].spec.containers[0].image}'
```

**Esperado:** imagem `nginxinc/nginx-unprivileged:1.27-alpine` (ou SHA digest equivalente no ECR).

```bash
kubectl describe pod -n darwin-infra-backoffice | grep "Port:"
```

**Esperado:** `containerPort: 8080`.

### 4.3 Após login — funcionalidades

| Página | URL | Validação |
|---|---|---|
| Compose | `/compose` | Campo de mensagem carrega targets do bot |
| Histórico | `/history` | Lista mensagens com paginação |
| Canais & Grupos | `/targets` | Lista targets registrados |
| Templates | `/templates` | Lista templates do Backstage |
| FinOps | `/finops` | Página carrega sem erro JS |

---

## 5. CI/CD — GitHub Actions

### 5.1 Fluxo GitOps (darwin-bot e darwin-infra-backoffice)

**Fluxo esperado após push na `main`:**
1. GitHub Actions: build Docker → push ECR com tags `latest` e `$GITHUB_SHA`
2. GitHub Actions: `git clone helm-apps && sed image.tag → commit → push`
3. ArgoCD: detecta mudança no helm-apps → sync automático → rolling update

**Sem secrets adicionais** — usa o OIDC AWS já configurado para fazer `kubectl rollout restart` após o push no ECR.

### 5.2 Validar último run

```bash
gh run list -R darwin-seguros/darwin-bot --limit 3
gh run list -R darwin-seguros/darwin-infra-backoffice --limit 3
gh run list -R darwin-seguros/darwin-backstage --limit 3
```

**Esperado:** `completed success` no run mais recente de cada repo.

### 5.3 Verificar que o deploy Helm foi removido dos Actions

Confirmar que os workflows NÃO contêm mais passos `helm upgrade --install`.

```bash
grep -r "helm upgrade" darwin-bot/.github darwin-infra-backoffice/.github
```

**Esperado:** sem resultados (deploy é feito pelo ArgoCD).

---

## 6. Backstage — darwin-backstage

### 6.1 Service Token (Platônico → Backstage)

O darwin-bot usa `BACKSTAGE_SERVICE_TOKEN` para chamar a Catalog API do Backstage com autenticação service-to-service.

**Validar secret no cluster:**
```bash
kubectl get secret backstage-microsoft-oauth -n shared -o jsonpath='{.data.BACKSTAGE_SERVICE_TOKEN}' | base64 -d | wc -c
```

**Esperado:** token com 64 chars (hex).

### 6.2 Templates disponíveis no Backstage

Abrir `https://backstage.production.cloud.darwinseguros.com/create` e verificar:

- [ ] Template `example-nodejs-template` listado
- [ ] Template `Platônico: Enviar mensagem no Teams` listado (tag: `platonico`)

### 6.3 Scaffolder action `platonico:send-message`

No Backstage, executar o template `platonico-send-message` com:
```json
{
  "targets": ["<id de um target existente>"],
  "text": "Teste QA via Backstage Scaffolder"
}
```

**Validar no darwin-bot:**
```bash
curl -H "Authorization: Bearer <token>" \
  "https://bot.shared.cloud.darwinseguros.com/api/messages/history?limit=1"
```

**Esperado:** mensagem com `sent_by: backstage-scaffolder` no histórico.

### 6.4 Vars de ambiente do Backstage (PLATONICO_*)

```bash
kubectl exec -n shared deploy/backstage -- env | grep PLATONICO
```

**Esperado:**
```
PLATONICO_BOT_URL=https://bot.shared.cloud.darwinseguros.com
PLATONICO_INTERNAL_TOKEN=<token>
```

---

## 7. Software Catalog — darwin-backstage-catalog

### 7.1 Repo

URL: `https://github.com/darwin-seguros/darwin-backstage-catalog`

**Estrutura esperada:**
```
catalog-info.yaml          ← Location root
org/groups.yaml            ← Group: platform-team
systems/infra-platform.yaml ← Domain: platform + System: infra-platform
components/
  platonico/catalog-info.yaml              ← Component + API
  darwin-infra-backoffice/catalog-info.yaml ← Component
  darwin-backstage/catalog-info.yaml        ← Component
templates/
  platonico-notification/template.yaml     ← Template Scaffolder
```

### 7.2 Registrar no Backstage

Para registrar o catalog no Backstage (uma vez), abrir o portal e ir em:
**Create → Register Existing Component**

URL para registrar:
```
https://raw.githubusercontent.com/darwin-seguros/darwin-backstage-catalog/main/catalog-info.yaml
```

**OU** adicionar ao `app-config.production.yaml` em `catalog.locations`:
```yaml
catalog:
  locations:
    - type: url
      target: https://github.com/darwin-seguros/darwin-backstage-catalog/blob/main/catalog-info.yaml
      rules:
        - allow: [Component, System, API, Group, Domain, Location, Template]
```

**Após registro, verificar no Backstage:**
- [ ] Component `platonico` visível no catálogo
- [ ] Component `darwin-infra-backoffice` visível
- [ ] Component `darwin-backstage` visível
- [ ] System `infra-platform` visível
- [ ] Group `platform-team` visível
- [ ] API `platonico-rest-api` com spec OpenAPI visível
- [ ] Template `platonico-send-message` listado em Create

---

## 8. AWS Secrets Manager

Secret: `darwin-bot/shared` — Chaves esperadas:

| Chave | Descrição |
|---|---|
| `MICROSOFT_APP_ID` | App ID do Azure Bot |
| `MICROSOFT_APP_PASSWORD` | Secret do Azure Bot |
| `MICROSOFT_APP_TENANT_ID` | Tenant ID Azure |
| `POSTGRES_HOST` | Endpoint RDS shared |
| `POSTGRES_PORT` | 5432 |
| `POSTGRES_USER` | platonico_user |
| `POSTGRES_PASSWORD` | senha do user da aplicação |
| `POSTGRES_DB` | platonico |
| `BACKSTAGE_BASE_URL` | https://backstage.production.cloud.darwinseguros.com |
| `BACKSTAGE_SERVICE_TOKEN` | Token 64-char para auth service-to-service |
| `AZURE_AD_TENANT_ID` | Tenant para validação JWT dos usuários |
| `AZURE_AD_CLIENT_ID` | Client ID para validação JWT |
| `BOT_INTERNAL_TOKEN` | Token compartilhado Backstage → bot (endpoint /api/internal) |
| `PORT` | 3978 |

---

## 9. Pendências conhecidas

| Item | Detalhe | Responsável |
|---|---|---|
| Bot instalado no Teams | O bot precisa estar instalado em pelo menos um canal para testes de envio | DevOps |
| Backstage build — validar | Verificar que o build do Backstage completou com sucesso após commit do Dockerfile fix | QA |
| Registrar catalog no Backstage | Executar o registro da URL do darwin-backstage-catalog no portal | DevOps |
| Validar `platonico:send-message` end-to-end | Executar template e confirmar mensagem no Teams e no histórico do bot | QA |
| Bot instalado no Teams | O bot precisa estar instalado em pelo menos um canal para testes de envio | DevOps |

---

## 10. Arquitetura resumida

```
GitHub (main push)
  └─► GitHub Actions
        ├─► docker build + push ECR (SHA tag + latest)
        └─► commit image.tag no helm-apps
              └─► ArgoCD detecta → sync
                    ├─► darwin-bot (namespace: darwin-bot)
                    └─► darwin-infra-backoffice (namespace: darwin-infra-backoffice)

Browser (equipe DevOps)
  └─► Darwin Infra Backoffice (MSAL Azure AD)
        └─► darwin-bot REST API (JWT Azure AD)
              ├─► PostgreSQL RDS (database: platonico)
              ├─► AWS Secrets Manager (darwin-bot/shared via ExternalSecret)
              └─► Backstage API (BACKSTAGE_SERVICE_TOKEN)

Backstage Scaffolder
  └─► platonico:send-message action
        └─► darwin-bot /api/internal/messages/send (BOT_INTERNAL_TOKEN)
              └─► Microsoft Teams Bot Framework → canal/grupo
```

---

## 11. Comandos úteis de diagnóstico

```bash
# Pods
kubectl get pods -n darwin-bot -n darwin-infra-backoffice

# Logs do bot (últimas 50 linhas)
kubectl logs -n darwin-bot deploy/darwin-bot --tail=50

# Logs do backoffice
kubectl logs -n darwin-infra-backoffice deploy/darwin-infra-backoffice --tail=20

# ExternalSecret status
kubectl describe externalsecret darwin-bot-secrets -n darwin-bot

# ArgoCD apps
kubectl get applications -n argocd

# Forçar sync ArgoCD (sem aguardar próximo ciclo)
kubectl patch application darwin-bot -n argocd \
  --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'

# Verificar imagem deployada
kubectl get deployment darwin-bot -n darwin-bot \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```
