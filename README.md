# 前言

本环境采用kubeadm安装(为了简单,非二进制环境安装),master节点未做高可用.经典的1master/2node模式.

[dawdler-series](https://github.com/srchen1987/dawdler-series) 终究还是要是要容器化.大势所趋所以被迫搞一下子.

全球第一个关于buildkit+nexus私有库实现的部署文档.

## 1.环境准备

### 1.1 四台虚拟机

| 操作系统 | cpu | 内存大小 | ip地址 | 描述 |
| :-: | :-: | :-: | :-: | :-: |
| Fedora-Server-dvd-x86_64-36-1.5 | 4u | 4G | 192.168.43.144 | k8s-master |
| Fedora-Server-dvd-x86_64-36-1.5 | 4u | 4G | 192.168.43.146 | k8s-node1 |
| Fedora-Server-dvd-x86_64-36-1.5 | 4u | 4G | 192.168.43.147 | k8s-node2 |
| Fedora-Server-dvd-x86_64-36-1.5 | 4u | 8G | 192.168.43.137 | nexus |

### 1.2 设置基础环境

三台k8s的主机分别做以下设置

#### 1.2.1 设置host

vim /etc/hosts

```text
192.168.43.144 k8s-master1
192.168.43.146 k8s-node1
192.168.43.147 k8s-node2
192.168.43.137 my.nexus.org
```

也可以用 hostnamectl set-hostname   k8s-master1

#### 1.2.2 关闭防火墙与selinux

```shell
systemctl stop firewalld 

systemctl disable firewalld 

setenforce 0  #表示临时关闭selinux防火墙
```

永久关闭selinux
vim /etc/selinux/config

SELINUX=disabled

#### 1.2.3 LVS的管理工具ipvsadm ipset

```shell
yum install -y ipvsadm  ipset
```

#### 1.2.4 安装 iproute

```shell
yum install iproute-tc.x86_64 -y
```

#### 1.2.5 时间同步

```shell
yum install chrony -y 

systemctl enable chronyd 

systemctl start chronyd

chronyc sources 
```

#### 1.2.6 设置k8s.conf 二层网络走FORWARD

vim /etc/sysctl.d/k8s.conf

```conf
net.bridge.bridge-nf-call-ip6tables = 1 
net.bridge.bridge-nf-call-iptables = 1 
net.ipv4.ip_forward = 1 
vm.swappiness=0 #vm.swappiness是操作系统控制物理内存交换出去的策略.它允许的值是一个百分比的值,范围在0-100，该值默认为60.vm.swappiness设置为0表示尽量少swap,100表示尽量将inactive的内存页做交换.
```

```shell
sysctl -p /etc/sysctl.d/k8s.conf
```

#### 1.2.7 关闭swap

修改/etc/fstab文件,注释掉 SWAP 的自动挂载.

```shell
swapoff -a

sed -i '/swap/s/^/#/' /etc/fstab

systemctl mask dev-zram0.swap

sysctl -p
```

使用free -m确认 swap 已经关闭.

#### 1.2.8 内核加载所需模块

```shell
mkdir /etc/sysconfig/modules/
```

vim  /etc/sysconfig/modules/ipvs.modules

```conf
modprobe -- ip_vs 
modprobe -- ip_vs_rr 
modprobe -- ip_vs_wrr 
modprobe -- ip_vs_sh 
modprobe -- nf_conntrack
modprobe br_netfilter 
```

使用lsmod | grep -e ip_vs -e nf_conntrack_命令查看是否已经正确加载所需的内核模块.

改变权限

```shell
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack
```

#### 1.2.9 安装yum和lvm2

```shell
 yum install -y yum-utils device-mapper-persistent-data lvm2
```

#### 1.2.10 添加docker源

```shell
yum-config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo 

yum install containerd.io.x86_64 -y 
```

#### 1.2.11  创建containerd配置文件

```shell
mkdir -p /etc/containerd 
containerd config default > /etc/containerd/config.toml
```

如果不能科学上网的同学则需要替换配置文件

```shell
sed -i "s#k8s.gcr.io#registry.cn-hangzhou.aliyuncs.com/google_containers#g"  /etc/containerd/config.toml
sed -i 's#SystemdCgroup = false#SystemdCgroup = true#g' /etc/containerd/config.toml
sed -i "s#https://registry-1.docker.io#https://registry.cn-hangzhou.aliyuncs.com#g"  /etc/containerd/config.toml
```

启动Containerd 并设置自启动

```shell
systemctl daemon-reload

systemctl enable containerd

systemctl restart containerd
```

#### 1.2.12  加入kubernetes源

vim  /etc/yum.repos.d/kubernetes.repo

不能科学上网的同学用这个源即可

```repo
[kubernetes] 
name=Kubernetes 
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64 
enabled=1 
gpgcheck=0 
repo_gpgcheck=0 
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg 
```

可以科学上网的同学建议用下面的源

```repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
```

#### 1.2.13 安装kubelet kubeadm kubectl

```shell
yum install -y kubelet kubeadm kubectl

systemctl daemon-reload 

systemctl enable kubelet && systemctl start kubelet
```

#### 1.2.14 设置运行时的endpoint

```shell
crictl config runtime-endpoint /run/containerd/containerd.sock
```

#### 1.2.15 设置~/.docker/config.json

安装nexus之前可以跳过此步骤,安装之后再配置.

```shell
mkdir ~/.docker
```

通过base64输出nexus账号密码

```shell
echo -n "admin:123456" | base64
```

输出: YWRtaW46MTIzNDU2

vim ~/.docker/config.json

```json
{
  "auths": {
    "https://my.nexus.org:8083": {
      "auth": "YWRtaW46MTIzNDU2"
    }
  }
}
```

### 2. kubeadm初始化k8s环境

在master节点执行

#### 2.1 查看kubeadm 需要的镜像列表(可选)

```shell
 kubeadm config images list 
```

#### 2.2 拉取kubeadm需要的镜像列表

可以科学上网的可以去除--image-repository

```shell
 kubeadm config images pull --kubernetes-version=v1.24.2 --image-repository=registry.aliyuncs.com/google_containers
```

#### 2.3 kubeadm初始化

命令行初始化 可以科学上网的可以去除--image-repository

```shell
kubeadm init \
    --apiserver-advertise-address=192.168.43.144 \
    --image-repository registry.aliyuncs.com/google_containers \
    --pod-network-cidr=10.244.0.0/16 \
    --service-cidr=10.1.0.0/16 \
    --kubernetes-version=v1.24.2
```

配置文件初始化方式

```shell
kubeadm config print init-defaults > kubeadm.yaml 
```

编辑 kubeadm.yaml

不可以科学上网的同学请替换 registry.cn-hangzhou.aliyuncs.com/google_containers

```shell
kubeadm init --config=kubeadm.yaml
```

执行完成之后会有以下输出

```shell
kubeadm join 192.168.43.144:6443 --token ds62l9.scyt0ise8k9qakae \
        --discovery-token-ca-cert-hash sha256:c1d0577681ad0c5dacd7814fc60afd71fe9adb33b3c6168bf93854645d33e33e
```

### 3. node节点加入k8s集群

运行kubeadm init输出

在node节点中执行

```shell
kubeadm join 192.168.43.144:6443 --token ds62l9.scyt0ise8k9qakae \
        --discovery-token-ca-cert-hash sha256:c1d0577681ad0c5dacd7814fc60afd71fe9adb33b3c6168bf93854645d33e33e
```

#### 3.1 查看集群状态

在master中执行

```shell
kubectl get nodes
```

此时可以在列表中看到k8s-node1、k8s-node2 状态为NotReady.(因为未安装网络插件)

### 4. 安装calico插件

k8s网络插件有很多,推荐用calico.使用其他的如Flannel请自行google.

在master中执行

```shell
wget https://docs.projectcalico.org/manifests/calico.yaml

kubectl apply -f calico.yaml
```

安装完成之后可以看下集群状态,状态NotReady变为Ready.

如果有问题请自行排查,如:

```shell
kubectl get pod -n kube-system
```

### 5. 配置自动补全命令行(可选)

在master中执行

```shell
yum install -y bash-completion 
source /usr/share/bash-completion/bash_completion 
source <(kubectl completion bash) 
echo "source <(kubectl completion bash)" >> ~/.bashrc 
```

### 6. 部署nginx并测试

在master中执行

#### 6.1 通过kubectl运行nginx绑定80端口

```shell
kubectl run web-nginx --image=nginx --port=80
```

#### 6.2 验证nginx

查看镜像列表(默认namespace为default)

```shell
kubectl get pods -owide
```

响应如下：

```text
NAME        READY   STATUS    RESTARTS   AGE     IP              NODE    NOMINATED NODE   READINESS GATES
web-nginx   1/1     Running   0          20s     172.17.104.30   node2   <none>           <none>
```

```shell
wget 172.17.104.30
```

响应的index.html 通过cat可以查看为nginx的首页

#### 6.3 通过expose暴露端口在node

```shell
kubectl expose pod web-nginx --name=web-nginx  --port=80  --type=NodePort
```

查看服务列表

```shell
kubectl get service -owide
```

响应结果

```text
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE    SELECTOR
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP        7d1h   <none>
web-nginx    NodePort    10.96.181.25   <none>        80:32002/TCP   5s     run=web-nginx
```

查看所在node

```shell
 kubectl get pods -A -owide
```

通过 访问 ``` http://nodeIp:32002/ ``` 可验证nginx是否启动成功.

### 7. 制作证书

#### 7.1 生成caKey与证书

```shell
#输入密码 密码要记下来 例: password
openssl genrsa -des3 -out ca.key  2048 

#输入上面输入的密码 后面按需求输入
openssl req -x509 -key ca.key -out ca.crt
```

#### 7.2 pkiCa配置

```shell
mkdir -p /etc/pki/CA/newcerts
 
touch /etc/pki/CA/index.txt
 
echo '01' > /etc/pki/CA/serial
```

#### 7.3 生成nexus证书key

```shell
#输入密码 密码要记下来 例: password
openssl genrsa -des3 -out nexus.key 2048  
```

#### 7.4 生成证书请求文件

```shell
#输入nexus证书key的密码 后面按需求输入(与ca.crt一致)
openssl req -new -key nexus.key -reqexts SAN -config <(cat /etc/pki/tls/openssl.cnf \
        <(printf "[SAN]\nsubjectAltName=DNS:my.nexus.org,DNS:www.nexus.org"))     -out nexus.csr
```

#### 7.5 自签署证书

```shell
openssl x509 -req -days 3650 \
    -in nexus.csr -out nexus.crt \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -extensions SAN \
    -extfile <(cat /etc/pki/tls/openssl.cnf \
        <(printf "[SAN]\nsubjectAltName=DNS:my.nexus.org,DNS:www.nexus.org"))
```

注意: 如果多次签署报错请清理通过 vim etc/pki/CA/index.txt编辑文件删除文件内容.

#### 7.6 添加证书信任

三台机器(k8s)全部需要执行相同步骤,ca.crt复制到其他节点上即可.

```shell
chmod 644 /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem
        
cat ca.crt >> /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem

chmod 444 /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem
```

```shell
cp nexus.crt  /etc/pki/ca-trust/source/anchors/my.nexus.org.crt

update-ca-trust
```

#### 7.7 导出pfx文件

```shell
openssl pkcs12 -export -out nexus.pfx -in nexus.crt -inkey nexus.key
```

#### 7.8 通过java代码生成keystore.jks

```java
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.security.Key;
import java.security.KeyStore;
import java.security.cert.Certificate;
import java.util.Enumeration;

public class CoverP12TokeyStore {
 public static final String PKCS12 = "PKCS12";
 public static final String JKS = "JKS";
 public static final String PFX_KEYSTORE_FILE = "/home/srchen/my.pfx";// pfx文件位置
 public static final String PFX_PASSWORD = "password";// 导出为pfx文件的设的密码
 public static final String JKS_KEYSTORE_FILE = "/home/srchen/software/nexus/nexus-3.40.1-01/etc/ssl/keystore.jks"; // jks文件位置
 public static final String JKS_PASSWORD = "password";// JKS的密码

 public static void coverTokeyStore() {
  FileInputStream fis = null;
  FileOutputStream out = null;
  try {
   KeyStore inputKeyStore = KeyStore.getInstance("PKCS12");
   fis = new FileInputStream(PFX_KEYSTORE_FILE);
   char[] pfxPassword = null;
   if ((PFX_PASSWORD == null) || PFX_PASSWORD.trim().equals("")) {
    pfxPassword = null;
   } else {
    pfxPassword = PFX_PASSWORD.toCharArray();
   }
   char[] jksPassword = null;
   if ((JKS_PASSWORD == null) || JKS_PASSWORD.trim().equals("")) {
    jksPassword = null;
   } else {
    jksPassword = JKS_PASSWORD.toCharArray();
   }

   inputKeyStore.load(fis, pfxPassword);
   fis.close();
   KeyStore outputKeyStore = KeyStore.getInstance("JKS");
   outputKeyStore.load(null, jksPassword);
   Enumeration<String> enums = inputKeyStore.aliases();
   while (enums.hasMoreElements()) {
    String keyAlias = enums.nextElement();
    if (inputKeyStore.isKeyEntry(keyAlias)) {
     Key key = inputKeyStore.getKey(keyAlias, pfxPassword);
     Certificate[] certChain = inputKeyStore.getCertificateChain(keyAlias);
     outputKeyStore.setKeyEntry(keyAlias, key, jksPassword, certChain);
    }
   }
   out = new FileOutputStream(JKS_KEYSTORE_FILE);
   outputKeyStore.store(out, jksPassword);
   out.flush();
  } catch (Exception e) {
   e.printStackTrace();
  } finally {
   if (fis != null) {
    try {
     fis.close();
    } catch (IOException e) {
     e.printStackTrace();
    }
   }
   if (out != null) {
    try {
     out.close();
    } catch (IOException e) {
     e.printStackTrace();
    }
   }
  }
 }

 public static void main(String[] args) {
  coverTokeyStore();
 }
}
```

### 8. 私有镜像库搭建

#### 8.1 下载nexus

```shell
wget https://download.sonatype.com/nexus/3/nexus-3.40.1-01-unix.tar.gz

tar -zxvf nexus-3.40.1-01-unix.tar.gz
```

#### 8.2 启动nexuss

```shell
cd nexus-3.40.1-01/bin

./nexus run &
```

启动完成之后密码生成在sonatype-work/nexus3/admin.password文件中.

访问```http://192.168.43.137:8081/``` 输入密码即可.

#### 8.3 开启https服务

由于后面采用buildkit来构建镜像推送到私有库中,只能采用https方式,所以需要开启nexus的https服务.

进入nexus配置目录.

```shell
cd /home/srchen/software/nexus/nexus-3.40.1-01/etc
```

编辑nexus-default.properties配置文件,添加${jetty.etc}/jetty-https.xml.

```properties
exus-args=${jetty.etc}/jetty.xml,${jetty.etc}/jetty-https.xml,${jetty.etc}/jetty-http.xml,${jetty.etc}/jetty-requestlog.xml
```

编辑/home/srchen/software/nexus/nexus-3.40.1-01/etc/jetty/jetty-https.xml,配置上面设置的jks的密码.

```xml
<New id="sslContextFactory" class="org.eclipse.jetty.util.ssl.SslContextFactory$Server">
          <Set name="KeyStorePath"><Property name="ssl.etc"/>/keystore.jks</Set>
    <Set name="KeyStorePassword">password</Set>
    <Set name="KeyManagerPassword">password</Set>
    <Set name="TrustStorePath"><Property name="ssl.etc"/>/keystore.jks</Set>
    <Set name="TrustStorePassword">password</Set>
    <Set name="EndpointIdentificationAlgorithm"></Set>
    <Set name="NeedClientAuth"><Property name="jetty.ssl.needClientAuth" default="false"/></Set>
    <Set name="WantClientAuth"><Property name="jetty.ssl.wantClientAuth" default="false"/></Set>
    <Set name="IncludeProtocols">
      <Array type="java.lang.String">
        <Item>TLSv1.2</Item>
      </Array>
    </Set>
  </New>
```

重启nexus

```shell
./nexus restart
```

#### 8.4 开启containerd私有库的支持

登陆 ```http://192.168.43.137:8081/```

访问 ```http://192.168.43.137:8081/#admin/repository/repositories```

点击 Create repository、docker(hosted),

Name输入 Containerd

Online 选中

HTTPS 输入8083端口

剩下全部选中,如果需要清理策略请自行配置.

保存之后会开启https的支持端口为8083.

### 9. buildkit安装

[buildkit](https://github.com/moby/buildkit) 号称下一代构建docker的神器.

在master主机上下载

```shell
wget https://github.com/moby/buildkit/releases/download/v0.10.3/buildkit-v0.10.3.linux-arm64.tar.gz

```

解压并移到local下

```shell
tar -zxvf buildkit-v0.10.3.linux-arm64.tar.gz

mkdir -p  /usr/local/buildkit-v0.10.3

mv bin /usr/local/buildkit-v0.10.3
```

```shell
vim /etc/profile

#加入
PATH=$PATH:/usr/local/buildkit/bin

source /etc/profile
```

### 10. 配置buildkit

```shell
mkdir /etc/buildkit
 ```

vim /etc/buildkit/buildkitd.toml

```toml
debug = true
[worker.containerd]
  namespace = "k8s.io"

[registry."docker.io"]
  mirrors = ["my.nexus.org:8083"]
  http = false
  nsecure = true
  ca=["/home/srchen/keys/ca.key"]
  
[registry."my.nexus.org:8083"]
  http = false
  insecure = true
  ca=["/home/srchen/keys/ca.key"]
  [[registry."my.nexus.org".keypair]]
    key="/home/srchen/keys/nexus.key"
    cert="/home/srchen/keys/nexus.crt"
```

#### 10.1 启动buildkitd服务

```shell
buildkitd --oci-worker=false --containerd-worker=true & 
```

#### 10.2 构建自定义镜像

通过[dawdler-series](https://github.com/srchen1987/dawdler-series)来举例子.

```shell
wget https://github.com/srchen1987/dawdler-runtime/archive/refs/tags/0.0.2-RELEASES.tar.gz

tar -zxvf 0.0.2-RELEASES.tar.gz

cd dawdler-runtime-0.0.2-RELEASES

```

vim Dockerfile

```Dockerfile
FROM suranagivinod/openjdk8
MAINTAINER suxuan696@gmail.com
RUN mkdir /opt/dawdler-server-0.0.2
ADD ./dawdler-server-0.0.2 /opt/dawdler-server-0.0.2
WORKDIR /opt/dawdler-server-0.0.2/bin
ENTRYPOINT ["/bin/sh","dawdler.sh","run"]
```

#### 10.3 构建并推送到私有镜像库

```shell
buildctl build   --no-cache \
    --frontend=dockerfile.v0 \
    --local context=. \
    --local dockerfile=. \
    --output type=image,name=my.nexus.org:8083/dawdler:0.0.2,push=true \
    --export-cache type=inline
```

#### 10.4 通过kubectl发布dawdler服务

这里只是演示所以采用 run方式,如果是生产环境建议采用 deployment或apply -f dawdler.yaml方式来创建.

注意: 用run方式运行 node节点中镜像如果没更新则需要通过以下脚本清理.

```shell
#查看列表
crictl images

# 449cdf80490b5是 images查看到容器ID
crictl rmi 449cdf80490b5
```

也可以通过 imagePullPolicy: Always 配置每次私有仓拉取镜像.

```shell
kubectl run dawdler --image=my.nexus.org:8083/dawdler:0.0.2
```

#### 10.5 查看服务列表

```shell
kubectl get pods -n default -owide
```

响应

```text
NAME        READY   STATUS    RESTARTS   AGE     IP              NODE    NOMINATED NODE   READINESS GATES
dawdler     1/1     Running   0          11h     172.17.104.28   node2   <none>           <none>
```

### 10.6 查看日志

```shell
kubectl logs dawdler
```

响应

```text
Welcome to use dawdler!

  _____              __          __  _____    _        ______   _____
 |  __ \      /\     \ \        / / |  __ \  | |      |  ____| |  __ \
 | |  | |    /  \     \ \  /\  / /  | |  | | | |      | |__    | |__) |
 | |  | |   / /\ \     \ \/  \/ /   | |  | | | |      |  __|   |  _  /
 | |__| |  / ____ \     \  /\  /    | |__| | | |____  | |____  | | \ \
 |_____/  /_/    \_\     \/  \/     |_____/  |______| |______| |_|  \_\


OS arch: amd64
OS availableProcessors: 4
OS name: Linux
OS version: 5.17.5-300.fc36.x86_64
Jvm totalMemory: 5.20093696E8(507904.00K)
Jvm freeMemory: 5.01973992E8(490208.98K)
Heap Memory Usage:
init = 536870912(524288K) used = 18119704(17695K) committed = 520093696(507904K) max = 520093696(507904K)
Non-Heap Memory Usage:
init = 2555904(2496K) used = 11309544(11044K) committed = 11730944(11456K) max = -1(-1K)
Java options:
[-Xms512m, -Xmx512m, -Xmn128m]
ClassPath: .:./asm-7.1.jar:./cglib-3.3.0.jar:./commons-jexl3-3.2.jar:./commons-logging-1.2.jar:./commons-net-3.6.jar:./commons-pool2-2.10.0.jar:./consul-api-1.4.5.jar:./curator-client-5.1.0.jar:./curator-framework-5.1.0.jar:./curator-recipes-5.1.0.jar:./dawdler-core-0.0.2-RELEASES.jar:./dawdler-serialization-0.0.2-RELEASES.jar:./dawdler-server-0.0.2-RELEASES.jar:./dawdler-util-0.0.2-RELEASES.jar:./dom4j-2.1.3.jar:./jaxen-1.2.0.jar:./kryo-4.0.2.jar:./logback-access-1.2.7.jar:./logback-classic-1.2.7.jar:./logback-core-1.2.7.jar:./minlog-1.3.0.jar:./objenesis-2.5.1.jar:./reflectasm-1.11.3.jar:./slf4j-api-1.7.32.jar:./zookeeper-3.6.2.jar:./zookeeper-jute-3.6.2.jar
LibraryPath: /usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib
Server startup in 0 ms,Listening port: 9527!
```
