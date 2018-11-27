#### Binary Authorization
```bash
PROJECT_ID=PROJECT_ID
gcloud config set project ${PROJECT_ID}
gcloud services enable \
    container.googleapis.com \
    containeranalysis.googleapis.com \
    binaryauthorization.googleapis.com
```

```bash
gcloud beta container clusters create \
    --enable-binauthz \
    --zone us-central1-a \
    test-cluster

gcloud container clusters get-credentials \
    --zone us-central1-a \
    test-cluster

gcloud beta container binauthz policy export

```

output </br>
```yaml
admissionWhitelistPatterns:
- namePattern: gcr.io/google_containers/*
- namePattern: gcr.io/google-containers/*
- namePattern: k8s.gcr.io/*
- namePattern: gcr.io/stackdriver-agents/*
defaultAdmissionRule:
  evaluationMode: ALWAYS_ALLOW
  enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
name: projects/${PROJECT_ID}/policy
```

```bash
ATTESTOR=test-attestor
NOTE_ID=test-attestor-note
```

Attestor Note </br>
```bash
cat > /tmp/note_payload.json << EOM
{
  "name": "projects/${PROJECT_ID}/notes/${NOTE_ID}",
  "attestation_authority": {
    "hint": {
      "human_readable_name": "Attestor Note"
    }
  }
}
EOM
```
```bash
curl -X POST \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $(gcloud auth print-access-token)"  \
    --data-binary @/tmp/note_payload.json  \
    "https://containeranalysis.googleapis.com/v1beta1/projects/${PROJECT_ID}/notes/?noteId=${NOTE_ID}"
```

Verify Note </br>
```bash
curl \
-H "Authorization: Bearer $(gcloud auth print-access-token)" \
"https://containeranalysis.googleapis.com/v1beta1/projects/${PROJECT_ID}/notes/${NOTE_ID}"
```

Create Attestor </br>
```bash
gcloud beta container binauthz attestors create ${ATTESTOR} \
--attestation-authority-note=${NOTE_ID} \
--attestation-authority-note-project=${PROJECT_ID}
```

Verify
```bash
gcloud beta container binauthz attestors list

```

```bash
sudo apt-get install rng-tools -y

sudo rngd -r /dev/urandom

gpg --batch --gen-key <(
  cat <<- EOF
    Key-Type: RSA
    Key-Length: 2048
    Name-Real: "Test Attestor"
    Name-Email: "test-attestor@example.com"
    %commit
EOF
)
```
```bash
gpg --list-keys "test-attestor@example.com"
FINGERPRINT=PUBLIC_KEY_FINGERPRINT
gpg --armor --export ${FINGERPRINT} > /tmp/generated-key.pgp
gcloud beta container binauthz attestors public-keys add \
    --attestor=${ATTESTOR} \
    --public-key-file=/tmp/generated-key.pgp
```

Configure Policy </br>

```bash
cat > /tmp/policy.yaml << EOM
    admissionWhitelistPatterns:
    - namePattern: gcr.io/google_containers/
    - namePattern: gcr.io/google-containers/
    - namePattern: k8s.gcr.io/
    - namePattern: gcr.io/stackdriver-agents/
    defaultAdmissionRule:
      evaluationMode: REQUIRE_ATTESTATION
      enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
      requireAttestationsBy:
        - projects/${PROJECT_ID}/attestors/${ATTESTOR}
    name: projects/${PROJECT_ID}/policy
EOM
```

Import policy </br>
```bash
gcloud beta container binauthz policy import /tmp/policy.yaml

```


Test Policy </br>
```bash
kubectl run hello-server --image gcr.io/google-samples/hello-app:1.0 --port 8080
kubectl get pods
kubectl get event --template \
'{{range.items}}{{"\033[0;36m"}}{{.reason}}:{{"\033[0m"}}\{{.message}}{{"\n"}}{{end}}'
kubectl delete deployment hello-server

```
Trusting an image </br>

```bash
IMAGE_PATH="gcr.io/google-samples/hello-app"
IMAGE_DIGEST="sha256:c62ead5b8c15c231f9e786250b07909daf6c266d0fcddd93fea882eb722c3be4"
gcloud beta container binauthz create-signature-payload \
--artifact-url=${IMAGE_PATH}@${IMAGE_DIGEST} > /tmp/generated_payload.json

gpg \
    --local-user "test-attestor@example.com" \
    --armor \
    --output /tmp/generated_signature.pgp \
    --sign /tmp/generated_payload.json


gcloud beta container binauthz attestations create \
    --artifact-url="${IMAGE_PATH}@${IMAGE_DIGEST}" \
    --attestor="projects/${PROJECT_ID}/attestors/${ATTESTOR}" \
    --signature-file=/tmp/generated_signature.pgp \
    --pgp-key-fingerprint="${FINGERPRINT}"


gcloud beta container binauthz attestations list \
    --attestor=$ATTESTOR --attestor-project=$PROJECT_ID

kubectl run hello-server --image ${IMAGE_PATH}@${IMAGE_DIGEST} --port 8080

kubectl get pods

```

Clean Up </br>
```bash
gcloud container clusters delete \
    --zone=us-central1-a \
    test-cluster

curl -vvv -X DELETE  \
    -H "Authorization: Bearer $(gcloud auth print-access-token)" \
    "https://containeranalysis.googleapis.com/v1alpha1/projects/${PROJECT_ID}/notes/${NOTE_ID}"

gcloud beta container binauthz attestors delete $ATTESTOR
```
