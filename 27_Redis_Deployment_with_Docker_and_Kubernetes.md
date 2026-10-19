# Part 27: Redis Deployment with Docker and Kubernetes — Complete Deep Dive

> Redis deployment me sirf container start karna kaafi nahi hai. Production me persistence, password, health checks, resource limits, network isolation, backups aur Kubernetes probes configure karne hote hain.

---

## 1. Docker Redis Deployment

### Basic Development Container

```bash
docker run -d \
  --name redis-local \
  -p 6379:6379 \
  redis:7.4-alpine
```

### Production-Oriented Container

```bash
docker run -d \
  --name redis-prod \
  -p 127.0.0.1:6379:6379 \
  -v redis_data:/data \
  -e REDIS_ARGS="--requirepass strong-password --appendonly yes --appendfsync everysec" \
  redis:7.4-alpine
```

Docker me Redis data by default container removal par lost hota hai; persistence ke liye volume `/data` par mount karna aur AOF enable karna recommended hai. [379]

---

## 2. Docker Compose Production Setup

### `docker-compose.yml`

```yaml
services:
  redis:
    image: redis:7.4-alpine
    container_name: redis
    restart: unless-stopped
    ports:
      - "127.0.0.1:6379:6379"
    environment:
      REDIS_PASSWORD: ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD}
      --appendonly yes
      --appendfsync everysec
      --maxmemory 512mb
      --maxmemory-policy allkeys-lru
    healthcheck:
      test: ["CMD-SHELL", "redis-cli -a \"$$REDIS_PASSWORD\" ping | grep -q PONG"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    deploy:
      resources:
        limits:
          memory: 768M
          cpus: "1.0"
        reservations:
          memory: 256M
          cpus: "0.25"

volumes:
  redis_data:
```

### `.env`

```env
REDIS_PASSWORD=strong-production-password
```

### Start

```bash
docker compose up -d
```

### Check Health

```bash
docker compose ps
```

### Test Redis

```bash
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" ping
```

Expected:

```text
PONG
```

Named Docker volume `/data` par mount hota hai, aur AOF persistence har write ko disk par log karta hai. [379][387]

---

## 3. Docker Persistence

### Volume Types

| Type | Example | Use |
|---|---|---|
| Named volume | `redis_data:/data` | Recommended |
| Bind mount | `./redis-data:/data` | Local development |
| Host volume | `/var/lib/redis:/data` | Controlled host setup |

### Verify Persistence

```bash
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" SET test:key "hello"
docker compose restart redis
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" GET test:key
```

Expected:

```text
"hello"
```

---

## 4. Docker Security

### Good

```yaml
ports:
  - "127.0.0.1:6379:6379"
```

### Bad

```yaml
ports:
  - "0.0.0.0:6379:6379"
```

### Security Rules

- Redis ko public IP par expose mat karo.
- Password use karo.
- TLS enable karo for external/untrusted network.
- Resource limits set karo.
- Container image pinned version use karo.
- Redis container ko read-only root filesystem consider karo, `/data` writable rakho.

---

## 5. Kubernetes Redis Deployment

Kubernetes me Redis ko `Deployment` ke bajaye `StatefulSet` me deploy karna better hota hai jab stable identity aur persistent storage chahiye.

StatefulSet har Pod ko stable, unique identity deta hai aur persistent storage ke saath applications manage karne ke liye useful hota hai. [377]

### Redis ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: redis
data:
  redis.conf: |
    appendonly yes
    appendfsync everysec
    maxmemory 512mb
    maxmemory-policy allkeys-lru
    protected-mode yes
```

---

## 6. Redis Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: redis-secret
  namespace: redis
type: Opaque
stringData:
  REDIS_PASSWORD: strong-production-password
```

---

## 7. Redis StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: redis
spec:
  serviceName: redis-headless
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7.4-alpine
          command:
            - redis-server
            - /etc/redis/redis.conf
            - --requirepass
            - $(REDIS_PASSWORD)
          env:
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: redis-secret
                  key: REDIS_PASSWORD
          ports:
            - containerPort: 6379
              name: redis
          volumeMounts:
            - name: redis-config
              mountPath: /etc/redis
            - name: redis-data
              mountPath: /data
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "768Mi"
              cpu: "1"
          livenessProbe:
            exec:
              command:
                - redis-cli
                - -a
                - $(REDIS_PASSWORD)
                - ping
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          readinessProbe:
            exec:
              command:
                - redis-cli
                - -a
                - $(REDIS_PASSWORD)
                - ping
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
      volumes:
        - name: redis-config
          configMap:
            name: redis-config
  volumeClaimTemplates:
    - metadata:
        name: redis-data
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
```

Kubernetes me Redis persistence ke liye PersistentVolumeClaims use hote hain jo Pod restart/rescheduling ke baad bhi data retain karte hain. [388]

---

## 8. Kubernetes Services

### Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-headless
  namespace: redis
spec:
  clusterIP: None
  selector:
    app: redis
  ports:
    - name: redis
      port: 6379
      targetPort: 6379
```

### Client Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: redis
spec:
  selector:
    app: redis
  ports:
    - name: redis
      port: 6379
      targetPort: 6379
```

---

## 9. Kubernetes Probes

| Probe | Purpose | Redis Example |
|---|---|---|
| `livenessProbe` | Container alive hai ya stuck hai | `redis-cli ping` |
| `readinessProbe` | Pod traffic serve karne ke liye ready hai | `redis-cli ping` |
| `startupProbe` | Slow-starting container ko extra time dena | Initial startup |

Liveness probe fail hone par container restart hota hai; readiness probe fail hone par Pod Service endpoints se temporarily remove ho jata hai. [378][385]

### Probe Tuning

```yaml
livenessProbe:
  initialDelaySeconds: 10
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 3

