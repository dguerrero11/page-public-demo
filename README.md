# Clase 4 — GitOps CI/CD con Tekton + ArgoCD

## Objetivo

Implementar el ciclo completo de GitOps: un `git push` es el único comando necesario para llevar código a producción. Tekton construye y publica la imagen, ArgoCD detecta el cambio en el repositorio y despliega automáticamente.

---

## Arquitectura

```
  Developer
     │
     │  git push (modifica app/ o k8s/)
     ▼
  ┌─────────────────────────────────────────┐
  │  GitHub: dguerrero11/page-public-demo   │
  │  ├── app/          ← código fuente      │
  │  │   ├── index.html                     │
  │  │   └── Dockerfile                     │
  │  ├── k8s/          ← ArgoCD lee esto    │
  │  │   ├── namespace.yaml                 │
  │  │   ├── deployment.yaml  ← image tag   │
  │  │   └── service.yaml                   │
  │  ├── tekton/       ← pipeline CI        │
  │  └── argocd/       ← application CD     │
  └──────────────────────────────┬──────────┘
                                 │
          ┌──────────────────────┼──────────────────────┐
          │ CI (Tekton)          │         CD (ArgoCD)  │
          ▼                      │                       ▼
  ┌───────────────────┐          │         ┌────────────────────┐
  │ Task: git-clone   │          │         │ Application        │
  │ alpine/git        │          │         │ page-demo          │
  │ → /workspace NFS  │          │         │ path: k8s/         │
  └────────┬──────────┘          │         │ selfHeal: true     │
           │ workspace compartido│         │ prune: true        │
           ▼                     │         └────────┬───────────┘
  ┌───────────────────┐          │                  │ kubectl apply
  │ Task: build-push  │          │                  ▼
  │ Kaniko executor   │──push──► │ Docker Hub  Namespace: demo
  │ → image:v1/v2     │  image   │         Deployment: page-demo
  └───────────────────┘          │         Service: NodePort :31080
          │                      │
          │ actualizar image tag  │
          ▼                      │
   k8s/deployment.yaml ──────────┘
   git commit + push
```

### Principio GitOps fundamental

```
  app/          ← lo que el DEV toca (código, HTML, Dockerfile)
  k8s/          ← lo que el OPS toca (manifiestos, image tags)
```

**Nunca mezclar código con manifiestos.** ArgoCD solo mira `k8s/`. Si el Dev cambia `app/index.html` sin actualizar el tag en `k8s/deployment.yaml`, ArgoCD no despliega nada — la fuente de verdad es Git.

---

## Pre-requisitos

### Cluster
```bash
# Tekton instalado
kubectl get pods -n tekton-pipelines

# StorageClass nfs-csi disponible (workspace del Pipeline)
kubectl get storageclass nfs-csi

# ArgoCD instalado
kubectl get pods -n argocd
```

### Cuentas necesarias
- GitHub: repo `page-public-demo` accesible + Personal Access Token
- Docker Hub: usuario + token de acceso (no la contraseña)

### Generar token Docker Hub
```
Docker Hub → Account Settings → Security → New Access Token
Nombre: bootcamp-tekton
Permisos: Read & Write
```

### Generar token GitHub (PAT)
```
GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens
Repository access: Solo este repo
Permisos: Contents → Read & Write, Metadata → Read
```

### Puertos de firewall necesarios (Rocky Linux 9)

```bash
# Abrir todos los puertos del lab de una vez (ejecutar en TODOS los nodos)
ansible all -i /root/kubernetes/ansible-k8s/inventory/hosts.ini \
  -m firewalld \
  -a "port={{ item }}/tcp permanent=yes state=enabled immediate=yes" \
  --become \
  -e "item={{ item }}" \
  --loop "30443 31080"

# O manualmente nodo por nodo:
for port in 30443 31080; do
  firewall-cmd --add-port=${port}/tcp --permanent
done
firewall-cmd --reload
firewall-cmd --list-ports
```

| Puerto | Servicio | Descripción |
|--------|----------|-------------|
| 30443  | ArgoCD UI | HTTPS — acceso a la interfaz de ArgoCD |
| 31080  | App demo  | HTTP — página web desplegada por ArgoCD |

