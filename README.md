# Dashboard Hub

#### 1. Frontend (React-like SPA)
- **Technologia**: HTML5, CSS3, JavaScript (Vanilla)
- **Funkcjonalność**:
  - Dashboard z metrykami systemu
  - Task Manager z priorytetami
  - System notatek z localStorage
  - Monitoring stanu serwisów
- **Hosting**: Nginx w kontenerze
- **Port**: 80

#### 2. Backend API
- **Technologia**: Node.js + Express
- **Funkcjonalność**:
  - REST API dla zadań (CRUD)
  - Health check endpoints
  - Status systemu
  - Integracja z Redis i PostgreSQL
- **Port**: 3000

#### 3. Baza Danych
- **Technologia**: PostgreSQL 13
- **Zastosowanie**: Trwałe przechowywanie danych aplikacji
- **Storage**: PersistentVolume 5GB
- **Port**: 5432

#### 4. Cache
- **Technologia**: Redis Alpine
- **Zastosowanie**: Cache dla sesji i danych tymczasowych
- **Port**: 6379

### Architektura Sieciowa

```
Internet → LoadBalancer → Frontend (Nginx) → API Gateway → Redis
                                          ↓
                                    PostgreSQL
```

### Bezpieczeństwo
- **RBAC**: Role-based access control dla API
- **Network Policies**: Izolacja ruchu sieciowego
- **Secrets**: Hasła bazy danych w zaszyfrowanej formie
- **Service Accounts**: Dedykowane konta dla aplikacji

---

## Środowisko i Wymagania

### Infrastruktura
- **Platforma**: Ubuntu 24.04 na maszynie wirtualnej
- **Kubernetes**: minikube (single-node cluster)
- **Container Runtime**: Docker
- **Zasoby**:
  - RAM: minimum 4GB (zalecane 8GB)
  - CPU: minimum 2 rdzenie (zalecane 4)
  - Storage: 20GB wolnego miejsca

### Narzędzia
- **kubectl**: CLI Kubernetes
- **k9s**: Terminal UI dla Kubernetes
- **minikube**: Lokalne środowisko Kubernetes
- **Docker**: Konteneryzacja aplikacji

---

## Szybkie wdrożenie

### 1. Uruchomienie minikube
```bash
minikube start --memory=4096 --cpus=4 --disk-size=20g

# Sprawdź status
kubectl get nodes
```

### 2. Wdrożenie aplikacji
```bash
# Zastosuj wszystkie manifesty Kubernetes
kubectl apply -f k8s-manifests/ --recursive

# Sprawdź status wdrożenia (poczekaj na Running)
kubectl get pods --all-namespaces -w
```

### 3. Dostęp do aplikacji
```bash
# Opcja 1: Port forwarding (zalecane dla testów)
kubectl port-forward service/frontend-service 8080:80 -n dashboard-app

# Aplikacja dostępna pod: http://localhost:8080
```
## Konfiguracja i Zarządzanie

### ConfigMaps
1. **app-config**: Zmienne środowiskowe aplikacji
2. **api-code**: Kod źródłowy API Node.js
3. **frontend-code**: Pliki HTML/CSS/JS
4. **nginx-config**: Konfiguracja reverse proxy

### Secrets
1. **postgres-secret**: Dane dostępowe do bazy danych

### Persistent Volumes
1. **postgres-pvc**: 5GB storage dla PostgreSQL

### Services
1. **frontend-service**: LoadBalancer (port 80)
2. **api-service**: ClusterIP (port 3000)
3. **redis-service**: ClusterIP (port 6379)
4. **postgres-service**: ClusterIP (port 5432)

---

## Bezpieczeństwo i RBAC

### Service Accounts
- **dashboard-api-sa**: Konto dla API z ograniczonymi uprawnieniami

### Role Permissions
```yaml
dashboard-api-role:
  - get, watch, list: pods, services
  - get, watch, list: deployments
```

### Network Policies
1. **frontend-netpol**: Frontend może komunikować się z API
2. **api-netpol**: API może komunikować się z Redis i PostgreSQL
3. **postgres-netpol**: PostgreSQL przyjmuje tylko połączenia od API

---

## Monitoring i Debugging

### Monitorowanie z k9s
```bash
# Uruchom graficzny interfejs Kubernetes
k9s

# Przydatne skróty w k9s:
# :pods - zobacz wszystkie pody
# :services - zobacz serwisy  
# :namespaces - przełączaj między namespace'ami
# l - logi wybranego poda
# s - shell do poda
# d - usuń zasób
# e - edytuj zasób
```

### Sprawdzanie logów
```bash
# Logi z API
kubectl logs -l app=api -n dashboard-app --tail=50 -f

# Logi z frontendu
kubectl logs -l app=frontend -n dashboard-app --tail=50 -f

# Logi z bazy danych
kubectl logs -l app=postgres -n dashboard-db --tail=50 -f
```

### Health Checks
```bash
# Test API health
kubectl port-forward service/api-service 3001:3000 -n dashboard-app
curl http://localhost:3001/health

# Test frontend health
kubectl port-forward service/frontend-service 8080:80 -n dashboard-app
curl http://localhost:8080
```

---

## API Endpoints

### Health Check
- `GET /health` - Status aplikacji
- `GET /ready` - Gotowość aplikacji

### System Status
- `GET /api/status` - Status wszystkich komponentów

### Task Management
- `GET /api/tasks` - Lista wszystkich zadań
- `POST /api/tasks` - Dodaj nowe zadanie
- `PUT /api/tasks/:id` - Aktualizuj zadanie
- `DELETE /api/tasks/:id` - Usuń zadanie
