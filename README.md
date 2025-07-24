# 🚀 Spring Boot Deployment with Docker & K3s Kubernetes on AWS EC2

This guide walks you through deploying a Spring Boot application using **Docker** and **K3s Kubernetes** entirely within an **EC2 Ubuntu 22.04** instance.

---

## 🧰 Tech Stack

* Java 21, Spring Boot 3.5.4
* Thymeleaf, Spring Security, OpenAPI (Swagger)
* Docker (multi-stage build)
* K3s Kubernetes
* NGINX Ingress Controller (optional)
* Hosted on AWS EC2

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

**Dockerfile**

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

Build Docker image:

```bash
docker build -t springboot-demo .
docker run -p 8080:8080 springboot-demo
```

Export image for K3s:

```bash
docker save springboot-demo > springboot-demo.tar
```

---

## ☸️ 3. Install K3s on EC2

```bash
curl -sfL https://get.k3s.io | sh -
alias kubectl='sudo k3s kubectl'
```

Add the alias permanently:

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

### `k8s/ingress.yaml` (optional)

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

Example output:

```
NAME                 TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
springboot-service   NodePort   10.43.189.7     <none>        80:30218/TCP   5m
```

Your app is now available at:

```
http://<EC2_PUBLIC_IP>:30218
```

Ensure port `30218` is open in your EC2 Security Group.

---

## 🌐 7. Ingress Setup (Optional for Domain)

Install NGINX ingress controller:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/cloud/deploy.yaml
kubectl get pods -n ingress-nginx
```

Make sure DNS points to your EC2 public IP, then access:

```
http://yourdomain.com
```

---

## 🔍 Debug & Troubleshooting

✅ 1. Check Node Status:

```bash
kubectl get nodes
```

✅ 2. Check Pod Status:

```bash
kubectl get pods
```

✅ 3. Describe the Pod:

```bash
kubectl describe pod <your-pod-name>
```

✅ 4. View Logs:

```bash
kubectl logs <your-pod-name>
```

✅ 5. Check Services:

```bash
kubectl get svc
```

✅ 6. Check Ingress:

```bash
kubectl get ingress
```

✅ 7. Check Ingress Controller:

```bash
kubectl get pods -n ingress-nginx
kubectl describe pod -n ingress-nginx <nginx-pod-name>
```

---

## 📑 8. Swagger Docs

If configured in your Spring Boot app:

```
http://<EC2_PUBLIC_IP>:<NodePort>/swagger-ui.html
```

Or via ingress:

```
http://yourdomain.com/swagger-ui.html
```