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
| `backup/` | Backup diário (dump do MariaDB + config + apps + arquivos) com restic → MinIO e Cloudflare R2; aplicado com `kubectl apply -k backup` |
| `secret.oidc.local.yaml` | secret do client OIDC `nextcloud` no Keycloak — **em `.gitignore`** |

Entrada pública: `https://cloud.lynkado.com.br` → **Cloudflare Tunnel** (`cloudflared`
no ns `cloudflare`) → `Service/nextcloud:80`. Não há Ingress.

> **Troca de domínio (03/10/2026):** o hostname público no túnel passou de `lynkado.com.br` para
> `cloud.lynkado.com.br`. Além do `configmap.yaml` (`OVERWRITEHOST`; `NEXTCLOUD_TRUSTED_DOMAINS` só vale
> na instalação inicial), a instância já instalada foi ajustada com `occ`:
> `config:system:set trusted_domains 0`, `overwritehost`, `overwrite.cli.url` e
> `user_oidc:provider Keycloak --postlogouturi=https://cloud.lynkado.com.br/`. No Keycloak, o client
> `nextcloud` precisa aceitar `https://cloud.lynkado.com.br/*` em *Valid redirect URIs* e
> *Valid post logout redirect URIs*.

## Estado observado no cluster

| Item | Valor |
| --- | --- |
| Versão Nextcloud | 34.0.3 (imagem `docker.io/library/nextcloud:latest`) |
| Deployment revision | 49 (muitas edições manuais via Lens/kubectl) |
| Banco | MariaDB — `mariadb.mariadb.svc.cluster.local` |
| Cache / lock | Redis — `redis.redis.svc.cluster.local:6379`, **com senha** (ver projeto `redis`) |
| Domínio público | `https://cloud.lynkado.com.br` via **Cloudflare Tunnel** (`cloudflared` no ns `cloudflare`) |
| Recursos do pod | requests `300m`/`768Mi` · limits `1500m`/`3Gi` |

## Autenticação: Keycloak (`user_oidc`), desde 27/09/2026

O login do Nextcloud passa pelo **Keycloak** (`https://login.lynkado.com.br`, realm **`lynkado`**, client
**`nextcloud`**), com o app oficial **`user_oidc` 8.11.0**. O Google virou um provedor **do Keycloak**. O
antigo app *Social Login* foi desativado e o client antigo do Google foi apagado. Lado Keycloak:
`fabriziorocha/keycloak`, `migracao/README.md`.

### Como ficou

| Item | Valor |
|---|---|
| Provedor `user_oidc` | `Keycloak` (id 1), discovery `https://login.lynkado.com.br/realms/lynkado/.well-known/openid-configuration`, scope `openid email profile` |
| uid do Nextcloud | claim **`preferred_username`**, com **`unique-uid=0`** (sem hash) e *soft auto-provisioning* (padrão ligado): cai na conta que já existe com o mesmo uid, de qualquer backend |
| Nome / e-mail | claims `name` / `email` |
| Grupos | `--group-provisioning=1 --mapping-groups=groups --group-whitelist-regex='^nextcloud$'`: **só o grupo `nextcloud` é sincronizado**. Com whitelist, o código do `user_oidc` ignora (não remove) grupos que não casam com o regex, então os grupos do Nextcloud (Família, admin...) ficam intactos. Conferido no piloto |
| Restrição de acesso | `--group-restrict-login-to-whitelist=1`: sem o grupo `nextcloud` no token, o Nextcloud recusa o login |
| Bearer | `check-bearer=0` (API/WebDAV continuam por app password) |
| Logout | `post-logout` `https://cloud.lynkado.com.br/`; backchannel logout do Keycloak em `/apps/user_oidc/backchannel-logout/Keycloak` |
| Virada | `occ config:app:set user_oidc allow_multiple_user_backends --value=0 --lazy`: `/login` redireciona direto para o Keycloak (só há 1 provedor). O backend `Database` **continua registrado** |
| Texto do botão | **"Login com Lynkado"**: `occ config:system:set user_oidc login_label --value="Login com Lynkado"` (sem isso o `user_oidc` mostra "Login with <identificador>"; o identificador `Keycloak` foi mantido porque faz parte da URL de backchannel logout configurada no Keycloak). Aparece em `?direct=1` e depois do logout |
| Acesso de emergência | **`https://cloud.lynkado.com.br/login?direct=1`**: usuário e senha do Nextcloud (conta local `admin`, com TOTP; bot `ultro`) |
| 2FA | Login pelo Keycloak **não** pede o TOTP do Nextcloud; o segundo fator é o da conta Google. O TOTP do Nextcloud continua valendo no login por senha (`?direct=1`) |
| Log | `occ log:manage --level=warning` (2); estava em 0 (DEBUG), escrevendo sem parar no disco com defeito |

