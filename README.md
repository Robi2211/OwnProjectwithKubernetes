# Eigenes Projekt auf Kubernetes: Notenrechner mit Frontend, Backend und MongoDB

## 1. Projektidee
Dieses Projekt ist eine vollständige, containerisierte Anwendung mit:

- **Frontend** (Notenrechner UI + Load-Balancer-Visualisierung)
- **Backend** (Node.js/Express API)
- **Datenbank** (MongoDB)

Die Anwendung wurde gezielt für den Kubernetes-Betrieb aufgebaut und enthält zusätzlich eine visuelle Demonstration, dass Requests über einen Kubernetes-Service auf mehrere Backend-Pods verteilt werden.

## Ziel des Projekts
Ziel ist es, eine reale, mehrschichtige Anwendung (Frontend, API-Backend, Datenbank) auf Kubernetes zu deployen und nachvollziehbar zu dokumentieren. Zusätzlich zeigt das Projekt grafisch, dass mehrere Pods aktiv sind und Lastverteilung funktioniert.

## Verwendete Technologien
- **Frontend:** HTML, CSS, JavaScript, Nginx
- **Backend:** Node.js, Express, Mongoose
- **Datenbank:** MongoDB
- **Infrastruktur:** Docker, Kubernetes (Deployments, Services)

---

## 2. Aufbau der Anwendung & Ordnerstruktur

```text
.
├── front-end/              # Frontend, läuft mit Nginx (Port 80)
│   ├── public/pods.html    # Visualisierung für den Load-Balancer-Test
│   ├── Dockerfile          # Docker-Image für das Frontend
│   ├── index.html          # Hauptseite (Notenrechner)
│   ├── nginx.conf          # Proxy-Config, leitet /api/ an Backend weiter
│   └── package.json        # Build-Skript für statische Auslieferung
├── back-end/               # Node.js Express API (Port 5000)
│   ├── server.js           # API inklusive /api/pod-info Endpoint
│   ├── Dockerfile          # Docker-Image für das Backend
│   └── package.json        # Backend-Abhängigkeiten
├── k8s/                    # Kubernetes-Manifeste
│   ├── backend.yaml        # Backend Deployment (3 Replikas) + Service
│   ├── frontend.yaml       # Frontend Deployment (3 Replikas) + LoadBalancer Service
│   └── mongo.yaml          # MongoDB Deployment (1 Replika) + ClusterIP Service
└── README.md
```

---

## 3. Dockerfiles

### Backend-Dockerfile (`back-end/Dockerfile`)
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 5000
CMD ["node", "server.js"]
```

### Frontend-Dockerfile (`front-end/Dockerfile`)
```dockerfile
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

## 4. Kubernetes YAML-Dateien
Alle Manifeste liegen in `k8s/`.

### `k8s/mongo.yaml`
- **Deployment:** Startet genau 1 Replika (`mongo:7`)
- **Service:** `mongo-svc` als `ClusterIP` auf Port `27017`

### `k8s/backend.yaml`
- **Deployment:** `replicas: 3` für die Node.js API
- **Environment:** `MONGO_URI=mongodb://mongo-svc:27017/lms`
- **Service:** `lms-backend` als `ClusterIP` auf Port `5000`

### `k8s/frontend.yaml`
- **Deployment:** `replicas: 3` für das Frontend (Nginx)
- **Service:** `lms-frontend` als `LoadBalancer` auf Port `80`

---

## 5. Installationsschritte und verwendete Befehle

### 1) Docker Images bauen
```bash
cd front-end
docker build -t lms-frontend:latest .
cd ../back-end
docker build -t lms-backend:latest .
cd ..
```

### 2) Kubernetes Ressourcen anwenden
```bash
kubectl apply -f k8s/mongo.yaml
kubectl apply -f k8s/backend.yaml
kubectl apply -f k8s/frontend.yaml
```

### 3) Status prüfen
```bash
kubectl get deployments
kubectl get pods -o wide
kubectl get svc
```

### 4) Testen
Falls lokal keine External-IP bereitgestellt wird:
```bash
kubectl port-forward svc/lms-frontend 8080:80
```

Danach im Browser:
- Hauptseite: `http://localhost:8080/`
- Load-Balancer-Seite: `http://localhost:8080/pods.html`

---

## 6. Überprüfung: Laufen Replikas & Load Balancing?

Erwartete Kontrolle per Deployments/Pods:
```bash
kubectl get deployments
kubectl get pods
```

Der Nachweis für Backend-Load-Balancing erfolgt über den Pod-Info-Endpoint:
```bash
for i in 1 2 3; do curl -s http://127.0.0.1:8080/api/pod-info; echo ""; sleep 1; done
```

Beispielausgabe:
```json
{"podName":"lms-backend-79b8cd466b-ncvnl","timestamp":"2026-03-25T11:34:24.157Z"}
{"podName":"lms-backend-79b8cd466b-ncvnl","timestamp":"2026-03-25T11:34:25.239Z"}
{"podName":"lms-backend-79b8cd466b-5wcnv","timestamp":"2026-03-25T11:34:26.286Z"}
```

Wenn unterschiedliche `podName`-Werte erscheinen, verteilt Kubernetes die Requests auf mehrere Backend-Replikas.

---

## 7. Visualisierung (Load-Balancing-Test Seite)

Unter `http://localhost:8080/pods.html` befindet sich ein Dashboard.

- Mit **"Auto request every 1s"** sendet der Browser regelmäßig Requests an `/api/pod-info`.
- Nginx im Frontend leitet `/api/` transparent an den Kubernetes-Service `lms-backend` weiter.
- Der Service verteilt die Requests auf die Backend-Pods.
- Der antwortende Pod wird als Button angezeigt und kurz pulsierend hervorgehoben.

So kann die Lastverteilung auch visuell einfach nachvollzogen werden.

---

## Mögliche Probleme und Lösungen

- **Backend startet neu (CrashLoopBackOff):** Kann auftreten, wenn MongoDB beim ersten Start noch nicht bereit ist. Kubernetes startet die Pods automatisch neu.
- **Keine External-IP für LoadBalancer:** Bei lokalen Clustern (Docker Desktop/Minikube) ist oft Port-Forwarding der einfachste Weg:
  ```bash
  kubectl port-forward svc/lms-frontend 8080:80
  ```
