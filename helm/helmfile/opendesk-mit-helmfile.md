# Praxisbeispiel: openDesk mit Helmfile ausrollen

## Hintergrund

openDesk (ZenDiS/BMI) ist kein einzelnes Helm-Chart, sondern 35+ Charts (Keycloak/Nubus als IAM,
Nextcloud, OX App Suite, OpenProject, XWiki, Element/Synapse, Jitsi, Collabora, CryptPad,
Mailstack, Datenbanken, ...), orchestriert ueber [Helmfile](https://helmfile.readthedocs.io/).
Kein `helm install opendesk` - stattdessen `helmfile apply -e <environment> -n <namespace>`.

Dieses Dokument beschreibt den Rollout auf DigitalOcean als Referenz fuer aehnliche
Helmfile-basierte Multi-Chart-Deployments. **Ergebnis vorweg:** Auf einem Managed-Cluster (DOKS)
blieb der LDAP-Server (siehe Troubleshooting) instabil - der erfolgreiche, stabile Rollout lief
am Ende auf einem selbstverwalteten 3-Node-kubeadm-Cluster (Kubernetes 1.32.13 auf Ubuntu-24.04-
Droplets). Grund und Fix sind unten dokumentiert.

## Voraussetzungen

| Tool | Version | Hinweis |
|---|---|---|
| Kubernetes | >= 1.24, CNCF-zertifiziert | getestet mit 1.35.5 (DOKS, LDAP instabil) und 1.32.13 (kubeadm, stabil - siehe Troubleshooting) |
| Helm | >= 3.17.3, NICHT 3.18.0 / 3.20.1 | bekannte Helm-Bugs in diesen Versionen |
| Helmfile | >= 1.0.0 | |
| Helm Diff Plugin | >= 3.11.0 | `helm plugin install` |
| yq | >= 4.52.4 | |
| Ingress Controller | haproxy-ingress.github.io (Default seit openDesk >= 1.13) | ingress-nginx geht technisch auch (Override noetig), ist aber upstream deprecated und nicht mehr der getestete Pfad |
| cert-manager | mit CRDs | fuer Zertifikate |
| Volume Provisioner | ReadWriteOnce (min.) | z.B. `do-block-storage` |

Hardware-Minimum laut openDesk-Doku fuer eine Eval-Instanz: 12 CPU-Cores, 32 GB RAM.
In der Praxis (siehe Troubleshooting unten) sollte man deutlich mehr RAM einplanen.

## Ablauf

### Schritt 1: Ingress-Controller und cert-manager installieren

openDesk >= v1.13 verwendet standardmaessig `ingressClassName: haproxy` - das sollte man auch so
lassen, nicht auf ingress-nginx umsteigen. Ingress-nginx laesst sich zwar technisch per Override
anbinden (Erfahrung aus der Praxis: DNS, Zertifikate und Routing funktionieren grundsaetzlich
auch damit), es ist aber upstream deprecated und **nicht der von openDesk getestete Pfad** - man
weicht damit unnoetig vom Standard-Setup ab, ohne einen echten Vorteil zu haben.

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
die Default-Dateien in `helmfile/environments/default/` anfassen) - z.B. fuer
Ressourcen-Anpassungen (siehe Troubleshooting). Der Ingress muss dafuer nicht angefasst werden,
da `haproxy` bereits Default ist.

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

### `ums-ldap-server-primary` crasht mit OOMKilled (ungeloest auf DOKS)

Der Nubus-LDAP-Server (Univention-basiert, Chart `nubus`) kann beim Start extrem viel Speicher
allozieren - in einem Test: Wachstum von ~100MB auf ueber 11 GB in unter 3 Sekunden, live per
`kubectl exec ... cat /sys/fs/cgroup/memory.current` verfolgt. Der Standard-Wert des Charts
(`nubusDevelopmentResources`) liegt bei nur 1Gi Limit; Univention selbst nennt fuer Produktion
2048Mi als Default - der Hersteller geht also nicht von einem hohen Speicherbedarf aus.

Folgende Ursachen wurden systematisch ausgeschlossen:

- **Zu wenig RAM:** Getestet mit Limits von 1Gi bis 40Gi (dedizierter `m-8vcpu-64gb`-Node) - das
  Wachstum floppt bei keiner Grenze ab, sondern waechst kontinuierlich weiter (live gemessen:
  24GB -> 38GB in ~30 Sekunden), bis es die jeweilige Grenze erreicht und gekillt wird. Kein
  einmaliger Init-Peak, der irgendwann plateaut.
- **Alte/korrupte Daten:** Frische, leere PVCs (nach komplettem `helm uninstall` inkl.
  PVC-Loeschung) zeigen dasselbe Verhalten. Die eigentlichen LDAP-Datenverzeichnisse
  (`/var/lib/univention-ldap/ldap/`) blieben ueber alle Versuche hinweg leer, der Translog wuchs
  nie ueber 32KB - das Problem haengt nicht von angesammeltem Datenvolumen ab.
- **Chart-/openDesk-Versions-Regression:** Mit openDesk v1.16.1 (nubus 1.20.1) UND v1.14.3
  (nubus 1.19.1) reproduziert - identisches Verhalten in zwei unabhaengigen Versionskombinationen.

Ein echter (und notwendiger) Bug wurde nebenbei trotzdem gefunden: `ums-ldap-notifier` und
`ums-ldap-server-primary` teilen sich ein RWO-Volume und muessen daher zwingend auf demselben
Node laufen (der Chart definiert dafuer eine `podAffinity`). Wird der `ldap-server-primary`-Pod
manuell neu gestartet (z.B. `kubectl delete pod`), kann der Scheduler ihn auf einen anderen Node
setzen als den Notifier -> `Multi-Attach error for volume ... already used by pod(s)
ums-ldap-notifier-0`. Fix: betroffenen Notifier-Pod ebenfalls loeschen, damit er sich per
Required-Affinity wieder neben dem Primary einordnet.

