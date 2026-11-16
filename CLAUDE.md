# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## O que é este repositório

Homelab **GitOps**: um cluster **k3s** gerenciado declarativamente pelo **Argo CD**.
Não há código de aplicação nem build — o repositório é só manifests Kubernetes. A
fonte da verdade é o Git; o cluster converge sozinho. **Nunca aplique mudanças direto
no cluster como solução — corrija o manifest no repo e deixe o Argo sincronizar.**

## Arquitetura de sync (o fluxo que amarra tudo)

É um padrão **app-of-apps** em dois níveis:

1. `bootstrap/root-app.yaml` define a Application `root`, que aponta para `argocd/`
   com `directory.recurse: true`. Ela descobre e cria todas as outras Applications.
2. Cada arquivo em `argocd/<nome>.yaml` é uma `Application` que aponta para uma pasta
   em `apps/<nome>/` (renderizada por Kustomize) e a sincroniza no namespace do app.

Consequência prática: para adicionar um app, crie `apps/<nome>/` (com
`kustomization.yaml`) **e** `argocd/<nome>.yaml`. A Application `root` passa a gerenciá-lo
automaticamente no próximo sync — não é preciso aplicar nada à mão.

Todas as Applications usam `syncPolicy.automated` com `prune: true` e `selfHeal: true`:
**drift é revertido automaticamente** e recursos removidos do Git são apagados do cluster.
Por isso uma mudança só "pega" depois de **commit + push** para `main`.

`bootstrap/argocd/` é especial: instala o próprio Argo CD a partir do `install.yaml`
oficial de uma versão fixada, com patch local (`server.insecure: "true"`). Mudanças aqui
afetam a camada de controle — explique o impacto e a ordem de aplicação.

## Convenções (seguir à risca)

- **Kustomize** para compor cada app; todo app tem seu `kustomization.yaml`.
- **Ingress é Traefik** (padrão do k3s) com TLS via annotation. Não presuma nginx-ingress.
  `apps/whoami/` é o modelo de referência para um app novo.
- Imagens sempre com **tag de versão explícita** (nunca `latest`); o mesmo para o Argo CD.
- Defina `resources.requests`/`limits` e probes — é homelab, recursos são escassos.
- Commits em **pt-BR**, Conventional Commits (`feat:`, `refactor:`, `chore:`).

## Validação (não há CI nem testes)

Valide localmente antes de commitar, sem aplicar no cluster:

```bash
kustomize build apps/<nome>          # renderiza o Kustomize
kubectl kustomize apps/<nome>        # alternativa sem kustomize standalone
kubectl apply --dry-run=client -k apps/<nome>
```

Diagnóstico read-only do Argo (não mutar o cluster sem pedido explícito):

```bash
kubectl get applications -n argocd
argocd app get <nome> ; argocd app diff <nome>
kubectl get events -n <ns> --sort-by=.lastTimestamp
```

## Minecraft — detalhes não óbvios (`apps/minecraft/`)

- Server **`itzg/minecraft-server`** (Paper) rodando em **`ONLINE_MODE=FALSE`**
  (contas TLauncher/offline). Isso muda tudo sobre identidade de jogador.
- Whitelist e ops vêm de um **ConfigMap** (`seed-users.yaml`) montado em `/seed`, não
  de lookup no PlayerDB. As UUIDs nos arquivos **precisam ser UUIDs offline** —
  UUID v3 (MD5) de `OfflinePlayer:<nick>`, não a UUID real da conta Mojang. Gere a UUID
  por esse algoritmo ao adicionar um jogador.
- `OVERRIDE_OPS=TRUE`/`OVERRIDE_WHITELIST=TRUE`: os arquivos de `/seed` são reaplicados
  **no startup do pod**. Editar o ConfigMap não basta — é preciso **reiniciar o pod**
  para a mudança valer. `ENFORCE_WHITELIST=TRUE`: um OP que não está na whitelist não entra.
- Exposição externa via **playit.gg** (`playit.yaml`), não via Ingress. A credencial do
  agente (`playit-secret`) é criada fora do Git com `kubectl create secret` e referenciada
  por `secretKeyRef` — **não** está no repositório e não deve ser commitada.

## Subagents disponíveis

- `argo-k3s` — especialista em Argo CD/k3s/Kustomize/Traefik para criar e editar apps e
  diagnosticar sync/drift.
- `git-flow` — commits (Conventional Commits pt-BR), PRs e merges via `gh`.
