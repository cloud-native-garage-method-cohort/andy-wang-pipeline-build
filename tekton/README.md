Created date : 2021.05.26
Activity : Garage Cloud Native Bootcamp
Purpuse : 
    Use the Task Catalog to build a Pipeline that clones a Git repository, runs the tests, creates a Docker image, pushes the image to quay.io
Note : will need to update... starts from Kustomize from now.
-----
[+gitclone+]
 oc apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.3/git-clone.yaml

[+Test-npm+]
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/npm/0.1/npm.yaml

[+buildah+]
Is tool for building Open Container Initiative (OCI) containers.

Ref : https://github.com/cloud-native-garage-method-cohort/ibm-cloud-ci-cd-apac-1/blob/main/tekton/pipeline-build-03-buildah.md
---

[Step1]
$:>  oc apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/buildah/0.1/buildah.yaml 
---

[Step2]
Since our Dockerfile lives at the project root, we only need to include the <image> name and workspace named <source> to add the task to our pipeline:

   - name: build-image
      runAfter:
        - run-tests
      params:
        - name: IMAGE
          value: "$(params.image-repo):$(tasks.clone-repository.results.commit)"
      taskRef:
        kind: Task
        name: buildah
      workspaces:
        - name: source
          workspace: pipeline-shared-data
---

[Step3]
Need for authentication before pushing to Quay.io.

To store our password on the cluster we will use a <docker-registry Kubernetes Secret.> 
"A Secret is an object that contains a small amount of sensitive data such as a password, a token, or a key." (source). To do this, we will use the command line:

$:> export CONTAINER_REGISTRY_SERVER='https://quay.io'
    export CONTAINER_REGISTRY_USER='<your registry user>'
    export CONTAINER_REGISTRY_PASSWORD='<your registry user password>'

$:> kubectl create secret -n <your namespace or project name> docker-registry quay-io-xxx-password \
      --docker-server=$CONTAINER_REGISTRY_SERVER \
      --docker-username=$CONTAINER_REGISTRY_USER \
      --docker-password=$CONTAINER_REGISTRY_PASSWORD
---

[Step4]
* Accessible to the Pipeline. We create a <ServiceAccount> at the command line:

$:> kubectl create sa build-bot
$:> kubectl patch serviceaccount build-bot -p '{"secrets": [{"name": "quay-io-xxx-password"}]}'
$:> kubectl get sa -n <your namespace or project name> build-bot -o yaml

* Modify pipelineRun as follows: ( add serviceAccountName: build-bot in spec)

apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: andreas-kavountzis-pipeline-from-scratch-pipeline-run-
spec:
  serviceAccountName: build-bot
  pipelineRef:
    name: andreas-kavountzis-pipeline-from-scratch-pipeline
  workspaces:
    - name: pipeline-shared-data
      persistentVolumeClaim:
        claimName: andreas-kavountzis-pvc
---

[Step5]
Get permissions

The OpenShift and Kubernetes Objects we introduce are:

    >ImageStream - An image stream and its associated tags provide an abstraction for referencing container images from within OpenShift Container Platform. Read more here.
    >Role - a set of rules for permissions to resources. Only additive and start with zero permissions.
    >RoleBinding - assigns a Role to a given user, group, or service account

apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: andreas-kavountzis-pipeline-from-scratch-imagestream
  namespace: andreas-kavountzis-pipeline-from-scratch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: builder
rules:
  - apiGroups:
      - image.openshift.io
    resources:
      - imagestreams
    verbs:
      - "*"
  - apiGroups:
      - image.openshift.io
    resources:
      - "imagestreams/layers"
    verbs:
      - "*"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: builder
subjects:
  - kind: ServiceAccount
    name: build-bot
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: builder
---


[+Kustomize+]
THE NEXT : 

    We want to do is construct Kubernetes configuration. We use Kustomize.

My understanding, if we want to config K8s, then it must be configuring deployment,service,routing...etc

[Step1]
In the repository we are deploying <means that we need to generate> there is a <k8s directory> with the requisite files for Kustomize:

1. k8s/route.yaml
2. k8s/deployment.yaml
3. k8s/kustomization.yaml
4. k8s/service.yaml

[Step2]
Create Kustomize <Task> : 

apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: kustomize-build
spec:
  workspaces:
    - name: source
      description: contains the cloned git repo
  steps:
    - name: kustomize-build
      image: docker.io/enzobarrett/kustomize:latest
      script: |
        #!/bin/sh
        cd source
        kustomize build k8s/ > manifests.yaml
        if [ -f manifests.yaml ]; then
          echo "manifests.yaml successfully generated"
        else
          echo "ERROR: manifests.yaml not generated"
          exit 1
        fi

[Step3]
Create a script for <deployment via kubectl>:

apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: test-deploy
spec:
  params:
    - name: app-namespace
      description: namespace that deployment will be tested in
    - name: app-name
      description: the name of the app
  workspaces:
    - name: source
      description: contains the cloned git repo
  results:
    - name: service-port
    - name: resource-type
  steps:
    - name: deploy
      image: docker.io/enzobarrett/kubectl:latest
      script: |
        #!/bin/sh
        cd source
        set -e
        APP_NAMESPACE="$(params.app-namespace)"

        echo -e "Deploying into: ${APP_NAMESPACE}"
        kubectl apply -n ${APP_NAMESPACE} -f ./manifests.yaml --validate=false > results.out
        cat results.out
    - name: verify-deploy
      image: docker.io/enzobarrett/kubectl:latest
      script: |
        #!/bin/sh
        cd source
        set -e
        APP_NAMESPACE="$(params.app-namespace)"
        APP_NAME="$(params.app-name)"

        cat results.out | \
          grep -E "deployment|statefulset|integrationserver|queuemanager" | \
          sed "s/deployment.apps/deployment/g" | \
          sed "s/statefulset.apps/statefulset/g" | \
          sed "s/configured//g" | \
          sed "s/created//g" | \
          sed "s/unchanged//g" | while read target; do
            echo "Waiting for rollout of ${target} in ${APP_NAMESPACE}"
            kubectl rollout status -n ${APP_NAMESPACE} ${target}
          done


[Step4]
Creat these Task into <Pipeline>:

 - name: kustomize-build
      runAfter:
        - build-image
      params:
        - name: image-with-tag
          value: "quay.io/andreask/express-sample-app=$(params.image-repo):$(tasks.clone-repository.results.commit)"
      taskRef:
        kind: Task
        name: kustomize-build
      workspaces:
        - name: source
          workspace: pipeline-shared-data
    - name: test-deploy
      runAfter:
        - kustomize-build
      params:
        - name: app-namespace
          value: andreas-kavountzis-pipeline-from-scratch
        - name: app-name
          value: andreas-kavountzis-pipeline-from-scratch
      taskRef:
        kind: Task
        name: test-deploy
      workspaces:
        - name: source
          workspace: pipeline-shared-data

[Step5]
Permisson needed, This time will be <developer>:

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
rules:
  - apiGroups:
      - apps
    resources:
      - deployments
    verbs:
      - get
      - create
      - list
      - patch
      - watch
  - apiGroups:
      - route.openshift.io
    resources:
      - routes
    verbs:
      - get
      - create
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - patch

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployer
subjects:
  - kind: ServiceAccount
    name: build-bot
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: deployer

[Step6]
 By using the docker-registry credentials we set up earlier (aka quay-io-andreas-kavountzis-password), sticking with the command line:

$:> kubectl patch serviceaccount build-bot -p '{"imagePullSecrets": [{"name": "quay-io-andreas-kavountzis-password"}]}'
