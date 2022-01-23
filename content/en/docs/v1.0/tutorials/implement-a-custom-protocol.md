---
title: 如何开发一个自定义协议
description: 
weight: 1000
---

MetaProtocol Proxy 提供了一个良好的协议扩展机制，使得我们可以基于 MetaProtocol Proxy 快速实现一个自定义协议的七层代理。

由于 MetaProtocol Proxy 已经实现了一个七层协议代理所需的大部分功能，包括七层负载均衡、RDS 动态路由、本地限流、全局限流、请求 Metrics 收集等，更多丰富的功能还在持续开发中。因此基于 MetaProtocol 进行开发极大简化了实现一个七层网络代理的工作，我们只需要实现编解码的少量代码，即可得到一个自定义协议的七层代理。一般来说，实现一个自定义协议只需要数百行代码。

在基于 MetaProtocol 实现数据面代理后，无需任何控制面编码，我们就可以通过 Aeraki 在 Isito 服务网格中对使用自定义协议的服务进行管理，为服务提供流量拆分、灰度发布、流量镜像、监控图表等服务治理能力。这是因为 Aeraki 可以识别基于 MetaProtocol 的任何七层协议，并在控制面提供用户友好的流量规则。Aeraki 会将这些流量规则翻译为 MetaProtocol 的配置后下发到数据面。

除了快速开发，节省工作量之外，采用 MetaProtocol 为服务网格开发自定义协议的另一个好处是该方案对 Istio，Envoy，以及 MetaProtocol Proxy 自身等上游开源项目是完全无侵入的，可以跟随上游项目进行快速迭代，充分利用上游项目新版本提供的新增能力，无需维护自有的 fork 分支。

## 实现编解码接口