---

## PARTE 1 — Instalar Tekton

```bash
# Instalar Tekton Pipelines
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

# Esperar a que todos los pods estén Running (~2-3 min)
kubectl get pods -n tekton-pipelines --watch
# Ctrl+C cuando todos estén Running

# Verificar CRDs instalados
kubectl get crds | grep tekton
# Debe mostrar: tasks.tekton.dev, pipelines.tekton.dev, pipelineruns.tekton.dev, etc.

# Instalar CLI de Tekton
curl -LO https://github.com/tektoncd/cli/releases/download/v0.35.0/tkn_0.35.0_Linux_x86_64.tar.gz
tar xzf tkn_0.35.0_Linux_x86_64.tar.gz -C /usr/local/bin tkn
tkn version
```

### ⚠️ CRÍTICO — PodSecurity: etiquetar el namespace de Tekton

Kubernetes 1.25+ incluye **Pod Security Admission** activado por defecto. El perfil
`restricted` bloquea los pods de Tekton (Kaniko necesita permisos elevados para
construir imágenes). Sin este paso los pods quedan en `Pending` sin ningún mensaje
claro de error.

```bash
# Etiquetar el namespace como privileged (OBLIGATORIO antes de lanzar cualquier PipelineRun)
kubectl label namespace tekton-pipelines \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/warn=privileged \
  pod-security.kubernetes.io/audit=privileged \
  --overwrite

# Verificar que se aplicó
kubectl get namespace tekton-pipelines --show-labels | grep pod-security
```

**Síntoma sin este paso:** Los pods de los TaskRuns quedan en `Pending` indefinidamente.
Al describir el pod se ve un evento de tipo `Warning` con el mensaje:
```
Error from server (Forbidden): pods "..." is forbidden:
violates PodSecurity "restricted:latest"
```

---

## PARTE 2 — Crear Secret de Docker Hub

```bash
# Usar el token de Docker Hub (NO la contraseña de la cuenta)
# Generar en: Docker Hub → Account Settings → Security → New Access Token
kubectl create secret docker-registry docker-credentials \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=guerrero11dan \
  --docker-password=<TU_TOKEN_DOCKERHUB> \
  -n tekton-pipelines

# Verificar que se creó
kubectl get secret docker-credentials -n tekton-pipelines
```

> El Secret nunca se commitea a Git. Se crea directamente en el cluster.

---

## PARTE 3 — Aplicar Tasks y Pipeline

```bash
cd tekton/

kubectl apply -f 02-task-git-clone.yaml
kubectl apply -f 03-task-build-push.yaml
kubectl apply -f 04-pipeline.yaml

# Verificar
kubectl get tasks -n tekton-pipelines
kubectl get pipeline -n tekton-pipelines
```

### ⚠️ CRÍTICO — PipelineRun: usar nombre fijo, no `generateName`

Los PipelineRuns en este repo usan `metadata.name` con un nombre fijo
(`build-and-deploy-v1`, `build-and-deploy-v2`). **No usen `generateName`** porque
`kubectl apply` no lo soporta y lanza este error:

```
error: from: build-and-deploy-: cannot use generate name with apply
```

La solución es que el YAML tenga siempre un `name` fijo:
```yaml
metadata:
  name: build-and-deploy-v1   # ✅ nombre fijo → kubectl apply funciona
  # generateName: build-and-deploy-  ← ❌ solo funciona con kubectl create
  namespace: tekton-pipelines
```

Si quieren lanzar el mismo PipelineRun una segunda vez deben borrar el anterior:
```bash
kubectl delete pipelinerun build-and-deploy-v1 -n tekton-pipelines
kubectl apply -f tekton/05-pipelinerun-v1.yaml
```

---

## PARTE 4 — Primer PipelineRun (build v1)

```bash
# Lanzar el pipeline
kubectl apply -f 05-pipelinerun-v1.yaml

# Ver el estado
tkn pipelinerun list -n tekton-pipelines

# Ver logs en tiempo real
tkn pipelinerun logs --last -f -n tekton-pipelines
```

