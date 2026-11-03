# 12 · Secrets

---

## What is a Secret?

A **Secret** works almost exactly like a ConfigMap (key → value, consumed as env vars or files), but is meant for **sensitive data**: passwords, tokens, API keys, TLS certs, registry credentials.

| | ConfigMap | Secret |
|---|---|---|
| For | non-sensitive config (URLs, feature flags, usernames…) | sensitive data |
| Stored as | plain text | **base64** (+ optional encryption at rest in etcd) |
| Shown by `describe` | values | **only key names + sizes** |
| Extra protection | — | can be restricted separately with RBAC; mounted on tmpfs (RAM) |

Like ConfigMaps, Secrets are separate from the image and the pod: change the Secret without rebuilding anything (mounted files update automatically; env vars need a pod restart).

## Create a Secret from a file

`secret-files/test.txt`:
```
secret1=hola
secret2=adios
```
```bash
kubectl create secret generic mysecret --from-file=secret-files/test.txt
kubectl get secrets
kubectl describe secret mysecret         # only shows "test.txt: 27 bytes" — not the value
kubectl get secret mysecret -o yaml      # data: test.txt: c2VjcmV0MT1ob2xh...
```

Secret types: `generic` (**Opaque**, the most used), `docker-registry` (`kubernetes.io/dockerconfigjson`), `tls` (`kubernetes.io/tls`), `kubernetes.io/service-account-token`, …

## What is base64?

```bash
echo -n hola | base64                 # aG9sYQ==
echo aG9sYQ== | base64 --decode       # hola
kubectl get secret mysecret -o jsonpath='{.data.test\.txt}' | base64 --decode
```

> **base64 is an encoding, NOT encryption.** Anyone who can read the Secret can decode it. Protect Secrets with **RBAC**, enable **encryption at rest** (on GKE: Application-layer Secrets Encryption with Cloud KMS), and/or use an external secret manager (Google Secret Manager, Vault) via the Secrets Store CSI driver / External Secrets Operator.

## `data` vs `stringData`

`secret-data.yaml` — `data`: **you** must base64-encode the values:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=            # echo -n admin | base64
  password: MWYyZDFlMmU2N2Rm
```

`secret-stringdata.yaml` — `stringData`: write **plain text**, Kubernetes encodes it for you:
```yaml
stringData:
  username: usertest
  password: password
```
Either way, `kubectl get secret -o yaml` shows it under `data:` (base64).

## Tip: never commit a YAML with secrets to Git!

Both forms above put the real secret in the file → anyone with repo access has it.

Technique — keep **placeholders** in the committed file and substitute from env vars at deploy time with `envsubst`. `secure.yaml`:
```yaml
stringData:
  username: $USER
  password: $PASSWORD
```
```bash
kubectl apply -f secure.yaml                 # fails: stores the literal string "$USER"!
export USER=alice PASSWORD=ALICE
envsubst < secure.yaml > tmp.yaml            # tmp.yaml has real values — DON'T commit it (add to .gitignore)
kubectl apply -f tmp.yaml
rm tmp.yaml
# or with no temp file:
envsubst < secure.yaml | kubectl apply -f -
```
(`envsubst` is in the `gettext` package: `brew install gettext`.)

Other real-world options: Sealed Secrets, SOPS, External Secrets Operator + a cloud secret manager, CI/CD secret variables.

## Inject Secrets as volumes

```yaml
containers:
- name: mypod
  image: nginx:alpine
  volumeMounts:
  - name: test
    mountPath: "/opt"
    readOnly: true
volumes:
- name: test
  secret:
    secretName: secret1
    items:                 # optional: only some keys, custom file names
    - key: username
      path: user.txt
```
```bash
kubectl apply -f pod-vol-secret.yaml
kubectl exec -it mypod -- cat /opt/user.txt       # admin   (decoded!)
```
Inside the pod, files contain the **decoded** value. Your app just reads the file (e.g. Python `open('/opt/user.txt').read()`).

## Inject Secrets as env vars

```yaml
env:
  - name: USERTEST
    valueFrom:
      secretKeyRef:
        name: secret1
        key: username
  - name: PASSWORDTEST
    valueFrom:
      secretKeyRef:
        name: secret1
        key: password
```
```bash
kubectl exec -it mypod -- env | grep TEST       # USERTEST=admin PASSWORDTEST=12345678
```
All keys at once: `envFrom: [{secretRef: {name: secret1}}]`.

> Files are generally safer than env vars (env vars can leak into logs, crash dumps, child processes, `kubectl describe` of the image's entrypoint…).

---

## Cheat sheet

```bash
kubectl create secret generic <name> --from-file=<f> | --from-literal=k=v | --from-env-file=<f>
kubectl create secret docker-registry regcred --docker-server=… --docker-username=… --docker-password=…
kubectl create secret tls my-tls --cert=tls.crt --key=tls.key
kubectl get secrets | kubectl describe secret <n>
kubectl get secret <n> -o jsonpath='{.data.<key>}' | base64 --decode
envsubst < secret.tmpl.yaml | kubectl apply -f -
```
