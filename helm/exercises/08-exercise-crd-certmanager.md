# Exercise cert-manager - CRD's 

## Schritt 1: repo hinzufügen 
```
helm repo add jetstack https://charts.jetstack.io
```

## Schritt 2: chart runterladen und entpacken (zum Gucken) 

```
helm pull jetstack/cert-manager
ls -la
helm pull jetstack/cert-manager --untar
ls -la
cd cert-manager
ls -la
cd templates
ls -la crds.yaml 
```

## Schritt 3: Installieren 

```
cd 
mkdir cm-values
cd cm-values
nano values.yaml
```

```
crds:
  enabled: true
```

```
helm install cert-manager jetstack/cert-manager --namespace cert-manager-<namenskuerzel> --create-namespace -f values.yaml
kubectl -n cert-manager-<namenskuerzel> get all
```

## CRD's da ? 

```
kubectl get crds | grep cert
```


## Deinstallieren 

```
helm -n cert-manager-<namenskuerzel> uninstall cert-manager
```

## CRD's noch da ? 

```
kubectl get crds | grep cert 
```


## CRD's händisch löschen 

```
# Variante 1
kubectl delete crd certificaterequests.cert-manager.io certificates.cert-manager.io  challenges.acme.cert-manager.io  clusterissuers.cert-manager.io  issuers.cert-manager.io orders.acme.cert-manager.io
```