Os uids foram preservados: os 8 usuários `google-<sub>` (criados pelo Social Login) ganharam no Keycloak o
mesmo username, já vinculados à conta Google; `fabrizio` foi renomeado no Keycloak para bater com o uid
local. Piloto com `fabrizio`: mesma conta (backend `Database`), os 7 grupos intactos + `nextcloud`, nenhum
usuário novo criado. Antes da migração o administrador removeu 5 contas locais (juliana, estevao, moises,
paulobelesa, nina), junto com as pastas delas; essas pessoas voltam como usuários novos pelo Google.

### Como foi configurado

O secret do client fica em `secret.oidc.local.yaml` (fora do git). Para ele não aparecer em argumento de
comando (o audit do Kubernetes registra `kubectl exec`), o provedor foi criado num **pod temporário** com a
imagem do Nextcloud, o PVC montado, label `app: nextcloud-cron` (as NetworkPolicies do MariaDB e do Redis
já liberam esse label) e o secret por variável de ambiente:

```bash
php occ user_oidc:provider Keycloak \
  --clientid=nextcloud --clientsecret-env=NC_OIDC_SECRET \
  --discoveryuri=https://login.lynkado.com.br/realms/lynkado/.well-known/openid-configuration \
  --postlogouturi=https://cloud.lynkado.com.br/ --scope="openid email profile" \
  --unique-uid=0 --check-bearer=0 --bearer-provisioning=0 --send-id-token-hint=1 \
  --mapping-uid=preferred_username --mapping-display-name=name --mapping-email=email \
  --mapping-groups=groups --group-provisioning=1 \
  --group-whitelist-regex='^nextcloud$' --group-restrict-login-to-whitelist=1
```

(Sem segredo, dá para rodar direto: `occ app:install user_oidc`, `occ user_oidc:provider Keycloak
--output=json_pretty` para conferir.)

### Operação

- **Usuário novo:** entra pelo Google → recebe "sem permissão" → admin coloca no grupo `nextcloud` **no
  Keycloak** (Users → Groups → Join group) → no próximo login a conta é criada → adicionar aos grupos do
  Nextcloud aqui.
- **Tirar acesso:** remover do grupo `nextcloud` ou desativar no Keycloak; para cortar sessões abertas,
  `occ user:auth-tokens:delete <uid>` (ou pelas configurações de segurança do usuário).
- **Clientes desktop/celular** já conectados usam app passwords e continuam funcionando; conexões novas
  passam pelo navegador (Login Flow) e, portanto, pelo Keycloak.

### Rollback

1. `occ config:app:set user_oidc allow_multiple_user_backends --value=1 --lazy` (volta a tela de senha).
2. Se preciso: `occ app:enable sociallogin`. Os dados dele (`oc_sociallogin_connect`, config cifrada)
   continuam no banco, **mas o client do Google que ele usava foi apagado**: seria preciso criar outro no
   Google Cloud e atualizar o provider no Social Login.
3. `occ app:disable user_oidc`.

Depois de algumas semanas estável: `occ app:remove sociallogin` para limpar o app e as tabelas dele.