**¿Qué está pasando?**
```
1. Kubernetes crea un PVC de 500Mi en NFS (el workspace compartido)
2. Pod del Task "clone" arranca → clona el repo en /workspace
3. Pod del Task "build-push" arranca → Kaniko lee /workspace/app/
4. Kaniko construye la imagen y la pushea a Docker Hub como guerrero11dan/page-public-demo:v1
5. El PVC se libera
```

```bash
# Ver los pods que crea cada Task
kubectl get pods -n tekton-pipelines
# Verás: build-and-deploy-XXXX-clone-pod y build-and-deploy-XXXX-build-push-pod

# Verificar que la imagen llegó a Docker Hub
# Ir a: hub.docker.com/r/guerrero11dan/page-public-demo/tags
```

---

## PARTE 5 — Instalar ArgoCD

```bash
# Crear namespace
kubectl create namespace argocd

# Instalar ArgoCD
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Esperar a que todos los pods estén Running (~3-4 min)
kubectl get pods -n argocd --watch

# Exponer con NodePort
kubectl patch svc argocd-server -n argocd \
  -p '{"spec":{"type":"NodePort","ports":[{"port":443,"nodePort":30443,"targetPort":8080,"protocol":"TCP","name":"https"}]}}'

# Obtener contraseña inicial
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

**Acceder a la UI:**
```
https://192.168.109.143:30443
Usuario:  admin
Password: (la del comando anterior)
Aceptar el certificado auto-firmado → Advanced → Proceed
```

---

## PARTE 6 — Crear la Application en ArgoCD

```bash
cd argocd/
kubectl apply -f application.yaml

# Verificar estado
kubectl get application -n argocd
# SYNC STATUS:   Synced
# HEALTH STATUS: Healthy
```

**¿Qué hizo ArgoCD?**
1. Leyó `k8s/` del repo
2. Encontró `namespace.yaml`, `deployment.yaml`, `service.yaml`
3. Aplicó los tres manifiestos en el cluster
4. El Deployment intentó arrancar con `guerrero11dan/page-public-demo:v1`

```bash
# Ver los pods desplegados
kubectl get pods -n demo
kubectl get svc -n demo

# Acceder a la app
# http://192.168.109.143:31080
```

---

## DEMO EN VIVO — El ciclo completo: v1 → v2

> Esta es la parte más importante de la clase. Hacerlo despacio.

### Paso 1 — Modificar la app (simular trabajo del Dev)

Editar `app/index.html` — cambiar algo visible:
```html
<!-- Buscar esta línea y cambiarla: -->
<div class="version-badge">v1.0.0</div>
<!-- Por: -->
<div class="version-badge">v2.0.0</div>

<!-- Y en la info-grid cambiar el valor de Versión: -->
<div class="value">v2.0.0</div>

<!-- Opcional: cambiar el color del badge en el CSS: -->
background: #1f6feb;   /* azul en vez de verde */
```

### Paso 2 — Actualizar el tag de imagen en el manifiesto K8s

Editar `k8s/deployment.yaml`:
```yaml
# Cambiar:
image: guerrero11dan/page-public-demo:v1
# Por:
image: guerrero11dan/page-public-demo:v2
```

### Paso 3 — Commit y push

```bash
git add app/index.html k8s/deployment.yaml
git commit -m "feat: update to v2 - new color and version"
git push origin main
```

### Paso 4 — Lanzar Tekton para construir v2

```bash
kubectl apply -f tekton/05-pipelinerun-v2.yaml
tkn pipelinerun logs --last -f -n tekton-pipelines
# Esperar a que termine (2-3 min)
```

### Paso 5 — ArgoCD detecta el cambio y despliega

ArgoCD hace polling cada 3 minutos. Para no esperar:
```bash
# Forzar sync inmediato (requiere CLI de ArgoCD)
argocd app sync page-demo

