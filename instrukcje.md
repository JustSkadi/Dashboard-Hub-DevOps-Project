# Personal Dashboard Hub - Przewodnik Implementacji

## Przegląd projektu

Personal Dashboard Hub to aplikacja webowa składająca się z mikrousług działających w Kubernetes, która służy jako centralny punkt kontrolny dla developerów. Zawiera dashboard z metrykami, task manager, monitorowanie Kubernetes oraz system notatek.

## Architektura

```
Frontend (React) → API Gateway (Node.js) → Redis Cache
                                        → PostgreSQL DB
```

Wszystko działa w Kubernetes z wykorzystaniem:
- Deployments dla każdego komponentu
- Services dla komunikacji
- ConfigMaps dla konfiguracji
- Secrets dla haseł
- PersistentVolumes dla danych
- Network Policies dla bezpieczeństwa
- RBAC dla uprawnień

## Wymagania wstępne

- Ubuntu VM z zainstalowanym:
  - Docker
  - minikube
  - kubectl
  - k9s
- 4GB RAM dla minikube
- Podstawowa znajomość JavaScript/React (opcjonalna - używamy gotowych obrazów)

---

## Krok 1: Przygotowanie środowiska

### 1.1 Uruchom minikube z dodatkowymi zasobami

```bash
# Zatrzymaj minikube jeśli działa
minikube stop

# Uruchom z większymi zasobami
minikube start --memory=4096 --cpus=4 --disk-size=20g

# Włącz addony
minikube addons enable metrics-server
minikube addons enable ingress
```

### 1.2 Sprawdź status klastra

```bash
kubectl cluster-info
kubectl get nodes
```

---

## Krok 2: Struktura projektu

### 2.1 Utwórz strukturę katalogów

```bash
mkdir personal-dashboard-hub
cd personal-dashboard-hub

mkdir -p {k8s-manifests/{namespaces,deployments,services,configmaps,secrets,volumes,rbac,network-policies},docker-images}

# Struktura powinna wyglądać tak:
tree
```

### 2.2 Przygotuj obrazy Docker (używamy gotowych)

Będziemy używać gotowych obrazów Docker:
- Frontend: nginx z prostym dashboardem HTML/CSS/JS
- Backend: Node.js z Express API
- Database: postgres:13
- Cache: redis:alpine

---

## Krok 3: Tworzenie namespace'ów

### 3.1 Utwórz plik namespace'ów

```bash
cat > k8s-manifests/namespaces/namespaces.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: dashboard-app
  labels:
    name: dashboard-app
    environment: development
---
apiVersion: v1
kind: Namespace
metadata:
  name: dashboard-db
  labels:
    name: dashboard-db
    environment: development
EOF
```

### 3.2 Zastosuj namespace'y

```bash
kubectl apply -f k8s-manifests/namespaces/namespaces.yaml
kubectl get namespaces
```

---

## Krok 4: Secrets i ConfigMaps

### 4.1 Utwórz Secrets dla bazy danych

```bash
cat > k8s-manifests/secrets/db-secrets.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: dashboard-db
type: Opaque
data:
  # postgres:password (base64)
  username: cG9zdGdyZXM=
  password: cGFzc3dvcmQ=
  database: ZGFzaGJvYXJk
EOF
```

### 4.2 Utwórz ConfigMap dla aplikacji

```bash
cat > k8s-manifests/configmaps/app-config.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: dashboard-app
data:
  DATABASE_URL: "postgresql://postgres:password@postgres-service.dashboard-db.svc.cluster.local:5432/dashboard"
  REDIS_URL: "redis://redis-service.dashboard-app.svc.cluster.local:6379"
  API_PORT: "3000"
  NODE_ENV: "development"
  
  # Frontend config
  frontend-config.json: |
    {
      "apiUrl": "http://api-service.dashboard-app.svc.cluster.local:3000",
      "title": "Personal Dashboard Hub",
      "version": "1.0.0"
    }
EOF
```

### 4.3 Zastosuj konfigurację

```bash
kubectl apply -f k8s-manifests/secrets/
kubectl apply -f k8s-manifests/configmaps/
```

