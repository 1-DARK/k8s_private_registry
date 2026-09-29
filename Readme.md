#  Kubernetes Private Container Registry

A hands-on implementation of a **private Docker container registry deployed on Kubernetes** with TLS security, authentication, persistent storage, and private image management.

This project demonstrates how to build a self-hosted container registry, push Docker images to it, and configure Kubernetes to securely pull and deploy private images.

---

##  Technologies Used

| Technology                        | Purpose                                                                     |
| --------------------------------- | --------------------------------------------------------------------------- |
| **Kubernetes**                    | Container orchestration, Deployments, Services, Secrets, and Pod management |
| **Docker**                        | Building, tagging, authenticating, pushing, and managing container images   |
| **containerd**                    | Container runtime used by Kubernetes nodes                                  |
| **Docker Registry**               | Self-hosted private container image repository                              |
| **TLS / SSL**                     | Secures communication between nodes and the private registry                |
| **OpenSSL**                       | Generates self-signed TLS certificates                                      |
| **htpasswd**                      | Provides username/password authentication for the registry                  |
| **Kubernetes Secrets**            | Stores TLS credentials, authentication data, and registry credentials       |
| **Persistent Volume (PV)**        | Provides persistent storage for registry data                               |
| **Persistent Volume Claim (PVC)** | Requests and connects persistent storage to the registry                    |
| **Linux / Ubuntu**                | Kubernetes node and registry environment                                    |
| **Bash**                          | Used for cluster configuration and automation commands                      |
| **Nginx**                         | Sample container image used to test the private registry                    |
| **kubectl**                       | CLI used to manage and troubleshoot Kubernetes resources                    |

---

##  Features

*  Private Docker Registry with **TLS encryption**
*  Username/password authentication using **htpasswd**
*  Kubernetes Secrets for secure credential management
*  Persistent storage using **PV/PVC**
*  Docker image push and pull workflow
*  Kubernetes deployment of private images
*  `imagePullSecrets` for authenticated image pulls
*  Registry DNS configuration across cluster nodes
*  Container image versioning using tags such as `v1` and `v2`
*  Kubernetes image version updates and rollouts
*  Image and application verification using Docker, `kubectl`, and `curl`

---

#  Architecture

```text
                         Kubernetes Cluster
                                
        ┌───────────────────────────────────────────┐
        │               Master Node                 │
        │                                           │
        │       Kubernetes API / kubectl            │
        │                  │                        │
        │                  ▼                        │
        │       Registry Deployment                 │
        │                  │                        │
        │                  ▼                        │
        │       Private Docker Registry             │
        │        my-registry:5000                   │
        │                  │                        │
        │                  ▼                        │
        │       Persistent Volume / Storage         │
        │                                           │
        └──────────────────┬────────────────────────┘
                           │
                    Push / Pull Images
                           │
        ┌──────────────────▼────────────────────────┐
        │                Worker Node                │
        │                                           │
        │              Docker CLI                   │
        │                  │                        │
        │             nginx:latest                  │
        │                  │                        │
        │             Docker Tag                    │
        │                  ▼                        │
        │    my-registry:5000/mynginx:v1            │
        │                  │                        │
        │                  ▼                        │
        │            Kubernetes Pod                 │
        │                Nginx                      │
        └───────────────────────────────────────────┘
```

---

#  Project Structure

```text
k8s_private_registry/
└── demo-reg/
    └── registry/
        ├── auth/
        │   └── htpasswd
        ├── certs/
        │   ├── tls.crt
        │   └── tls.key
        ├── volume.yaml
        ├── deploy.yaml
        └── service.yaml
```

### Configuration Files

* **`volume.yaml`** — Defines persistent storage for the registry.
* **`deploy.yaml`** — Deploys the private Docker registry.
* **`service.yaml`** — Exposes the registry inside the Kubernetes cluster.
* **`auth/htpasswd`** — Stores registry authentication credentials.
* **`certs/tls.crt`** — TLS certificate.
* **`certs/tls.key`** — TLS private key.

---

#  Implementation

## 1. Clone the Repository

```bash
git clone https://github.com/1-DARK/k8s_private_registry.git

cd k8s_private_registry
cd demo-reg
cd registry

alias k=kubectl
```

---

## 2. Generate TLS Certificate

Generate a self-signed certificate for the registry:

```bash
openssl req -x509 -newkey rsa:4096 -days 365 -nodes -sha256 \
-keyout certs/tls.key \
-out certs/tls.crt \
-subj "/CN=my-registry" \
-addext "subjectAltName = DNS:my-registry"
```

Verify:

```bash
ls certs
```

Expected:

```text
tls.crt
tls.key
```

---

## 3. Create Registry Authentication

Install Docker:

```bash
sudo apt update
sudo apt install -y docker.io
```

Generate the `htpasswd` authentication file:

```bash
sudo docker run --entrypoint htpasswd \
docker.io/library/httpd:2 \
-Bbn myuser mypasswd > auth/htpasswd
```

Verify:

```bash
cat auth/htpasswd
```

Example credentials:

```text
Username: myuser
Password: mypasswd
```

> For production environments, use stronger credentials and avoid storing plaintext passwords in shell history or public repositories.

---

#  4. Create Kubernetes Secrets

### TLS Secret

```bash
kubectl create secret tls certs-secret \
--cert=certs/tls.crt \
--key=certs/tls.key
```

### Authentication Secret

```bash
kubectl create secret generic auth-secret \
--from-file=auth/htpasswd
```

Check:

```bash
kubectl get secrets
```

---

#  5. Configure Persistent Storage

Create the registry storage directory:

```bash
mkdir -p /home/ubuntu/repos
```

The registry will persist image data under:

