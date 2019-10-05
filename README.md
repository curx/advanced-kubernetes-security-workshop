# Advanced Kubernetes Security Workshop

This repository contains the sample code and scripts for Kubernetes Security
Best Practices, which is a guided workshop to showcase some of Kubernetes and
GKE's best practices with respect to security.

## Setup

1.  The sample code and scripts use the following environment variables. You
    should set these to your associated values:

    ```text
    PROJECT_ID="..."
    ```

1.  Configure the project:

    ```shell
    ./bin/01-create-service-account.sh
    ```

1.  Create a GKE cluster which will run as the attached service account:

    ```shell
    ./01-create-cluster.sh
    ```

1.  Show that GKE nodes are not publicly accessible:

    ```text
    gcloud compute instances list --project $PROJECT_ID
    ```

1.  SSH into the bastion host:

    ```text
    gcloud compute ssh my-bastion \
      --project $PROJECT_ID \
      --zone us-central1-b
    ```

1.  Install `kubectl` command line tool:

    ```text
    sudo apt-get -yqq install kubectl
    ```

1.  Authenticate to talk to the GKE cluster:

    ```text
    gcloud container clusters get-credentials my-cluster \
      --region us-central1
    ```

1.  Explore cluster:

    ```text
    kubectl get po

    kubectl get po -n kube-system
    ```


### Audit Logging

1.  Enable system-level audit logs:

    ```text
    curl -sf https://raw.githubusercontent.com/GoogleCloudPlatform/k8s-node-tools/master/os-audit/cos-auditd-logging.yaml | kubectl apply -f -
    ```

    Events will show up as "linux-auditd" events in Stackdriver under "GCE VM Instance".

1.  To disable system audit logs:

    ```text
    kubectl delete ns cos-auditd
    ```


### Network Policy

1.  Deploy and expose an nginx container:

    ```text
    kubectl create deployment nginx --image nginx
    ```

    ```text
    kubectl expose deployment nginx --port 80
    ```

1.  Exec into a shell container and show that nginx is accessible:

    ```text
    kubectl run --generator run-pod/v1 busybox --rm -it --image busybox /bin/sh
    ```

    ```text
    wget --spider --timeout 2 nginx
    ```

1.  Create a network policy that restricts access to the nginx pod:

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: access-nginx
    spec:
      podSelector:
        matchLabels:
          app: nginx
      ingress:
      - from:
        - podSelector:
            matchLabels:
              can-nginx: "true"
    ```

    Apply this with `kubectl apply -f`.

1.  Exec into a shell container and show that nginx is no longer accessible:

    ```text
    kubectl run --generator run-pod/v1 busybox --rm -it --image busybox /bin/sh
    ```

    ```text
    wget --spider --timeout 2 nginx
    ```

1.  Start pod with label and try again:

    ```text
    kubectl run --generator run-pod/v1 busybox --rm -it --labels "can-nginx=true" --image busybox /bin/sh
    ```

    ```text
    wget --spider --timeout 2 nginx
    ```

1.  Delete the nginx deployment:

    ```text
    kubectl delete deployment nginx
    ```

    ```text
    kubectl delete svc nginx
    ```

    ```text
    kubectl delete networkpolicy access-nginx
    ```

### Pod Security Policy

1.  Create a psp that prevents pods from running as root:

    ```yaml
    apiVersion: extensions/v1beta1
    kind: PodSecurityPolicy
    metadata:
      name: restrict-root
    spec:
      privileged: false
      runAsUser:
        rule: MustRunAsNonRoot
      seLinux:
        rule: RunAsAny
      fsGroup:
        rule: RunAsAny
      supplementalGroups:
        rule: RunAsAny
      volumes:
      - '*'
    ```

    Apply this with `kubectl apply -f`.

1.  Update the cluster to start enforcing psp:

    ```text
    gcloud beta container clusters update my-cluster \
        --project $PROJECT_ID \
        --region us-central1 \
        --enable-pod-security-policy
    ```

    Note: this process can take many minutes on an existing cluster.


### Container Security Context

1.  First, demonstrate that a container will run as root unless otherwise
    specified:

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: demo
    spec:
      containers:
      - name: demo
        image: busybox
        command: ["sh", "-c", "sleep 1h"]
    ```

    Apply this with `kubectl apply -f`.

1.  Show that the container is running as root:

    ```text
    kubectl exec -it demo /bin/sh
    ```

    ```text
    ps
    # ...

    id
    uid=0(root) gid=0(root) groups=10(wheel)

    touch foo
    # succeeds
    ```

    ```text
    kubectl delete po demo
    ```


