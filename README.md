# 🚀 Spring Boot Deployment with Docker & K3s Kubernetes on AWS EC2

This guide walks you through deploying a Spring Boot application using **Docker** and **K3s Kubernetes** entirely within an **EC2 Ubuntu 22.04** instance.

---

## 🧰 Tech Stack

* Java 21, Spring Boot 3.5.4
* Thymeleaf, Spring Security, OpenAPI (Swagger)
* Docker for containerization
* K3s for lightweight Kubernetes
* NGINX Ingress Controller
* Hosted on AWS EC2

---

## 📁 Project Structure

```
demo/
├── src/
├── pom.xml
├── Dockerfile
├── k8s/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
└── README.md
```

---

## ⚙️ 1. Build Spring Boot JAR

```bash
./mvnw clean install
```

This creates:

```
target/demo-0.0.1-SNAPSHOT.jar
```

---

## 🐳 2. Create Docker Image

**Dockerfile**

```Dockerfile
FROM eclipse-temurin:21-jdk
VOLUME /tmp
COPY target/demo-0.0.1-SNAPSHOT.jar app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

Build the image:

```bash
docker build -t springboot-demo .
```

---

## ☸️ 3. Install K3s on EC2

```bash
curl -sfL https://get.k3s.io | sh -
sudo k3s kubectl get nodes
alias kubectl='sudo k3s kubectl'
```

(Optional) Add to `~/.bashrc`:

```bash
echo "alias kubectl='sudo k3s kubectl'" >> ~/.bashrc
```

---

## 📦 4. Load Docker Image into K3s

If building inside EC2:

```bash
k3s ctr images import springboot-demo.tar
```

Or:

```bash
docker save springboot-demo > springboot-demo.tar
sudo k3s ctr images import springboot-demo.tar
```

---

## 📄 5. Kubernetes Manifests

### `k8s/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: springboot-demo
  template:
    metadata:
      labels:
        app: springboot-demo
    spec:
      containers:
        - name: springboot-demo
          image: springboot-demo:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
```

### `k8s/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: springboot-service
spec:
  selector:
    app: springboot-demo
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
```

### `k8s/ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: springboot-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: springboot-service
                port:
                  number: 80
```

---

## 🌐 6. Install Ingress Controller (NGINX)

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/cloud/deploy.yaml
kubectl get pods -n ingress-nginx
```

---

## 🚀 7. Apply Manifests

```bash
kubectl apply -f k8s/
```

Check status:

```bash
kubectl get pods
kubectl get svc
kubectl get ingress
```

---

## 🌍 8. Access the App

* Point your domain to your EC2 public IP.
* Ensure port 80 is open in EC2 Security Group.
* Visit: `http://yourdomain.com`

---

## 📑 9. Swagger Documentation

Once deployed:

```
http://yourdomain.com/swagger-ui.html
```