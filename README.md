# XPU Share Scheduler Extender

昆仑设备的 K8S scheduler 插件，允许用户以显存为粒度创建 pod

# 安装说明

## 0.在物理机安装昆仑驱动
```console
# 确保安装完成后，通过 xpu_smi 查看卡的状态, 输出类似：
$ xpu_smi
Runtime Version: 4.0
Driver Version: 4.0
  DEVICES
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
| DevID |   PCI Addr   | Model |        SN        |    INODE   | State | UseRate |    L3     |   Memory    | Power(W) | Temp | Freq(MHz) | Firmware Version | CPLD Version |
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
|     0 | 0000:86:00.0 | K200  | 000000000007a136 | /dev/xpu0  |     N |     0 % | 0 / 16 MB | 0 / 8064 MB |       37 |   41 |       900 | 0001.0015.0022   |       e71702 |
|     1 | 0000:86:00.0 | K200  | 000000000007a136 | /dev/xpu1  |     N |     0 % | 0 / 16 MB | 0 / 8064 MB |       37 |   39 |       900 | 0001.0015.0022   |       e71702 |
|     2 | 0000:af:00.0 | K200  | 000000000007a135 | /dev/xpu2  |     N |     0 % | 0 / 16 MB | 0 / 8064 MB |       36 |   39 |       900 | 0001.0015.0022   |       e71702 |
|     3 | 0000:af:00.0 | K200  | 000000000007a135 | /dev/xpu3  |     N |     0 % | 0 / 16 MB | 0 / 8064 MB |       36 |   39 |       900 | 0001.0015.0022   |       e71702 |
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

## 1. 部署 XPU share scheduler extender
``` console
$ kubectl create -f ./deployment/xpushare-scheduler-extender.yaml
clusterrole.rbac.authorization.k8s.io/xpushare-scheduler-extender created
serviceaccount/xpushare-scheduler-extender created
clusterrolebinding.rbac.authorization.k8s.io/xpushare-scheduler-extender created
deployment.apps/xpushare-scheduler-extender created
service/xpushare-scheduler-extender created
```

## 2. 更新 scheduler configuration
```console
# 2.1 检查版本
$ kubectl version

# 2.2 更新配置

# 2.2.a 如果返回结果为 GitVersion:v1.23.x
$ cp deployment/config/v1.23.x/scheduler-policy-config.yaml /etc/kubernetes/

# 对 /etc/kubernetes/manifests/kube-scheduler.yaml 进行更新
# 在 pod spec 的部分添加：
- mountPath: /etc/kubernetes/scheduler-policy-config.yaml
  name: scheduler-policy-config
  readOnly: true

- hostPath:
      path: /etc/kubernetes/scheduler-policy-config.yaml
      type: FileOrCreate
  name: scheduler-policy-config

# 在 scheduler arguments 的部分添加：
- --config=/etc/kubernetes/scheduler-policy-config.yaml

# 可以参考 deployment/config/v1.23.x/kube-scheduler.yaml

# 2.2.b 如果版本低于 v1.23.x
$ cp deployment/config/before_v1.11.x/scheduler-policy-config.json /etc/kubernetes/

# 在 pod spec 的部分添加：
- mountPath: /etc/kubernetes/scheduler-policy-config.json
  name: scheduler-policy-config
  readOnly: true

- hostPath:
      path: /etc/kubernetes/scheduler-policy-config.json
      type: FileOrCreate
  name: scheduler-policy-config

# 在 scheduler arguments 的部分添加：
- --policy-config-file=/etc/kubernetes/scheduler-policy-config.json

# 可以参考 deployment/config/before_v1.11.x/kube-scheduler.yaml
```

## 3. Deploy Device Plugin
```console
kubectl create -f deployment/device-plugin-rbac.yaml
kubectl create -f deployment/device-plugin-ds.yaml
```

## 4. 给 XPU 节点增加 labels
```console
# target node 是插有昆仑卡的机器节点名
kubectl label node <target_node> xpushare=true
```

## 5. 安装 kubectl extension
```console
cp ./deployment/kubectl-inspect-xpushare /usr/bin/kubectl-inspect-xpushare
chmod u+x /usr/bin/kubectl-inspect-xpushare

```

# 使用说明
1. 通过增加 `baidu.com/xpu-mem` 配置来为pod申请xpu-mem资源（单位为 MiB）:
```
        resources:
          limits:
            # MiB
            baidu.com/xpu-mem: 4500
```
2. 通过设置 `baidu.com/xpu-count` 标签来指定pod使用的xpu个数，每个xpu提供xpu-mem/xpu-count大小显存，若不设置该标签则所有显存由1个xpu提供:
```
    metadata:
      labels:
        baidu.com/xpu-count: "1"
```

具体配置可以参考`./deployment/samples/xpu_pod.yaml`

xpu-mem申请成功后，可通过`kubectl-inspect-xpushare`工具查看node上的xpu-mem资源使用情况:

```console
$ kubectl create -f ./deployment/samples/xpu_pod.yaml

$ ./deployment/kubectl-inspect-xpushare
# 或 kubectl inspect xpushare
# 输出类似
NAME      IPADDRESS     XPU0(Allocated/Total)  XPU1(Allocated/Total)  XPU2(Allocated/Total)  XPU3(Allocated/Total)  XPU Memory(MiB)
minikube  192.168.49.2  4500/8064              4500/8064              0/8064                 0/8064                 9000/32256
---------------------------------------------------------------------------------------
Allocated/Total XPU Memory In Cluster:
9000/32256 (27%)

# 注意正常状态下，可以看到相关的 pods 都在 running 的状态，如下：
$ kubectl get po -A
NAMESPACE     NAME                                           READY   STATUS             RESTARTS        AGE
default       xpu-pod-6bf885cf6d-2vzgz                       1/1     Running            0               2m24s
default       xpu-pod-6bf885cf6d-m6vf6                       1/1     Running            0               2m24s
kube-system   kube-scheduler-minikube                        1/1     Running            0               14h
kube-system   xpushare-device-plugin-ds-p5kdn                1/1     Running            0               123m
kube-system   xpushare-scheduler-extender-77b8688cf9-t2h5j   1/1     Running            0               29m
...

```

# 编译开发

```console
$ make
mkdir -p /mnt/weihaoji/icode-xpushare-sche-extender/baidu/personal-code/xpushare-sche-extender/gen
go build -o /mnt/weihaoji/icode-xpushare-sche-extender/baidu/personal-code/xpushare-sche-extender/gen/xpushare-scheduler-extender /mnt/weihaoji/icode-xpushare-sche-extender/baidu/personal-code/xpushare-sc
he-extender/cmd/*.go
docker build \
        -t kunlunxpu/xpushare-scheduler-extender:v0.1.0 \
        -f Dockerfile \
        .
Sending build context to Docker daemon   72.9MB
Step 1/3 : FROM debian:bullseye-slim
 ---> bfb07174f19e
Step 2/3 : COPY ./gen/xpushare-scheduler-extender /usr/bin/xpushare-scheduler-extender
 ---> 673971bafabd
Step 3/3 : CMD xpushare-scheduler-extender
 ---> Running in 0df87e8ecdfe
Removing intermediate container 0df87e8ecdfe
 ---> 296fa4647f82
Successfully built 296fa4647f82
Successfully tagged kunlunxpu/xpushare-scheduler-extender:v0.1.0

$ ls gen/
xpushare-scheduler-extender

$ docker images
REPOSITORY                                                   TAG                 IMAGE ID            CREATED             SIZE
kunlunxin/xpushare-scheduler-extender                         v0.1.0              296fa4647f82        2 minutes ago       112MB
```