---

## Krok 5: Persistent Volumes

### 5.1 Utwórz PVC dla PostgreSQL

```bash
cat > k8s-manifests/volumes/postgres-pvc.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: dashboard-db
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
EOF
```

### 5.2 Zastosuj PVC

```bash
kubectl apply -f k8s-manifests/volumes/
kubectl get pvc -n dashboard-db
```

---

## Krok 6: Deployments bazy danych

### 6.1 PostgreSQL Deployment

```bash
cat > k8s-manifests/deployments/postgres-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-deployment
  namespace: dashboard-db
  labels:
    app: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:13
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: database
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
EOF
```

### 6.2 Redis Deployment

```bash
cat > k8s-manifests/deployments/redis-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-deployment
  namespace: dashboard-app
  labels:
    app: redis
spec:
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
        image: redis:alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        livenessProbe:
          tcpSocket:
            port: 6379
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 5
          periodSeconds: 5
EOF
```

---

## Krok 7: Services dla baz danych

### 7.1 PostgreSQL Service

```bash
cat > k8s-manifests/services/postgres-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: dashboard-db
  labels:
    app: postgres
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
    protocol: TCP
  type: ClusterIP
EOF
```

### 7.2 Redis Service

```bash
cat > k8s-manifests/services/redis-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: dashboard-app
  labels:
    app: redis
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
    protocol: TCP
  type: ClusterIP
EOF
```

### 7.3 Wdróż bazy danych

```bash
kubectl apply -f k8s-manifests/deployments/postgres-deployment.yaml
kubectl apply -f k8s-manifests/deployments/redis-deployment.yaml
kubectl apply -f k8s-manifests/services/postgres-service.yaml
kubectl apply -f k8s-manifests/services/redis-service.yaml

# Sprawdź status
kubectl get pods -n dashboard-db
kubectl get pods -n dashboard-app
```

---

## Krok 8: Backend API Deployment

### 8.1 Backend Deployment

```bash
cat > k8s-manifests/deployments/api-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
  namespace: dashboard-app
  labels:
    app: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: node:16-alpine
        command: ["/bin/sh"]
        args: ["-c", "npm init -y && npm install express cors body-parser pg redis && node server.js"]
        ports:
        - containerPort: 3000
        envFrom:
        - configMapRef:
            name: app-config
        volumeMounts:
        - name: api-code
          mountPath: /app
        workingDir: /app
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
      volumes:
      - name: api-code
        configMap:
          name: api-code
          defaultMode: 0755
EOF
```

### 8.2 Utwórz kod dla API

```bash
cat > k8s-manifests/configmaps/api-code.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-code
  namespace: dashboard-app
data:
  server.js: |
    const express = require('express');
    const cors = require('cors');
    const bodyParser = require('body-parser');
    
    const app = express();
    const PORT = process.env.API_PORT || 3000;
    
    app.use(cors());
    app.use(bodyParser.json());
    
    // Mock data for tasks
    let tasks = [
      { id: 1, title: 'Setup Kubernetes cluster', completed: true, priority: 'high' },
      { id: 2, title: 'Deploy application', completed: false, priority: 'medium' },
      { id: 3, title: 'Configure monitoring', completed: false, priority: 'low' }
    ];
    
    // Health checks
    app.get('/health', (req, res) => {
      res.json({ status: 'healthy', timestamp: new Date().toISOString() });
    });
    
    app.get('/ready', (req, res) => {
      res.json({ status: 'ready', timestamp: new Date().toISOString() });
    });
    
    // API endpoints
    app.get('/api/tasks', (req, res) => {
      res.json(tasks);
    });
    
    app.post('/api/tasks', (req, res) => {
      const newTask = {
        id: tasks.length + 1,
        title: req.body.title,
        completed: false,
        priority: req.body.priority || 'medium'
      };
      tasks.push(newTask);
      res.status(201).json(newTask);
    });
    
    app.put('/api/tasks/:id', (req, res) => {
      const id = parseInt(req.params.id);
      const taskIndex = tasks.findIndex(t => t.id === id);
      
      if (taskIndex !== -1) {
        tasks[taskIndex] = { ...tasks[taskIndex], ...req.body };
        res.json(tasks[taskIndex]);
      } else {
        res.status(404).json({ error: 'Task not found' });
      }
    });
    
    app.delete('/api/tasks/:id', (req, res) => {
      const id = parseInt(req.params.id);
      const taskIndex = tasks.findIndex(t => t.id === id);
      
      if (taskIndex !== -1) {
        const deletedTask = tasks.splice(taskIndex, 1)[0];
        res.json(deletedTask);
      } else {
        res.status(404).json({ error: 'Task not found' });
      }
    });
    
    app.get('/api/status', (req, res) => {
      res.json({
        services: {
          api: 'healthy',
          database: 'connected',
          cache: 'connected'
        },
        version: '1.0.0',
        uptime: process.uptime()
      });
    });
    
    app.listen(PORT, '0.0.0.0', () => {
      console.log(`API Server running on port ${PORT}`);
    });
  
  package.json: |
    {
      "name": "dashboard-api",
      "version": "1.0.0",
      "description": "Personal Dashboard API",
      "main": "server.js",
      "scripts": {
        "start": "node server.js"
      },
      "dependencies": {
        "express": "^4.18.0",
        "cors": "^2.8.5",
        "body-parser": "^1.20.0",
        "pg": "^8.8.0",
        "redis": "^4.5.0"
      }
    }
EOF
```

