# 1. 只更新 master 证书  

小于等于 `v1.9` 版本，etcd 默认不使用 TLS 连接，没有 etcd 相关证书，只需要更新 master 证书即可

如果有多个 master 节点，在每个 master 节点都执行一次  

```
./update-kubeadm-cert.sh master
```

将更新以下 master 证书和 kubeconfig 配置文件  

```
/etc/kubernetes
├── admin.conf
├── controller-manager.conf
├── scheduler.conf
├── kubelet.conf
└── pki
    ├── apiserver.crt
    ├── apiserver-kubelet-client.crt
    └── front-proxy-client.crt
```

# 2. 使用脚本处理后证书是延续 10 年吗？

准确来说并不是  

kubeadm 签发的 CA 默认有效期是 10 年 (从 init 集群那一刻开始算)，当 CA 到期后，整套证书体系都失效了  

也就是说，10 年有效期是从 init 集群那一刻开始算的，不是从执行脚本更新证书那一刻开始算  


# 3. kubeadm 证书相关命令发展

- `v1.8` 版开始提供了证书生成命令 `kubeadm alpha phase certs <cert_name>`
- `v1.13` 版开始证书生成命令改为 `kubeadm init phase certs <cert_name>`
- `v1.15` 版增加了证书更新命令 `kubeadm alpha certs renew <cert_name>`（这个命令与上面两个区别是：上面两个是生成证书，这个是更新证书），`v1.15` 版之后可使用 `kubeadm alpha certs renew <cert_name>` 来更新证书


# 4. kubeadm 命令更新证书手动处理

使用该脚本更新证书，不涉及下面这个 bug，无需手动处理

bug 见 https://github.com/kubernetes/kubeadm/issues/1753 ，这个bug 在 `1.17` 版修复

针对小于  `1.17版本` ，使用  `kubeadm alpha certs renew <cert_name>`  来更新证书

`kubeadm alpha certs renew`  并不会更新 kubelet 证书（kubelet.conf 文件里面写的客户端证书），因为 kubelet 证书是默认开启自动轮回更新的，但是在执行 `kubeadm init` 的 master 节点的 kubelet.conf 文件里面的证书是以 base64 编码写死的 (类似 controller-manager.conf 里面的证书)

在用 `kubeadm`  命令更新 master 证书时需要手动将 kubelet.conf 文件的  `client-certificate-data`  和  `client-key-data`  改为：

```yaml
client-certificate: /var/lib/kubelet/pki/kubelet-client-current.pem
client-key: /var/lib/kubelet/pki/kubelet-client-current.pem
```

# 5. 更新slave证书
在`v1.7.5`版本使用 #1 中`./update-kubeadm-cert.sh master`,没有自动更新 slave 节点。（高版本不会出现这个问题，会自动更新 slave 节点）。

slave 中具体报错为`Failed to list *v1.Pod: the server has asked for the client to provide credentials (get pods)`

解决办法: 在 master 中查看 token, 在 slave 中使用 `kubeadm join`。
```bash
root@master:~# kubeadm token list
TOKEN                     TTL         EXPIRES   USAGES                   DESCRIPTION                                                EXTRA GROUPS
27c516.e6ef88e9f68908e1   <forever>   <never>   authentication,signing   The default bootstrap token generated by 'kubeadm init'.   <none>
[root@slave ~]# rm /etc/kubernetes/kubelet.conf
[root@slave ~]# kubeadm1.7 join --token 27c516.e6ef88e9f68908e1 master_ip:6443 --node-name master_hostname --skip-preflight-checks
[root@slave ~]# systemctl restart kubelet
[root@slave ~]# systemctl status kubelet -l
```

