# nextcloud-asis — snapshot do que está publicado no Kubernetes

Reconstrução **fiel ("as-is")** dos manifestos que compõem a aplicação
`nextcloud` rodando no cluster Talos, capturada em **2026-09-03**.

> Não é uma versão limpa/endurecida. O objetivo é ter no Git exatamente o que
> está no cluster, incluindo os problemas conhecidos (seção "Pendências").
> A versão "para arrumar" é o projeto `nextcloud`.

## Estrutura

| Arquivo | Recurso no cluster |
| --- | --- |
| `namespace.yaml` | `Namespace/nextcloud` (sem Pod Security Admission) |
| `configmap.yaml` | `ConfigMap/nextcloud-config` (12 chaves) |
| `configmap-remoteip.yaml` | `ConfigMap/nextcloud-remoteip` (`remoteip.conf` do Apache) |
| `secret.example.yaml` | template do `Secret/nextcloud-secret` (commitado) |
| `secret.local.yaml` | valores em claro para gerar o sealed — **em `.gitignore`** |
| `sealed-secret.yaml` | `SealedSecret/nextcloud-secret` (único segredo versionado) |
| `pvc.yaml` | `PersistentVolumeClaim/nextcloud` (256Gi, `local-path`) |
| `deployment.yaml` | `Deployment/nextcloud` (1 réplica, imagem/tag definidas no `kustomization.yaml`) |
| `cronjob.yaml` | `CronJob/nextcloud-cron` (`*/5 * * * *`, `php cron.php`) |
| `service.yaml` | `Service/nextcloud` ClusterIP (80) |
| `networkpolicy.yaml` | default-deny + ingress só do `cloudflared` + egress mínimo |
| `kustomization.yaml` | junta tudo no ns `nextcloud` + labels `app.kubernetes.io/*` (Lens agrupa como "Application") |

Entrada pública: `https://lynkado.com.br` → **Cloudflare Tunnel** (`cloudflared`
no ns `cloudflare`) → `Service/nextcloud:80`. Não há Ingress.

## Estado observado no cluster

| Item | Valor |
| --- | --- |
| Versão Nextcloud | 34.0.3 (imagem `docker.io/library/nextcloud:latest`) |
| Deployment revision | 49 (muitas edições manuais via Lens/kubectl) |
| Banco | MariaDB — `mariadb.mariadb.svc.cluster.local` |
| Cache / lock | Redis — `redis.redis.svc.cluster.local:6379`, **com senha** (ver projeto `redis`) |
| Domínio público | `https://lynkado.com.br` via **Cloudflare Tunnel** (`cloudflared` no ns `cloudflare`) |
| Recursos do pod | requests `300m`/`768Mi` · limits `1500m`/`3Gi` |

## Aplicar

```bash
# 1. Segredo (só ao rotacionar senha):
cp secret.example.yaml secret.local.yaml         # preencha os valores
kubeseal --format yaml --controller-namespace kube-system \
  < secret.local.yaml > sealed-secret.yaml       # commite só o sealed-secret.yaml

# 2. Aplicar:
kubectl apply -k .
kubectl -n nextcloud rollout status deploy/nextcloud
```

> **Migração do Secret (feita em 2026-09-03):** o `nextcloud-secret` era um
> `Secret` comum criado à mão; o controller de sealed-secrets não sobrescreve
> um Secret que não é dele. Sequência usada:
> `kubectl -n nextcloud delete secret nextcloud-secret` e
> `kubectl apply -f sealed-secret.yaml`. Agora `SYNCED=True`.
>
> **Senha do admin:** `NEXTCLOUD_ADMIN_PASSWORD` só vale na instalação inicial.
> A instância já estava instalada, então a troca real foi
> `occ user:resetpassword admin` dentro do pod; o valor no Secret foi alinhado
> à nova senha. O usuário `admin` tem **2FA (TOTP) ativo** — login por senha
> pura (WebDAV/basic-auth) não funciona sem app-password ou o código TOTP.
>
> **Senha do Redis (2026-09-03):** o Redis passou a exigir `requirepass`
> (projeto `redis`). A senha entra pela env **`REDIS_HOST_PASSWORD`** (no
> `SealedSecret`), porque a imagem oficial roda `config/redis.config.php` a cada
> request e ele sobrescreve o bloco `redis` do `config.php` — `occ
> config:system:set redis password` é ignorado. Trocar a senha do Redis exige
> re-selar este secret + `rollout restart`.

## Pendências (drift / problemas capturados no snapshot)

1. ~~Imagem sem tag~~ — **resolvido em 2026-09-19**: tag fixada em
   `34.0.4-apache` via `images:` no `kustomization.yaml` (aplica-se a
   `deployment.yaml` e `cronjob.yaml`). `pullPolicy` continua `Always`.
