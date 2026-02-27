# CREATE KUBECONFIG

## Step 1: Create Namespace

#### Play the role of Admin (Cluster Owner)

```
kubectl create namespace go-redis-env
```

## Step 2: Key Generation and Certificate Application (CSR)

#### Play the role of Developer

##### Create Private Key

```
openssl genrsa -out dev-user.key 2048
```

##### Create file CSR 
```
openssl req -new -key dev-user.key -out dev-user.csr -subj "/CN=dev-user/O=developers"
```

#### Notes: May be copy file to server
```
scp -i ~/.ssh/ssh-key file.txt user@server_ip:/path
```

## Step 3: Admin approves and issues Certificate (Certificate API)

### 1. Get the Base64 encoded string of the CSR file:
```
cat dev-user.csr | base64 | tr -d "\n"
```

### 2. Create a dev-csr.yaml file with the following content:
```
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: dev-user-access
spec:
  request: <STRING-OF-STEP1>
  signerName: kubernetes.io/kube-apiserver-client
  usages:`
  - client auth
```

### 3. Submit request and Approval:

#### Submit to K8s:
```
kubectl apply -f dev-csr.yaml
```

#### Check status (Pending status will be seen)
```
kubectl get csr
```

#### Admin officially approved
```
kubectl certificate approve dev-user-access
```

#### Extract the signed certificate into a .crt file
```
kubectl get csr dev-user-access -o jsonpath='{.status.certificate}' | base64 --decode > dev-user.crt
```

## Step 4: Set up RBAC (Power Limit)

### 1. Create Role: Allow all operations (create, get, list, update, delete) on Pods, Services, Deployments
```
kubectl create role go-redis-manager \
  --verb=* \
  --resource=pods,services,deployments \
  -n go-redis-env
```

### 2. Create RoleBinding: Attach the newly created Role to the user "dev-user"
```
kubectl create rolebinding dev-user-binding \
  --role=go-redis-manager \
  --user=dev-user \
  -n go-redis-env
```

## Step 5: Package Kubeconfig file for Developer

### 1. Declare Cluster (Replace <MASTER_IP> with Public IP of Master node):
```
kubectl config set-cluster my-cluster \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=https://<MASTER_IP>:6443 \
  --kubeconfig=dev-kubeconfig
```

### 2. Declare User:
```
kubectl config set-credentials dev-user \
  --client-certificate=dev-user.crt \
  --client-key=dev-user.key \
  --embed-certs=true \
  --kubeconfig=dev-kubeconfig
```

### 3. Create a Context and set the default namespace go-redis-env:
```
kubectl config set-context dev-context \
  --cluster=my-cluster \
  --user=dev-user \
  --namespace=go-redis-env \
  --kubeconfig=dev-kubeconfig
```

### 4. Activate context:
```
kubectl config use-context dev-context --kubeconfig=dev-kubeconfig
```

### 5. Add Certificate and Private key to dev-kubeconfig file:

#### Make sure you are in the correct directory containing the `dev-user.crt` and `dev-user.key` file

```
kubectl config set-credentials dev-user \
  --client-certificate=dev-user.crt \
  --client-key=dev-user.key \
  --embed-certs=true \
  --kubeconfig=dev-kubeconfig
```

## Step 6: Testing

### Test 1: Can dev-user create a Pod in the project's namespace?
```
kubectl auth can-i create pods -n go-redis-env --as dev-user
```
#### Expectation: yes

### Test 2: Is dev-user allowed to delete Pods in the default namespace?
```
kubectl auth can-i delete pods -n default --as dev-user
```
#### Expectation: no

### Test 3: Can dev-users see the Node list?
```
kubectl auth can-i get nodes --as dev-user
```
#### Expectation: no


## There may be a TLS Certificate SAN Mismatch error

### 1. Backup old certificate:

```
cd /etc/kubernetes/pki/

sudo mv apiserver.crt apiserver.crt.bak
sudo mv apiserver.key apiserver.key.bak
```

### 2. Use Kubeadm to regenerate a new certificate:
```
sudo kubeadm init phase certs apiserver --apiserver-cert-extra-sans=<PUBLIC-IP>
```

#### Check
```
openssl x509 -in apiserver.crt -text -noout | grep <PUBLIC-IP>
```

### 3. Restart API Server:

#### Find ID of kube-apiserver container
```
sudo crictl ps | grep kube-apiserver
```

#### Remove Container
```
sudo crictl stop <CONTAINER_ID>
sudo crictl rm <CONTAINER_ID>
```

### 4. Test
```
kubectl --kubeconfig=dev-kubeconfig get pods -n go-redis-env
```

#### Expectation: No resources found in go-redis-env namespace.