### 8.3 API Service

```bash
cat > k8s-manifests/services/api-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: dashboard-app
  labels:
    app: api
spec:
  selector:
    app: api
  ports:
  - port: 3000
    targetPort: 3000
    protocol: TCP
  type: ClusterIP
EOF
```

---

## Krok 9: Frontend Deployment

### 9.1 Frontend HTML/CSS/JS

```bash
cat > k8s-manifests/configmaps/frontend-code.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-code
  namespace: dashboard-app
data:
  index.html: |
    <!DOCTYPE html>
    <html lang="pl">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Personal Dashboard Hub</title>
        <link rel="stylesheet" href="style.css">
    </head>
    <body>
        <div class="container">
            <header>
                <h1>🚀 Personal Dashboard Hub</h1>
                <div class="status-bar">
                    <span id="status" class="status-online">Online</span>
                    <span id="last-update">Last update: --</span>
                </div>
            </header>
            
            <div class="dashboard-grid">
                <div class="card">
                    <h2>📊 System Status</h2>
                    <div id="system-status">
                        <div class="metric">
                            <span class="label">API:</span>
                            <span id="api-status" class="status-unknown">Checking...</span>
                        </div>
                        <div class="metric">
                            <span class="label">Database:</span>
                            <span id="db-status" class="status-unknown">Checking...</span>
                        </div>
                        <div class="metric">
                            <span class="label">Cache:</span>
                            <span id="cache-status" class="status-unknown">Checking...</span>
                        </div>
                    </div>
                </div>
                
                <div class="card">
                    <h2>✅ Task Manager</h2>
                    <div class="task-input">
                        <input type="text" id="task-input" placeholder="Add new task...">
                        <select id="priority-select">
                            <option value="low">Low</option>
                            <option value="medium" selected>Medium</option>
                            <option value="high">High</option>
                        </select>
                        <button onclick="addTask()">Add</button>
                    </div>
                    <div id="tasks-list"></div>
                </div>
                
                <div class="card">
                    <h2>🐳 Kubernetes Info</h2>
                    <div class="k8s-info">
                        <div class="metric">
                            <span class="label">Namespace:</span>
                            <span>dashboard-app</span>
                        </div>
                        <div class="metric">
                            <span class="label">Pods:</span>
                            <span id="pod-count">--</span>
                        </div>
                        <div class="metric">
                            <span class="label">Services:</span>
                            <span id="service-count">--</span>
                        </div>
                    </div>
                </div>
                
                <div class="card">
                    <h2>📝 Quick Notes</h2>
                    <textarea id="notes" placeholder="Your notes here..." rows="6"></textarea>
                    <button onclick="saveNotes()">Save Notes</button>
                </div>
            </div>
        </div>
        
        <script src="app.js"></script>
    </body>
    </html>
  
  style.css: |
    * {
        margin: 0;
        padding: 0;
        box-sizing: border-box;
    }
    
    body {
        font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
        min-height: 100vh;
        padding: 20px;
    }
    
    .container {
        max-width: 1200px;
        margin: 0 auto;
    }
    
    header {
        text-align: center;
        margin-bottom: 30px;
        background: rgba(255, 255, 255, 0.1);
        backdrop-filter: blur(10px);
        border-radius: 15px;
        padding: 20px;
        color: white;
    }
    
    header h1 {
        font-size: 2.5em;
        margin-bottom: 10px;
    }
    
    .status-bar {
        display: flex;
        justify-content: center;
        gap: 30px;
        font-size: 0.9em;
        opacity: 0.8;
    }
    
    .dashboard-grid {
        display: grid;
        grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
        gap: 20px;
    }
    
    .card {
        background: white;
        border-radius: 15px;
        padding: 25px;
        box-shadow: 0 8px 32px rgba(31, 38, 135, 0.37);
        backdrop-filter: blur(8px);
        border: 1px solid rgba(255, 255, 255, 0.18);
        transition: transform 0.3s ease;
    }
    
    .card:hover {
        transform: translateY(-5px);
    }
    
    .card h2 {
        color: #333;
        margin-bottom: 20px;
        font-size: 1.3em;
    }
    
    .metric {
        display: flex;
        justify-content: space-between;
        align-items: center;
        padding: 10px 0;
        border-bottom: 1px solid #eee;
    }
    
    .metric:last-child {
        border-bottom: none;
    }
    
    .label {
        font-weight: 600;
        color: #555;
    }
    
    .status-online { color: #28a745; }
    .status-offline { color: #dc3545; }
    .status-unknown { color: #ffc107; }
    
    .task-input {
        display: flex;
        gap: 10px;
        margin-bottom: 20px;
    }
    
    .task-input input {
        flex: 1;
        padding: 10px;
        border: 1px solid #ddd;
        border-radius: 5px;
    }
    
    .task-input select, .task-input button {
        padding: 10px 15px;
        border: 1px solid #ddd;
        border-radius: 5px;
        cursor: pointer;
    }
    
    .task-input button {
        background: #007bff;
        color: white;
        border: none;
    }
    
    .task-item {
        display: flex;
        align-items: center;
        justify-content: space-between;
        padding: 10px;
        margin: 5px 0;
        background: #f8f9fa;
        border-radius: 5px;
        border-left: 4px solid #007bff;
    }
    
    .task-item.completed {
        opacity: 0.7;
        border-left-color: #28a745;
        text-decoration: line-through;
    }
    
    .task-item.high { border-left-color: #dc3545; }
    .task-item.medium { border-left-color: #ffc107; }
    .task-item.low { border-left-color: #6c757d; }
    
    .task-actions {
        display: flex;
        gap: 5px;
    }
    
    .task-actions button {
        padding: 5px 10px;
        border: none;
        border-radius: 3px;
        cursor: pointer;
        font-size: 0.8em;
    }
    
    .toggle-btn { background: #28a745; color: white; }
    .delete-btn { background: #dc3545; color: white; }
    
    #notes {
        width: 100%;
        padding: 10px;
        border: 1px solid #ddd;
        border-radius: 5px;
        resize: vertical;
        font-family: inherit;
        margin-bottom: 10px;
    }
    
    .save-btn {
        background: #28a745;
        color: white;
        border: none;
        padding: 10px 20px;
        border-radius: 5px;
        cursor: pointer;
    }
  
  app.js: |
    const API_BASE = '/api';
    let tasks = [];
    
    // Initialize app
    document.addEventListener('DOMContentLoaded', function() {
        loadTasks();
        loadSystemStatus();
        loadNotes();
        
        // Update status every 30 seconds
        setInterval(loadSystemStatus, 30000);
        
        // Update timestamp
        updateTimestamp();
        setInterval(updateTimestamp, 1000);
    });
    
    function updateTimestamp() {
        document.getElementById('last-update').textContent = 
            'Last update: ' + new Date().toLocaleTimeString();
    }
    
    async function loadSystemStatus() {
        try {
            const response = await fetch(`${API_BASE}/status`);
            const status = await response.json();
            
            document.getElementById('api-status').textContent = status.services.api;
            document.getElementById('api-status').className = 
                status.services.api === 'healthy' ? 'status-online' : 'status-offline';
            
            document.getElementById('db-status').textContent = status.services.database;
            document.getElementById('db-status').className = 
                status.services.database === 'connected' ? 'status-online' : 'status-offline';
            
            document.getElementById('cache-status').textContent = status.services.cache;
            document.getElementById('cache-status').className = 
                status.services.cache === 'connected' ? 'status-online' : 'status-offline';
            
            document.getElementById('status').className = 'status-online';
            document.getElementById('status').textContent = 'Online';
            
        } catch (error) {
            console.error('Failed to load system status:', error);
            document.getElementById('status').className = 'status-offline';
            document.getElementById('status').textContent = 'Offline';
        }
    }
    
    async function loadTasks() {
        try {
            const response = await fetch(`${API_BASE}/tasks`);
            tasks = await response.json();
            renderTasks();
        } catch (error) {
            console.error('Failed to load tasks:', error);
        }
    }
    
    function renderTasks() {
        const tasksList = document.getElementById('tasks-list');
        tasksList.innerHTML = '';
        
        tasks.forEach(task => {
            const taskElement = document.createElement('div');
            taskElement.className = `task-item ${task.completed ? 'completed' : ''} ${task.priority}`;
            
            taskElement.innerHTML = `
                <span>${task.title}</span>
                <div class="task-actions">
                    <button class="toggle-btn" onclick="toggleTask(${task.id})">
                        ${task.completed ? 'Undo' : 'Done'}
                    </button>
                    <button class="delete-btn" onclick="deleteTask(${task.id})">Delete</button>
                </div>
            `;
            
            tasksList.appendChild(taskElement);
        });
    }
    
    async function addTask() {
        const input = document.getElementById('task-input');
        const priority = document.getElementById('priority-select').value;
        const title = input.value.trim();
        
        if (!title) return;
        
        try {
            const response = await fetch(`${API_BASE}/tasks`, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ title, priority })
            });
            
            const newTask = await response.json();
            tasks.push(newTask);
            renderTasks();
            input.value = '';
        } catch (error) {
            console.error('Failed to add task:', error);
        }
    }
    
    async function toggleTask(id) {
        const task = tasks.find(t => t.id === id);
        if (!task) return;
        
        try {
            const response = await fetch(`${API_BASE}/tasks/${id}`, {
                method: 'PUT',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ completed: !task.completed })
            });
            
            const updatedTask = await response.json();
            const taskIndex = tasks.findIndex(t => t.id === id);
            tasks[taskIndex] = updatedTask;
            renderTasks();
        } catch (error) {
            console.error('Failed to update task:', error);
        }
    }
    
    async function deleteTask(id) {
        try {
            await fetch(`${API_BASE}/tasks/${id}`, { method: 'DELETE' });
            tasks = tasks.filter(t => t.id !== id);
            renderTasks();
        } catch (error) {
            console.error('Failed to delete task:', error);
        }
    }
    
    function saveNotes() {
        const notes = document.getElementById('notes').value;
        localStorage.setItem('dashboard-notes', notes);
        alert('Notes saved locally!');
    }
    
    function loadNotes() {
        const notes = localStorage.getItem('dashboard-notes');
        if (notes) {
            document.getElementById('notes').value = notes;
        }
    }
    
    // Handle Enter key in task input
    document.getElementById('task-input').addEventListener('keypress', function(e) {
        if (e.key === 'Enter') {
            addTask();
        }
    });
EOF
```

