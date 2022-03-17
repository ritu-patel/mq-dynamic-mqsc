# Dynamic MQSC

<img src="/readme-images/dynamic-mqsc-pipeline.png" width="55%" height="25%">

**Description**: This repo has a similar folder structure as our [gitops repo](https://github.com/IBMMQAutomation/mq-pipeline) for 5 different evironments such as DEV, SIT, UAT, PTE and PRD. Each environment has different queue manager folders and each queue manager folder has dynamic.mqsc which contains mqsc changes. Notice that `prd` and `pte` doesn't have dynamic mqsc files but have `.env` files because we will be promoting UAT mqsc to PTE then PRD

```
├── dev
│   ├── qm01
│   │   └── dynamic.mqsc
│   │   └── .env
│   └── qm02
│       └── dynamic.mqsc
│       └── .env
└── uat
    ├── qm01
    │   └── dynamic.mqsc
    │   └── .env
    └── qm02
        └── dynamic.mqsc
        └── .env

├── prd
│   ├── qm01
│       └── .env
│   └── qm02
│       └── .env
├── pte
│   ├── qm01
│       └── .env
│   └── qm02
│       └── .env

```

- Suggested access for this repo: developer and admins

## Steps

### 1. Copy this repo to your enterprise repository

### 2. Make necessary changes to MQSC files

### 3. Create Tekton Pipeline and Tasks

- Create `pipeline.yaml` with the following content and replace necesary parameters
<details>
<summary>pipeline.yaml</summary>

```
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: dynamic-mqsc
  namespace: mq
spec:
  workspaces:
    - name: source

  params:
    - name: GIT_SECRET
      default: "git-credentials"
    - name: source-dir
      default: /source
    - name: git-cli-image
      default: docker.io/alpine/git:v2.26.2@sha256:23618034b0be9205d9cc0846eb711b12ba4c9b468efdd8a59aac1d7b1a23363f
      # mqsc repo
    - name: git-mqsc-url
      default: github.com/IBMMQAutomation/dynamic-mqsc.git
    - name: mqsc-git-branch
      default: master
    - name: mqsc-subdirectory
      default: dynamic-mqsc
      # kustomize repo
    - name: kustomize-subdirectory
      default: mq-pipeline
    - name: git-kustomize-url
      default: github.com/IBMMQAutomation/mq-pipeline.git
    - name: kustommize-git-branch
      default: main
  tasks:
    - name: git-task-dev
      params:
        - name: git-mqsc-clone
          value: "git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@$(params.git-mqsc-url)"
        - name: mqsc-subdirectory
          value: $(params.mqsc-subdirectory)
        - name: GIT_SECRET
          value: $(params.GIT_SECRET)
        - name: source-dir
          value: $(params.source-dir)
        - name: git-cli-image
          value: $(params.git-cli-image)
        - name: kustomize-subdirectory
          value: $(params.kustomize-subdirectory)
        - name: git-kustomize-clone
          value: "git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@$(params.git-kustomize-url)"
        - name: kustommize-git-branch
          value: $(params.kustommize-git-branch)
      taskRef:
        kind: Task
        name: git-task-dev
      workspaces:
        - name: source
          workspace: source
```

</details>

- Create `git-task.yaml` with the following content
<details>
<summary>git-task.yaml</summary>

```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: git-task-dev
  namespace: mq
spec:
  params:
    - name: git-mqsc-clone
    - name: mqsc-subdirectory
    - name: GIT_SECRET
    - name: source-dir
    - name: git-cli-image
    - name: kustomize-subdirectory
    - name: git-kustomize-clone
    - name: kustommize-git-branch

  workspaces:
    - name: source
      mountPath: $(params.source-dir)

  steps:
    - name: git-pull
      image: $(params.git-cli-image)
      env:
        - name: GIT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: $(params.GIT_SECRET)
              key: password
              optional: true
        - name: GIT_USERNAME
          valueFrom:
            secretKeyRef:
              name: $(params.GIT_SECRET)
              key: username
              optional: true
        - name: GIT_EMAIL
          valueFrom:
            secretKeyRef:
              name: $(params.GIT_SECRET)
              key: email
              optional: true
      resources: {}
      script: |
        git config --global user.email "$GIT_EMAIL"
        git config --global user.name "$GIT_USERNAME"
        $(params.git-mqsc-clone)
        $(params.git-kustomize-clone)
        rm -rf $(params.mqsc-subdirectory)/.git
        cd $(params.mqsc-subdirectory)
        cp -u -a uat/. pte
        cp -u -a uat/. prd
        cd ..
        for env in $(params.mqsc-subdirectory)/*
        do
          for qm in $env/*
          do
              source $qm/.env
              eval "echo \"$(cat $qm/dynamic.mqsc)\"" > $qm/dynamic.mqsc
              rm $qm/.env
          done
        done
        cp -R $(params.mqsc-subdirectory)/dev $(params.kustomize-subdirectory)/
        cp -R $(params.mqsc-subdirectory)/sit $(params.kustomize-subdirectory)/
        cp -R $(params.mqsc-subdirectory)/uat $(params.kustomize-subdirectory)/
        cp -R $(params.mqsc-subdirectory)/pte $(params.kustomize-subdirectory)/
        cp -R $(params.mqsc-subdirectory)/prd $(params.kustomize-subdirectory)/
        cd $(params.kustomize-subdirectory)
        git add .
        git commit -m "tekton added dynamic mqsc"
        git push
      workingDir: $(params.source-dir)
```

</details>

### 4. Apply tekton pipeline on openshift

Now, lets apply both of these files.

```
oc apply -f pipeline.yaml
oc apply -f git-task.yaml
```

Make sure you do not check in above files in this repo. Git-task script goes through every folder/every QM folder to check for .env files so the script will error out if you check in yaml files in this repo

### 5. Run the pipeline

Recommended: you can add webhook to trigger the pipeline run

Next Steps:

- Gitops (ArgoCD) - https://github.com/ritu-patel/mq-gitops