```text
/home/ubuntu/repos/docker/registry/v2/
```

---

#  6. Deploy the Private Registry

Apply the Persistent Volume configuration:

```bash
kubectl apply -f volume.yaml
```

Deploy the registry:

```bash
kubectl apply -f deploy.yaml
```

Create the registry Service:

```bash
kubectl apply -f service.yaml
```

Verify:

```bash
kubectl get pv
kubectl get pvc
kubectl get pods
kubectl get svc
```

---

#  7. Configure Registry DNS

Set the registry name:

```bash
export REGISTRY_NAME="my-registry"
```

Find the Service IP:

```bash
kubectl get svc
```

Add the registry name to `/etc/hosts` on **both Master and Worker nodes**:

```bash
sudo vi /etc/hosts
```

Add:

```text
<SERVICE-IP> my-registry
```

For example:

```text
10.96.100.50 my-registry
```

Test:

```bash
ping my-registry
```

---

#  8. Configure TLS Trust

On the Master Node, display the certificate:

```bash
cat certs/tls.crt
```

Copy the certificate to:

```text
/usr/local/share/ca-certificates/tls.crt
```

Then run:

```bash
sudo update-ca-certificates
```

Perform the same configuration on the Worker Node.

---

#  9. Configure Docker Registry Certificates

On the required nodes:

```bash
sudo mkdir -p /etc/docker/certs.d/my-registry:5000
```

Copy the certificate:

```bash
sudo cp /usr/local/share/ca-certificates/tls.crt \
/etc/docker/certs.d/my-registry:5000/tls.crt
```

Verify:

```bash
ls /etc/docker/certs.d/my-registry:5000/
```

Expected:

```text
tls.crt
```

---

#  10. Login to Private Registry

On the Worker Node:

```bash
docker login my-registry:5000 \
-u myuser \
-p mypasswd
```

Expected:

```text
Login Succeeded
```

---

#  11. Push an Image to the Registry

Pull Nginx:

```bash
sudo docker pull nginx
```

Tag the image:

```bash
sudo docker tag nginx:latest \
my-registry:5000/mynginx:v1
```

Push the image:

```bash
sudo docker push my-registry:5000/mynginx:v1
```

Verify:

```bash
sudo docker images
```

The image should now be available as:

```text
my-registry:5000/mynginx:v1
```

---

#  12. Verify Registry Storage

Check the registry storage:

```bash
cd /home/ubuntu/repos/docker/registry/v2/repositories/mynginx
ls -lrt
```

You can also inspect the registry Pod:

```bash
kubectl get pods
```

```bash
kubectl exec -it <registry-pod-name> -- sh
```

Inside the Pod:

```bash
cd /var/lib/registry/docker/registry/v2/repositories/mynginx
ls -lrt
```

Exit:

```bash
exit
```

---

#  13. Create Kubernetes Image Pull Secret

Create a registry authentication secret:

```bash
kubectl create secret docker-registry nginx-secret \
--docker-server=my-registry:5000 \
--docker-username=myuser \
--docker-password=mypasswd
```

Verify:

```bash
kubectl get secrets
```

This allows Kubernetes to authenticate with the private registry when pulling images.

---

#  14. Configure Container Runtime

Restart containerd on the cluster nodes:

```bash
sudo systemctl restart containerd
```

Verify:

```bash
sudo systemctl status containerd
```

---

#  15. Deploy Private Image to Kubernetes

Configure the Deployment to use the private registry:

```yaml
spec:
  containers:
    - name: nginx
      image: my-registry:5000/mynginx:v1

  imagePullSecrets:
    - name: nginx-secret
```

Apply:

```bash
kubectl apply -f new.yaml
```

Check:

```bash
kubectl get pods
```

---

#  16. Test the Deployment

Get the Pod IP:

```bash
kubectl get pods -o wide
```

Test from the Worker Node:

```bash
curl <POD-IP>
```

The Nginx response confirms that the private image was successfully pulled and deployed.

---

#  17. Image Versioning

Create a second version:

```bash
sudo docker tag nginx:latest \
my-registry:5000/mynginx:v2
```

Push:

```bash
sudo docker push my-registry:5000/mynginx:v2
```

Now the registry contains:

```text
my-registry:5000/mynginx:v1
my-registry:5000/mynginx:v2
```

---

#  18. Deploy New Image Version

Change:

```yaml
image: my-registry:5000/mynginx:v1
```

to:

```yaml
image: my-registry:5000/mynginx:v2
```

Apply:

```bash
kubectl apply -f new.yaml
```

Check:

```bash
kubectl get pods
```

Verify the image:

```bash
kubectl describe pod <pod-name>
```

Or:

```bash
kubectl get pod <pod-name> \
-o jsonpath='{.spec.containers[0].image}'
```

Expected:

```text
my-registry:5000/mynginx:v2
```

---

#  Key Kubernetes Concepts Demonstrated

```text
                    Private Registry
                           │
             ┌─────────────┴─────────────┐
             │                           │
          Security                  Persistence
             │                           │
       TLS + htpasswd                 PV/PVC
             │                           │
             └─────────────┬─────────────┘
                           │
                     Kubernetes
                           │
                    imagePullSecrets
                           │
                           ▼
                       Private Image
                           │
                           ▼
                       Nginx Pod
```

#  Project Outcome

By completing this project, you build a working workflow where:

```text
Docker Image
     │
     ▼
Docker Tag
     │
     ▼
Private Registry
     │
     │ Authentication + TLS
     ▼
Kubernetes
     │
     │ imagePullSecrets
     ▼
Private Image Pull
     │
     ▼
Nginx Pod
```

This project provides practical experience with **private container registries, Kubernetes storage, secrets, TLS security, container runtimes, and private image deployments**.