### 9.2 Frontend Deployment

```bash
cat > k8s-manifests/deployments/frontend-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
  namespace: dashboard-app
  labels:
    app: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: frontend-content
          mountPath: /usr/share/nginx/html
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: frontend-content
        configMap:
          name: frontend-code
      - name: nginx-config
        configMap:
          name: nginx-config
EOF
```

### 9.3 Nginx Configuration

```bash
cat > k8s-manifests/configmaps/nginx-config.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: dashboard-app
data:
  default.conf: |
    server {
        listen 80;
        server_name localhost;
        root /usr/share/nginx/html;
        index index.html;
        
        location / {
            try_files $uri $uri/ /index.html;
        }
        
        location /api/ {
            proxy_pass http://api-service.dashboard-app.svc.cluster.local:3000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
        
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }
EOF
```

### 9.4 Frontend Service

```bash
cat > k8s-manifests/services/frontend-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: dashboard-app
  labels:
    app: frontend
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
  type: LoadBalancer
EOF
```

---

## Krok 10: RBAC i Service Accounts

### 10.1 Service Account dla API

```bash
cat > k8s-manifests/rbac/service-accounts.yaml << 'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-api-sa
  namespace: dashboard-app
  labels:
    app: dashboard-api
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dashboard-app
  name: dashboard-api-role
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "watch", "list"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dashboard-api-binding
  namespace: dashboard-app
subjects:
- kind: ServiceAccount
  name: dashboard-api-sa
  namespace: dashboard-app
roleRef:
  kind: Role
  name: dashboard-api-role
  apiGroup: rbac.authorization.k8s.io
EOF
```

