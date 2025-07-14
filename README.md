# Komplet Guide: Lokal Kubernetes Cluster med Ingress, Monitoring & Autoscaling

Denne guide viser hele processen:
1.  Opsæt en lokal Kubernetes-cluster med `kind`.
2.  Installer en NGINX Ingress Controller.
3.  Deploy en test-applikation.
4.  Installer en monitoring stack med Prometheus & Grafana.
5.  Konfigurer automatisk skalering (HPA) baseret på CPU-forbrug.

## Forudsætninger
Sørg for, at du har følgende værktøjer installeret:
* Docker Desktop
* `kubectl` (Kubernetes Command-line Tool)
* `kind` (Kubernetes in Docker)
* `helm` (The Kubernetes Package Manager)

---
## Trin 1: Opret Cluster med Port-mapping

1.  **Opret `kind-config.yaml`:**
    ```yaml
    # kind-config.yaml
    kind: Cluster
    apiVersion: kind.x-k8s.io/v1alpha4
    nodes:
    - role: control-plane
      extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
    ```

2.  **Opret clusteren:**
    ```bash
    kind create cluster --config=kind-config.yaml
    ```
---
## Trin 2: Installer NGINX Ingress Controller

1.  **Installer controlleren** fra det `kind`-optimerede manifest:
    ```bash
    kubectl apply -f [https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml](https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml)
    ```

2.  **Vent på, at controlleren er klar:**
    ```bash
    kubectl wait --namespace ingress-nginx \
      --for=condition=ready pod \
      --selector=app.kubernetes.io/component=controller \
      --timeout=90s
    ```
---
## Trin 3: Deploy Test-applikation med Ressource-anmodninger

For at autoskalering kan virke, *skal* vi specificere `resources.requests`.

1.  **Opret `app-deployment.yaml`:**
    ```yaml
    # app-deployment.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: my-nginx-app
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: my-nginx
      template:
        metadata:
          labels:
            app: my-nginx
        spec:
          containers:
          - name: nginx
            image: nginx:latest
            ports:
            - containerPort: 80
            resources:
              requests:
                cpu: "100m"
                memory: "128Mi"
              limits:
                cpu: "200m"
                memory: "256Mi"
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: my-nginx-service
    spec:
      selector:
        app: my-nginx
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
    ```

2.  **Opret `app-ingress.yaml`:**
    ```yaml
    # app-ingress.yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: my-nginx-ingress
    spec:
      ingressClassName: nginx
      rules:
      - http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-nginx-service
                port:
                  number: 80
    ```
3.  **Anvend filerne:**
    ```bash
    kubectl apply -f app-deployment.yaml
    kubectl apply -f app-ingress.yaml
    ```
---
## Trin 4: Verificer Applikationen

Åbn din browser og gå til **http://localhost**. Du skulle se "Welcome to nginx!".

---
## Trin 5: Installer Monitoring (Prometheus & Grafana)

1.  **Tilføj Helm Repository:**
    ```bash
    helm repo add prometheus-community [https://prometheus-community.github.io/helm-charts](https://prometheus-community.github.io/helm-charts)
    helm repo update
    ```
2.  **Installer `kube-prometheus-stack`:**
    ```bash
    helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace
    ```
3.  **Få adgang til Grafana:** Åbn en **ny terminal** og kør:
    ```bash
    kubectl --namespace monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
    ```
4.  **Find adgangskode (i PowerShell):**
    ```powershell
    [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($(kubectl get secret --namespace monitoring kube-prometheus-stack-grafana -o jsonpath='{.data.admin-password}')))
    ```
5.  **Log ind på Grafana:**
    * **URL:** http://localhost:3000
    * **Bruger:** `admin`
    * **Kode:** Koden fra kommandoen ovenfor.

---
## Trin 6: Opsæt Autoskalering (HPA)

1.  **Installer Metrics Server.** HPA'en har brug for denne for at hente CPU-data.
    ```bash
    kubectl apply -f [https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml](https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml)
    kubectl patch -n kube-system deployment metrics-server --type=json -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'
    ```

2.  **Opret `app-hpa.yaml`:**
    ```yaml
    # app-hpa.yaml
    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    metadata:
      name: my-nginx-hpa
    spec:
      scaleTargetRef:
        apiVersion: apps/v1
        kind: Deployment
        name: my-nginx-app
      minReplicas: 2
      maxReplicas: 10
      metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 50
    ```

3.  **Anvend HPA-filen:**
    ```bash
    kubectl apply -f app-hpa.yaml
    ```
---
## Trin 7: Test Autoskalering

1.  **Åbn `k9s`** og gå til HPA-viewet (`:hpa`) for at observere.

2.  **Generer belastning (i PowerShell):** Start 10 parallelle jobs.
    ```powershell
    1..10 | ForEach-Object {
        Start-Job -ScriptBlock {
            while ($true) {
                try {
                    Invoke-WebRequest -Uri http://localhost -UseBasicParsing | Out-Null
                } catch {}
            }
        }
    }
    ```
    Du vil se antallet af pods stige.

3.  **Stop belastningen:** Når du er færdig, skal du stoppe og rydde op i dine jobs.
    ```powershell
    Get-Job | Stop-Job
    Get-Job | Remove-Job
    ```
    Efter 5 minutter vil HPA'en skalere antallet af pods ned til 2 igen.

---
## Trin 8: Ryd op

Når du er helt færdig, kan du slette alt med én kommando:
```bash
kind delete cluster
```