## Backup

Criado em 2026-09-27. CronJob `nextcloud/nextcloud-backup`, diário às **04:00** (America/Sao_Paulo),
depois do Mailu (03:00) e do Keycloak (03:30).

| Destino | O que vai | Por quê |
|---|---|---|
| **MinIO** `nextcloud-backup` (ns `minio`, disco físico **sdc**, separado do sdb) | dump do banco + `config/` + `custom_apps/` + **arquivos** (`data/`, sem miniaturas/cache) | Restauração completa; ~135 GiB, 117 GB guardados |
| **Cloudflare R2** `nextcloud-backup` (fora de casa) | dump do banco + `config/` + `custom_apps/` (**sem** os arquivos) | Cabe na franquia gratuita (10 GB). Decisão: os arquivos (~135 GB) ficam só no MinIO |

Se a máquina for perdida, o R2 permite reconstruir contas, grupos, compartilhamentos e configurações,
mas **não os arquivos**.

Como funciona (`backup/cronjob.yaml`, `backup/configmap.yaml`):

1. initContainer `db-dump` (`mariadb:11.8.9`): `mariadb-dump --single-transaction --quick --routines
   --triggers` como o usuário `nextcloud` (senha do secret `nextcloud-secret`). Transação única: dump
   consistente sem travar o Nextcloud. Falha se o arquivo não terminar com `Dump completed`.
2. container `restic` (0.19.1), com o PVC `nextcloud` montado **somente leitura** em `/nc`:
   `restic backup` no MinIO (tudo) e no R2 (sem `data/`), exclusões em `excludes.txt` (miniaturas,
   css/js gerados, cache, uploads parciais), retenção 7 diárias / 4 semanais / 12 mensais e, aos
   domingos, `restic check --read-data-subset` (2% no MinIO, 10% no R2).
3. Alertas `NextcloudBackupFailed` e `NextcloudBackupStale` (mais de 26 h sem sucesso) no Prometheus
   (o Alertmanager do cluster ainda não notifica ninguém).

Segurança: PSA `restricted` (roda como `www-data`, uid 33, dono dos arquivos; rootfs só leitura;
`drop ALL`). `backup/networkpolicy.yaml` libera só DNS, MariaDB (regra aditiva no ns `mariadb`), MinIO
(regra no ns `minio`) e IPs públicos na 443 (R2). Segredos: `backup/secret.local.yaml` (fora do git) →
`backup/sealed-secret.yaml`. **Guarde o `RESTIC_PASSWORD` fora do cluster.**

```bash
kubectl apply -k backup
kubectl -n nextcloud get cronjob,jobs -l app=nextcloud-backup
kubectl -n nextcloud create job --from=cronjob/nextcloud-backup nextcloud-backup-manual-$(date +%s)
kubectl -n nextcloud logs job/<nome> --all-containers
```

> Neste cluster `kubectl exec -i`/`kubectl cp` não entregam stdin (kubectl 1.34 × servidor 1.37).
> Scripts vão por ConfigMap montado ou em base64 no argumento; resultados saem por `kubectl logs`.

### Teste de restauração (feito em 2026-09-27)

Pod temporário no ns `nextcloud` (label `app: nextcloud-backup`, para usar as NetworkPolicies):

- initContainer `restic`: `restic restore latest --verify` do MinIO só de `/dump`, `/nc/config` e 3
  arquivos de amostra (não cabe restaurar 135 GB num emptyDir: o disco do sistema tem 118 GB);
- container `mariadb`: `mariadb-install-db` + `mariadbd --skip-networking` num emptyDir (uid 999;
  conectar com `-u root`), importa o dump e conta os registros.