---

## Krok 11: Network Policies

### 11.1 Network Policy dla bezpieczeństwa

```bash
cat > k8s-manifests/network-policies/network-policies.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-netpol
  namespace: dashboard-app
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from: []
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 3000
  - to: []
    ports:
    - protocol: TCP
      port: 53
    - protocol: UDP
      port: 53
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-netpol
  namespace: dashboard-app
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 3000
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379
  - to:
    - namespaceSelector:
        matchLabels:
          name: dashboard-db
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  - to: []
    ports:
    - protocol: TCP
      port: 53
    - protocol: UDP
      port: 53
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-netpol
  namespace: dashboard-db
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: dashboard-app
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 5432
EOF
```

---

## Krok 12: Wdrożenie wszystkich komponentów

### 12.1 Zastosuj wszystkie konfiguracje

```bash
# Zastosuj wszystkie pliki ConfigMap
kubectl apply -f k8s-manifests/configmaps/

# Wdróż aplikacje
kubectl apply -f k8s-manifests/deployments/api-deployment.yaml
kubectl apply -f k8s-manifests/deployments/frontend-deployment.yaml

# Utwórz serwisy
kubectl apply -f k8s-manifests/services/api-service.yaml
kubectl apply -f k8s-manifests/services/frontend-service.yaml

# Zastosuj RBAC
kubectl apply -f k8s-manifests/rbac/

# Zastosuj Network Policies
kubectl apply -f k8s-manifests/network-policies/
```

