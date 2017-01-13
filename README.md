Calico demo
-----------

On target node (the last node in `kubectl get nodes` output) install BCC tools:

```bash
echo "deb [trusted=yes] https://repo.iovisor.org/apt/xenial xenial-nightly main" | sudo tee /etc/apt/sources.list.d/iovisor.list
sudo apt-get update
sudo apt-get install -y bcc-tools
```

Run exec snoop

```bash
cd /usr/share/bcc/tools/
./execsnoop | tee /tmp/exec.log
```

On k8s-master node run

```bash
node=$(kubectl get nodes | tail -n1 | awk '{print $1}')
kubectl label node $node test=nginx1
kubectl get nodes --show-labels | grep --color nginx1
kubectl create -f k8s/nginx_calico.yaml
kubectl get pods -o wide
```