```
kubectl -n opendesk delete pod ums-ldap-notifier-0
```

**Geloest.** Ursache war ein Kubernetes-cgroup-v2-Verhalten: `memory.oom.group` (seit K8s 1.28
aktiv) killt bei cgroup v2 den **kompletten Container**, sobald irgendein einzelner Prozess
darin kurzzeitig OOM geht - auch bei einer voellig harmlosen, transienten Speicherspitze, wie
sie LMDB (das Speicherformat von OpenLDAP) beim Start durch memory-mapped I/O erzeugt. Seit
K8s 1.32 gibt es das Kubelet-Flag `singleProcessOOMKill`, das genau das verhindert (nur der
einzelne Prozess wird gekillt, nicht der ganze Container) - Default ist aber weiterhin "aus"
fuer cgroup v2.

Da DOKS als Managed-Service keine Kubelet-Konfiguration erlaubt, war das dort nicht loesbar.
Auf einem selbstverwalteten kubeadm-Cluster (3 Droplets, Kubernetes 1.32.13, `kubeadm init`
mit einer `KubeletConfiguration` inkl. `singleProcessOOMKill: true`) lief `ums-ldap-server-primary`
sofort stabil: tatsaechlicher Speicherbedarf liegt bei ~86 MB (!) statt der zuvor gemessenen 40GB+.
Bestaetigt der urspruenglichen Vermutung: der "hohe Speicherbedarf" war nie real, sondern ein
Kill-Artefakt der cgroup-v2-Policy.

```
# kubeadm-config.yaml (Auszug) - vor "kubeadm init" setzen, gilt dann automatisch
# fuer alle spaeter beitretenden Nodes (aus der kubelet-config ConfigMap)
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
singleProcessOOMKill: true
```

**Praktische Konsequenz:** Fuer openDesk (bzw. jeden anderen LMDB/mmap-lastigen Workload) auf
Kubernetes >= 1.28 mit cgroup v2 ist ein Managed-Kubernetes-Anbieter, der keine Kubelet-Config
erlaubt (wie DOKS), ungeeignet - zumindest ohne diesen Fix upstream im Chart/Image. Alternativen:
selbstverwalteter Cluster (kubeadm/kubespray, wie hier), oder ein Managed-Angebot, das
Kubelet-Extra-Args/-Config zulaesst.

### `helmfile apply` erneut ausfuehren startet Job-Hooks (z.B. Schema-Ladejobs) neu

Jobs, die als Helm-Hooks definiert sind (z.B. `ums-stack-data-ums`, `ums-provisioning-register-
consumers`, `ums-keycloak-bootstrap`), laufen bei **jedem** `helmfile apply` erneut, sobald der
zugehoerige Release ein Diff hat (auch ein voellig harmloses, wie ein Secret-Rotationswert). Das
kann fuer ein paar Minuten kurzzeitig 401/503-Fehler bei abhaengigen Consumern verursachen (z.B.
`ums-portal-frontend`, `ums-provisioning-udm-transformer`, `ox-connector`), bis die Hooks
durchgelaufen sind und die Subscriptions neu registriert wurden. Kein Bug, nur bei einem erneuten
`helmfile apply` auf ein bereits laufendes System zu beachten - i.d.R. loest es sich innerhalb
weniger Minuten von selbst; falls nicht, betroffene Pods einmal `kubectl delete pod` neu starten.

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
State bei Retries, Cross-Pod-Volume-Affinity und - am aufwendigsten zu diagnostizieren -
Kubernetes-Plattform-Verhalten (cgroup v2 OOM-Handling), das mit bestimmten Workloads
(memory-mapped I/O wie LMDB) kollidiert.

Auf einem DigitalOcean-Managed-Kubernetes-Cluster (DOKS) blieb der LDAP-Server trotz umfangreicher
Diagnose (Ressourcen, PVCs, Node-Affinity, Chart-Versionen) instabil, weil DOKS als Managed-Service
keinen Zugriff auf die Kubelet-Konfiguration erlaubt. Der Wechsel auf einen selbstverwalteten
kubeadm-Cluster (3 Droplets, `singleProcessOOMKill: true`) - genau der Weg, gegen den openDesk
selbst testet (kubespray-basiert) - hat das Problem sofort behoben.

**Praxis-Lehre:** Bei mysterioesen OOM-Kills mit scheinbar absurd hohem, nicht plateauendem
Speicherbedarf lohnt sich der Blick auf die cgroup-v2-OOM-Policy des Clusters, bevor man Limits
immer weiter hochdreht - insbesondere bei Workloads mit memory-mapped I/O (LMDB, RocksDB, u.ae.)
und auf Managed-Kubernetes-Angeboten ohne Kubelet-Zugriff.

Fuer ein Trainings-Lab ohne diesen Anspruch lohnt sich ggf. trotzdem der Blick auf die
leichtgewichtigeren Alternativen (K3s-Guide, SCS-Dokumentation von openDesk).

**Endergebnis:** Auf dem kubeadm-Cluster laeuft openDesk (26 Helm-Releases, kein einziger dauerhaft
fehlgeschlagen) stabil, das Portal ist unter der konfigurierten Domain per HTTPS mit gueltigem
Let's-Encrypt-Zertifikat erreichbar, `ums-ldap-server-primary` blieb ueber Stunden durchgehend
`Ready` ohne einen einzigen Neustart.
