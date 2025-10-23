# rke2 setup

Load balancer:

- rke2.sikademo.com

Masters:

- rke2-ma-0.sikademo.com
- rke2-ma-1.sikademo.com
- rke2-ma-2.sikademo.com

Workers:

- rke2-wo-0.sikademo.com
- rke2-wo-1.sikademo.com
- rke2-wo-2.sikademo.com

Install `rke2` on first master

```
apt-get install -y open-iscsi
curl -sfL https://get.rke2.io | sh -
mkdir -p /etc/rancher/rke2/
cat << EOF > /etc/rancher/rke2/config.yaml
token: supersecuretoken
tls-san:
  - rke2-ma-0.sikademo.com
  - rke2-ma-1.sikademo.com
  - rke2-ma-2.sikademo.com
node-taint:
    - "CriticalAddonsOnly=true:NoExecute"
EOF
systemctl enable rke2-server.service --now
```

Install `kubectl`

```
slu install-bin kubectl
```

Create `~/.kube/config` symlink

```
mkdir -p ~/.kube
sudo ln -s /etc/rancher/rke2/rke2.yaml ~/.kube/config
```

Install `helm`

```
slu install-bin helm
```

Add other masters

```
apt-get install -y open-iscsi
curl -sfL https://get.rke2.io | sh -
mkdir -p /etc/rancher/rke2/
cat << EOF > /etc/rancher/rke2/config.yaml
server: https://rke2-ma-0.sikademo.com:9345
token: supersecuretoken
tls-san:
  - rke2-ma-0.sikademo.com
  - rke2-ma-1.sikademo.com
  - rke2-ma-2.sikademo.com
node-taint:
    - "CriticalAddonsOnly=true:NoExecute"
EOF
systemctl enable rke2-server.service --now
```

Add workers

```
apt-get install -y open-iscsi
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -
mkdir -p /etc/rancher/rke2/
cat << EOF > /etc/rancher/rke2/config.yaml
server: https://rke2-ma-0.sikademo.com:9345
token: supersecuretoken
EOF
systemctl enable rke2-agent.service --now
```

Install cert-manager via helm

```
helm upgrade --install \
cert-manager cert-manager \
--repo https://charts.jetstack.io \
--create-namespace \
--namespace cert-manager \
--set crds.enabled=true \
--wait
```

Create ClusterIssuer for cert-manager

```
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    email: lets-encrypt-slu@sikamail.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-issuer-account-key
    solvers:
      - http01:
          ingress:
            class: nginx
EOF
```

Verify installation by sikalabs/hello-world chart

```
helm upgrade --install \
hello-world \
--repo https://helm.sikalabs.io \
hello-world \
--set host=hello-world.rke2-wo-0.sikademo.com \
--set TEXT="Hello rke2" \
--wait
```

Install load balancer

```
apt-get install -y haproxy
cat << EOF > /etc/haproxy/haproxy.cfg
defaults
  mode tcp
  timeout client 10s
  timeout connect 5s
  timeout server 10s
  timeout http-request 10s

frontend http
  bind 0.0.0.0:80
  default_backend http

frontend https
  bind 0.0.0.0:443
  default_backend https

backend http
  server backend-wo-0 rke2-wo-0.sikademo.com:80 check port 80
  server backend-wo-1 rke2-wo-1.sikademo.com:80 check port 80
  server backend-wo-2 rke2-wo-2.sikademo.com:80 check port 80

backend https
  server backend-wo-0 rke2-wo-0.sikademo.com:443 check port 443
  server backend-wo-1 rke2-wo-1.sikademo.com:443 check port 443
  server backend-wo-2 rke2-wo-2.sikademo.com:443 check port 443
EOF
systemctl enable haproxy
systemctl restart haproxy
```

Verify load balancer by sikalabs/hello-world chart

```
helm upgrade --install \
hello-world \
--repo https://helm.sikalabs.io \
hello-world \
--set host=hello-world.rke2.sikademo.com \
--set TEXT="Hello rke2 (lb)" \
--wait
```
