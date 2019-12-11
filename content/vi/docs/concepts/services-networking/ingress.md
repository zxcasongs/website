---
reviewers:
- huynguyennovem
title: Ingress
content_template: templates/concept
weight: 40
---

{{% capture overview %}}
{{< feature-state for_k8s_version="v1.1" state="beta" >}}
{{< glossary_definition term_id="ingress" length="all" >}}
{{% /capture %}}

{{% capture body %}}

## Thuật ngữ

Để cho rõ ràng, hướng dẫn này định nghĩa những thuật ngữ sau:

Node
: Một node worker trong Kubernetes, một phần của cluster.

Cluster
: Một tập hợp các Node chạy những ứng dụng được đóng gói (containerized applications) được quản lý bởi Kubernetes. Trong ví dụ này và trong hầu hết các deployment phổ biến của Kubernetes, các node trong cluster không là một phần của Internet.

Edge router
: Một router thực thi firewall policy cho cluster. Đây có thể là một gateway được quản lý bởi một cloud provider hoặc một phần cứng vật lý.

Cluster network
: Một tập các links, logic hoặc vật lý, giúp giao tiếp dễ dàng trong một cluster theo [mô hình mạng](/docs/concepts/cluster-administration/networking/) Kubernetes. 

Service
: Một Kubernetes {{< glossary_tooltip term_id="service" >}} xác định một tập các Pods sử dụng {{< glossary_tooltip text="label" term_id="label" >}} selectors. Trừ khi được đề cập theo một cách khác, Services được coi như là có các virtual IPs mà chỉ có thể định tuyến trong cluster network.

## Ingress là gì?

Ingress exposes các HTTP và HTTPS routes từ bên ngoài cluster vào
{{< link text="services" url="/docs/concepts/services-networking/service/" >}} bên trong cluster.
Việc định tuyến các traffic được kiểm soát bởi các rule được xác định trong Ingress resource.

```none
    internet
        |
   [ Ingress ]
   --|-----|--
   [ Services ]
```

Một Ingress có thể được cấu hình để cung cấp cho các Service các URL có thể truy cập từ bên ngoài, lưu lượng cân bằng tải, terminate SSL / TLS, và cung cấp dịch vụ lưu trữ ảo dựa theo tên (name based virtual hosting). Một [Ingress controller](/docs/concepts/services-networking/ingress-controllers) chịu trách nhiệm hoàn thành Ingress, thường là với một bộ cân bằng tải, mặc dù nó cũng có thể cấu hình edge router hoặc các frontends bổ sung để giúp xử lý lưu lượng.