# O desde la UI: Applications → page-demo → Sync → Synchronize
```

### Paso 6 — Verificar el despliegue

```bash
kubectl rollout status deployment/page-demo -n demo
kubectl get pods -n demo
# Los pods nuevos deben estar Running con la imagen :v2
```

**Abrir el navegador en `http://192.168.109.143:31080`** — el badge debe decir v2.0.0.

### Pregunta para la clase
> "¿Cuántos `kubectl apply` hicimos para desplegar la nueva versión?"
> **Cero. Solo un `git push`.**

---

## DEMO — selfHeal en vivo

```bash
# Borrar manualmente un pod (simular fallo)
kubectl delete pod -l app=page-demo -n demo

# ArgoCD lo detecta en ~10s y lo restaura automáticamente
kubectl get pods -n demo -w
```

```bash
# Borrar el Deployment completo
kubectl delete deployment page-demo -n demo

# ArgoCD lo restaura en ~10s
kubectl get deployment -n demo -w
```

> "selfHeal: true significa que ArgoCD es la fuente de verdad. Lo que no esté en Git, no existe en el cluster."

---

## DEMO — Rollback desde ArgoCD UI

```
ArgoCD UI → page-demo → History and Rollback
→ Seleccionar la revisión anterior (v1)
→ Rollback
```

```bash
# Verificar que volvió a v1
kubectl get deployment page-demo -n demo \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
# Debe mostrar: guerrero11dan/page-public-demo:v1
```

---

## LABORATORIO — Para los alumnos (70 min)

### Paso 1 — Clonar el repo base y crear su propia rama

```bash
git clone https://github.com/dguerrero11/page-public-demo
cd page-public-demo
git checkout -b alumno-<tunombre>
```

### Paso 2 — Personalizar la app

Editar `app/index.html`:
- Cambiar el título
- Cambiar colores
- Agregar su nombre en el footer

### Paso 3 — Crear el Secret de Docker Hub

```bash
kubectl create secret docker-registry docker-credentials \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<TU_USUARIO> \
  --docker-password=<TU_TOKEN> \
  -n tekton-pipelines
```

### Paso 4 — Actualizar los YAMLs con su imagen

Editar `tekton/04-pipeline.yaml` y `05-pipelinerun-v1.yaml`:
```yaml
# Cambiar la imagen por la suya
value: "TU_USUARIO_DOCKERHUB/page-demo:v1"
```

Editar `k8s/deployment.yaml`:
```yaml
image: TU_USUARIO_DOCKERHUB/page-demo:v1
```

### Paso 5 — Aplicar Tasks y Pipeline

```bash
kubectl apply -f tekton/02-task-git-clone.yaml
kubectl apply -f tekton/03-task-build-push.yaml
kubectl apply -f tekton/04-pipeline.yaml
```

### Paso 6 — Lanzar el Pipeline

```bash
kubectl apply -f tekton/05-pipelinerun-v1.yaml
tkn pipelinerun logs --last -f -n tekton-pipelines
```

### Paso 7 — Crear Application en ArgoCD

Editar `argocd/application.yaml` con su nombre:
```yaml
metadata:
  name: page-demo-<tunombre>
```

```bash
kubectl apply -f argocd/application.yaml
```

### Paso 8 — Verificar y hacer el ciclo v1 → v2

```bash
# App corriendo
kubectl get pods -n demo

# Modificar, push, pipeline v2, sync ArgoCD
# Ver el cambio en el navegador
```

---

## Limpieza — Primero solo (instructor), luego con alumnos

### Limpiar la app desplegada

```bash
# Opción A: borrar solo los recursos de la app
kubectl delete namespace demo

# Opción B: dejar que ArgoCD lo gestione
kubectl delete application page-demo -n argocd
# ArgoCD borra todo lo que gestionaba (prune: true)
```

### Limpiar Tekton

```bash
# Borrar los PipelineRuns (liberan los PVCs NFS)
kubectl delete pipelineruns --all -n tekton-pipelines

# Borrar Tasks y Pipeline
kubectl delete -f tekton/02-task-git-clone.yaml
kubectl delete -f tekton/03-task-build-push.yaml
kubectl delete -f tekton/04-pipeline.yaml

# Borrar el Secret (nunca se deja en el cluster después del lab)
kubectl delete secret docker-credentials -n tekton-pipelines

# Desinstalar Tekton completo
kubectl delete -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
```

