# K8s-raspberrypi8bg-setup
Before start, you maight want to check this other link: [home lab setup](https://github.com/Harguer/home-lab-setup  )

One of the turorials I followed: [here](https://www.shogan.co.uk/kubernetes/building-a-raspberry-pi-kubernetes-cluster-part-2-master-node/)

# Master and node setup  

I have used this [OS image](https://downloads.raspberrypi.org/raspios_arm64/images/raspios_arm64-2020-05-28/2020-05-27-raspios-buster-arm64.zip )
``` 
2020-05-27-raspios-buster-arm64.img 
```

Since I'm using Linux on my laptop, I used followin dd to burn the image:   

```
dd bs=4M if=2020-05-27-raspios-buster-arm64.img of=/dev/mmcblk0 conv=fsync
```

Update OS
```
sudo apt-get update; sudo apt upgrade -y
```

install docker 
```
curl -sSL get.docker.com | sh && sudo usermod pi -aG docker && newgrp docker
```

disable swap (as per k8s requeriments), and enable cgroups for resource isolation
```
sudo dphys-swapfile swapoff
sudo dphys-swapfile uninstall
sudo update-rc.d dphys-swapfile remove
sudo systemctl disable dphys-swapfile.service
sudo sed -i -e 's/$/ cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1/' /boot/cmdline.txt
```

Install K8s (kubeadm, kubelet, kubectl and kubernetes-cni)
```
sudo bash -c "echo deb http://apt.kubernetes.io/ kubernetes-xenial main > /etc/apt/sources.list.d/kubernetes.list"
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo apt-get update
sudo apt-get install -qy kubelet=1.18.5-00 kubectl=1.18.5-00 kubeadm=1.18.5-00 kubernetes-cni=0.8.6-00
```

Set Iptables to legacy:
```
sudo sysctl net.bridge.bridge-nf-call-iptables=1
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
```

# Just Master node setup:

```
sudo kubeadm config images pull -v3
sudo kubeadm init--pod-network-cidr 10.244.0.0/16
# this produce this output:
# capture text and run as normal user. e.g.:
# mkdir -p $HOME/.kube
# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
# sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Installing CNI (flannel) for our pod network
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

# on worker-nodes:
```
kubeadm join 10.0.0.50:6443 --token yi4hzn.glushkg39orzx0fk \
    --discovery-token-ca-cert-hash sha256:xyz0721e03e1585f86e46e477de0bdf32f59e0a6083f0e16871ababc123missing here more documentation....
```

Check the status of the k8s env  
```
kubectl get nodes
kubectl get pods --all-namespaces
```

My cluster looks like this:
```
$ kubectl get nodes
NAME                   STATUS   ROLES    AGE   VERSION
k8s-pi-master1          Ready    master   78d   v1.18.5
k8s-pi-node1            Ready    <none>   78d   v1.18.5
k8s-pi-node2   Ready    <none>   37d   v1.18.5
$ 
```

NOTE:   
If you to have re-do - several times I re-set the k8s cluster because i screwed up it, here how to do it in the best way possible:
```
sudo kubeadm reset - on workers node too
rm -fr .kube
sudo rm -fr /etc/cni/net.d/* 
sudo iptables -F    - on worker node too
docker rmi $(docker images| awk '{print $3}')   - on worker node too, be carful with this, it will remove all docker images
sudo kubeadm init phase certs all
sudo kubeadm init phase kubeconfig all
sudo kubeadm init phase control-plane all --pod-network-cidr 10.244.0.0/16
sudo sed -i 's/initialDelaySeconds: [0-9][0-9]/initialDelaySeconds: 240/g' /etc/kubernetes/manifests/kube-apiserver.yaml
sudo sed -i 's/failureThreshold: [0-9]/failureThreshold: 18/g'             /etc/kubernetes/manifests/kube-apiserver.yaml
sudo sed -i 's/timeoutSeconds: [0-9][0-9]/timeoutSeconds: 20/g'            /etc/kubernetes/manifests/kube-apiserver.yaml
sudo kubeadm init --v=1 --skip-phases=certs,kubeconfig,control-plane --ignore-preflight-errors=all --pod-network-cidr 10.244.0.0/16
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

I tried to install calico first, but it failed with following error, some how i think it keept the old config, anyway if you see a timeout like this from any pod:
```
time out [ERROR] plugin/errors: 2 checkpoint-api.weave.works. A: read udp 10.244.0.2:59346
```

Run this: 
```
sudo iptables -P FORWARD ACCEPT
```
I have created systemd script to have this survive after reboot. I did not have a time to figured out where k8s save the firewall rules, work-around just works
```
[Unit]
Description=iptables rule for K8s to allow traffic between pods systemd service.
After=network.target containerd.service kubelet.service

[Service]
Type=oneshot
ExecStart=/bin/bash /home/pi/firewall_forward.sh start
ExecStop=/bin/bash /home/pi/firewall_forward.sh stop

[Install]
WantedBy=multi-user.target
```
and the script that actually set the firewall rule:
```
$ cat /home/pi/firewall_forward.sh

#!/bin/bash
case $1 in
   start)
         while true; do
              sleep 15
              echo status:
              /usr/sbin/iptables -L 2>/dev/null | grep DROP | grep policy 
              myeval=$?
              if [[ $myeval -eq 0 ]] ;then
                    echo applying FW rule
                    /usr/sbin/iptables -P FORWARD ACCEPT 
                    /usr/sbin/iptables -L 2>/dev/null | grep FORW | grep policy 
                    echo firewall rule applied successfully
         	    exit 0
              else
         	    echo forward is accept
                    /usr/sbin/iptables -L 2>/dev/null | grep FORW | grep policy 
                    sleep 2
              fi
         done
	 ;;
    stop)
         echo nothing to do
         ;;
    *)
         echo -n use start or stop
	 exit 1
	 ;;
esac

```


During the installation process, I ran in to different issues, I cannot recall most of them, but i dealed with re-reinitializing, and also where this configuration helped to sorted out the installation process:
```
for file /boot/cmdline.txt 
console=serial0,115200 console=tty1 root=PARTUUID=3f672e03-02 rootfstype=ext4 elevator=deadline fsck.repair=yes rootwait quiet splash plymouth.ignore-serial-consoles cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1
```

In order to access easily to any service (like jenkins) running on the K8s env, I have installed MetalLB  
MetalLB is a load-balancer implementation for bare metal Kubernetes clusters, using standard routing protocols. You can read more about it in this link:  
https://metallb.universe.tf/

To install it, use kubectl to apply the deployment, i was using latest version of it, but i was running into several issues, so I tried with this version and worked fine without any issues:  
```
kubectl apply -f https://gist.githubusercontent.com/Shogan/d418190a950a1d6788f9b168216f6fe1/raw/ca4418c7167a64c77511ba44b2c7736b56bdad48/metallb.yaml
```
wait till it is deployed and then you will need to deploy something similar that apply your network:  I'm giving half of  my subnet to it, LoadBalancer IPs will start on 10.0.0.128
```
$ cat network_home_lab.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: address-pool-1
      protocol: layer2
      addresses:
      - 10.0.0.128/25
$ 

```

Right now my loadbalancer is giving two IPs, one for jenkins and another for mysql:  
```
$ k get svc
NAME    TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
mysql   LoadBalancer   10.110.207.184   10.0.0.129    3306:30007/TCP   3d22h
pi@k8s-pi-master1:~ $ kn jenkins
Context "kubernetes-admin@kubernetes" modified.
Active namespace is "jenkins".
$ k get svc
NAME            TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
jenkins         LoadBalancer   10.105.114.186   10.0.0.128    80:32190/TCP   7d
jenkins-agent   ClusterIP      10.104.229.69    <none>        50000/TCP      7d
$ 
```

