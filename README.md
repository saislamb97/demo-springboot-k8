# 🚀 Spring Boot Deployment with Docker & K3s Kubernetes on AWS EC2

This guide walks you through deploying a Spring Boot application using **Docker** and **K3s Kubernetes**, fully within an **Ubuntu 22.04 EC2** instance.

---

## 🧰 Tech Stack

* Java 21, Spring Boot 3.5.4
* Thymeleaf, Spring Security, OpenAPI (Swagger)
* Docker (multi-stage build)
* K3s Kubernetes (lightweight)
* NGINX Ingress Controller (optional)
* Hosted on AWS EC2 (Ubuntu 22.04)

---

## 📁 Project Structure

```
demo-springboot-k8/
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

## ⚙️ 1. Build the Spring Boot JAR

```bash
./mvnw clean package -DskipTests
```

Result:

```
target/demo-0.0.1-SNAPSHOT.jar
```

---

## 🐳 2. Dockerize with Multi-Stage Build

```Dockerfile
# Stage 1: Build the JAR
FROM maven:3.9.6-eclipse-temurin-21 AS builder
WORKDIR /app
COPY . .
RUN mvn clean package -DskipTests

# Stage 2: Run the JAR
FROM eclipse-temurin:21-jdk
WORKDIR /app
COPY --from=builder /app/target/demo-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Build and test locally:

```bash
docker build -t springboot-demo .
docker run -p 8080:8080 springboot-demo
```

Export the image:

```bash
docker save springboot-demo > springboot-demo.tar
```

---

## ☸️ 3. Install K3s on EC2

```bash
curl -sfL https://get.k3s.io | sh -
alias kubectl='sudo k3s kubectl'
```

Make alias permanent:

```bash
echo "alias kubectl='sudo k3s kubectl'" >> ~/.bashrc
source ~/.bashrc
```

---

## 📦 4. Load Docker Image into K3s

```bash
sudo k3s ctr images import springboot-demo.tar
```

---

## 📄 5. Kubernetes Manifests

**`k8s/deployment.yaml`**

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

**`k8s/service.yaml`**

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

**`k8s/ingress.yaml` (optional)**

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

## 🚀 6. Apply K8s Resources

```bash
kubectl apply -f k8s/
```

---

## ✅ NodePort Exposure (Quick Access)

```bash
kubectl patch svc springboot-service -p '{"spec": {"type": "NodePort"}}'
kubectl get svc springboot-service
```

Then access your app via:

```
http://<EC2_PUBLIC_IP>:<NodePort>
```

Make sure the **NodePort** is open in your EC2 Security Group.

---

## 🔁 7. Redeploy or Update the App

When making code changes or updating the image:

```bash
# Rebuild and save new Docker image
docker build -t springboot-demo .
docker save springboot-demo > springboot-demo.tar
sudo k3s ctr images import springboot-demo.tar

# Reapply Kubernetes resources
kubectl delete -f k8s/
kubectl apply -f k8s/
```

Alternatively, for rolling restart:

```bash
kubectl rollout restart deployment springboot-demo
```

To delete everything:

```bash
kubectl delete all --all
```

---

## 🌐 8. Ingress Setup (Optional)

Install NGINX Ingress:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/cloud/deploy.yaml
kubectl get pods -n ingress-nginx
```

Then access your app via:

```
http://yourdomain.com
```

Make sure DNS points to your EC2 public IP.

---

## 🔍 9. Debug & Troubleshooting

```bash
kubectl get nodes               # Node status
kubectl get pods                # Pod status
kubectl describe pod <pod>      # Inspect pod
kubectl logs <pod>              # View logs
kubectl get svc                 # List services
kubectl get ingress             # List ingress routes
kubectl get pods -n ingress-nginx
kubectl describe pod -n ingress-nginx <nginx-pod>
```

---

## 📑 10. Swagger Docs

If enabled in your Spring Boot app:

* Via NodePort: `http://<EC2_PUBLIC_IP>:<NodePort>/swagger-ui.html`
* Via Ingress: `http://yourdomain.com/swagger-ui.html`