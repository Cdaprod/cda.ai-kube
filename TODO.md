# Instructions for Deploying AI Application with Ansible

This guide provides step-by-step instructions to deploy the AI application using Ansible, including setting up MinIO as `ai-s3`, Weaviate as `ai-vdb`, a Python environment for the AI application, and a frontend environment.

#### Prerequisites

1. **Ansible Installed**: Ensure Ansible is installed on your local machine.
2. **Kubernetes Cluster**: A running Kubernetes cluster where the deployments will be executed.
3. **Environment Variables**: Set `MINIO_ACCESS_KEY` and `MINIO_SECRET_KEY` in your environment.
4. **NFS Server**: An NFS server configured with the appropriate path for persistent storage.

### Directory Structure

Create the following directory structure for organizing the Ansible playbooks and roles:

```
ansible/
├── group_vars/
│   └── all.yml
├── inventory/
│   └── hosts
├── roles/
│   ├── common/
│   │   └── tasks/
│   │       └── main.yml
│   ├── ai-s3/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── templates/
│   │   │   ├── deployment.yml.j2
│   │   │   ├── pvc.yml.j2
│   │   │   ├── pv.yml.j2
│   │   │   └── secret.yml.j2
│   ├── ai-vdb/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── templates/
│   │   │   └── deployment.yml.j2
│   ├── ai-py/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── templates/
│   │   │   └── deployment.yml.j2
│   ├── ai-frontend/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── templates/
│   │   │   └── deployment.yml.j2
└── playbook.yml
```

### Group Variables

Create the `group_vars/all.yml` file to store common variables.

```yaml
# group_vars/all.yml
minio_access_key: "{{ lookup('env', 'MINIO_ACCESS_KEY') }}"
minio_secret_key: "{{ lookup('env', 'MINIO_SECRET_KEY') }}"
nfs_server: "cda_ds"
nfs_path: "/volume1/cda-storage/minio/data"
namespace: "cdaprod-dev"
```

### Inventory File

Define the hosts in `inventory/hosts`.

```ini
# inventory/hosts
[all]
localhost ansible_connection=local
```

### Common Tasks

Create the `roles/common/tasks/main.yml` file to handle common tasks.

```yaml
# roles/common/tasks/main.yml
- name: Create namespace
  kubernetes.core.k8s:
    api_version: v1
    kind: Namespace
    name: "{{ namespace }}"
```

### MinIO (ai-s3) Role

Create tasks and templates for the MinIO deployment.

#### Tasks

```yaml
# roles/ai-s3/tasks/main.yml
- name: Create MinIO secret
  kubernetes.core.k8s:
    definition: "{{ lookup('template', 'secret.yml.j2') }}"
    
- name: Create NFS persistent volume
  kubernetes.core.k8s:
    definition: "{{ lookup('template', 'pv.yml.j2') }}"

- name: Create NFS persistent volume claim
  kubernetes.core.k8s:
    definition: "{{ lookup('template', 'pvc.yml.j2') }}"

- name: Deploy MinIO
  kubernetes.core.k8s:
    definition: "{{ lookup('template', 'deployment.yml.j2') }}"
```

#### Templates

##### `secret.yml.j2`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: minio-credentials
  namespace: {{ namespace }}
type: Opaque
stringData:
  accesskey: "{{ minio_access_key }}"
  secretkey: "{{ minio_secret_key }}"
```

##### `pv.yml.j2`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  nfs:
    path: {{ nfs_path }}
    server: {{ nfs_server }}
```

##### `pvc.yml.j2`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
  namespace: {{ namespace }}
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
```

##### `deployment.yml.j2`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-s3
  namespace: {{ namespace }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ai-s3
  template:
    metadata:
      labels:
        app: ai-s3
    spec:
      containers:
      - name: minio
        image: ghcr.io/Cdaprod/minio:latest
        args:
        - server
        - /data
        env:
        - name: MINIO_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: minio-credentials
              key: accesskey
        - name: MINIO_SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: minio-credentials
              key: secretkey
        volumeMounts:
        - name: nfs-storage
          mountPath: /data
      volumes:
      - name: nfs-storage
        persistentVolumeClaim:
          claimName: nfs-pvc
```

### Next Steps
Next, we will create the roles and templates for `ai-vdb`, `ai-py`, and `ai-frontend`.

---

### Updated Instructions for Deploying AI Application with Ansible (Continued)

Let's continue with the roles and templates for `ai-vdb`, `ai-py`, and `ai-frontend`.

### Weaviate (ai-vdb) Role

Create tasks and templates for the Weaviate deployment.

#### Tasks

```yaml
# roles/ai-vdb/tasks/main.yml
- name: Deploy Weaviate
  kubernetes.core.k8s:
    definition: "{{ lookup('template', 'deployment.yml.j2') }}"
```

#### Templates