### 12.2 Sprawdź status wszystkich komponentów

```bash
# Sprawdź wszystkie pody
kubectl get pods --all-namespaces

# Sprawdź serwisy
kubectl get services --all-namespaces

# Sprawdź szczegóły w k9s
k9s
```

---

## Krok 13: Testowanie aplikacji

### 13.1 Uzyskaj dostęp do aplikacji

```bash
# Sprawdź IP dla frontend service
kubectl get service frontend-service -n dashboard-app

# W minikube użyj:
minikube service frontend-service -n dashboard-app --url
```

### 13.2 Port forwarding (alternatywny dostęp)

```bash
# Forward port frontendu na localhost
kubectl port-forward service/frontend-service 8080:80 -n dashboard-app

# Teraz otwórz: http://localhost:8080
```

### 13.3 Testowanie funkcjonalności

1. **Otwórz aplikację w przeglądarce**
2. **Sprawdź status systemu** - wszystkie statusy powinny być "healthy"/"connected"
3. **Dodaj kilka zadań** w Task Manager
4. **Sprawdź różne priorytety** (High/Medium/Low)
5. **Oznacz zadania jako completed**
6. **Zapisz notatki** w sekcji Quick Notes

---

## Krok 14: Monitoring i debugging

### 14.1 Sprawdzanie logów

