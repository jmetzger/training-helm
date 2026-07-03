# Praxisbeispiel: openDesk mit Helmfile ausrollen

## Hintergrund

openDesk (ZenDiS/BMI) ist kein einzelnes Helm-Chart, sondern 35+ Charts (Keycloak/Nubus als IAM,
Nextcloud, OX App Suite, OpenProject, XWiki, Element/Synapse, Jitsi, Collabora, CryptPad,
Mailstack, Datenbanken, ...), orchestriert ueber [Helmfile](https://helmfile.readthedocs.io/).
Kein `helm install opendesk` - stattdessen `helmfile apply -e <environment> -n <namespace>`.

Dieses Dokument beschreibt den Rollout auf einem frischen DOKS-Cluster (DigitalOcean) als
Referenz fuer aehnliche Helmfile-basierte Multi-Chart-Deployments.

## Voraussetzungen

| Tool | Version | Hinweis |
|---|---|---|
| Kubernetes | >= 1.24, CNCF-zertifiziert | getestet mit 1.35.5 (DOKS) |
| Helm | >= 3.17.3, NICHT 3.18.0 / 3.20.1 | bekannte Helm-Bugs in diesen Versionen |
| Helmfile | >= 1.0.0 | |
| Helm Diff Plugin | >= 3.11.0 | `helm plugin install` |
| yq | >= 4.52.4 | |
| Ingress Controller | ingress-nginx oder haproxy-ingress.github.io | openDesk >= 1.13 nutzt haproxy als Default |
| cert-manager | mit CRDs | fuer Zertifikate |
| Volume Provisioner | ReadWriteOnce (min.) | z.B. `do-block-storage` |

Hardware-Minimum laut openDesk-Doku fuer eine Eval-Instanz: 12 CPU-Cores, 32 GB RAM.
In der Praxis (siehe Troubleshooting unten) sollte man deutlich mehr RAM einplanen.

## Ablauf

### Schritt 1: Ingress-Controller und cert-manager installieren

openDesk >= v1.13 verwendet standardmaessig `ingressClassName: haproxy` (nginx-ingress gilt als
deprecated). Wenn man bei nginx bleiben will, muss man das explizit in einer eigenen
Environment-Values-Datei setzen (siehe Schritt 3).

```
helm repo add haproxy-ingress https://haproxy-ingress.github.io/charts
helm upgrade -i haproxy-ingress haproxy-ingress/haproxy-ingress \
  -n haproxy-ingress --create-namespace \
  --set controller.ingressClass=haproxy \
  --set controller.ingressClassResource.enabled=true \
  --set controller.ingressClassResource.default=true \
  --set controller.config.config-global="tune.bufsize 65536\ntune.http.maxhdr 256"
```

```
helm repo add jetstack https://charts.jetstack.io
helm upgrade -i cert-manager jetstack/cert-manager \
  -n cert-manager --create-namespace --set crds.enabled=true
```

Danach einen `ClusterIssuer` fuer Let's Encrypt anlegen (HTTP-01, passend zur Ingress-Klasse):

```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: <deine-email>
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - http01:
          ingress:
            ingressClassName: haproxy
```

### Schritt 2: DNS

openDesk braucht ein Wildcard-DNS auf die Ingress-LoadBalancer-IP - jede Komponente bekommt eine
eigene Subdomain (z.B. `office.<domain>`, `id.<domain>`, `pad.<domain>`).

```
# LoadBalancer-IP ermitteln
kubectl -n haproxy-ingress get svc haproxy-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

DNS-Eintraege: `<domain>` und `*.<domain>` als A-Record auf diese IP.

### Schritt 3: Repo klonen und konfigurieren

```
git clone https://gitlab.opencode.de/bmi/opendesk/deployment/opendesk.git
cd opendesk
git checkout v1.16.1
```

Eigene Einstellungen kommen als `.yaml.gotmpl`-Dateien in `helmfile/environments/dev/` (nicht
die Default-Dateien in `helmfile/environments/default/` anfassen). Beispiel, um bei nginx statt
haproxy zu bleiben:

```
# helmfile/environments/dev/ingress-override.yaml.gotmpl
ingress:
  ingressClassName: "nginx"
  controller: "nginx"
```

### Schritt 4: Deployen

```
export DOMAIN=meine-domain.de
export MASTER_PASSWORD=<sicheres-passwort-per-openssl-rand-generiert>

helmfile apply -e dev -n opendesk
```

`MASTER_PASSWORD` ist Pflicht (kein Default mehr seit v1.14) und wird intern per
`derivePassword` fuer alle Sub-Secrets (Datenbanken, LDAP, OX AppSuite, ...) verwendet - ein
Passwort reicht fuer die komplette Eval-Instanz.

**Sicherheitshinweis:** `MASTER_PASSWORD` niemals im Klartext in Chat/Terminal-History landen
lassen - per `openssl rand` generieren, SOPS-verschluesselt ablegen (`.env.enc`), zur Laufzeit
per `sops --decrypt` in eine Shell-Variable einlesen.

### Schritt 5: Default-User importieren

Ueber die UDM-REST-API mittels Docker-Container:

```
docker run --rm registry.opencode.de/bmi/opendesk/components/platform-development/images/user-import:3.0.0 \
  ./user_import_udm_rest_api.py \
  --import_domain ${DOMAIN} \
  --udm_api_password ${MASTER_PASSWORD} \
  --set_default_password <user-pw> \
  --import_filename template.ods \
  --create_admin_accounts True
```

## Troubleshooting (Praxis-Erfahrungen)

### `helm upgrade` schlaegt fehl mit `jobs.batch "<name>-N" not found`

Trat bei `opendesk-migrations-pre` und `opendesk-nextcloud-management` auf, wenn `helmfile apply`
nach einem Fehler erneut ausgefuehrt wird. Ursache: die ConfigMap `migrations-status` im
Ziel-Namespace haelt den Migrations-Status fest und verwirrt den Job-Hook bei einem Retry
("Current Stage PRE is the same as PRE - this is suspicious").

Fix: ConfigMap loeschen und erneut `helmfile apply` ausfuehren.

```
kubectl -n opendesk delete configmap migrations-status
```

### `ums-ldap-server-primary` crasht mit OOMKilled

Der Nubus-LDAP-Server (Univention-basiert, Chart `nubus`) kann beim Start extrem viel Speicher
allozieren (in einem Test: ueber 11 GB in unter 3 Sekunden), unabhaengig vom gesetzten Memory-
Limit (getestet bis 15Gi). Weder frische PVCs noch ein kompletter `helm uninstall`/Neuinstall
des `nubus`-Release behoben es zuverlaessig in unserem Setup (DOKS 1.35.5, sehr neue Kernel-
Version). Das deutet auf eine Inkompatibilitaet zwischen genau dieser Chart-/Image-Version und
der Kubernetes-/Kernel-Kombination hin, nicht auf einen simplen Konfigurationsfehler.

Ein echter (und notwendiger) Bug wurde dabei trotzdem gefunden: `ums-ldap-notifier` und
`ums-ldap-server-primary` teilen sich ein RWO-Volume und muessen daher zwingend auf demselben
Node laufen (der Chart definiert dafuer eine `podAffinity`). Wird der `ldap-server-primary`-Pod
manuell neu gestartet (z.B. `kubectl delete pod`), kann der Scheduler ihn auf einen anderen Node
setzen als den Notifier -> `Multi-Attach error for volume ... already used by pod(s)
ums-ldap-notifier-0`. Fix: betroffenen Notifier-Pod ebenfalls loeschen, damit er sich per
Required-Affinity wieder neben dem Primary einordnet.

```
kubectl -n opendesk delete pod ums-ldap-notifier-0
```

Wenn danach weiterhin OOMKilled auftritt: naechster Schritt waere ein Test mit einer aelteren,
etablierteren Kubernetes-Minor-Version (z.B. 1.33.x statt 1.35.x) oder einer aelteren
openDesk-Version, um die Kombination einzugrenzen. Noch nicht final geloest.

### Geloeschte PVCs haengen in `Terminating`

Auf DigitalOcean kann das automatische Detachen eines Block-Storage-Volumes vom Node mehrere
Minuten dauern oder haengen bleiben. Kein K8s-Fehler, sondern eine asynchrone Cloud-Operation.
Falls es nach 5+ Minuten immer noch haengt: Volume manuell per `doctl` detachen, dann die
`VolumeAttachment`-Objekte in Kubernetes loeschen.

```
doctl compute volume-action detach <volume-id> <droplet-id>
kubectl delete volumeattachment <name>
```

## Fazit

Ein Helmfile-Deployment dieser Groessenordnung (35+ Charts, SSO/LDAP/Datenbanken) ist deutlich
komplexer als ein einzelnes `helm install` und bringt eigene Fehlerklassen mit sich: Job-Hook-
State bei Retries, Cross-Pod-Volume-Affinity, und - wie hier beobachtet - potenziell chart-
spezifische Ressourcenprobleme, die sich nicht durch simples Hochsetzen von Limits loesen lassen.
Fuer ein Trainings-Lab lohnt sich ggf. der Blick auf die leichtgewichtigeren Alternativen
(K3s-Guide, SCS-Dokumentation von openDesk).
