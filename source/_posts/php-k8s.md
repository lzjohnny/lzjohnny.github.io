---
title: PHP 容器化 K8S 迁移问题排查解决
date: 2019-07-19 09:06:47
tags:
---
# PHP 容器化 K8S 迁移问题排查解决

------

基础环境：
> * PHP 7.2.7
> * Docker 18.09.2
> * Kubernetes 1.12


## PHP 开启 huge_code_pages 报错解决方式
PHP 开启 opcache.huge_code_pages=1 后，运行时报错：Zend OPcache huge_code_pages: mmap(HUGETLB) failed: Cannot allocate memory (12)

### 问题原因：
系统 Hugepage 不足，CentOS 系统中 Hugepage 是没有默认开启的，可以通过下面命令查看，可见 HugePages_Total 是 0
```shell
cat /proc/meminfo | grep Huge
```
![meminfo](meminfo.png)

### 解决步骤：
**(1) 开启宿主机 Hugepage**

/etc/sysctl.conf 添加配置 `vm.nr_hugepages = 512`，保存退出，执行 sysctl -p 使配置生效，再次执行 cat /proc/meminfo | grep Huge 查看已分配 HugePages_Total 为 512。
如果查看结果与设置数值相差较大，则可能为系统已经无法分配出连续的这些内存，重启服务器后再重复操作即可。
注意上图 Hugepagesize 是 2048KB，也就是单个 Hugepage 是 2M，分配 512 个 Hugepage，所以总共分配的内存是 1G。

补充说明：单个 Hugepage 有 2048kB 和 1048576kB 两种，执行 ls /sys/devices/system/node/node0/hugepages，可见：
![hugepages](hugepages.png)

**(2) 为 nodes 分配 Hugepage**

可参考 Kubernetes 官方文档“[管理巨页](https://kubernetes.io/zh/docs/tasks/manage-hugepages/scheduling-hugepages/)”一节。首先 Kubernetes 节点宿主机必须预先分配巨页，这在上一步已经完成。
需要将开关 HugePages 设置为 true：`--feature-gates=HugePages=true`
在 master 和 node 各个节点上都修改 /etc/systemd/system/kubelet.service.d/10-kubeadm.conf 文件，增加启动参数 `--feature-gates=HugePages=true` 。如图所示：
![kubeadm-conf](kubeadm-conf.png)

修改后需要重启kubelet使参数生效：
systemctl daemon-reload
systemctl restart kubelet
此时在 master 执行 kubectl describe nodes，hugepages-2Mi 配置已经由 0 变为 1Gi：
![hugepages-allocated](hugepages-allocated.png)

**(3) 在 Deployment中使用 hugepages**

如果不进行该步骤直接部署容器会导致执行 PHP 任何命令均报错 Bus error (core dumped)
参考 Kubernetes 官方文档 “[管理巨页](https://kubernetes.io/docs/tasks/manage-hugepages/scheduling-hugepages/#api)” 中的 yaml 文件示例。
按照官方示例 yaml，添加 spec.containers.volumeMounts、spec.containers.resources、spec.volumes 即可，虽然官方是 Pod 的 yaml 做示例，但 Deployment、DaemonSet 修改方式都相同。

**(4) 部署完成**

部署完成后进入 Docker，执行 php -i | grep -i huge 检查 huge_code_pages 确实已经开启，且命令运行正常，问题解决。

**(5) 巨页分配注意事项**

其中 memory: 1024Mi 的数值要按照第一步分配的内存确定，512 * 2048KB = 1024M
需注意，一方面对于 PHP-FPM 应用，分配内存不宜过小。实测当分配 100M 但 FPM 启动多达 200 个进程时会由于内存不足频繁报错 child exited on signal 9 (SIGKILL) after xxx seconds from start。
另一方面，所有需要使用 hugepage 的应用 **分配并独占** 宿主机相应大小的 hugepage 空间，如果一个 Deployment 已分配了全部的 1024M 内存，另一个也需要 hugepage 的 Deployment 会处于 Pending 状态而不会启动成功。

### 参考资料：

 - https://juejin.im/entry/5bd9d4786fb9a0228b409420
 - https://v1-14.docs.kubernetes.io/docs/tasks/manage-hugepages/scheduling-hugepages/
 - https://kubernetes.io/zh/docs/tasks/manage-hugepages/scheduling-hugepages/
 - https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/

## PHP 上传文件报错
PHP Web 服务容器化部署后，上传文件报错：file_get_contents(): Filename cannot be empty
根据 php.ini 中 `upload_tmp_dir = /tmp/php7temp` 配置，检查容器中 /tmp/php7temp 目录是否存在、PHP-FPM 用户是否可写，若不存在可通过 K8S 的 emptyDir 挂载：

```yaml
containers:
  - name: php-container
    volumeMounts:
      - mountPath: /tmp/php7temp
        name: php-upload-tmp-dir

  volumes:
    - name: php-upload-tmp-dir
      emptyDir: {}
```
参考 Kubernetes 官方文档 “[volumes](https://kubernetes.io/docs/concepts/storage/volumes/#example-pod)” 一节 emptyDir 的示例。

## 采用K8S部署后配置、代码自动更新

应用配置、Nginx 配置、PHP-FPM 配置均是采用 ConfigMap 指定 subPath 的方式挂载到容器，一般环境变量或是 subPath 方式设置的配置不能自动更新。

滚动更新思路：Deployment 等 yaml 文件 template 字段下的内容发生变化时可以触发滚动更新，所以增加在 `annotations` 字段下增加配置版本信息即可，配置变更时由配置管理系统或是 githook 触发，自动更新生成新 yaml 并通过 kubectl apply -f 命令滚动更新。

```yaml
spec:
  template:
    metadata:
      annotations:
        codeVersionId: CODE_VERSION_ID # 供代码版本变化滚动更新
        configVersionId: CONFIG_VERSION_ID # 供配置变化滚动更新
```
同理可以实现代码版本变化自动滚动更新，只需通过 githook 每次提交代码时把 commit_id 或其他代码版本标识放入 yaml 文件即可。

参考他人文章：[ConfigMap的热更新](https://jimmysong.io/kubernetes-handbook/concepts/configmap-hot-update.html)

## 指定容器内进程启动用户
默认容器内进程以 root 用户启动，为避免安全风险等目的，可以使用非 root 用户启动。
```yaml
spec:
  template:
    spec:
      securityContext:
        runAsUser: 498
      containers:
	    - name: php-container
          securityContext:
            privileged: false
            allowPrivilegeEscalation: false
```
其中 runAsUser 为执行用户的 UID，该 UID 必须预先存在于镜像中，如 498 为本人镜像中 nginx 用户 UID。

privileged 配置禁止从容器内访问宿主机任何设备（默认配置），allowPrivilegeEscalation 配置禁止子进程获取比主进程更高的权限。
根据官方文档的 yaml 文件，如果不希望使用 root 用户启动，推荐配置 privileged 和 allowPrivilegeEscalation 都为 false。

参考 Kubernetes 官方文档 “[Pod 安全策略](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#example-policies)” 一节中的示例。