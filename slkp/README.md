# slkp setup

Prepare requirements

```
slkp prepare
```

Install haproxy

```
slkp run haproxy
```

Install rke2 cluster

```
slkp run rke2
```

Verify cluster installation by sikalabs/hello-world chart

```
export KUBECONFIG=kubeconfig.yml

helm upgrade --install \
hello-world \
--repo https://helm.sikalabs.io \
hello-world \
--set host=hello-world.slkp.sikademo.com \
--set TEXT="Hello SLKP" \
--wait
```
