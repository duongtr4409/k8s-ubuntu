# Kubernetes \_ Ubuntu (Devops mentor):

link: https://www.youtube.com/watch?v=xvVZCt5QR7k&list=PLxPA4dviHjbDzTU1pnDvn5TGhwt5l1A-O&index=13

## Khởi chạy các máy chủ:

- có 6 máy chủ:

1: LB: máy chủ loadblance<br>
2: master<br>
3: addMaster1<br>
4: addMaster2<br>
5: addWorker1<br>
6: addWorker2<br>

## Cấu hình load balancer

- tại máy 'lb' cài đặt nginx

```command
apt install nginx -y
```

- tạo file cấu hình load stream
  với nội dung sau

```conf
stream {
    upstream kubernetes {
        server master1_ip:6443 max_fails=3 fail_timeout=30s;
        server master2_ip:6443 max_fails=3 fail_timeout=30s;
        server master3_ip:6443 max_fails=3 fail_timeout=30s;
    }
server {
        listen 6443;
        #listen 443;
        proxy_pass kubernetes;
    }
}
```

ps: master1_ip | master2_ip | master3_ip : là ip của các máy master

```command
cd /etc/nginx/
mkdir k8s-lb.d
cd /etc/nginx/k8s-lb.d
touch apiserver.conf
vi /etc/nginx/k8s-lb./apiserver.conf
```

- apply file config 'apiserver.conf' cho nginx

thêm nội dung sau vào cuối file '/etc/nginx/nginx.conf'

```conf
...

include /etc/nginx/k8s-lb.d/*;
```

-- restart lại nginx

```command
nginx -s reload
```

## Khởi tại cluster

- Tại máy "master" chạy lệnh:

```command
kubeadm init --control-plane-endpoint=lb_ip:6443 --upload-certs --pod-network-cidr=10.0.0.0/8
```

```command
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

export KUBECONFIG=/etc/kubernetes/admin.conf
```

ps: "lb_ip" là ip của máy load balance (máy "lb")

- Dùng "Heml" để cài CNI

```command
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

```command
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --version 1.11.6 --namespace kube-system
```

- lấy thông tin join master node

```command
kubeadm join apiserver.lb:6443 --token [token] \
        --discovery-token-ca-cert-hash sha256:[sha256] \
        --control-plane --certificate-key [cert-key]
```

ps:<br> -[token] chạy lệnh để lấy

```command
kubeadm token create
```

-[sha256] chạy lệnh để lấy

```command
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
openssl dgst -sha256 -hex | sed 's/^.* //'
```

-[token] chạy lệnh để lấy

```command
kubeadm init phase upload-certs --upload-certs
```

vd:

```command
kubeadm join apiserver.lb:6443 --token dj5j6p.0dpca6cmty9ezirk \
        --discovery-token-ca-cert-hash sha256:104b74f95a8001eec1286c3696ce34a8c77bc9f3b054142bba5d90a5d4b90bdf \
        --control-plane --certificate-key c670b997b1b5df0d78368c4a0f6bd5d507c2052312d98b1428a6d746f2979808
```

### PS: nếu kuberlet không chạy:

- kiểm tra bằng lệnh sau:

```command
systemctl status kubelet
```

- xem log kubelet

```command
journalctl -exu kubelet
```

- chạy lệnh sau để fix:

```command
sudo swapoff -a
```

```command
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
