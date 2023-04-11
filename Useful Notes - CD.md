Useful Notes on Continuous Delivery and Deployment

ArgoCD & ArgoRollouts
---------------------
Check this out...

https://argo-rollouts.readthedocs.io/en/stable/concepts/#concepts

Automated controlled blue-green and canary deployments to production. Embedded tests during deployment process.

- Can't use autopilot because of restrictions on kube-system namespace
- Setup argocd in K8s

kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

- Get initial password

argocd admin initial-password -n argocd
argocd login 35.246.45.171
argocd account update-password
admin/password

- Register your clusters
kubectl config get-contexts -o name
argocd cluster add gke_k8s-play-unique_europe-west2-a_cd-manager

- Create apps using...

kubectl create namespace apps

argocd app create guestbook \
    --dest-name gke_k8s-play-unique_europe-west2-a_cd-manager \
    --repo https://github.com/seventieskid/argocd-example-apps \
    --path guestbook \
    --dest-namespace apps \
    --sync-policy automated

- Deploy the application...

argocd app set guestbook --sync-policy automated

argocd app sync guestbook

https://console.cloud.google.com/gcr/images/heptio-images/global/ks-guestbook-demo

gcr.io/heptio-images/ks-guestbook-demo:0.2

- Delete the application...

argocd app delete guestbook

(Removes it from the cluster)

