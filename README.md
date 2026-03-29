# Eigenes Kubernetes-Projekt: Kleiner Notenrechner

In diesem Projekt geht es darum, eine kleine eigene Web-Applikation als Container in Kubernetes zu betreiben und sauber zu dokumentieren.  
Statt einer großen Fremd-Anwendung wird hier bewusst ein **einfacher Notenrechner** eingesetzt, damit der Fokus auf Kubernetes liegt.

## Projektidee

Die Anwendung ist eine kleine HTML/JavaScript-Seite, auf der Noten eingetragen und als Durchschnitt berechnet werden können.  
Die Seite läuft in einem Nginx-Container und wird über Kubernetes mit mehreren Replikas bereitgestellt.

## Ziel des Projekts

Das Ziel ist, eine reale Kubernetes-Bereitstellung nachvollziehbar zu zeigen:

- eigene Anwendung containerisieren
- Deployment mit Replikas erstellen
- Service im Cluster veröffentlichen
- Anwendung im Browser aufrufen
- Zustand und Pods per `kubectl` prüfen

Damit ist das Kriterium „Eigenes Projekt auf Kubernetes installiert und dokumentiert“ erfüllt.

## Verwendete Technologien

- **Frontend:** HTML, CSS, JavaScript
- **Webserver/Container:** Nginx (Alpine)
- **Containerisierung:** Docker
- **Orchestrierung:** Kubernetes (Namespace, Deployment, Service)

## Aufbau der Anwendung & Ordnerstruktur

```text
.
├── app
│   ├── Dockerfile           # Container-Image für den Notenrechner
│   └── index.html           # Kleine Notenrechner-Webseite
├── k8s
│   ├── namespace.yaml       # Namespace ownproject
│   ├── deployment.yaml      # Deployment mit 2 Replikas
│   └── service.yaml         # NodePort-Service auf 30080
└── README.md
```

## Dockerfile

Die Anwendung wird über ein kleines Nginx-Image bereitgestellt:

```dockerfile
FROM nginx:1.27-alpine
COPY index.html /usr/share/nginx/html/index.html
```

## Kubernetes-Manifeste

Alle Manifeste liegen im Ordner `k8s/`:

- **namespace.yaml:** legt den Namespace `ownproject` an
- **deployment.yaml:** startet 2 Replikas des Notenrechners (`ownproject-grade-calculator:latest`)
- **service.yaml:** veröffentlicht die App als `NodePort` auf Port `30080`

## Installationsschritte und Befehle

### 1) Voraussetzungen

Benötigt werden:

- Docker
- kubectl
- Minikube (oder ein anderer lokaler Kubernetes-Cluster)

### 2) Cluster starten

```bash
minikube start
kubectl cluster-info
kubectl get nodes
```

### 3) Docker-Image bauen

Im Projekt-Root:

```bash
docker build -t ownproject-grade-calculator:latest ./app
```

> Hinweis: Für Minikube kann je nach Setup `eval $(minikube docker-env)` nötig sein, damit das Image direkt im Cluster verfügbar ist.

### 4) Kubernetes-Ressourcen anwenden

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

Alternativ:

```bash
kubectl apply -f k8s/
```

### 5) Status prüfen

```bash
kubectl get namespaces
kubectl get deployments -n ownproject
kubectl get pods -n ownproject
kubectl get svc -n ownproject
```

### 6) Anwendung testen

Direkt mit Minikube öffnen:

```bash
minikube service ownproject-service -n ownproject
```

Oder manuell aufrufen:

```bash
minikube ip
kubectl get svc ownproject-service -n ownproject
```

Danach ist die Anwendung unter `http://<MINIKUBE_IP>:30080` erreichbar.

## Funktionsweise des Notenrechners

Die Seite erlaubt:

1. Note eingeben (1.0 bis 6.0)
2. Mehrere Noten sammeln
3. Durchschnitt berechnen
4. Eingaben zurücksetzen

Ungültige Werte werden abgefangen, damit nur sinnvolle Noten in die Berechnung einfließen.

## Fehlerdiagnose

Wenn Pods nicht starten oder nicht erreichbar sind:

```bash
kubectl describe pod -n ownproject <POD_NAME>
kubectl logs -n ownproject <POD_NAME>
kubectl get events -n ownproject --sort-by=.metadata.creationTimestamp
```

## Aufräumen

Ressourcen entfernen:

```bash
kubectl delete -f k8s/
```

Minikube stoppen:

```bash
minikube stop
```