### Limpiar ArgoCD

```bash
# Borrar la Application (ArgoCD borra los recursos gestionados)
kubectl delete application page-demo -n argocd

# Desinstalar ArgoCD completo
kubectl delete namespace argocd
```

### Verificar limpieza total

```bash
kubectl get all -n demo          # debe dar error (namespace borrado)
kubectl get pods -n tekton-pipelines
kubectl get pods -n argocd
kubectl get pvc -n tekton-pipelines  # verificar que se liberaron los PVCs NFS
```

---

## Troubleshooting

> Esta sección recoge los problemas encontrados en el despliegue real del lab.
> Todos los errores siguientes ocurrieron y fueron resueltos durante la preparación.

---

### ❌ Pods de Tekton en `Pending` — PodSecurity violation

**Síntoma:**
```bash
kubectl get pods -n tekton-pipelines
# NAME                                  READY   STATUS    RESTARTS
# build-and-deploy-v1-clone-pod         0/1     Pending   0
```

```bash
kubectl describe pod <nombre-pod> -n tekton-pipelines | grep -A5 Events
# Warning  FailedCreate ... violates PodSecurity "restricted:latest"
```

**Causa:** Kubernetes 1.25+ bloquea pods con privilegios elevados en namespaces sin
etiqueta de PodSecurity. Kaniko (el builder de imágenes) necesita capacidades de root.

**Solución:**
```bash
kubectl label namespace tekton-pipelines \
  pod-security.kubernetes.io/enforce=privileged \
  --overwrite
```

---

### ❌ `cannot use generate name with apply`

**Síntoma:**
```bash
kubectl apply -f tekton/05-pipelinerun-v1.yaml
# error: from: build-and-deploy-: cannot use generate name with apply
```

**Causa:** El YAML usaba `generateName` en lugar de `name`. `kubectl apply` requiere
un nombre determinista para calcular el diff; `generateName` solo funciona con
`kubectl create`.

**Solución:** Cambiar el YAML de:
```yaml
metadata:
  generateName: build-and-deploy-   # ❌
```
a:
```yaml
metadata:
  name: build-and-deploy-v1         # ✅
```

---

### ❌ `git push` rechazado — autenticación GitHub

**Síntoma:**
```
remote: Support for password authentication was removed on August 13, 2021.
fatal: Authentication failed for 'https://github.com/...'
```

**Causa:** GitHub eliminó la autenticación por contraseña vía HTTPS. Es necesario
usar un Personal Access Token (PAT).

**Solución:**
```bash
# 1. Generar token en: GitHub → Settings → Developer settings → Personal access tokens → Fine-grained
#    Permisos: Contents (Read & Write), Metadata (Read)

# 2. Actualizar la URL del remote con el token
git remote set-url origin https://<USUARIO>:<TOKEN>@github.com/<USUARIO>/page-public-demo.git

# 3. Verificar
git remote -v

# 4. Hacer push normalmente
git push
```

> Para el lab con Rocky Linux 9, el credential helper no está configurado por
> defecto, por lo que incluir el token en la URL es el método más directo.

---

### ❌ Namespace `demo` no existe al sincronizar ArgoCD

**Síntoma:** ArgoCD muestra la Application en estado `Missing` y los recursos no
se crean.

**Causa:** El namespace `demo` no existía antes de que ArgoCD intentara crear el
Deployment y el Service.

**Solución:** El archivo `argocd/application.yaml` ya incluye:
```yaml
syncOptions:
  - CreateNamespace=true
```
Si el namespace sigue sin crearse, aplicarlo manualmente:
```bash
kubectl apply -f k8s/namespace.yaml
```

---

### ❌ ArgoCD NodePort 30443 no accesible desde fuera del cluster

**Síntoma:** La UI de ArgoCD no carga en el navegador en `https://<IP>:30443`.

**Causa:** El firewall de Rocky Linux 9 (firewalld) bloquea el puerto.

