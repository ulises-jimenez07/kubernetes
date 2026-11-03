# 11 · ConfigMaps & Environment Variables

---

## Part 1 — Environment variables

### Your first env vars

```yaml
containers:
- name: envar-demo-container
  image: nginx:alpine
  env:                      # a LIST of {name, value}
  - name: VAR1
    value: "valor de prueba 1"
  - name: VAR2
    value: "test2"
```
```bash
kubectl apply -f env.yaml
kubectl exec -it envar-demo -- env | grep VAR
kubectl exec -it envar-demo -- sh -c 'echo $VAR1'
```

### Inject pod info into env vars — Downward API

`kubectl get pod <p> -o yaml` shows tons of info about the pod (node, IP, namespace…). You can expose those fields to the container with `fieldRef`:

```yaml
env:
- name: MY_NODE_NAME
  valueFrom:
    fieldRef:
      fieldPath: spec.nodeName
- name: MY_POD_NAME
  valueFrom: {fieldRef: {fieldPath: metadata.name}}
- name: MY_POD_NAMESPACE
  valueFrom: {fieldRef: {fieldPath: metadata.namespace}}
- name: MY_POD_IP
  valueFrom: {fieldRef: {fieldPath: status.podIP}}
```
(`resourceFieldRef` does the same for a container's requests/limits.)

---

## Part 2 — ConfigMaps

### What is a ConfigMap?

Classic Docker approach: `COPY nginx.conf` into the image → to change one setting you must **rebuild and redeploy** a new image (v2).

A **ConfigMap** is a separate, namespaced object of **key → value** pairs holding **non-sensitive configuration**. The pod references it, so you change config **without rebuilding images**, and the same image runs in dev/prod with different config.

> Sensitive data (passwords, tokens) → use **Secrets** instead ([12-secrets](../12-secrets/README.md)).

### How a pod consumes a ConfigMap

Ways to create one:
- from a **file** / directory / literal with `kubectl create configmap`
- from a **YAML manifest** (preferred: versionable, portable)

Ways to use it in a pod:
1. as **environment variables** (`configMapKeyRef` / `envFrom`)
2. as **files** via a **volume** (each key → a file)

### Explore nginx's default config

```bash
kubectl run --rm -it nginx-test --image=nginx:alpine -- sh
cat /etc/nginx/conf.d/default.conf    # listen 80; root /usr/share/nginx/html; …
```
Editing it inside the container works (`nginx -s reload`) but is lost when the pod dies — that's what ConfigMaps fix. This `default.conf` is what we'll put in a ConfigMap.

### Create a ConfigMap from a file

`configmaps-examples/nginx.conf` (listens on 8080):
```bash
kubectl create configmap nginx-config --from-file=configmaps-examples/nginx.conf
kubectl get cm
kubectl describe cm nginx-config        # key = file name "nginx.conf", value = file content
kubectl get cm nginx-config -o yaml

# other variants
kubectl create configmap vars --from-env-file=configmaps-examples/vars.example   # each line → a key
kubectl create configmap lit --from-literal=key1=value1 --from-literal=key2=value2
```

### Mount a ConfigMap as a volume

The ConfigMap is written in YAML (key `nginx` with a `listen 9090` server block, plus key `test: hola`), and a Deployment mounts it:

```yaml
spec:
  containers:
    - name: nginx
      image: nginx:alpine
      volumeMounts:
      - name: nginx-vol
        mountPath: /etc/nginx/conf.d      # directory inside the container
  volumes:
    - name: nginx-vol
      configMap:
        name: nginx-config
        items:                            # pick specific keys…
        - key: nginx
          path: default.conf              # …and choose the file name
```
```bash
kubectl apply -f cm-nginx-vol.yaml
kubectl exec -it <pod> -- ls /etc/nginx/conf.d       # default.conf
kubectl exec -it <pod> -- cat /etc/nginx/conf.d/default.conf
curl <pod-ip>:9090                                   # nginx now listens on 9090
```

> Mounting over a directory **hides** whatever was there in the image.
> Mounted ConfigMap files update automatically (after a short delay) when the ConfigMap changes — but the app must re-read them (nginx needs a reload). Env vars **never** update without a pod restart.

### Mount without `items`

Remove `items:` → **every key becomes a file** named after the key:
```
/etc/nginx/conf.d/nginx    ← content of key "nginx"
/etc/nginx/conf.d/test     ← "hola"
```
Use `items` to choose keys and give them proper file names (e.g. `default.conf`).

### ConfigMap → environment variables

A second ConfigMap `vars`:
```yaml
data:
  db_host: dev.host.local
  db_user: dev_user
  script: |
    echo DB host es $DB_HOST y DB user es $DB_USER > /usr/share/nginx/html/test.html
```
Consumed both ways in the same pod:
```yaml
env:
  - name: DB_HOST                 # env var name (arbitrary)
    valueFrom:
      configMapKeyRef:
        name: vars                # ConfigMap name
        key: db_host              # key inside it
  - name: DB_USER
    valueFrom:
      configMapKeyRef: {name: vars, key: db_user}
volumeMounts:
  - name: nginx-vol
    mountPath: /etc/nginx/conf.d
  - name: script-vol
    mountPath: /opt
volumes:
  - name: nginx-vol
    configMap: {name: nginx-config, items: [{key: nginx, path: default.conf}]}
  - name: script-vol
    configMap: {name: vars, items: [{key: script, path: script.sh}]}
```

Load **all** keys as env vars at once:
```yaml
envFrom:
  - configMapRef:
      name: vars
```

### Validate

```bash
kubectl apply -f cm-nginx-env.yaml
kubectl exec -it <pod> -- sh
env | grep DB_            # DB_HOST=dev.host.local, DB_USER=dev_user
sh /opt/script.sh         # writes test.html using the env vars
# browser / curl: <pod-ip>:9090/test.html → "DB host es dev.host.local y DB user es dev_user"
```
Env vars are visible to any process in the container (shell, Python `os.environ`, …).

---

## Cheat sheet

```bash
kubectl create cm <name> --from-file=<file|dir> | --from-env-file=<f> | --from-literal=k=v
kubectl get cm | kubectl describe cm <name> | kubectl get cm <name> -o yaml
kubectl edit cm <name>
kubectl rollout restart deployment <dep>      # pick up env-var changes
```