| Verificação | Resultado |
|---|---|
| `restic --verify` | ✅ 17 arquivos, 88 MiB |
| Tabelas | ✅ 177 (igual à produção) |
| Usuários / grupos | ✅ 11 / 10 (igual à produção) |
| `oc_filecache` / compartilhamentos | 37.074 / 9 |
| Vínculo do Social Login | ✅ `fabrizio` |
| `config.php` | ✅ `instanceid`, `passwordsalt`, `secret`, `dbpassword` presentes |
| 3 arquivos de amostra (vídeo, JPG de pasta de grupo, foto de usuário) | ✅ SHA-256 idêntico ao de produção |

### Restauração completa (roteiro, não executado)

1. `occ maintenance:mode --on` (ou escalar o Deployment e o CronJob `nextcloud-cron` a zero).
2. Banco: restaurar `/dump/nextcloud.sql` no MariaDB (`mariadb -u nextcloud nextcloud < nextcloud.sql`,
   num pod com o label de backup e o dump restaurado por restic).
3. Arquivos: pod com o PVC `nextcloud` montado **com escrita**, `restic restore <snapshot> --target /
   --include /nc/data --include /nc/config --include /nc/custom_apps` (montando o PVC em `/nc`), depois
   `chown -R 33:33`.
4. Subir, `occ maintenance:mode --off`, `occ files:scan --all` e `occ maintenance:data-fingerprint`
   (avisa os clientes de sincronização de que o servidor foi restaurado).

## Incidente: disco sdb com setores ilegíveis (2026-09-27)

O primeiro backup completo encontrou `input/output error`. O kernel confirma
`Medium Error / Unrecovered read error - auto reallocate failed` no **sdb** (SSD GN-512-2280, 512 GB,
M.2 2280 SATA, porta `ata2`), que guarda **todos** os volumes `local-path` do cluster
(`/var/mnt/local-storage`, inclusive o PVC `nextcloud`).

SMART (valores brutos): `Reallocated_Sector_Ct` 14, `Current_Pending_Sector` 14,
`Offline_Uncorrectable` 24, ~7.800 h ligado. O "PASSED" do SMART é enganoso (o fabricante não ajusta os
valores normalizados). Diagnóstico: defeito da memória flash, não desgaste. **Troca necessária.**

**10 arquivos ilegíveis** (não entram no backup; só voltam se houver originais para reenviar):

- pasta de grupo 1 (Marketing): `Ensaio Fotos - 1/260201 Secundo/BFV00233.ARW`;
  `MARKETING/ENSAIO EXTERNA - MAIO - Luma, Alexa, Marina e olivia/DSC09078.ARW` e `DSC09086.JPG`;
  `MARKETING/ENSAIO FOTOS ESTUDIO 4 - MAIO/05_11_2026_NINA GYN/DSC00948.ARW`;
  `MARKETING/ENSAIO FOTOS ESTUDIO 4 - MAIO/CAMISAS E BONÉS COPA/05_11_2026_NINA GYN/DSC00589.ARW`;
- Ana Trindade (`google-112467776291511528952`), `UploadInstantâneo/Camera/`: `20250320_124332.mp4`,
  `20250420_195928.mp4`, `20250508_102507.mp4`, `20250614_180101.mp4`, `20260117_135602.mp4`.

Acompanhamento: se a contagem de `input/output error` no log do backup noturno crescer, a degradação
está avançando (troca urgente).

Plano de troca (clone): o `UserVolumeConfig local-storage` do Talos escolhe o disco pela **porta**
(`disk.bus_path` do `ata2`, `minSize: 512GB`) e acha o volume pelo rótulo `u-local-storage`. Um clone
feito com **GNU ddrescue** (não `dd`), conferido com `xfs_repair -n`, com a partição e o XFS aumentados
(`growpart` + `xfs_growfs`) e colocado **no mesmo slot**, monta sem mudar a configuração do Talos. O disco
novo precisa ter **≥ 512 GB reais** (1 TB recomendado; "500 GB" não passa no `minSize`). Nunca ligar o
disco antigo e o clone juntos. Antes da troca: backup manual dos três apps (Mailu, Keycloak, Nextcloud).

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