2. ~~Credenciais de admin em ConfigMap~~ — **resolvido em 2026-09-03**: senha
   trocada via `occ`, movida para o `SealedSecret`, removida do ConfigMap.
3. **Recursos divergentes** do manifesto original versionado (que pedia
   2CPU/4Gi–4CPU/8Gi). Aqui já estão os valores reais do cluster —
   `limits.memory: 3Gi`. O pod foi **OOMKilled em 2026-09-03T22:57** (rodou
   ~9h e estourou). Avaliar subir para 4Gi ou investigar vazamento.
   ⚠️ Todo restart do pod re-gera o `config.php` a partir das envs — se faltar
   `REDIS_HOST_PASSWORD` no Secret, o Redis quebra (NOAUTH) silenciosamente.
4. ~~Namespace sem Pod Security Admission~~ — **resolvido em 2026-09-19**:
   `pod-security.kubernetes.io/enforce=restricted` ativo em `namespace.yaml`,
   alinhado com n8n/minio. Pod web e cron agora rodam 100% non-root
   (`runAsUser/Group: 33`, `drop: ["ALL"]`, `seccompProfile: RuntimeDefault`;
   o pod web soma `NET_BIND_SERVICE` para bindar a porta 80). Dois problemas
   apareceram no meio do caminho e foram corrigidos:
   - O entrypoint da imagem oficial escreve
     `/usr/local/etc/php/conf.d/redis-session.ini` (dir `root:root`) sempre
     que `REDIS_HOST` está definido — é a única escrita fora do
     `/var/www/html` no fluxo normal de start (auditado no `entrypoint.sh`).
     Corrigido com o `initContainer redis-session-ini` em `deployment.yaml`,
     que replica essa lógica escrevendo num `emptyDir` montado via `subPath`
     só nesse arquivo. **Esse initContainer é permanente** — roda em todo pod
     novo, não é correção de uma vez só (o diretório é do filesystem da
     imagem, não do PVC).
   - `/var/www/html/nextcloud-init-sync.lock` tinha ficado `root:root` no PVC
     (resíduo de quando o pod sempre rodava como root) e bloqueava o
     non-root. Corrigido uma única vez via `chown www-data:www-data` — esse,
     sim, foi definitivo (é dado no PVC, não recriado a cada pod).
   ⚠️ Não testado: o caminho de **upgrade de major** do entrypoint
   (`NEXTCLOUD_UPDATE=1`) sob non-root — só será exercitado no próximo bump
   de major version. Se falhar, o rollout trava com o `enforce: restricted`
   já ativo; ter esse Pendência em mente antes do próximo upgrade de major.
5. **`strategy` padrão (RollingUpdate)** com PVC RWO — pode travar rollout.
6. Sem ServiceAccount dedicada (usa `default`).
7. **NetworkPolicy não é aplicada pelo Flannel** (CNI atual). `networkpolicy.yaml`
   fica como intenção declarativa até haver um enforcer no cluster.
8. Config de *trusted proxy* espalhada em 3 lugares não idênticos
   (`TRUSTED_PROXIES` no env, `remoteip.conf`, `trusted_proxies` no `config.php`
   aplicado via `occ` — este último fora do Git).

## Fora deste snapshot (existem no repo `nextcloud` mas não no cluster)

- `ingress.yaml` / `certificate.yaml` — o Ingress `nextcloud-ingress`
  (`linkado.ddns.net`) era o acesso antigo, antes do Cloudflare Tunnel.
  Removido do cluster em 2026-09-03. cert-manager nunca foi instalado.
- `label-transformer.yaml` — estava comentado no kustomization.
- `values.yaml` — values do Helm chart; o deploy não usa Helm.
- Assets de tema (`favicon.ico`, `lynkado.png/webp`, textos legais) — enviados
  manualmente ao pod / app Theming, não versionados como manifesto.

## Limpezas já aplicadas no cluster

- **2026-09-03**: `kubectl -n nextcloud delete ingress nextcloud-ingress`
  (host `linkado.ddns.net`, sem uso desde a migração para o Cloudflare Tunnel).
- **2026-09-03**: removida a porta `443` do `Service/nextcloud` (sem backend —
  o container só escuta na `80`).
- **2026-09-03**: adicionadas as 4 NetworkPolicies de `networkpolicy.yaml`
  (default-deny, ingress só do `cloudflared`, egress web e egress do cron).
  O pod do CronJob passou a ter o label `app: nextcloud-cron` (antes o
  jobTemplate não tinha label nenhum) só para casar com a policy de egress.
  Sem efeito prático enquanto o CNI for Flannel.
