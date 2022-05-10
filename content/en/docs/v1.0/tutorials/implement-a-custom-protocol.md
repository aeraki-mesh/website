---
title: How to implement a custom protocol
description: 
weight: 1000
---

MetaProtocol Proxy provides a well-designed mechanism for protocol extensions, which allows us to quickly implement a seven-layer proxy for a custom protocol.


MetaProtocol Proxy already implements most of the features needed for a Layer 7 proxy, including load balancing, fusion, RDS dynamic routing, local flow restriction, fully restricted flow, Header modification, request Metrics collection, etc., more rich features are still under development. Therefore, developing based on MetaProtocol greatly simplifies the work of implementing a Layer 7 web proxy, and we only need to implement a small amount of code for coding and decoding to get a Layer 7 proxy with a custom protocol. In general, only a few hundred lines of code are needed to implement a custom protocol.

After implementing a data proxy based on MetaProtocol, without any control coding, we can manage services using custom protocols in the Isito service grid through Aeraki, providing service governance capabilities such as traffic splitting, gray release, traffic mirroring, monitoring charts, etc. for services. This is because Aeraki recognizes any Layer 7 protocol based on MetaProtocol and provides user-friendly traffic rules at the control plane. aeraki translates these traffic rules into MetaProtocol configurations and then dispatches them to the data plane.

Another benefit of using MetaProtocol to develop custom protocols for the service grid is that the solution is completely non-intrusive to upstream open source projects such as Istio, Envoy, and MetaProtocol Proxy itself. It allows rapid iterations to follow the upstream projects and take advantage of the new capabilities provided by new versions of the upstream projects without maintaining their fork branches.

## Implementation of codec interface
Aeraki provides an example application protocol extension [awesomerpc](https://github.com/aeraki-mesh/meta-protocol-awesomerpc). The example contains a program framework that you can modify to write your private protocol.

```bash
git clone https://github.com/aeraki-mesh/meta-protocol-awesomerpc.git my-protocol-proxy
```

We need focus on `awesomerpc_codec.h` and `awesomerpc_codec.cc` in the `src/application_protocols/awesomerpc/` directory. These two files define the interface and implementation code for the codec of the application protocol.

The definition of `awesomerpc_codec.h` is as follows, you can see that the application protocol codec is extended from `MetaProtocolProxy::Codec`. To implement the application protocol, you only need to implement the `decode`, `encode` and `onError` methods.

```c++
/**
 * Codec for Awesomerpc protocol.
 */
class AwesomerpcCodec : public MetaProtocolProxy::Codec,
                  public Logger::Loggable<Logger::Id::misc> {
public:
  AwesomerpcCodec() {};
  ~AwesomerpcCodec() override = default;

  //For protocol decoding, the buffer needs to be parsed and populated with Metadata, which will be used for MetaProtocol Proxy filters, such as flow restriction and matching conditions for routes.
  MetaProtocolProxy::DecodeStatus decode(Buffer::Instance& buffer,
                                         MetaProtocolProxy::Metadata& metadata) override;

  //Protocol encoding, the request or response packet can be modified according to Mutation, such as adding, deleting or modifying the header, which needs to be written back to the buffer after modification
  void encode(const MetaProtocolProxy::Metadata& metadata,
              const MetaProtocolProxy::Mutation& mutation, Buffer::Instance& buffer) override;

  //the framework uses err code to return error messages to the client, such as route not found or connection creation failure, etc., the encoded data needs to be written to the buffer
  void onError(const MetaProtocolProxy::Metadata& metadata, const MetaProtocolProxy::Error& error,
               Buffer::Instance& buffer) override;

...
```

You can also refer to the [dubbo](https://github.com/aeraki-mesh/meta-protocol-proxy/tree/master/src/application_protocols/dubbo) and [thrift](https://github.com/aeraki-mesh/meta-protocol-proxy/tree/master/src/application_protocols/thrift) implementation when writing the codec.

## Configuration WORKSPACE

Configure the metaProtocol, envoy and Istio-Proxy dependencies in the WORKSPACE file in the root directory.

You need to configure the git commit of metaProtocol in WORKSPACE, and the dependencies of envoy and Istio-Proxy refer to the configuration in WORKSPACE in the commit of metaProtocol. See [Version compatibility between Aeraki and MetaProtocol and Istio](https://github.com/aeraki-mesh/website/blob/86dcc4d3fdebde9cec3f428dc7f2aca2b73713f9/content/en/docs/v1.0/install.md) for version dependencies.

```Starlark

# Get the dependency version information of envoy from the meta_protocol_proxy code
ENVOY_SHA = "98c1c9e9a40804b93b074badad1cdf284b47d58b"
ENVOY_SHA256 = "4365a4c09b9a8b3c4ae34d75991fcd046f3e19d53d95dfd5c89209c30be94fe6"
......

# Get the istio_proxy dependency version information from the meta_protocol_proxy code
http_archive(
    name = "io_istio_proxy",
    strip_prefix = "proxy-1.10.0",
    sha256 = "19d13bc4859dc8422b91fc28b0d8d12a34848393583eedfb08af971c658e7ffb",
    url = "https://github.com/istio/proxy/archive/refs/tags/1.10.0.tar.gz",   
)
...... 

# set metaProtocol git commit
git_repository(
  name = "meta_protocol_proxy",
  remote = "https://github.com/aeraki-mesh/meta-protocol-proxy.git",
  commit = "5ae1d11",  
)
```

## Compilation

It is recommended to use Ubuntu 18.04 as the build environment.

Install the required software for compiling:

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

Build binary:

```bash
./build.sh
```

## Define a  ApplicationProtocol

To identify custom protocols in Istio, you need to create an Aeraki's ApplicationProtocol CRD resource.

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

## Protocol Selection

Aeraki identifies MetaProtocol-based application protocols by the service prefix, and the port name of the service must follow this naming convention: tcp-metaprotocol-{application protocol}-xxx.
Service definitions can be K8s service or Istio ServiceEntry.

The following ServiceEntry defines a dubbo service:

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

The following Service defines a Thrift service:

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

> Note: The prefix of the port definition needs to contain "tcp", this is because Istio will treat the service as a tcp service. Aeraki will recognize the application protocol that follows and process it accordingly at Layer 7.
