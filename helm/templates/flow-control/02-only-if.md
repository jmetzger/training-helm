# If 

## Prepare (if not done yet)

```
helm create iftest
cd iftest/templates
rm -f *.yaml
```


## Step 2: values-file erweitern 

```
nano ../values.yaml
```

```
# Adjust values.yaml file accordingly
favorite:
  food: PIZZA
  drink: coffee
```

## Step 3: Probably the best solution 

```
nano cm.yaml
```

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- if eq .Values.favorite.drink "coffee"}}
  {{ "mug: true" }}
  {{- end }}
```

```
helm template ..
```

## Step 4: change favorite drin 

```
nano ../values.yaml
```

```
# Adjust values.yaml file accordingly
favorite:
  food: PIZZA
  drink: tea 
```

```
helm template ..
```


## Reference

  * https://helm.sh/docs/chart_template_guide/control_structures/