##### `deployment.yml.j2`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-vdb
  namespace: {{ namespace }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ai-vdb
  template:
    metadata:
      labels:
        app: ai-vdb
    spec:
      containers:
      - name: weaviate
        image: ghcr.io/Cdaprod/weaviate-app:latest
        env:
        - name: STORAGE_BACKEND
          value: "s3"
        - name: S3_ENDPOINT
          value: "ai-s3:9000"
        - name: S3_BUCKET
          value: "weaviate-data"
        - name: S3_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: minio-credentials
              key: accesskey
        - name: S3_SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: minio-credentials
              key: secretkey
      imagePullSecrets:
      - name: ghcr-secret
```

### Python Environment for AI Application (ai-py)

Create tasks and templates for the AI application deployment.

#### Tasks

```yaml
# roles/ai-py/tasks/main.yml
- name: Deploy AI Python Application
  kubernetes.core.k8s:
    definition: "{{ lookup('template', 'deployment.yml.j2') }}"
```

#### Templates

##### `deployment.yml.j2`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-py
  namespace: {{ namespace }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ai-py
  template:
    metadata:
      labels:
        app: ai-py
    spec:
      containers:
      - name: ai-py
        image: ghcr.io/Cdaprod/ai-py:latest
        env:
        - name: MINIO_ENDPOINT
          value: "ai-s3:9000"
        - name: MINIO_BUCKET
          value: "weaviate-data"
        - name: MINIO_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: minio-credentials
              key: accesskey
        - name: MINIO_SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: minio-credentials
              key: secretkey
        volumeMounts:
        - name: nfs-storage
          mountPath: /app/data
      volumes:
      - name: nfs-storage
        persistentVolumeClaim:
          claimName: nfs-pvc
      imagePullSecrets:
      - name: ghcr-secret
```

### Frontend Environment for AI Application (ai-frontend)

Create tasks and templates for the frontend application deployment.

#### Tasks

```yaml
# roles/ai-frontend/tasks/main.yml
- name: Deploy AI Frontend Application
  kubernetes.core.k8s:
    definition: "{{ lookup('template', 'deployment.yml.j2') }}"
```

#### Templates

##### `deployment.yml.j2`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-frontend
  namespace: {{ namespace }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ai-frontend
  template:
    metadata:
      labels:
        app: ai-frontend
    spec:
      containers:
      - name: ai-frontend
        image: ghcr.io/Cdaprod/ai-frontend:latest
        ports:
        - containerPort: 80
      imagePullSecrets:
      - name: ghcr-secret

---
apiVersion: v1
kind: Service
metadata:
  name: ai-frontend
  namespace: {{ namespace }}
spec:
  selector:
    app: ai-frontend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: LoadBalancer
```

### Playbook File

Create the `playbook.yml` to execute all roles.

```yaml
# playbook.yml
- hosts: all
  become: yes
  roles:
    - role: common
    - role: ai-s3
    - role: ai-vdb
    - role: ai-py
    - role: ai-frontend
```

### Running the Playbook

1. **Navigate to the Ansible directory**:
   ```sh
   cd ansible
   ```

2. **Run the Ansible playbook**:
   ```sh
   ansible-playbook -i inventory/hosts playbook.yml
   ```

This completes the instructions for deploying the AI application using Ansible. Each component, including MinIO (`ai-s3`), Weaviate (`ai-vdb`), the Python AI application (`ai-py`), and the frontend (`ai-frontend`), is organized into roles and templates for a scalable and programmatic deployment. If you need further customization or adjustments, you can modify the respective templates and task files accordingly.

---

### Group Variables

Create the `group_vars/all.yml` file to store common variables.

```yaml
# group_vars/all.yml
minio_access_key: "{{ lookup('env', 'MINIO_ACCESS_KEY') }}"
minio_secret_key: "{{ lookup('env', 'MINIO_SECRET_KEY') }}"
nfs_server: "cda_ds"
nfs_path: "/volume1/cda-storage/minio/data"
namespace: "cdaprod-dev"
```

### Inventory File

Define the hosts in `inventory/hosts`.

```ini
# inventory/hosts
[all]
localhost ansible_connection=local
```

### Common Tasks

Create the `roles/common/tasks/main.yml` file to handle common tasks.

```yaml
# roles/common/tasks/main.yml
- name: Create namespace
  kubernetes.core.k8s:
    api_version: v1
    kind: Namespace
    name: "{{ namespace }}"
```

### MinIO (ai-s3) Role

Create tasks and templates for the MinIO deployment.

#### Tasks

```yaml
# roles/ai-s3/tasks/main.yml
- name: Create MinIO secret
  kubernetes.core.k8s:
    definition: "{{ lookup('template', 'secret.yml.j2') }}"
    
- name: Create NFS persistent volume
  kubernetes.core.k8s:
    definition: "{{ lookup('template', 'pv.yml.j2') }}"

- name: Create NFS persistent volume claim
  kubernetes.core.k8s:
    definition: "{{ lookup('template', 'pvc.yml.j2') }}"

- name: Deploy MinIO
  kubernetes.core.k8s:
    definition: "{{ lookup('template', 'deployment.yml.j2') }}"
```

#### Templates

##### `secret.yml.j2`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: minio-credentials
  namespace: {{ namespace }}
type: Opaque
stringData:
  accesskey: "{{ minio_access_key }}"
  secretkey: "{{ minio_secret_key }}"
```

##### `pv.yml.j2`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  nfs:
    path: {{ nfs_path }}
    server: {{ nfs_server }}
```

##### `pvc.yml.j2`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
  namespace: {{ namespace }}
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
```

##### `deployment.yml.j2`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-s3
  namespace: {{ namespace }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ai-s3
  template:
    metadata:
      labels:
        app: ai-s3
    spec:
      containers:
      - name: minio
        image: ghcr.io/Cdaprod/minio:latest
        args:
        - server
        - /data
        env:
        - name: MINIO_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: minio-credentials
              key: accesskey
        - name: MINIO_SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: minio-credentials
              key: secretkey
        volumeMounts:
        - name: nfs-storage
          mountPath: /data
      volumes:
      - name: nfs-storage
        persistentVolumeClaim:
          claimName: nfs-pvc
```

### Weaviate (ai-vdb) Role

Create tasks and templates for the Weaviate deployment.

#### Tasks

```yaml
# roles/ai-vdb/tasks/main.yml
- name: Deploy Weaviate
  kubernetes.core.k8s:
    definition: "{{ lookup('template', 'deployment.yml.j2') }}"
```

#### Templates

##### `deployment.yml.j2`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-vdb
  namespace: {{ namespace }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ai-vdb
  template:
    metadata:
      labels:
        app: ai-vdb
    spec:
      containers:
      - name: weaviate
        image: ghcr.io/Cdaprod/weaviate-app:latest
        env:
        - name: STORAGE_BACKEND
          value: "s3"
        - name: S3_ENDPOINT
          value: "ai-s3:9000"
        - name: S3_BUCKET
          value: "weaviate-data"
        - name: S3_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: minio-credentials
              key: accesskey
        - name: S3_SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: minio-credentials
              key: secretkey
      imagePullSecrets:
      - name: ghcr-secret
```

### Python Environment for AI Application (ai-py)

Create tasks and templates for the AI application deployment.

#### Tasks

```yaml
# roles/ai-py/tasks/main.yml
- name: Deploy AI Python Application
  kubernetes.core.k8s:
    definition: "{{ lookup('template', 'deployment.yml.j2') }}"
```

#### Templates

##### `deployment.yml.j2`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-py
  namespace: {{ namespace }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ai-py
  template:
    metadata:
      labels:
        app: ai-py
    spec:
      containers:
      - name: ai-py
        image: ghcr.io/Cdaprod/ai-py:latest
        env:
        - name: MINIO_ENDPOINT
          value: "ai-s3:9000"
        - name: MINIO_BUCKET
          value: "weaviate-data"
        - name: MINIO_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: minio-credentials
              key: accesskey
        - name: MINIO_SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: minio-credentials
              key: secretkey
        volumeMounts:
        - name: nfs-storage
          mountPath: /app/data
      volumes:
      - name: nfs-storage
        persistentVolumeClaim:
          claimName: nfs-pvc
      imagePullSecrets:
      - name: ghcr-secret
```

### Frontend Environment for AI Application (ai-frontend)

Create tasks and templates for the frontend application deployment.

#### Tasks

```yaml
# roles/ai-frontend/tasks/main.yml
- name: Deploy AI Frontend Application
  kubernetes.core.k8s:
    definition: "{{ lookup('template', 'deployment.yml.j2') }}"
```

#### Templates

##### `deployment.yml.j2`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-frontend
  namespace: {{ namespace }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ai-frontend
  template:
    metadata:
      labels:
        app: ai-frontend
    spec:
      containers:
      - name: ai-frontend
        image: ghcr.io/Cdaprod/ai-frontend:latest
        ports:
        - containerPort: 80
      imagePullSecrets:
      - name: ghcr-secret

---
apiVersion: v1
kind: Service
metadata:
  name: ai-frontend
  namespace: {{ namespace }}
spec:
  selector:
    app: ai-frontend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: LoadBalancer
```

### Playbook File

Create the `playbook.yml` to execute all roles.

```yaml
# playbook.yml
- hosts: all
  become: yes
  roles:
    - role: common
    - role: ai-s3
    - role: ai-vdb
    - role: ai-py
    - role: ai-frontend
```

### Running the Playbook

1. **Navigate to the Ansible directory**:
   ```sh
   cd ansible
   ```

2. **Run the Ansible playbook**:
   ```sh
   ansible-playbook -i inventory/hosts playbook.yml
   ```

This completes the instructions for deploying the AI application using Ansible. Each component, including MinIO (`ai-s3`), Weaviate (`ai-vdb`), the Python AI application (`ai-py`), and the frontend (`ai-frontend`), is organized into roles and templates for a scalable and programmatic deployment. If you need further customization or adjustments, you can modify the respective templates and task files accordingly.