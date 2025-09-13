# Personal Dashboard Hub - Dokumentacja

### Komponenty Aplikacji

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

## Struktura Namespace'ów

### dashboard-app
- **Przeznaczenie**: Warstwa aplikacyjna
- **Komponenty**:
  - Frontend deployment + service
  - API deployment + service
  - Redis deployment + service
  - ConfigMaps z kodem aplikacji
  - Service Account dla API

### dashboard-db
- **Przeznaczenie**: Warstwa danych
- **Komponenty**:
  - PostgreSQL deployment + service
  - PersistentVolumeClaim
  - Secrets z danymi dostępowymi

---

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


## Zasoby i narzędzia

**Wymagane:**
- Ubuntu VM (4GB RAM, 2 CPU, 20GB storage)
- minikube, kubectl, k9s
- Git, edytor tekstu

**Opcjonalne:**
- Docker Desktop (dla lokalnego developmentu)
- Postman (testowanie API)
- Browser Developer Tools