1.  Create a container with a securityContext:

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: demo
    spec:
      securityContext:
        runAsUser: 1000
        runAsGroup: 2000
        fsGroup: 3000

      containers:
      - name: demo
        image: busybox
        command: ["sh", "-c", "sleep 1h"]
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
    ```

    Apply this with `kubectl apply -f`.

1.  Show that the container is running as an unprivileged user:

    ```text
    kubectl exec -it demo /bin/sh
    ```

    ```text
    ps
    # ...

    id
    uid=1000 gid=2000

    touch foo
    Read-only file system
    ```

    ```text
    kubectl delete po demo
    ```

1.  Create a container with a apparmor, seccomp, and selinux options:

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: demo
      annotations:
        seccomp.security.kubernetes.io/demo: runtime/default
        container.apparmor.security.kubernetes.io/demo: runtime/default
    spec:
      securityContext:
        runAsUser: 1000
        runAsGroup: 2000
        fsGroup: 3000
        seLinuxOptions:
          level: "s0:c123,c456"

      containers:
      - name: demo
        image: busybox
        command: ["sh", "-c", "sleep 1h"]
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
    ```

    Apply this with `kubectl apply -f`.

    Delete it when you're done:

    ```text
    kubectl delete po demo
    ```

### Sandbox

1.  Deploy a container:

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: demo
    spec:
      securityContext:
        runAsUser: 1000
        runAsGroup: 2000

      containers:
      - name: demo
        image: busybox
        command: ["sh", "-c", "sleep 1h"]
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
    ```

    Apply this with `kubectl apply -f`.

1.  Show that the container is not running under gvisor:

    ```text
    kubectl get po demo -o yaml
    # runtimeClassName should not be present
    ```

1.  Delete the pod and redeploy with gvisor:

    ```text
    kubectl delete po demo
    ```

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: demo
    spec:
      runtimeClassName: gvisor
      securityContext:
        runAsUser: 1000
        runAsGroup: 2000

      containers:
      - name: demo
        image: busybox
        command: ["sh", "-c", "sleep 1h"]
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
    ```

    Apply this with `kubectl apply -f`.

1.  Show that the container is running under gvisor:

    ```text
    kubectl get po demo -o yaml
    # runtimeClassName should be gvisor
    ```

    ```text
    kubectl delete po demo
    ```


### Workload Identity

Suppose I want to give a Kubernetes service account permissions to talk to a Google Cloud API. I can do this using Workload Identity!

Note that IAM is eventually consistent, so permission changes on the GCP service
account will not be immediately available to pods.

1.  Ensure the `--identity-namespace` flag was passed into the cluster.

1.  Create a Google service account which will be mapped to a Kubernetes Service
    Account shortly:

    ```text
    gcloud iam service-accounts create my-gcp-sa \
      --project $PROJECT_ID
    ```

1.  Give the Google service account the ability to mint tokens:

    ```text
    gcloud projects add-iam-policy-binding $PROJECT_ID \
      --role roles/iam.serviceAccountTokenCreator \
      --member "serviceAccount:my-gcp-sa@${PROJECT_ID}.iam.gserviceaccount.com"
    ```

1.  Give the Google service account viewer permissions (so we can test them
    later inside a pod):

    ```text
    gcloud projects add-iam-policy-binding $PROJECT_ID \
      --role roles/viewer \
      --member "serviceAccount:my-gcp-sa@${PROJECT_ID}.iam.gserviceaccount.com"
    ```

1.  Create a Kubernetes Service Account:

    ```text
    kubectl create serviceaccount my-k8s-sa
    ```

1.  Allow the KSA to use the GSA:

    ```text
    gcloud iam service-accounts add-iam-policy-binding \
      --project $PROJECT_ID \
      --role roles/iam.workloadIdentityUser \
      --member "serviceAccount:${PROJECT_ID}.svc.id.goog[default/my-k8s-sa]" \
      my-gcp-sa@${PROJECT_ID}.iam.gserviceaccount.com
    ```

    ```text
    kubectl annotate serviceaccount my-k8s-sa \
      iam.gke.io/gcp-service-account=my-gcp-sa@${PROJECT_ID}.iam.gserviceaccount.com
    ```

1.  Deploy a pod with the attached service account:

    ```text
    kubectl run -it --rm \
      --generator run-pod/v1 \
      --image google/cloud-sdk \
      --serviceaccount my-k8s-sa \
      demo
    ```

    ```text
    gcloud auth list
    ```

    ```text
    gcloud compute instances list
    ```

    ```text
    gcloud compute instances create foo --zone us-central1-b # fails
    ```


### Vulnerability Scanning and Binary Authorization

Okay, so we are scanning images, but what prevents someone from deploying a
container image that has not been scanned? Binary authorization enables admins
to restrict the container images that run on the platform by requiring
verification via attestors.

