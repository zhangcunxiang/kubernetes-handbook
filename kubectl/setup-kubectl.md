# Setup Kubectl

## Export environment variables
```shell
source /usr/k8s/bin/env.sh
export KUBE_APISERVER="https://${MASTER1_IP}:6443"
```
## Download and install Kubectl
```shell
curl -O -L https://dl.k8s.io/v1.9.3/kubernetes-client-linux-amd64.tar.gz 
tar -xzvf kubernetes-client-linux-amd64.tar.gz
sudo cp kubernetes/client/bin/kube* /usr/k8s/bin/
sudo chmod a+x /usr/k8s/bin/kube*
sudo chown $USER:USER /usr/k8s/bin/kube*
export PATH=/usr/k8s/bin:$PATH
```

## Create admin certificates

Because we could run kubectl on any machine (no necessarily on master nodes), hence, we need to encrypt the network between kubectl and kube-api-server. 


```shell
# Create admin-csr.json
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF

# Generate certificates
cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/ca-config.json \
  -profile=kubernetes admin-csr.json | cfssljson -bare admin

sudo mv admin*.pem /etc/kubernetes/ssl/
```

## Use kubectl to generate kubeconfig file
```shell
# Configure the certificates and the cluster
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER}

# Configure client side certificates
kubectl config set-credentials admin \
  --client-certificate=/etc/kubernetes/ssl/admin.pem \
  --embed-certs=true \
  --client-key=/etc/kubernetes/ssl/admin-key.pem \
  --token=${BOOTSTRAP_TOKEN}

# Create a new context
kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=admin

# Set the context as default context
kubectl config use-context kubernetes
```
kubeconfig is saved in `~/.kube/config`


## Check Cluster Status
Once the Kubectl is configured, we should be able to check the cluster's status:
```shell
kubectl get componentstatuses
```

> **Note:** 
  At the moment, `Kubectl` only points to a specific API server `https://MASTER1_IP:6443` to interact with the cluster. Once we have a high available cluster setup later, we will come back and make adjustments to the configuration.