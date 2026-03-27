# Darwin Backstage Catalog

Software Catalog centralizado da Darwin Seguros.

## Registrar no Backstage

**Create → Register Existing Component → URL:**

```
https://github.com/darwin-seguros/darwin-backstage-catalog/blob/main/catalog-info.yaml
```

## Estrutura

```
catalog-info.yaml                              ← Location root (importa tudo)
org/groups.yaml                                ← Teams
systems/infra-platform.yaml                    ← Domain + System
components/
  platonico/catalog-info.yaml                  ← Bot + API OpenAPI
  darwin-infra-backoffice/catalog-info.yaml    ← Portal
  darwin-backstage/catalog-info.yaml           ← Developer Portal
templates/
  platonico-notification/template.yaml         ← Enviar mensagem Teams via Scaffolder
QA-VALIDATION.md                               ← Checklist de validação
```

## Adicionar novo componente

1. Criar `components/<nome>/catalog-info.yaml`
2. Adicionar a referência em `catalog-info.yaml` (seção `spec.targets`)
3. Push na `main` — Backstage sincroniza automaticamente
