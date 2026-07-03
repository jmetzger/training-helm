# Named Template Exercise 

##  

## Step 0: create project 

```
cd
mkdir -p helm-exercises 
cd helm-exercises
helm create range 
cd range
```


## Step 1: Put in file starting _helpers.tpl 

```
cd templates
rm -fR *.yaml 
nano _helpers.tpl
```

  * Das am Ende einfügen -> 

```
{{/* Definiere ein named template namens "mychart.containerPort" */}}
{{- define "range.containerPort" -}}
- name: http
  containerPort: 80
  protocol: TCP
{{- end }}
```

## Step 2: templates/deployment.yaml 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx 
    spec:
      containers:
        - name: web
          image: "nginx:latest"
          ports:
            {{ include "range.containerPort" . }}
```

```
helm template ..
```

```
# Problem, Zeile 1 o.k., nächste Zeile nicht richtig eingerückt
```

![image](https://github.com/user-attachments/assets/8bfe07d5-59f9-4fc6-87d0-3aeff93c2acb)



## Step 3: Einrückung richtig setzen 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx 
    spec:
      containers:
        - name: web
          image: "nginx:latest"
          ports:
            {{- include "range.containerPort" . | nindent 12 }}
```

```
helm template ..
```