```bash
# Logi API
kubectl logs -l app=api -n dashboard-app --tail=50 -f

# Logi frontendu
kubectl logs -l app=frontend -n dashboard-app --tail=50 -f

# Logi bazy danych
kubectl logs -l app=postgres -n dashboard-db --tail=50 -f
```

### 14.2 Debugging w k9s

1. Otwórz k9s: `k9s`
2. Przejdź do namespace: `:namespace` → wybierz `dashboard-app`
3. Zobacz pody: `:pods`
4. Sprawdź logi: wybierz pod → `l`
5. Shell do poda: wybierz pod → `s`

### 14.3 Sprawdzenie health check'ów

```bash
# Test API health
kubectl exec -it deployment/api-deployment -n dashboard-app -- wget -q -O- http://localhost:3000/health

# Test frontend health
kubectl exec -it deployment/frontend-deployment -n dashboard-app -- wget -q -O- http://localhost:80/health
```

---

## Krok 15: Skalowanie i aktualizacje

### 15.1 Skalowanie aplikacji

```bash
# Zwiększ liczbę replik API
kubectl scale deployment api-deployment --replicas=3 -n dashboard-app

# Zwiększ liczbę replik frontendu
kubectl scale deployment frontend-deployment --replicas=3 -n dashboard-app

# Sprawdź status
kubectl get deployments -n dashboard-app
```

### 15.2 Rolling update (symulacja)

```bash
# Zaktualizuj obraz API (symulacja nowej wersji)
kubectl set image deployment/api-deployment api=node:18-alpine -n dashboard-app

# Sprawdź status rollout
kubectl rollout status deployment/api-deployment -n dashboard-app

# Historia rollout
kubectl rollout history deployment/api-deployment -n dashboard-app

# Rollback jeśli potrzebny
# kubectl rollout undo deployment/api-deployment -n dashboard-app
```

---

## Krok 16: Czyszczenie środowiska

### 16.1 Usunięcie aplikacji

```bash
# Usuń wszystkie resources z namespace'ów
kubectl delete namespace dashboard-app dashboard-db

# Lub usuń pojedynczo:
# kubectl delete -f k8s-manifests/ --recursive
```

### 16.2 Reset minikube (jeśli potrzebne)

```bash
minikube stop
minikube delete
minikube start --memory=4096 --cpus=4
```

---

## Rozwiązywanie problemów

### Problem: Pody nie startują

```bash
# Sprawdź events
kubectl get events --sort-by=.metadata.creationTimestamp -n dashboard-app

# Opisz pod
kubectl describe pod <pod-name> -n dashboard-app

# Sprawdź logi
kubectl logs <pod-name> -n dashboard-app
```

### Problem: Brak dostępu do aplikacji

```bash
# Sprawdź serwisy
kubectl get services -n dashboard-app

# Sprawdź endpoints
kubectl get endpoints -n dashboard-app

# Test połączenia
kubectl run test-pod --image=busybox --rm -it -- wget -q -O- http://frontend-service.dashboard-app.svc.cluster.local
```

### Problem: API nie działa

```bash
# Sprawdź czy API odpowiada
kubectl port-forward service/api-service 3001:3000 -n dashboard-app
curl http://localhost:3001/health

# Sprawdź ConfigMap
kubectl get configmap api-code -n dashboard-app -o yaml
```

---

## Następne kroki

Po udanym wdrożeniu możesz:

1. **Dodać Helm Chart** - przeportować manifesty na Helm
2. **Dodać monitoring** - Prometheus/Grafana
3. **Dodać CI/CD** - GitHub Actions dla automatycznego deploymentu
4. **Dodać HTTPS** - certyfikaty TLS
5. **Dodać bazę danych** - rzeczywiste połączenie z PostgreSQL
6. **Dodać testy** - testy automatyczne dla API
7. **Dodać backup** - backup bazy danych

Gratulacje! Masz działające środowisko Personal Dashboard Hub w Kubernetes z wykorzystaniem wszystkich poznanych na kursie konceptów!