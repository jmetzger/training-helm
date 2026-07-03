# Range 

## Preparation

```
cd
mkdir -p helm-exercises
cd helm-exercises 
helm create range
cd range/templates
rm -f *.yaml
cd ..
nano values.yaml 
```

## Step 1: values.yaml 

```
# unten mit ran schreiben 
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions
```

## Step 2 (Version 1):

```
cd templates
nano cm.yaml
```

```
# nano cm.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
  toppings: |-
    {{- range .Values.pizzaToppings }}
    - {{ . | title | quote }}
    {{- end }}    
```

## Step 3 (Version 2 - works as well) 

  * Accessing the parent scope

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  toppings: |-
    {{- range $.Values.pizzaToppings }}
    - {{ . | title | quote }}
    {{- end }}    
  {{- end }}
```
