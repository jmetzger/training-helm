# Example gitlab ci-cd pipeline Kubernetes 

## Step 1: Import gitlab - repo and pipeline 

```
1. Create new repo on gitlab -> do import
https://gitlab.com/jmetzger/training-helm-chart-kubernetes-gitlab-ci-cd.git
 

```

## Step 2: in .gitlab-ci.yaml

```
von ---_>
APP_NAME: my-first-app

in ----->
APP_NAME: my-first-app<namenskuerzel>
```

## Step 3: Add your KUBECONFIG as Variable (type: File) to Variables 

  * https://gitlab.com/jmetzger/training-helm-chart-kubernetes-gitlab-ci-cd/-/settings/ci_cd#js-cicd-variables-settings

![image](https://github.com/user-attachments/assets/b5168cf3-dd74-4d86-becf-e807985dd471)

## Step 4: Create a pipeline for deployment 

```
stages:          # List of stages for jobs, and their order of execution
  - deploy

variables:
  APP_NAME: my-first-app

deploy:
  stage: deploy
  image: 
    name: alpine/stages:          # List of stages for jobs, and their order of execution
  - deploy

variables:
  APP_NAME: my-first-app

deploy:
  stage: deploy
  image: 
    name: alpine/k8s:1.31.13
# Important to unset entrypoint 
    entrypoint: [""]
  script:
    - ls -la
    - cd; mkdir .kube; cd .kube; cat $KUBECONFIG_SECRET > config; ls -la;
    - cd $CI_PROJECT_DIR; helm upgrade ${APP_NAME} ./charts/my-app --install --namespace ${APP_NAME} --create-namespace -f ./config/values.yaml
  rules:
    - if: $CI_COMMIT_BRANCH == 'master'
      when: always

# Important to unset entrypoint 
    entrypoint: [""]
  script:
    - ls -la
    - cd; mkdir .kube; cd .kube; cat $KUBECONFIG_SECRET > config; ls -la;
    - cd $CI_PROJECT_DIR; helm upgrade ${APP_NAME} ./charts/my-app --install --namespace ${APP_NAME} --create-namespace -f ./config/values.yaml
  rules:
    - if: $CI_COMMIT_BRANCH == 'master'
      when: always

```


## Reference: Example Project (Public)

  * https://gitlab.com/jmetzger/training-helm-chart-kubernetes-gitlab-ci-cd