Aeraki 提供了一个应用协议扩展的示例 [awesomerpc](https://github.com/aeraki-mesh/meta-protocol-awesomerpc)。示例中包含了实现自定义协议的程序框架，可以该示例为基础进行修改，编写你自己的私有协议。

```bash
git clone https://github.com/aeraki-mesh/meta-protocol-awesomerpc.git my-protocol-proxy
```

我们主要需要关注的是 `src/application_protocols/awesomerpc/` 目录下的 `awesomerpc_codec.h` 和 `awesomerpc_codec.cc` 。这两个文件定义了应用协议的 codec 的接口和实现代码。

`awesomerpc_codec.h` 的定义如下，可以看到应用协议的 codec 继承于 `MetaProtocolProxy::Codec`。实现应用协议只需要实现 `decode`，`encode` 和 `onError` 三个方法即可。

```c++
/**
 * Codec for Awesomerpc protocol.
 */
class AwesomerpcCodec : public MetaProtocolProxy::Codec,
                  public Logger::Loggable<Logger::Id::misc> {
public:
  AwesomerpcCodec() {};
  ~AwesomerpcCodec() override = default;

  //协议解码，需要解析 buffer 并填充 Metadata， Metadata 将被用于 MetaProtocol Proxy 的 filter，例如限流，路由的匹配条件
  MetaProtocolProxy::DecodeStatus decode(Buffer::Instance& buffer,
                                         MetaProtocolProxy::Metadata& metadata) override;

  //协议编码，可以根据 Mutation 对请求或者响应数据包进行修改，例如增加、删除或者修改 header，修改后需要回写到 buffer 中
  void encode(const MetaProtocolProxy::Metadata& metadata,
              const MetaProtocolProxy::Mutation& mutation, Buffer::Instance& buffer) override;

  //错误编码，用于框架向客户端返回错误信息，例如未找到路由或者连接创建失败等，编码的数据需要写入到 buffer 中
  void onError(const MetaProtocolProxy::Metadata& metadata, const MetaProtocolProxy::Error& error,
               Buffer::Instance& buffer) override;

...
```

在编写 codec 时也可以参考 [dubbo](https://github.com/aeraki-mesh/meta-protocol-proxy/tree/master/src/application_protocols/dubbo) 和 [thrift](https://github.com/aeraki-mesh/meta-protocol-proxy/tree/master/src/application_protocols/thrift) 的 codec 实现。

## 配置 WORKSPACE

在根目录的 WORKSPACE 文件中配置 metaProtocol, envoy 和 Istio-Proxy 的依赖。

需要在 WORKSPACE 中配置 metaProtocol 的 git commit，envoy 和 Istio-Proxy 的依赖参考 metaProtocol 该 commit 中的 WORKSPACE 中的配置。版本依赖关系参见 [Aeraki 和 MetaProtocol 以及 Istio 的版本兼容性](/zh/docs/v1.0/install/#aeraki-%E5%92%8C-metaprotocol-%E4%BB%A5%E5%8F%8A-istio-%E7%9A%84%E7%89%88%E6%9C%AC%E5%85%BC%E5%AE%B9%E6%80%A7)。

```Starlark

# 从 meta_protocol_proxy 的代码中获取 envoy 的依赖版本信息
ENVOY_SHA = "98c1c9e9a40804b93b074badad1cdf284b47d58b"
ENVOY_SHA256 = "4365a4c09b9a8b3c4ae34d75991fcd046f3e19d53d95dfd5c89209c30be94fe6"
......

# 从 meta_protocol_proxy 的代码中获取 istio_proxy 的依赖版本信息
http_archive(
    name = "io_istio_proxy",
    strip_prefix = "proxy-1.10.0",
    sha256 = "19d13bc4859dc8422b91fc28b0d8d12a34848393583eedfb08af971c658e7ffb",
    url = "https://github.com/istio/proxy/archive/refs/tags/1.10.0.tar.gz",   
)

...... 

# 设置 metaProtocol 的 git commit
git_repository(
  name = "meta_protocol_proxy",
  remote = "https://github.com/aeraki-mesh/meta-protocol-proxy.git",
  commit = "5ae1d11",  
)
```

## 编译

建议使用 Ubuntu 18.04 作为编译环境。

安装编译所需软件：

```bash
sudo wget -O /usr/local/bin/bazel https://github.com/bazelbuild/bazelisk/releases/latest/download/bazelisk-linux-$([ $(uname -m) = "aarch64" ] && echo "arm64" || echo "amd64")
sudo chmod +x /usr/local/bin/bazel

sudo apt-get install \
autoconf \
automake \
cmake \
curl \
libtool \
make \
ninja-build \
patch \
python3-pip \
unzip \
virtualenv

sudo add-apt-repository ppa:ubuntu-toolchain-r/test
sudo apt update
sudo apt-get install llvm-10 lldb-10 llvm-10-dev libllvm10 llvm-10-runtime clang-10 clang++-10 lld-10 gcc-10 g++-10
```

构建二进制：

```bash
./build.sh
```

## 定义一个 ApplicationProtocol

要在 Istio 中识别自定义协议，需要创建一个 Aeraki 的 ApplicationProtocol CRD 资源。

```yaml
apiVersion: metaprotocol.aeraki.io/v1alpha1
kind: ApplicationProtocol
metadata:
  name: my-protocol
  namespace: istio-system
spec:
  protocol: my-protocol
  codec: aeraki.meta_protocol.codec.my_protocol
```

## 协议选择

Aeraki 通过服务的前缀来识别基于 MetaProtocol 的应用协议，服务的端口名必须遵从该命名规则： tcp-metaprotocol-{application protocol}-xxx。
服务定义可以采用 K8s service 或者 Istio ServiceEntry。

下面的 ServiceEntry 定义了一个 dubbo 服务：
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: dubbo-demoservice
  namespace: meta-dubbo
spec:
  hosts:
    - org.apache.dubbo.samples.basic.api.demoservice
  ports:
    - number: 20880
      name: tcp-metaprotocol-dubbo
      protocol: TCP
  workloadSelector:
    labels:
      app: dubbo-sample-provider
  resolution: STATIC
```

下面的 Service 定义了一个 Thrift 服务：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: thrift-sample-server
spec:
  selector:
    app: thrift-sample-server
  ports:
    - name: tcp-metaprotocol-thrift-hello-server
      protocol: TCP
      port: 9090
      targetPort: 9090
```

> 备注：端口定义的前缀需要包含 "tcp"，这是因为 Istio 会将该服务作为 tcp 服务进行处理。 而 Aeraki 则会识别后面的应用协议并进行相应的七层处理。






