# Eigenes Projekt auf Kubernetes installiert und dokumentiert

Dieses Repository zeigt ein vollständiges, lauffähiges Beispiel, wie ein eigenes Projekt mit Kubernetes bereitgestellt wird.  
Die Anwendung wird als Container (Nginx) im Cluster ausgeführt und über einen Service im Netzwerk erreichbar gemacht.

## Ziel des Projekts

Das Ziel ist eine nachvollziehbare End-to-End-Anleitung:

1. Kubernetes-Umgebung starten
2. Ressourcen deployen
3. Bereitstellung prüfen
4. Anwendung aufrufen
5. Projekt sauber wieder entfernen

Die Anleitung ist absichtlich konkret formuliert und nicht als To-do-Liste gehalten.

## Projektstruktur

```text
.
├── README.md
└── k8s
    ├── deployment.yaml
    ├── namespace.yaml
    └── service.yaml
```

## Voraussetzungen

Für die Ausführung werden folgende Werkzeuge benötigt:

- Docker
- kubectl
- ein lokaler Kubernetes-Cluster (empfohlen: Minikube)

### Minikube einmalig installieren

Die Installation ist je nach Betriebssystem unterschiedlich. Die offizielle Dokumentation findet sich hier:  
https://minikube.sigs.k8s.io/docs/start/

## Kubernetes lokal starten

Nach der Installation wird der Cluster gestartet und als aktueller Kontext gesetzt:

```bash
minikube start
kubectl cluster-info
kubectl get nodes
```

Wenn mindestens ein Node im Status `Ready` angezeigt wird, ist die Umgebung bereit.

## Deployment des Projekts

Die Kubernetes-Ressourcen werden in der richtigen Reihenfolge angewendet:

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

Alternativ kann alles in einem Schritt angewendet werden:

```bash
kubectl apply -f k8s/
```

## Laufzeitprüfung

Nach dem Deployment wird der Zustand geprüft:

```bash
kubectl get namespaces
kubectl get deployments -n ownproject
kubectl get pods -n ownproject
kubectl get svc -n ownproject
```

Der Pod sollte nach kurzer Zeit den Status `Running` besitzen.

## Anwendung aufrufen

Da der Service als `NodePort` bereitgestellt wird, kann die Anwendung mit Minikube direkt geöffnet werden:

```bash
minikube service ownproject-service -n ownproject
```

Alternativ können URL und Port manuell abgefragt werden:

```bash
minikube ip
kubectl get svc ownproject-service -n ownproject
```

Danach ist die Anwendung unter `http://<MINIKUBE_IP>:30080` erreichbar.

## Technische Beschreibung der Ressourcen

Die Konfiguration ist bewusst einfach gehalten:

- **Namespace**: trennt die Ressourcen logisch vom Rest des Clusters (`ownproject`)
- **Deployment**: startet zwei Replikas der Container-Anwendung
- **Service (NodePort)**: stellt die Anwendung über Port `30080` extern erreichbar bereit

## Fehlerdiagnose

Wenn Pods nicht starten, helfen diese Befehle bei der Analyse:

```bash
kubectl describe pod -n ownproject <POD_NAME>
kubectl logs -n ownproject <POD_NAME>
kubectl get events -n ownproject --sort-by=.metadata.creationTimestamp
```

## Aufräumen

Zum Entfernen aller Ressourcen:

```bash
kubectl delete -f k8s/
```

Falls der lokale Cluster nicht mehr benötigt wird:

```bash
minikube stop
```