readinessProbe:
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3
```

---

## 10. Kubernetes Commands

### Deploy

```bash
kubectl apply -f redis-configmap.yaml
kubectl apply -f redis-secret.yaml
kubectl apply -f redis-statefulset.yaml
kubectl apply -f redis-service.yaml
```

### Verify

```bash
kubectl get pods -n redis
kubectl get pvc -n redis
kubectl get svc -n redis
```

### Check Logs

```bash
kubectl logs -f redis-0 -n redis
```

### Test Redis

```bash
kubectl exec -it redis-0 -n redis -- redis-cli -a strong-production-password ping
```

### Persistence Test

```bash
kubectl exec -it redis-0 -n redis -- redis-cli -a strong-production-password SET test:key "hello"

kubectl delete pod redis-0 -n redis

kubectl exec -it redis-0 -n redis -- redis-cli -a strong-production-password GET test:key
```

Expected:

```text
"hello"
```

Pod deletion ke baad bhi PVC-backed data survive karta hai — ye persistence validation ka important test hai. [388]

---

## 11. Redis Helm Chart

Bitnami Redis Helm chart standalone, replication aur Sentinel modes support karta hai; persistence, secrets, ConfigMaps aur Services automatically manage karta hai. [383][389]

### Add Helm Repo

```bash
helm repo add bitnami [https://charts.bitnami.com/bitnami](https://charts.bitnami.com/bitnami)
helm repo update
```

### Install Standalone Redis

```bash
helm install redis bitnami/redis \
  --namespace redis \
  --create-namespace \
  --set architecture=standalone \
  --set auth.password=strong-production-password \
  --set master.persistence.enabled=true \
  --set master.persistence.size=10Gi
```

### Install Replication with Sentinel

```bash
helm install redis bitnami/redis \
  --namespace redis \
  --create-namespace \
  --set architecture=replication \
  --set auth.password=strong-production-password \
  --set sentinel.enabled=true \
  --set replica.replicaCount=2 \
  --set master.persistence.size=10Gi
```

Bitnami Redis chart master-replica cluster deploy karta hai aur Sentinel enable karne ka option deta hai. [383][386]

---

## 12. Production Kubernetes Checklist

- [ ] Redis StatefulSet use ho raha hai.
- [ ] PersistentVolumeClaim configured hai.
- [ ] StorageClass fast SSD-backed hai.
- [ ] Password Kubernetes Secret me hai.
- [ ] Redis publicly exposed nahi hai.
- [ ] NetworkPolicy configured hai.
- [ ] Liveness/readiness probes set hain.
- [ ] Resource requests/limits set hain.
- [ ] AOF/RDB persistence enabled hai.
- [ ] Backup CronJob configured hai.
- [ ] Monitoring/Prometheus integration hai.
- [ ] PodDisruptionBudget configured hai.
- [ ] Failover testing done hai.

---

## 13. Kubernetes NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: redis-network-policy
  namespace: redis
spec:
  podSelector:
    matchLabels:
      app: redis
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: user-service
      ports:
        - protocol: TCP
          port: 6379
```

---

## 14. Common Deployment Mistakes

| Mistake | Problem | Correct Approach |
|---|---|---|
| No persistent volume | Pod restart par data loss | PVC + StatefulSet |
| Redis public exposure | Security breach | Private service/NetworkPolicy |
| No health probes | Unhealthy Pod traffic receive karta hai | Liveness + readiness |
| No resource limits | Node resource exhaustion | Requests + limits |
| Deployment for Redis | Unstable identity/storage | StatefulSet |
| No backup | Disaster recovery missing | Backup CronJob |
| Latest Redis tag | Unexpected upgrade behavior | Pinned version |

---

## 15. Interview Questions

| Question | Answer |
|---|---|
| Docker me Redis data persist kaise karte hain? | Volume ko `/data` par mount karke aur AOF/RDB enable karke. |
| Redis container me AOF kya karta hai? | Har write operation ko disk par log karta hai. |
| Kubernetes me Redis ke liye StatefulSet kyun use karte hain? | Stable pod identity, ordered management aur persistent storage ke liye. |
| PVC kya hai? | PersistentVolumeClaim — Pod ke liye persistent storage request. |
| Liveness probe kya karta hai? | Container stuck/unhealthy hone par restart karta hai. |
| Readiness probe kya karta hai? | Pod ready na ho to Service se traffic remove karta hai. |
| Helm chart kya hai? | Kubernetes applications ko parameterized packages me deploy karne ka tool. |
| Kubernetes me Redis security kaise karte hain? | Secret, NetworkPolicy, private Service aur authentication se. |

---

## 16. Final Mental Model

```text
Docker:
Container + Volume + AOF + Healthcheck

Kubernetes:
StatefulSet + PVC + Secret + Service + Probes

Production:
Private network
+
Persistence
+
Backups
+
Monitoring
+
Resource limits
```

---

## Next Part

**Part 28: Redis Testing in Spring Boot — Unit, Integration, Testcontainers and Load Testing — Complete Deep Dive**

Isme hum cover karenge:

- Redis testing strategy
- Unit testing Redis services
- Mocking RedisTemplate
- Integration testing
- Testcontainers Redis
- Cache testing
- TTL testing
- Pub/Sub testing
- Stream testing
- Distributed lock testing
- Load testing
- Production testing checklist
-