1.  Set some environment variables for easier access later:

    ```text
    export PROJECT_NUMBER="$(gcloud projects describe "${PROJECT_ID}" --format 'value(projectNumber)')"
    ```

1.  Allow GKE to pull images from GCR:

    ```text
    gsutil iam ch serviceAccount:my-node-sa@${PROJECT_ID}.iam.gserviceaccount.com:objectViewer gs://artifacts.${PROJECT_ID}.appspot.com
    ```

1.  Allow Cloud Build to deploy to GKE:

    ```text
    gcloud projects add-iam-policy-binding $PROJECT_ID \
      --member "serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com" \
      --role roles/container.developer
    ```

1.  Create and export PGP keys that will sign the attestation:

    ```text
    gpg --batch --gen-key <(
      cat <<- EOF
        Key-Type: RSA
        Key-Length: 2048
        Name-Real: "vulnz-attestor"
        Name-Email: "vulnz-attestor@example.com"
        %commit
    EOF
    )
    ```

    ```text
    gpg --armor --export vulnz-attestor@example.com > vulnz-attestor.asc
    gpg --armor --export-secret-keys vulnz-attestor@example.com > vulnz-attestor.gpg
    echo "password" > vulnz-attestor.pass
    gpg --list-secret-keys | grep -B1 vulnz-attestor | head -n1 | awk '{print $1}' > vulnz-attestor.fpr
    ```

1.  Encrypt and upload keys to Cloud Storage:

    Create the bucket:

    ```text
    gsutil mb gs://${PROJECT_ID}-keys
    gsutil iam ch serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com:objectViewer gs://${PROJECT_ID}-keys
    ```

    Create a key to encrypt the keys:

    ```text
    gcloud kms keyrings create binauth \
      --project $PROJECT_ID \
      --location global

    gcloud kms keys create binauth \
      --project $PROJECT_ID \
      --location global \
      --keyring binauth \
      --purpose encryption
    ```

    Allow Cloud Build to decrypt the keys:

    ```text
    gcloud kms keys add-iam-policy-binding binauth \
      --project $PROJECT_ID \
      --location global \
      --keyring binauth \
      --member serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com \
      --role roles/cloudkms.cryptoKeyDecrypter
    ```

    Encrypt the keys using KMS:

    ```text
    gcloud kms encrypt \
      --plaintext-file vulnz-attestor.gpg \
      --ciphertext-file vulnz-attestor.gpg.enc \
      --project $PROJECT_ID \
      --location global \
      --keyring binauth \
      --key binauth

    gcloud kms encrypt \
      --plaintext-file vulnz-attestor.pass \
      --ciphertext-file vulnz-attestor.pass.enc \
      --project $PROJECT_ID \
      --location global \
      --keyring binauth \
      --key binauth
    ```

    Upload the encrypted secrets and fingerprints:

    ```text
    gsutil cp *.enc gs://${PROJECT_ID}-keys/
    gsutil cp *.fpr gs://${PROJECT_ID}-keys/
    gsutil cp *.asc gs://${PROJECT_ID}-keys/
    ```


1.  Create a Container Analysis note:

    ```text
    curl "https://containeranalysis.googleapis.com/v1/projects/${PROJECT_ID}/notes/?noteId=vulnz-attestor" \
      -X POST \
      -H "Content-Type: application/json" \
      -H "Authorization: Bearer $(gcloud auth print-access-token)" \
      -H "X-Goog-User-Project: ${PROJECT_ID}" \
      --data-binary @- <<EOF
        {
          "name": "projects/${PROJECT_ID}/notes/vulnz-attestor",
          "attestation": {
            "hint": {
              "human_readable_name": "Vulnerability scanner attestor"
            }
          }
        }
    EOF
    ```

1.  Grant Cloud Build IAM permissions to view and attach notes to container
    images:

    ```text
    curl "https://containeranalysis.googleapis.com/v1beta1/projects/${PROJECT_ID}/notes/vulnz-attestor:setIamPolicy" \
      -X POST \
      -H "Content-Type: application/json" \
      -H "Authorization: Bearer $(gcloud auth print-access-token)" \
      -H "X-Goog-User-Project: ${PROJECT_ID}" \
      --data-binary @- <<EOF
        {
          "resource": "projects/${PROJECT_ID}/notes/${NOTE_ID}",
          "policy": {
            "bindings": [
              {
                "role": "roles/containeranalysis.notes.occurrences.viewer",
                "members": [
                  "serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com"
                ]
              },
              {
                "role": "roles/containeranalysis.notes.attacher",
                "members": [
                  "serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com"
                ]
              }
            ]
          }
        }
    EOF
    ```