Một Ingress không expose các ports hoặc các giao thức một cách tùy ý. Việc expose các service khác với HTTP và HTTPS tới internet thường
sử dụng loại service: [Service.Type=NodePort](/docs/concepts/services-networking/service/#nodeport) hoặc
[Service.Type=LoadBalancer](/docs/concepts/services-networking/service/#loadbalancer).

## Prerequisites

Bạn phải có một [ingress controller](/docs/concepts/services-networking/ingress-controllers) để đáp ứng một Ingress. Việc chỉ tạo một Ingress resource sẽ không có hiệu quả.

Bạn có thể cần triển khai một Ingress controller như [ingress-nginx](https://kubernetes.github.io/ingress-nginx/deploy/). Bạn có thể chọn từ một số
[Ingress controllers](/docs/concepts/services-networking/ingress-controllers).

Lý tưởng nhất, tất cả các Ingress controllers nên phù hợp với các specs tham chiếu. Trong thực tế, các Ingress
controllers khác nhau sẽ hoạt động khác nhau đôi chút.

{{< note >}}
Hãy chắc chắn rằng bạn xem lại tài liệu về Ingress controller của bạn để hiểu được sự thận trọng khi chọn nó.
{{< /note >}}

## The Ingress Resource

Một ví dụ Ingress resource đơn giản:

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /testpath
        backend:
          serviceName: test
          servicePort: 80
```

 Cũng như các resources khác của Kubernetes, một Ingress cần các trường `apiVersion`, `kind`, và `metadata`.
 Để biết thêm về cách làm việc với các file cấu hình, xem [triển khai ứng dụng](/docs/tasks/run-application/run-stateless-application-deployment/), [cấu hình containers](/docs/tasks/configure-pod-container/configure-pod-configmap/), [quản lý resources](/docs/concepts/cluster-administration/manage-deployment/).
 Ingress thường xuyên sử dụng các annotation để cấu hình một số tùy chọn phụ thuộc vào Ingress controller, một ví dụ
 là [rewrite-target annotation](https://github.com/kubernetes/ingress-nginx/blob/master/docs/examples/rewrite/README.md).
 Các [Ingress controller](/docs/concepts/services-networking/ingress-controllers) khác nhau hỗ trợ các annotation khác nhau. Xem lại tài liệu cho
 sự lựa chọn Ingress controller của bạn để biết annotations nào được hỗ trợ.

Ingress [spec](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status)
có tất cả thông tin cần thiết để cấu hình một bộ cân bằng tải (load balancer) hoặc proxy server. Quan trọng nhất, nó
chứa một danh sách các rules phù hợp với tất cả các incoming requests. Ingress resource chỉ hỗ trợ các rule
cho việc điều hướng HTTP traffic.

### Ingress rules

Mỗi một HTTP rule chứa những thông tin sau:

* Một host tùy chọn. Trong ví dụ này, không có host nào được chỉ định, vì vậy rule áp dụng cho tất cả các
  HTTP traffic gửi đến thông qua địa chỉ IP được chỉ định. Nếu một host được cung cấp (ví dụ,
  foo.bar.com), các rules chỉ áp dụng cho host đó.
* Một danh sách các path (ví dụ, `/testpath`), mỗi trong số đó có một backend được xác định với `serviceName`
  và `servicePort`. Cả host và path phải khớp với nội dung của một incoming request trước khi
  bộ cân bằng tải điều hướng lưu lượng (traffic) đến Service được tham chiếu.
* Một backend là sự kết hợp giữa Service và port names như được mô tả trong
  [Service doc](/docs/concepts/services-networking/service/). HTTP (và HTTPS) requests đến
  Ingress khớp với host và path của rule được gửi đến backend đã được liệt kê.

Một backend mặc định thường được cấu hình trong một Ingress controller để phục vụ mọi request không
khớp với một path trong spec.

### Backend mặc định

Một Ingress không có rules sẽ gửi tất cả traffic đến một backend mặc định duy nhất. Backend
mặc định thường là một tùy chọn cấu hình của [Ingress controller](/docs/concepts/services-networking/ingress-controllers) và nó không được chỉ định trong Ingress resources.

Nếu khôn có host hoặc path nào khớp với HTTP request trong các đối tượng Ingress, lưu lượng (traffic)
được định tuyến đến backend mặc định.

## Types of Ingress

### Single Service Ingress

There are existing Kubernetes concepts that allow you to expose a single Service
(see [alternatives](#alternatives)). You can also do this with an Ingress by specifying a
*default backend* with no rules.

{{< codenew file="service/networking/ingress.yaml" >}}

If you create it using `kubectl apply -f` you should be able to view the state
of the Ingress you just added:

```shell
kubectl get ingress test-ingress
```

```
NAME           HOSTS     ADDRESS           PORTS     AGE
test-ingress   *         107.178.254.228   80        59s
```

Where `107.178.254.228` is the IP allocated by the Ingress controller to satisfy
this Ingress.

{{< note >}}
Ingress controllers and load balancers may take a minute or two to allocate an IP address.
Until that time, you often see the address listed as `<pending>`.
{{< /note >}}

### Simple fanout

A fanout configuration routes traffic from a single IP address to more than one Service,
based on the HTTP URI being requested. An Ingress allows you to keep the number of load balancers
down to a minimum. For example, a setup like:

```none
foo.bar.com -> 178.91.123.132 -> / foo    service1:4200
                                 / bar    service2:8080
```

would require an Ingress such as:

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: simple-fanout-example
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        backend:
          serviceName: service1
          servicePort: 4200
      - path: /bar
        backend:
          serviceName: service2
          servicePort: 8080
```

When you create the Ingress with `kubectl apply -f`:

```shell
kubectl describe ingress simple-fanout-example
```

```
Name:             simple-fanout-example
Namespace:        default
Address:          178.91.123.132
Default backend:  default-http-backend:80 (10.8.2.3:8080)
Rules:
  Host         Path  Backends
  ----         ----  --------
  foo.bar.com
               /foo   service1:4200 (10.8.0.90:4200)
               /bar   service2:8080 (10.8.0.91:8080)
Annotations:
  nginx.ingress.kubernetes.io/rewrite-target:  /
Events:
  Type     Reason  Age                From                     Message
  ----     ------  ----               ----                     -------
  Normal   ADD     22s                loadbalancer-controller  default/test
```

The Ingress controller provisions an implementation-specific load balancer
that satisfies the Ingress, as long as the Services (`service1`, `service2`) exist.
When it has done so, you can see the address of the load balancer at the
Address field.

{{< note >}}
Depending on the [Ingress controller](/docs/concepts/services-networking/ingress-controllers)
you are using, you may need to create a default-http-backend
[Service](/docs/concepts/services-networking/service/).
{{< /note >}}

### Name based virtual hosting

Name-based virtual hosts support routing HTTP traffic to multiple host names at the same IP address.

```none
foo.bar.com --|                 |-> foo.bar.com service1:80
              | 178.91.123.132  |
bar.foo.com --|                 |-> bar.foo.com service2:80
```

The following Ingress tells the backing load balancer to route requests based on
the [Host header](https://tools.ietf.org/html/rfc7230#section-5.4).

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: name-virtual-host-ingress
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - backend:
          serviceName: service1
          servicePort: 80
  - host: bar.foo.com
    http:
      paths:
      - backend:
          serviceName: service2
          servicePort: 80
```

If you create an Ingress resource without any hosts defined in the rules, then any
web traffic to the IP address of your Ingress controller can be matched without a name based
virtual host being required.

For example, the following Ingress resource will route traffic
requested for `first.bar.com` to `service1`, `second.foo.com` to `service2`, and any traffic
to the IP address without a hostname defined in request (that is, without a request header being
presented) to `service3`.

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: name-virtual-host-ingress
spec:
  rules:
  - host: first.bar.com
    http:
      paths:
      - backend:
          serviceName: service1
          servicePort: 80
  - host: second.foo.com
    http:
      paths:
      - backend:
          serviceName: service2
          servicePort: 80
  - http:
      paths:
      - backend:
          serviceName: service3
          servicePort: 80
```

### TLS

You can secure an Ingress by specifying a {{< glossary_tooltip term_id="secret" >}}
that contains a TLS private key and certificate. Currently the Ingress only
supports a single TLS port, 443, and assumes TLS termination. If the TLS
configuration section in an Ingress specifies different hosts, they are
multiplexed on the same port according to the hostname specified through the
SNI TLS extension (provided the Ingress controller supports SNI). The TLS secret
must contain keys named `tls.crt` and `tls.key` that contain the certificate
and private key to use for TLS. For example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: testsecret-tls
  namespace: default
data:
  tls.crt: base64 encoded cert
  tls.key: base64 encoded key
type: kubernetes.io/tls
```

Referencing this secret in an Ingress tells the Ingress controller to
secure the channel from the client to the load balancer using TLS. You need to make
sure the TLS secret you created came from a certificate that contains a Common
Name (CN), also known as a Fully Qualified Domain Name (FQDN) for `sslexample.foo.com`.

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: tls-example-ingress
spec:
  tls:
  - hosts:
    - sslexample.foo.com
    secretName: testsecret-tls
  rules:
    - host: sslexample.foo.com
      http:
        paths:
        - path: /
          backend:
            serviceName: service1
            servicePort: 80
```

{{< note >}}
There is a gap between TLS features supported by various Ingress
controllers. Please refer to documentation on
[nginx](https://git.k8s.io/ingress-nginx/README.md#https),
[GCE](https://git.k8s.io/ingress-gce/README.md#frontend-https), or any other
platform specific Ingress controller to understand how TLS works in your environment.
{{< /note >}}

### Loadbalancing

An Ingress controller is bootstrapped with some load balancing policy settings
that it applies to all Ingress, such as the load balancing algorithm, backend
weight scheme, and others. More advanced load balancing concepts
(e.g. persistent sessions, dynamic weights) are not yet exposed through the
Ingress. You can instead get these features through the load balancer used for
a Service.

It's also worth noting that even though health checks are not exposed directly
through the Ingress, there exist parallel concepts in Kubernetes such as
[readiness probes](/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)
that allow you to achieve the same end result. Please review the controller
specific documentation to see how they handle health checks (
[nginx](https://git.k8s.io/ingress-nginx/README.md),
[GCE](https://git.k8s.io/ingress-gce/README.md#health-checks)).

## Updating an Ingress

To update an existing Ingress to add a new Host, you can update it by editing the resource:

```shell
kubectl describe ingress test
```

```
Name:             test
Namespace:        default
Address:          178.91.123.132
Default backend:  default-http-backend:80 (10.8.2.3:8080)
Rules:
  Host         Path  Backends
  ----         ----  --------
  foo.bar.com
               /foo   service1:80 (10.8.0.90:80)
Annotations:
  nginx.ingress.kubernetes.io/rewrite-target:  /
Events:
  Type     Reason  Age                From                     Message
  ----     ------  ----               ----                     -------
  Normal   ADD     35s                loadbalancer-controller  default/test
```

```shell
kubectl edit ingress test
```

This pops up an editor with the existing configuration in YAML format.
Modify it to include the new Host:

```yaml
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - backend:
          serviceName: service1
          servicePort: 80
        path: /foo
  - host: bar.baz.com
    http:
      paths:
      - backend:
          serviceName: service2
          servicePort: 80
        path: /foo
..
```

After you save your changes, kubectl updates the resource in the API server, which tells the
Ingress controller to reconfigure the load balancer.

Verify this:

```shell
kubectl describe ingress test
```

```
Name:             test
Namespace:        default
Address:          178.91.123.132
Default backend:  default-http-backend:80 (10.8.2.3:8080)
Rules:
  Host         Path  Backends
  ----         ----  --------
  foo.bar.com
               /foo   service1:80 (10.8.0.90:80)
  bar.baz.com
               /foo   service2:80 (10.8.0.91:80)
Annotations:
  nginx.ingress.kubernetes.io/rewrite-target:  /
Events:
  Type     Reason  Age                From                     Message
  ----     ------  ----               ----                     -------
  Normal   ADD     45s                loadbalancer-controller  default/test
```

You can achieve the same outcome by invoking `kubectl replace -f` on a modified Ingress YAML file.

## Failing across availability zones

Techniques for spreading traffic across failure domains differs between cloud providers.
Please check the documentation of the relevant [Ingress controller](/docs/concepts/services-networking/ingress-controllers) for details. You can also refer to the [federation documentation](https://github.com/kubernetes-sigs/federation-v2)
for details on deploying Ingress in a federated cluster.

## Future Work

Track [SIG Network](https://github.com/kubernetes/community/tree/master/sig-network)
for more details on the evolution of Ingress and related resources. You may also track the
[Ingress repository](https://github.com/kubernetes/ingress/tree/master) for more details on the
evolution of various Ingress controllers.

## Alternatives

You can expose a Service in multiple ways that don't directly involve the Ingress resource:

* Use [Service.Type=LoadBalancer](/docs/concepts/services-networking/service/#loadbalancer)
* Use [Service.Type=NodePort](/docs/concepts/services-networking/service/#nodeport)

{{% /capture %}}

{{% capture whatsnext %}}
* Learn about [ingress controllers](/docs/concepts/services-networking/ingress-controllers/)
* [Set up Ingress on Minikube with the NGINX Controller](/docs/tasks/access-application-cluster/ingress-minikube)
{{% /capture %}}
