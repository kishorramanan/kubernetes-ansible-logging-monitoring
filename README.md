# Deploy kubernetes via ansible (on cloudstack servers) with logging (efk) & monitoring (prometheus) support #

![k8s_Infra1.jpg](https://github.com/gregbkr/kubernetes-ansible-logging-monitoring/raw/master/media/k8s-infra1.JPG)

## Updates ;-)
An even better setup with kargo (deploy to aws, azure, baremetal, and hot k8s migration) is available here --> https://github.com/gregbkr/kubernetes-kargo-logging-monitoring

## What you will get:
- 1 master node running : k8s for container orchestration, it will pilot and gives work to the minions
- 2(or more) minion/slave/worker nodes : running the actual containers and doing the actual work
- Efk: we will send all k8s container logs to an elasticsearch DB, via fluentd, and visualize dashboards with kibana
- Prometheus will monitoring all this infra, with grafana dashbaord
- Heapster is an alternative for monitoring your k8s cluster
- K8s dashboard addon (not efk dashboard), where you can visualize k8s component in a GUI
- Service-loadbalancer (static haproxy): which is the public gateway to access your internal k8s services (kibana, grafana)
- Dynamic loadbalancer (traefik): an alternative to haproxy, quite powerful with its dynamic service discovery and auto certification

*Prerequisit:*
- Cloudstack cloud provider (ex: exoscale) / but you can deploy anywhere else with a bit of adaptation in ansible. Deploying logging and monitoring stay the same at the moment you have k8s running
- A vm (ubuntu) with ansible installed, where you will run recipes, and manage k8s with kubeclt

More info: you can find an overview of that setup on my blog: https://greg.satoshi.tech/

# 1. Deploy kubernetes

### 1.1 Clone repo

    git clone https://github.com/gregbkr/kubernetes-ansible-logging-monitoring.git k8s && cd k8s

### 1.2 Install kubectl

Kubeclt is your admin local client to pilot the k8s cluster.
Please use the same version as server. You will be able to talk and pilot k8s with this tool.

    curl -O https://storage.googleapis.com/kubernetes-release/release/v1.4.6/bin/linux/amd64/kubectl
    chmod +x kubectl
    mv kubectl /usr/local/bin/kubectl

Autocompletion

    source <(kubectl completion bash)
    kubeclt get nod +[TAB]
    
If issues, see troubleshooting section. 


### 1.3 Deploy k8s for a cloudstack infra

I will use the nice setup made by Seb: https://www.exoscale.ch/syslog/2016/05/09/kubernetes-ansible/
I just added few lines in file: ansible/roles/k8s/templates/k8s-node.j2 to be able to collect logs with fluentd
(# In order to have logs in /var/log/containers to be pickup by fluentd
    Environment="RKT_OPTS=--volume dns,kind=host,source=/etc/resolv.conf --mount volume=dns,target=/etc/resolv.conf --volume var-log,kind=host,source=/var/log --mount volume=var-log,target=/var/log" )

    nano ansible/k8s.yml      <-- edit k8s version, num_node, ssh_key if you want to use your own

Next step will create firewall rules k8s, master and minion nodes, and install k8s components
Run recipe:

	ansible-playbook ansible/k8s.yml
	watch kubectl get node    <-- wait for the nodes to be up

### 1.4 Checks:
	
	kubectl get all --all-namespaces    <-- should have no error here
	
# 2. Deploy logging (efk) to collect k8s & containers events
	
### 2.1 Deploy elasticsearch, fluentd, kibana

    kubectl apply -f logging    <-- all deployment declarations and configurations are here

    kubectl get all --all-namespaces      <-- if you see elasticsearch container restarting, please restart all nodes one time only (setting vm.max_map_count, see troubleshooting section)

### 2.2 Access services

From here, you should be able to access our services from your laptop, as long as your cloud server ip are public:

- kibana: http://any_minion_node_ip:30601
- ES: http://any_minion_node_ip:30200

To enable that access, we had set Type=NodePort and nodePort:35601/39200 in kibana/elasticsearch-service.yaml, to make it easier to learn at this point.
Because we want to control how and from where we should be accessing our public services, we will set in a later section a loadbalancer.

### 2.3 See logs in kibana

Check logs coming in kibana, you just need to refresh, select Time-field name : @timestamps + create

Load and view your first dashboard: management > Saved Object > Import > logging/dashboards/elk-v1.json

![k8s-kibana.jpg](https://github.com/gregbkr/kubernetes-ansible-logging-monitoring/raw/master/media/k8s-kibana.JPG)


# 3. Monitoring services and containers

It seems like two schools are gently fighting for container monitoring:

- Heapster: this new player now comes as a kind of k8s addon (you can deploy it via a simple switch in some setup). It seems to be better integrated at the moment, and even more in the future with k8s component depending on it, but still young and few features
- Prometheus: it has been around for some times, lots of nice features (alerting, application metrics) and community resources available (see the public dashboards for example)

More on which one to choose: https://github.com/kubernetes/heapster/issues/645

### 3.1 Monitoring with prometheus

Create monitoring containers

    kubectl apply -f monitoring
    kubectl get all --namespace=monitoring

**Prometheus**

Access the gui: http://any_minion_node_ip:30090

Go to status > target : you should see only some green. 
*We got one false positive error scaping k8s-node with 2 ports 9102 and 80. As long as 9102 is good, we got the data.
If you got some "context deadline exceeded" or "getsockopt connection refused", you will have to open firewall rule between the nodes. For exemple in security group k8s, you need to open 9100 and 10255.*

Try a query: "node_memory_Active" > Execute > Graph --> you should see 2 lines representing both nodes.

![prometheus.jpg](https://github.com/gregbkr/kubernetes-ansible-logging-monitoring/raw/master/media/prometheus.JPG)



**Grafana**

Login to the interface with login:admin | pass:admin) :   http://any_minion_node_ip:30000
Load some dashboards: dashboard > home

**Kubernetes pod resources**
![grafana-k8s-pod-resources1.jpg](https://github.com/gregbkr/kubernetes-ansible-logging-monitoring/raw/master/media/grafana-k8s-pod-resources1.JPG)
![grafana-k8s-pod-resources2.jpg](https://github.com/gregbkr/kubernetes-ansible-logging-monitoring/raw/master/media/grafana-k8s-pod-resources2.JPG)


**Prometheus stats**
![grafana-prometheus-stats.jpg](https://github.com/gregbkr/kubernetes-ansible-logging-monitoring/raw/master/media/grafana-prometheus-stats.JPG)

**Load other public dashboards**

Grafana GUI > Dashboards > Import

Already loaded:
- prometheus stats: https://grafana.net/dashboards/2
- kubernetes cluster : https://grafana.net/dashboards/162

Other good dashboards :

- node exporter: https://grafana.net/dashboards/704 - https://grafana.net/dashboards/22

- deployment: pod metrics: https://grafana.net/dashboards/747 - pod resources: https://grafana.net/dashboards/737

### 3.2 Monitoring2 with heapster

    kubectl apply -f monitoring2
    kubectl get all --namespace=monitoring2

**Access services**

- Grafana2: http://any_minion_node_ip:30002

You can load Cluster or Pods dashboards. When viewing Pods, type manually "namespace=monitoring2" to view stats for the related containers.

![grafana2-pods.jpg](https://github.com/gregbkr/kubernetes-ansible-logging-monitoring/raw/master/media/grafana2-pods.JPG)
![grafana2-cluster1.jpg](https://github.com/gregbkr/kubernetes-ansible-logging-monitoring/raw/master/media/grafana2-cluster1.JPG)
![grafana2-cluster1.jpg](https://github.com/gregbkr/kubernetes-ansible-logging-monitoring/raw/master/media/grafana2-cluster2.JPG)
![grafana2-cluster1.jpg](https://github.com/gregbkr/kubernetes-ansible-logging-monitoring/raw/master/media/grafana2-cluster3.JPG)

# 4. Kubenetes dashboard addon (not logging efk)

Dashboard addon let you see k8s services and containers via a nice GUI.

    kubectl apply -f kubernetes-dashboard
    kubectl get all --namespace=kube-system     <-- carefull dashboard is running in namespace=kube-system
    
Access GUI: http://any_minion_node_ip:30999 

![kubernetes-dashboard.jpg](https://github.com/gregbkr/kubernetes-ansible-logging-monitoring/raw/master/media/kubernetes-dashboard-addon.JPG)

# 5. LoadBalancers

If you are on aws or google cloud, these provider we automatically set a loadbalancer matching the *-ingress.yaml configuration. For all other cloud provider and baremetal, you will have to take care of that step. Luckyly, I will present you two types of loadlancer below ;-)
- service-loadbalancer (static haproxy) https://github.com/kubernetes/contrib/tree/master/service-loadbalancer
- traefik (dynamic proxy) https://github.com/containous/traefik

### 5.1 Service-loadbalancer

Create the load-balancer to be able to connect your service from the internet.
Give 1 or more nodes the loadbalancer role:

    kubectl label node 185.19.30.121 role=loadbalancer
    kubectl apply -f service-loadbalancer-daemonset.yaml

*If you change the config, use "kubectl delete -f service-loadbalancer.yaml" to force a delete/create, then the discovery of the newly created service.
Add/remove services? please edit service-loadbalancer.yaml*

**Access services**

- kibana (logging): http://lb_node_ip:5601
- grafana (monitoring): http://lb_node_ip:3000   (admin/admin)
- prometheus (monitoring): http://lb_node_ip:3000
- grafana2 (monitoring2): http://lb_node_ip:3002
- kubernetes-dashboard: http://lb_node_ip:9999

### 5.2 Traefik

Any news services, exposed by *-ingress.yaml, will be caught by traefik and made available without restart.

To experience the full power of traefik, please purchase a domain name (ex: satoshi.tech), and point that record to the node you choose to be the lb. This record will help create the automatic certificate via the acme standard.

- satoshi.tech --> lb_node_ip

Then for each services you will use, create a dns A record:

- kibana.satoshi.tech --> lb_node_ip
- grafana.satoshi.tech --> lb_node_ip
- prometheus.satoshi.tech --> lb_node_ip
- grafana2.satoshi.tech --> lb_node_ip
- kubernetes-dashboard.satoshi.tech --> lb_node_ip
- traefik-console.satoshi.tech --> lb_node_ip

Based on which name you use to access the lb_node, traefik will forward to the right k8s service.

Now you need to edit the configuration:

    nano traefik/traefik-daemonset.yaml
        [acme]   <-- set your data for auto certification

Create the dynamic proxy to be able to connect your service from the internet.

    kubectl apply -f traefik    <-- if error, probably because you didn't deploy other namespaces, so can ignore
    kubectl get all --all-namespaces  <-- if traefik pod can't get created, probably issue with port 443 on loadbalancer --> see troubleshooting section

**Access services**
If set in traefik, please use login/pass: test/test
You can use http or https

- kibana (logging): http://kibana.satoshi.tech
- grafana (monitoring): http:grafana.satoshi.tech   (admin/admin)
- prometheus (monitoring): http://prometheus.satoshi.tech
- grafana2 (monitoring2): http://grafana2.satoshi.tech
- kubernetes-dashboard: http://kubernetes-dashboard.satoshi.tech
- traefik-console: http://traefik-console.satoshi.tech or http://any_minion_node_ip:30080

![traefik.jpg](https://github.com/gregbkr/kubernetes-ansible-logging-monitoring/raw/master/media/traefik.JPG)

### 5.3 Security considerations

These lb nodes are some kind of DMZ servers where you could balance later your DNS queries.
For production environment, I would recommend that only DMZ services (service-loadbalancer, traefik, nginx, ...) could run in here, because these servers will apply some less restrictive firewall rules (ex: open 80, 433, 5601, 3000) than other minion k8s nodes. 
So I would create a second security group (sg): k8s-dmz with same rules as k8s, and rules between both zone, so k8s services can talk to  k8s and k8s-dmz. Then open 80, 433, 5601, 3000 for k8s-dmz only. Like this, k8s sg still protect more sensitive containers from direct public access/scans.

The same applies for the master node. I would create a new sg for it: k8s-master, so only this group will permit access from kubeclt (port 80, 443).

Then you should remove all NodePort from the services configuration, so no service will be available when scanning a classic minion. For that please comment the section "# type: NodePort" for all *-service.yaml

### 5.4 Scaling loadbalancers

Add more loadbalancers, by adding more loadbalancers nodes. Because we use Daemonset type of job, all new nodes tagged with loadbalancer will spawn a loadbalancer container.

Use ansible to add a node

    nano ansible/k8s.yml     <-- edit:  k8s_num_nodes: 3
    ansible-playbook ansible/k8s.yml
    kubeclt get node         <-- wait for it!
	
Label it as a loadbalancer node

    kubectl label node your_new_lb_minion_node role=loadbalancer

Then just check the new containers getting created 
    kubectl get all --all-namespaces

For service-loadbalancer, try to access new_lb_minion_ip:5601

For trafik, add a dns A-record kibana.satoshi.tech --> new_lb_minion_ip so we will balance dns resolution to the old and new lb_node.
Test some ping, and access kibana.satoshi.tech few times...	


# 6. Secure your k8s access with certificates (optional demonstration)

kubectl pilot k8s via the api server already on a secured port 443 in https.
We will now create a certicate autority, to issue a certificate for the api, and for your admin client, to get even higher level of authentification.

The lines below are based on the work of Kelsey and the cloudflare easy CA.
https://github.com/kelseyhightower/docker-kubernetes-tls-guide
https://github.com/cloudflare/cfssl

Install your own PKI infra tool: cfssl 
You need GO 1.6+ and the GOPATH set

    apt install golang-go
    export GOPATH=$HOME/work
    export PATH=$PATH:/usr/local/go/bin
    export PATH=$PATH:$GOPATH/bin

    go get -u github.com/cloudflare/cfssl/cmd/cfssl
    go get -u github.com/cloudflare/cfssl/cmd/...

    cd tls
    mkdir kubectl master minion

Declare your master Ip (or domain) to the server cert

    nano kube-apiserver-server-csr.json     <-- add your master_ip in hosts section

**Initialize a CA**

    cfssl gencert -initca ca-csr.json | cfssljson -bare ca/ca

**Create an api server cert**

```
cfssl gencert \
-ca=ca/ca.pem \
-ca-key=ca/ca-key.pem \
-config=ca-config.json \
-profile=server \
kube-apiserver-server-csr.json | cfssljson -bare master/kube-apiserver-server
```

    cp ca/ca.pem master

**Create kubeclt client cert**

```
cfssl gencert \
-ca=ca/ca.pem \
-ca-key=ca/ca-key.pem \
-config=ca-config.json \
-profile=client \
kubernetes-admin-user.csr.json | cfssljson -bare kubectl/kubernetes-admin-user
```
     
    kubectl config set-cluster secure --server=https://185.19.30.189:443 --certificate-authority=master/ca.pem --embed-certs=true

```
kubectl config set-credentials admin \
--client-key=kubectl/kubernetes-admin-user-key.pem \
--client-certificate=kubectl/kubernetes-admin-user.pem \
--embed-certs=true
```

    kubectl config set-context secure --cluster=secure --user=admin
    kubectl config use-context secure

Copy tls/master folder to node master

    scp -r -i ~/.ssh/id_rsa_foobar master core@185.19.30.189:/home/core

Edit master

    ssh -i ~/.ssh/id_rsa_foobar core@185.19.30.189
     
    mkdir /etc/kubernetes
    mv master/* /etc/kubernetes/.

```
sudo vim /etc/systemd/system/kube-apiserver.service

--client-ca-file=/etc/kubernetes/ca.pem \
--tls-cert-file=/etc/kubernetes/kube-api-server-server.pem \
--tls-private-key-file=/etc/kubernetes/kube-apiserver-server-key.pem \
```

    sudo systemctl daemon-reload
    sudo systemctl restart kube-apiserver.service
	
*Note: this systemclt config will be erased at the next reboot. If you want to use cert, edit ansible master node template with the right config, then deploy a new master. Then generate certs...*

Test

    curl --cert kubectl/kubernetes-admin-user.pem --key kubectl/kubernetes-admin-user-key.pem --cacert master/ca.pem https://185.19.30.189/api -v
    kubectl get node


**Create kubelet client cert**

```
cfssl gencert \
-ca=ca/ca.pem \
-ca-key=ca/ca-key.pem \
-config=ca-config.json \
-profile=client \
kubelet-client-csr.json | cfssljson -bare minion/kubelet/kubelet-client
```

Edit minion node

Copy and mv the file:

    scp -r -i ~/.ssh/id_rsa_foobar minion core@185.19.30.189:/home/core
    ssh -i ~/.ssh/id_rsa_foobar core@185.19.3.31
     
    mkdir /etc/kubernetes
    mv minion/* /etc/kubernetes/.

```
sudo vim /etc/systemd/system/kube-kubelet.service	  
--api-servers=https://185.19.30.189:443 \
--kubeconfig=/etc/kubernetes/kubelet/kubelet.kubeconfig \
```

    sudo systemctl daemon-reload
    sudo systemctl restart kube-apiserver.service
    
Check logs:

    journalctl --since "10 minutes ago" -u kube-kubelet --no-pager
    
Try 

    kubectl get node

*All services (kube-proxy, kube-client, kube-controller) can be set to use certificate. But this is a subject for another setup.*

# 7. Troubleshooting

### Kubectl autocompletion not working

Probably your custom bash profile is interfering with the sourcing. 
- Use kubectl completion bash > /etc/bash_completion.d/kubectl
- And make sure that lines are uncomment in 
```
nano  ~/.bashrc
if [ -f /etc/bash_completion ] && ! shopt -oq posix; then
    . /etc/bash_completion
fi
```
Then logon and try again.

### If problem starting elasticsearch v5: (fix present in roles/k8s/templates/k8s-node.j2)
- manually on all node: fix an issue with hungry es v5
```
ssh -i ~/.ssh/id_rsa_foobar core@185.19.29.212
sudo sysctl -w vm.max_map_count=262144
```

- make it persistent:
```
sudo vi /etc/sysctl.d/elasticsearch.conf
vm.max_map_count=262144
sudo sysctl --system
```

### If issue connecting to svc (for example elasticsearch), use ubuntu container: 
- First see if ubuntu will be in the same namespace as the service you want to check:

```
nano utils/ubuntu.yaml
kubectl apply -f utils/ubuntu.yaml
```

- Depending in which namespace ubuntu runs, you can check services with one of these commands:

```
kubectl exec ubuntu -- curl elasticseach:9200   <-- should returns ... "cluster_name" : "elasticsearch"...
kubectl exec ubuntu -- curl kibana:5601         <-- should returns ... var defaultRoute = '/app/kibana'...
    
kubectl exec ubuntu -- curl elasticsearch.logging.svc.cluster.local:9200         <-- ubuntu in default namespace
kubectl exec ubuntu --namespace=logging -- nslookup elasticsearch               <-- ubuntu in logging namespace
kubectl exec ubuntu --namespace=logging -- nslookup kubernetes.default.svc.cluster.local     <-- ubuntu in logging namespace
```

- Check port 9200 on the node running elasticsearch container: ssh -i ~/.ssh/id_rsa_foobar core@185.19.29.212 netstat -plunt
- Uncomment type: NodePort and nodePort: 39200 if you want to access elasticsearch from any node_ip
- Check data in elasticsearch
    kubectl exec ubuntu -- curl es:9200/_search?q=*
    curl node_ip:39200/_search?q=*       <-- if type: NodePort set in es.yaml

### No log coming in kibana:
- check that there are file in node: ssh -i ~/.ssh/id_rsa_foobar core@185.19.29.212 ls /var/log/containers

Can't connect to k8s-dashboard addon:
- Carefull, addon are in kube-system namespace!
If stuck use type: NodePort and
- Find the node public port: kubectl describe service kubernetes-dashboard --namespace=kube-system
- Access it from nodes : http://185.19.30.220:31224/

### DNS resolution not working? Svc kube-dns.kube-system should take care of the resolution

    kubectl exec ubuntu -- nslookup google.com
    kubectl exec ubuntu -- nslookup kubernetes
    kubectl exec ubuntu -- nslookup kubernetes.default
    kubectl exec ubuntu -- nslookup kubernetes-dashboard.kube-system

### Pod can't get created? See more logs:

    kubectl describe po/elastcisearch
    kubectl logs -f elasticsearch-ret5zg

### Prometheus can't scrape node_exporter
Possibly firewall issues!
You need to open firewall internal rules between all nodes port 9100 (endpoint) and 10255 (node)

### Check influxdb

    kubectl exec ubuntu --namespace=monitoring2 -- curl -sl -I influxdb:8086/ping

### Traefik pod cant get created: port 443 already used

Not very sure yet what the issue on lb_node, but try to start traefik on another minion:

    kubectl label node 185.19.29.192 role=minion    <-- give another node the minion role
    nano traefik/traefik-deployment.yaml   <-- and edit section    spec > nodeSelector: > role: minion

Then delete and recreate traefik, should be all good.

### Check traefik protected access

    apt install httpie
    http --verify=no --auth test:test https://kibana.satoshi.tech -v

# 8. Annexes

### Shell Alias for K8s
```
alias k='kubectl'
alias kk='kubectl get all'
alias wk='watch kubectl get all'
alias ka='kubectl get all --all-namespaces'
alias kc='kubectl create -f'
alias kdel='kubectl delete -f'
alias kcdel='kubectl delete configmap'
alias kd='kubectl describe'
alias kl='kubectl logs'

```

### Need another slave node?
Edit ansible-cloudstack/k8s.yml and run again the deploy

### Want to start from scrash? 

Delete the corresponding namespace, all related containers/services will be destroyed.

    kubectl delete namespace monitoring
    kubectl delete namespace logging

# 9. Future work

- Use different firewalls security group: k8s, k8s-dmz, k8s-master, to be ready for production
- Use persistent data for Elasticsearch and prometheus
- Fix prometheus k8s_pod scraping both port 80 and 9102...