- Event driven CD via github webhooks (otherwise it's a 3 minute poll from argocd)

- Nice easy way to access the argo API server from outside world:-

kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

Helm
----

argocd app create helm-guestbook \
    --dest-name gke_k8s-play-unique_europe-west2-a_argocd \
    --repo https://github.com/seventieskid/argocd-example-apps \
    --path helm-guestbook \
    --dest-namespace apps \
    --sync-policy automated

That targets a git repo. 

It's also possible to target a HELM repo, and for that, type HELM is selected.

argocd app create helm-sealed-secrets \
    --dest-name gke_k8s-play-unique_europe-west2-a_argocd \
    --repo https://bitnami-labs.github.io/sealed-secrets \
    --helm-chart sealed-secrets \
    --revision 1.16.1 \
    --release-name sealed-secrets-release \
    --dest-namespace apps \
    --sync-policy automated

    (Fails to deploy because UNAUTHORISED on image pull)

Blue-Green
----------

Install argo rollouts on apple, m1 max:-

sudo curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-darwin-arm64
sudo chmod +x ./kubectl-argo-rollouts-darwin-arm64
sudo mv ./kubectl-argo-rollouts-darwin-arm64 ./kubectl-argo-rollouts

kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

argocd app create --name blue-green \
        --dest-name gke_k8s-play-unique_europe-west2-a_argocd \
        --repo https://github.com/seventieskid/argocd-example-apps \
        --dest-namespace apps \
        --path blue-green

argocd app sync blue-green

REM Bump the image version and create a temporary blue/green scenario
Commit a change to the image version in github (01 --> 0.2)

REM Rollout forward manually
kubectl argo rollouts promote blue-green-helm-guestbook -n blue-green

REM Rollback
kubectl argo rollouts undo blue-green-helm-guestbook -n blue-green

Observations
------------

- Out-of-Sync only appears to work pop up on "top level" k8s artefacts like Deployments or Rollouts etc. Not on direct changes replica sets or pods. Works real nice on those top level ones though.
- 3 min poll from argocd to github. No evidence of that, it's rapid. Github webhooks could easily be used to push events to argocd, but it doesn't appear to be necessary.

Pre-Post Sync Webhooks ?? Use for HSBC CR validation ?? RESOURCE HOOKS

- Extensible via it's plugins to reach SCRIPTED config (theoretically no programming needed)

- Onboarding via the app-of-apps paradigm

Initial ArgoCD Bootstrapping - App of Apps (alternative to appset)
------------------------------------------
argocd app create apps \
    --dest-namespace apps \
    --dest-server https://kubernetes.default.svc \
    --repo https://github.com/seventieskid/argocd-example-apps.git \
    --path apps 

argocd app sync -l app.kubernetes.io/instance=apps -n apps

ApplicationSet
--------------
https://github.com/argoproj/argo-cd/tree/master/applicationset/examples

Sync-Waves
----------
Powerful way of making sure that things get synced in the correct sequence by argocd

Notifications
-------------
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-notifications/release-1.0/manifests/install.yaml
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/notifications_catalog/install.yaml

export EMAIL_USER=<fullemailaddress>
export PASSWORD=<special16charapppasswordfromgoogleaccount>
kubectl apply -n argocd -f - << EOF
apiVersion: v1
kind: Secret
metadata:
  name: argocd-notifications-secret
stringData:
  email-username: $EMAIL_USER
  email-password: $PASSWORD
type: Opaque
EOF

kubectl edit cm argocd-notifications-cm -n argocd

apiVersion: v1
data:
  service.email.gmail: "username: $email-username \npassword: $email-password \nhost:
    smtp.gmail.com \nport: 465 \nfrom: $email-username\n"

Can fire notifications if an application becomes out-of-sync between TARGET (what's deployed on K8s) and SOURCE (GitHub).

kubectl edit cm argocd-notifications-cm -n argocd

  template.app-sync-status-outofsync: |
    email:
      subject: Application {{.app.metadata.name}} Out-Of-Sync .
    message: Application {{.app.metadata.name}} is out of sync.

  trigger.on-sync-status-outofsync: | 
    - description: Application status is 'Out Of Sync'
      send:
      - app-sync-status-outofsync
      when: app.status.sync.status == 'OutOfSync'

[Manually add subscription notfication to application]

annotation:
  notifications.argoproj.io/subscribe.on-sync-status-outofsync.gmail=grees@azuron.co.uk

Useful Background
-----------------

https://codefresh.io/learn/continuous-delivery/continuous-delivery-vs-continuous-deployment/

Spinnaker
---------

- Specifically for CD (continuous delivery/deployment)
- Beyond Kubernetes to manage cloud infrastructure
- Built in Automated Canary Analysis and "red/black" or bluegreen support
- Steeper learning curve
- Heavyweight K8s install (compared to ArgoCD) - needs 3 e2-mediums
- GitOps yes
- Needs 3 x e2-mediums to run (argocd is on there too mind you)

Install....    https://github.com/armory/spinnaker-operator

# Pick a release from https://github.com/armory/spinnaker-operator/releases (or clone the repo and use the master branch for the latest development work)
$ mkdir -p spinnaker-operator && cd spinnaker-operator
$ bash -c 'curl -L https://github.com/armory/spinnaker-operator/releases/latest/download/manifests.tgz | tar -xz'
 
# Install or update CRDs cluster wide
$ kubectl apply -f deploy/crds/

# Install operator in namespace spinnaker-operator, see below if you want a different namespace
$ kubectl create ns spinnaker-operator
$ kubectl -n spinnaker-operator apply -f deploy/operator/cluster

kubectl patch svc spinnaker-operator -n spinnaker-operator -p '{"spec": {"type": "LoadBalancer"}}'

# Clone the project
git clone https://github.com/armory/spinnaker-kustomize-patches.git
cd spinnaker-kustomize-patches

# Delete default recipe
rm kustomization.yml

# Create symlink for oss recipe
ln -s ./recipes/kustomization-oss-minimum.yml kustomization.yml

# Create the spinnaker namespace
kubectl create ns spinnaker

# Build the kustomize template and deploy in kubernetes
kustomize build . | kubectl apply -f -

# Watch the install progress, check out the pods being created too!
$ kubectl -n spinnaker get spinsvc spinnaker -w

kubectl patch svc spin-deck -n spinnaker -p '{"spec": {"type": "LoadBalancer"}}'

WAIT 15 mins

- Java based app

- failed to get productive after half a day

- Jenkins like in it's pipeline setup config

- External storage provider for app settings and pipelines (e.g. GCS)

- Manual judgement stage could be useful

- Very simple. Designed for the command line, and hence pipelines

- Seems heavy on kustomize

FluxCD
------

- Flux installed in-situ on a cluster and manages it exclusively
- Must have correct git structure to support multiple clusters or multi-tenancy
- No UI shipped with Flux
- Helm and Kustomize, but no kustomization.yaml present, does usual k8s deploy I think
- Can generate commits from creation of new image versions in container reg. Very nice, but dangerous without auto/testing. Need to add receivers and then a webhook to container reg.
- Easy getting started here: https://fluxcd.io/flux/get-started/
- No opportunity for Automated Continuous Deployment Testing (e.g. Automated Canary Testing)
- GitOps via addition of receivers

brew install fluxcd/tap/flux

export GITHUB_TOKEN=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

flux bootstrap github \
  --owner=seventieskid \
  --repository=my-repository \
  --path=clusters/my-cluster \
  --personal

flux create source git podinfo \
  --url=https://github.com/stefanprodan/podinfo \
  --branch=master \
  --interval=30s \
  --export > ./clusters/my-cluster/podinfo-source.yaml

flux create kustomization podinfo \
  --target-namespace=default \
  --source=podinfo \
  --path="./kustomize" \
  --prune=true \
  --interval=5m \
  --export > ./clusters/my-cluster/podinfo-kustomization.yaml

git add -A && git commit -m "Add podinfo Kustomization"
git push

flux uninstall --namespace=flux-system

Image automation....https://fluxcd.io/flux/guides/image-update/#imagepolicy-examples

export GITHUB_TOKEN=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

flux bootstrap github \
  --components-extra=image-reflector-controller,image-automation-controller \
  --owner=seventieskid \
  --repository=flux-image-updates \
  --branch=main \
  --path=clusters/argocd \
  --read-write-key \
  --personal

flux create image repository podinfo \
--image=ghcr.io/stefanprodan/podinfo \
--interval=1m \
--export > ./clusters/argocd/podinfo-registry.yaml

flux create image policy podinfo \
--image-ref=podinfo \
--select-semver=5.0.x \
--export > ./clusters/argocd/podinfo-policy.yaml

Annotate the deployment manifest like this...

      containers:
        - name: podinfod
          image: ghcr.io/stefanprodan/podinfo:5.0.0 # {"$imagepolicy": "flux-system:podinfo"}

flux create image update flux-system \
--git-repo-ref=flux-system \
--git-repo-path="./clusters/argocd" \
--checkout-branch=main \
--push-branch=main \
--author-name=fluxcdbot \
--author-email=fluxcdbot@users.noreply.github.com \
--commit-template="{{range .Updated.Images}}{{println .}}{{end}}" \
--export > ./clusters/argocd/flux-system-automation.yaml

Tekton CI and CD
----------------

- https://tekton.dev/docs/

- Kubernetes only, in-cluster deployment

- Build, TEST and deploy

- Integration with Skaffold, ArgoCD ?!

- Google backed

- Tekton Catalog

- Install - kubectl apply --filename \
https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml


kubectl apply --filename \
https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
kubectl apply --filename \
https://storage.googleapis.com/tekton-releases/triggers/latest/interceptors.yaml

kubectl apply --filename https://storage.googleapis.com/tekton-releases/dashboard/latest/release.yaml

brew install tektoncd-cli

- Heavy install

- Dashboard really appears to be a read-only tool out of the box. Useful, but question why you'd bother installing it.

- Tekton appears to be a way of creating any kind of generic pipeline without coding, using yaml. Seems flexible, but ultimately doesn't have a USP.

- Tekton is not specifically a continuous delivery tool. Which for this tranche of analysis, is what I'm specifically interested in.

- Examples using kaniko to build images and illustates how to perform a git clone using tekton hub. Limited. Could do with a trigger example.

gcloud projects add-iam-policy-binding k8s-play-unique \
  --member=serviceAccount:tekton-gsa@k8s-play-unique.iam.gserviceaccount.com \
  --role=roles/artifactregistry.reader \
  --role=roles/artifactregistry.writer

gcloud projects add-iam-policy-binding PROJECT_ID \
    --member "serviceAccount:GSA_NAME@GSA_PROJECT.iam.gserviceaccount.com" \
    --role "ROLE_NAME"

kubectl annotate serviceaccount \
tekton-ksa \
iam.gke.io/gcp-service-account=tekton-gsa@k8s-play-unique.iam.gserviceaccount.com

kubectl -n getting-started annotate serviceaccount \
tekton-triggers-example-sa \
iam.gke.io/gcp-service-account=tekton-gsa@k8s-play-unique.iam.gserviceaccount.com

gcloud iam service-accounts add-iam-policy-binding \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:k8s-play-unique.svc.id.goog[apps/tekton-ksa]" \
  tekton-gsa@k8s-play-unique.iam.gserviceaccount.com

gcloud beta container node-pools update default-pool \
  --cluster=cd-manager \
  --workload-metadata-from-node=GKE_METADATA_SERVER \
  --zone=europe-west2-a
  
- Triggers. Complex setup for even the most basic trigger on commit.

- Kaniko, build dockerfiles to images WITHOUT needing docker

- No in-built drift or sync management with Tekton. Possible to build it, but it's not it's primary purpose.