**Solución:**
```bash
# Opción A — Ansible (todos los nodos de golpe)
ansible all -i /root/kubernetes/ansible-k8s/inventory/hosts.ini \
  -m firewalld \
  -a "port=30443/tcp permanent=yes state=enabled immediate=yes" \
  --become

# Opción B — Manual en cada nodo
firewall-cmd --add-port=30443/tcp --permanent
firewall-cmd --reload
```

---

### PipelineRun falla en el paso `clone`

```bash
# Ver qué TaskRun falló
tkn taskrun list -n tekton-pipelines

# Ver logs del TaskRun específico
tkn taskrun logs <nombre> -n tekton-pipelines

# Causas comunes:
# 1. URL del repo mal escrita
# 2. PVC no se creó — verificar StorageClass
kubectl get pvc -n tekton-pipelines

# 3. Repo privado sin credenciales → crear Secret con token GitHub
kubectl create secret generic github-credentials \
  --from-literal=token=<TU_TOKEN_GITHUB> \
  -n tekton-pipelines
```

### Kaniko falla con error de autenticación

```bash
# Verificar que el Secret existe y tiene el formato correcto
kubectl get secret docker-credentials -n tekton-pipelines -o yaml

# Recrear el Secret
kubectl delete secret docker-credentials -n tekton-pipelines
kubectl create secret docker-registry docker-credentials \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<USUARIO> \
  --docker-password=<TOKEN> \
  -n tekton-pipelines
```

### ArgoCD muestra OutOfSync pero no sincroniza automáticamente

```bash
# Verificar que automated sync está configurado
kubectl get application page-demo -n argocd -o yaml | grep -A 5 syncPolicy

# Forzar refresh del repo
argocd app get --refresh page-demo

# Ver qué difiere entre Git y el cluster
argocd app diff page-demo
```

### La app desplegada muestra la imagen anterior (no se actualizó)

```bash
# Verificar que el tag en deployment.yaml fue actualizado y commiteado
git log k8s/deployment.yaml
git show HEAD:k8s/deployment.yaml | grep image

# Verificar que ArgoCD tomó el último commit
argocd app get page-demo | grep "Revision"

# Forzar sync
argocd app sync page-demo
```

### Nodos NotReady después de instalar ArgoCD

ArgoCD instala muchos pods a la vez y puede saturar los recursos temporalmente.
```bash
kubectl get nodes
kubectl describe node <nodo> | grep -A 10 "Conditions:"
# Esperar 2-3 min — suele resolverse solo
```

---

## Preguntas de cierre

1. ¿Cuál es la diferencia entre Tekton y ArgoCD? ¿Pueden usarse sin el otro?
2. ¿Por qué el workspace del Pipeline necesita un PVC en NFS y no un emptyDir?
3. Si borro manualmente un Deployment que ArgoCD gestiona, ¿qué pasa en los próximos 10 segundos?
4. ¿Qué significa `prune: true` en la Application? ¿Qué riesgo tiene?
5. ¿Por qué el Secret de Docker Hub no se commitea a Git?

---

## Estructura del repositorio

```
page-public-demo/
├── app/
│   ├── index.html              ← página web (modificar para v1→v2)
│   └── Dockerfile              ← FROM nginx:alpine, una línea
├── k8s/
│   ├── namespace.yaml          ← namespace "demo"
│   ├── deployment.yaml         ← actualizar image tag para cada versión
│   └── service.yaml            ← NodePort :31080
├── tekton/
│   ├── 01-secret-docker.yaml   ← TEMPLATE — no usar, crear con kubectl
│   ├── 02-task-git-clone.yaml  ← Task: clonar repo en workspace NFS
│   ├── 03-task-build-push.yaml ← Task: Kaniko build + push a Docker Hub
│   ├── 04-pipeline.yaml        ← Pipeline: encadena clone → build
│   ├── 05-pipelinerun-v1.yaml  ← Lanzar build de v1
│   └── 05-pipelinerun-v2.yaml  ← Lanzar build de v2
├── argocd/
│   └── application.yaml        ← Application: monitorea k8s/, sync automático
└── README.md                   ← esta guía
```