1.  Create the vulnerability scan attestor:

    ```text
    gcloud beta container binauthz attestors create "vulnz-attestor" \
      --project $PROJECT_ID \
      --attestation-authority-note "vulnz-attestor" \
      --attestation-authority-note-project $PROJECT_ID

    gcloud beta container binauthz attestors public-keys add \
      --project $PROJECT_ID \
      --attestor "vulnz-attestor" \
      --pgp-public-key-file vulnz-attestor.asc
    ```

1.  Allow Cloud Build to verify these attestations:

    ```text
    gcloud beta container binauthz attestors add-iam-policy-binding vulnz-attestor \
      --project $PROJECT_ID \
      --member "serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com" \
      --role roles/binaryauthorization.attestorsVerifier
    ```

1.  Set the Binary Authorization policy:

    ```text
    cat > binauth-policy.yaml <<EOF
    admissionWhitelistPatterns:
    - namePattern: docker.io/istio/*
    defaultAdmissionRule:
      evaluationMode: ALWAYS_DENY
      enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
    globalPolicyEvaluationMode: ENABLE
    clusterAdmissionRules:
      us-central1.my-cluster:
        evaluationMode: REQUIRE_ATTESTATION
        enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
        requireAttestationsBy:
        - projects/${PROJECT_ID}/attestors/vulnz-attestor
    EOF
    ```

    ```text
    gcloud beta container binauthz policy import binauth-policy.yaml \
      --project $PROJECT_ID
    ```

1.  Build and push the vulnerability scan attestor to GCR:

    ```text
    gcloud builds submit \
      --project $PROJECT_ID \
      --tag gcr.io/${PROJECT_ID}/cloudbuild-attestor \
      ./binauthz-tools/
    ```


### Binary Authorization Demo

1.  Create a Cloud Source repo:

    ```text
    gcloud source repos create hello-app \
      --project $PROJECT_ID

    gcloud source repos clone hello-app  \
      --project $PROJECT_ID

    cp -R ./binauthz-tools/examples/hello-app/* ./hello-app/
    ```

1.  Open the Cloud Build Triggers page in the GCP Console.

1.  Click "Create Trigger", choose "hello-app" Cloud Source Repository

1.  Fill the following:

    - Name: "hello-app-vulnz-deploy"
    - Branch: "master"
    - Build Configuration: "Cloud Build configuration file", location: "cloudbuild.yaml"

1.  Add the following substitution variables:

    - `_VULNZ_NOTE_ID=vulnz-attestor`
    - `_KMS_KEYRING=binauth`
    - `_KMS_KEY=binauth`
    - `_COMPUTE_ZONE=us-central1`
    - `_PROD_CLUSTER=my-cluster`

1.  Push a commit to hello-app:

    ```text
    pushd hello-app
    git add .
    git commit -m "Initial commit"
    git push -u origin master
    popd
    ```


1.  Deploy a signed image:

    ```text
    # Get the SHA256 digest from cloud build's output
    export DIGEST=...
    ```

    ```text
    kubectl apply -f - <<EOF
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: hello-app
        labels:
          app: hello-app
      spec:
        replicas: 1
        selector:
          matchLabels:
            app: hello-app
        template:
          metadata:
            labels:
              app: hello-app
          spec:
            containers:
            - name: hello-app
              image: gcr.io/${PROJECT_ID}/hello-app@sha256:${DIGEST}
    EOF
    ```

1.  Deploy an unsigned image:

    ```text
    kubectl apply -f - <<EOF
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: nginx
        labels:
          app: nginx
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
            - name: nginx
              image: nginx
    EOF
    ```

    ```text
    kubectl get deploy nginx -o yaml
    ```

    ```text
    kubectl delete deploy nginx
    ```

1.  Deploy an unsigned image (break glass):

    ```text
    kubectl apply -f - <<EOF
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: hello-app
        labels:
          app: hello-app
      spec:
        replicas: 1
        selector:
          matchLabels:
            app: hello-app
        template:
          metadata:
            labels:
              app: hello-app
            annotations:
              alpha.image-policy.k8s.io/break-glass: "true"
          spec:
            containers:
            - name: hello-app
              image: gcr.io/${PROJECT_ID}/hello-app@sha256:${DIGEST}
    EOF
    ```

1.  To disable binary authorization enforcement on the cluster:

    ```text
    cat > binauth-policy.yaml <<EOF
    admissionWhitelistPatterns:
    - namePattern: docker.io/istio/*
    defaultAdmissionRule:
      enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
      evaluationMode: ALWAYS_ALLOW
    globalPolicyEvaluationMode: DISABLE
    EOF
    ```

    ```text
    gcloud beta container binauthz policy import binauth-policy.yaml \
      --project $PROJECT_ID
    ```

    Alternatively, you can disable binauthz on the cluster entirely.
