# 设置环境

设置您的环境以构建operator

## 在 Linux 上安装 opeartor-sdk 的先决条件

要在 Linux 和 Windows 上开发基于 Golang 的Operator，您需要安装以下内容：

- [Git](https://git-scm.com/downloads)
- [Go](https://golang.org/dl/) v1.10+
- [Docker v17.03](https://docs.docker.com/get-docker/) +
- 已安装 OpenShift CLI (oc) v4.1+
- [Kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/)
- OpenShift CLI（**如果您计划部署到 OpenShift 集群**）[oc](https://docs.openshift.com/container-platform/4.5/cli_reference/openshift_cli/getting-started-cli.html)
- 要使用 OpenShift 集群，我们建议您使用**4.6+ 版本**

此外，您还需要：

- 访问 Kubernetes v1.11.3+ 集群（v1.16.0+，如果使用 apiextensions.k8s.io/v1 CRDs）。请参阅[CodeReady Containers](https://code-ready.github.io/crc/#installing-codeready-containers_gsg)以免费访问集群。
- 访问容器镜像仓库，例如[Quay.io](https://quay.io/)或[DockerHub](https://hub.docker.com/)

## 步骤

1. 安装podman-docker
2. 安装 Ansible 和模块依赖项
3. 安装 Operator SDK
4. 安装 OC
5. 安装 Go

### 1.安装podman-docker

```sh
sudo dnf -y install docker
sudo systemctl start docker
sudo systemctl enable docker
```

### 2.安装 Ansible 和模块依赖项

#### 安装pip3

使用dnf命令安装包pip:

```shell
dnf install python3-pip
```

通过查询版本号确认安装::

```shell
pip3 --version
```

pip 9.0.3 from /usr/lib/python3.6/site-packages (python 3.6)

#### 安装 ansible

`sudo dnf -y install ansible`

Ansible [runner](https://ansible-runner.readthedocs.io/en/stable/)和[http runner](https://github.com/ansible/ansible-runner-http)用于运行本地版本的 operator。这**对于开发和测试非常有用**。

`pip3 install --user ansible-runner`

`pip3 install --user ansible-runner-http`

#### 安装所需的python模块

`pip3 install --user requests`

`pip3 install --user openshift`

#### 安装Make

`sudo dnf install -y make`

### 3.安装Operator SDK

#### 安装适用于 Linux 或 Windows 的 Operator SDK

确保已经安装 Go v1.19.1和Curl

- 对于 Linux 或 Windows，从GitHub下载安装 Operator SDK（v1.23.0 ）。

#### 下载发布二进制文件

设置平台信息：

```sh
export ARCH=$(case $(uname -m) in x86_64) echo -n amd64 ;; aarch64) echo -n arm64 ;; *) echo -n $(uname -m) ;; esac)
export OS=$(uname | awk '{print tolower($0)}')
```

下载适用于您平台的二进制文件：

```sh
export OPERATOR_SDK_DL_URL=https://github.com/operator-framework/operator-sdk/releases/download/v1.23.0
curl -LO ${OPERATOR_SDK_DL_URL}/operator-sdk_${OS}_${ARCH}
curl -LO ${OPERATOR_SDK_DL_URL}/ansible-operator_${OS}_${ARCH}
curl -LO ${OPERATOR_SDK_DL_URL}/helm-operator_${OS}_${ARCH}
```

#### 验证下载的二进制文件

从以下位置导入 operator-sdk 发布 GPG 密钥`keyserver.ubuntu.com`：

```sh
gpg --keyserver keyserver.ubuntu.com --recv-keys 052996E2A20B5C7E
```

下载校验和文件及其签名，然后验证签名：

```sh
curl -LO ${OPERATOR_SDK_DL_URL}/checksums.txt
curl -LO ${OPERATOR_SDK_DL_URL}/checksums.txt.asc
gpg -u "Operator SDK (release) <cncf-operator-sdk@cncf.io>" --verify checksums.txt.asc
```

您应该会看到类似于以下内容的内容：

```console
gpg: assuming signed data in 'checksums.txt'
gpg: Signature made Fri 30 Oct 2020 12:15:15 PM PDT
gpg:                using RSA key ADE83605E945FA5A1BD8639C59E5B47624962185
gpg: Good signature from "Operator SDK (release) <cncf-operator-sdk@cncf.io>" [ultimate]
```

确保校验和匹配：

```sh
grep operator-sdk_${OS}_${ARCH} checksums.txt | sha256sum -c -
grep ansible-operator_${OS}_${ARCH} checksums.txt | sha256sum -c -
grep helm-operator_${OS}_${ARCH} checksums.txt | sha256sum -c -
```

您应该会看到类似于以下内容的内容：

```console
operator-sdk_linux_amd64: OK
ansible-operator_linux_amd64: OK
helm-operator_linux_amd64: OK
```

#### 在您的 PATH 中安装发布二进制文件

```sh
chmod +x operator-sdk_${OS}_${ARCH} && sudo mv operator-sdk_${OS}_${ARCH} /usr/local/bin/operator-sdk
chmod +x ansible-operator_${OS}_${ARCH} && sudo mv ansible-operator_${OS}_${ARCH} /usr/local/bin/ansible-operator
chmod +x helm-operator_${OS}_${ARCH} && sudo mv helm-operator_${OS}_${ARCH} /usr/local/bin/helm-operator
```

#### 测试 Operator SDK 的环境

1. 在您选择的终端中运行以下命令：
   
   ```
   $ operator-sdk version
   ```

2. 你应该看到这样的输出：
   
   ```
   operator-sdk version: "v1.23.0", commit: "1eaeb5adb56be05fe8cc6dd70517e441696846a4", kubernetes version: "1.24.2", go version: "go1.18.5", GOOS: "linux", GOARCH: "amd64"
   ```

3. 确保安装了 kustomize。通过下载预编译的[二进制文件]([Releases · kubernetes-sigs/kustomize · GitHub](https://github.com/kubernetes-sigs/kustomize/releases))来安装 Kustomize：

```
$ curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
$ chmod +x kustomize && sudo mv kustomize /usr/local/bin/kustomize
$ kustomize version
```

你应该看到这样的输出：

```
{Version:kustomize/v4.5.7 GitCommit:56d82a8378dfc8dc3b3b1085e5a6e67b82966bd7 BuildDate:2022-08-02T16:35:54Z GoOs:linux GoArch:amd64}
```

### 4.安装oc（环境已经预装）

如果您计划使用 OpenShift 集群，请使用[OpenShift 文档](https://docs.openshift.com/container-platform/latest/cli_reference/openshift_cli/getting-started-cli.html)`oc`中的说明安装 OpenShift CLI ( OC) 。或者，您可以按照[文章](https://developers.redhat.com/openshift/command-line-tools)所述通过 OpenShift Web 控制台安装 CLI 。

通过发出以下命令来测试 CLI 以查看 CLI 的版本：

```
$ oc version
Client Version: 4.6.20
Server Version: 4.6.20
Kubernetes Version: v1.19.0+2f3101c
```

### 5.安装go

下载安装包：

```shell
wget https://go.dev/dl/go1.19.1.linux-amd64.tar.gz
```

通过删除/usr/local/ Go文件夹(如果它存在的话)来删除之前的Go安装，然后将你刚刚下载的存档解压到/usr/local，在/usr/local/ Go中创建一个新的Go版本:

```shell
$ rm -rf /usr/local/go && tar -C /usr/local -xzf go1.19.1.linux-amd64.tar.gz
```

(您可能需要以root用户或通过sudo运行该命令)。  不要将存档解tar到现有的/usr/local/go目录中，这将会引起安装问题。

将/usr/local/go/bin添加到PATH环境变量中。您可以通过在$HOME/中添加以下行来实现这一点。HOME/.profile或/etc/profile(用于系统范围的安装):

```
export PATH=$PATH:/usr/local/go/bin
```

注意:对配置文件所作的更改可能要到下次登录计算机时才生效。要立即应用更改，只需直接运行shell命令，或使用源$HOME/.profile等命令从概要文件执行它们。

通过打开命令提示符并输入以下命令，验证您已经安装了Go:

```shell
$ go version
```

确认该命令打印已安装的Go版本。

## 结论

如果您已安装这些先决条件，则您的环境已设置为开发operator并将其部署到 OpenShift（或 Kubernetes）集群。
