# k3s setup


Install `k3s` cluster

```
curl -sfL https://get.k3s.io | sh -s - --disable traefik
```

Create `~/.kube/config` symlink

```
sudo ln -s /etc/rancher/k3s/k3s.yaml ~/.kube/config
```

Install `helm`

```
slu install-bin helm
```

Install ingress-nginx via helm

```
helm upgrade --install \
	ingress-nginx ingress-nginx \
	--repo https://kubernetes.github.io/ingress-nginx \
	--create-namespace \
	--namespace ingress-nginx \
	--set controller.service.type=ClusterIP \
	--set controller.ingressClassResource.default=true \
	--set controller.kind=DaemonSet \
	--set controller.hostPort.enabled=true \
	--set controller.metrics.enabled=true \
	--set controller.config.use-proxy-protocol=false \
	--wait
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
--set host=hello-world.k3s.sikademo.com \
--set TEXT="Hello k3s" \
--wait
```

Delete k3s cluster

```
k3s-uninstall.sh
```
