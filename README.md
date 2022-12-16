# Log Aggregation in Kubernetes with Traefik Proxy

When deployed as a Kubernetes ingress controller, Traefik can process and route
many thousands of requests without complaint. And yet, for the operations team,
visibility into what's happening behind the scenes is essential. Is the
application healthy? Is it working as intended? Monitoring distributed systems
is one of the core principles of the set of practices known as site reliability
engineering ([SRE](https://sre.google/sre-book/table-of-contents/)).

This first in a series of posts on Traefik and SRE techniques explores how
Traefik's built-in logging features can help to provide needed visibility. When
combined with a set of open-source observability projects like [Open 
Telemetry](https://opentelemetry.io/), [Elastic
Stack](https://www.elastic.co/what-is/elk-stack), [Grafana
Stack](https://grafana.com/products/enterprise/) or their competitors, Traefik
becomes part of a rich set of tools for network log analysis and visualization.

In order to stay simple, everything will be deployed on default namespace and
without any kind of protection on Traefik Dashboard. __On production, you should
use namespace and take care of how this Dashboard should be accessed.__

## Prerequisites

If you'd like to follow along with this tutorial on your own machine, you'll
need a few things first:

1. A Kubernetes cluster running at localhost. In this blog post, we will use
   [kind](https://kind.sigs.k8s.io). You may also use alternatives like
   [k3d](https://k3d.io)
2. Helm v3.9+ [installed](https://helm.sh/docs/using_helm/#installing-helm).
3. The `kubectl` command-line tool is installed and configured to access your
   cluster.

kind requires some config in order to use an IngressController on localhost:
```yaml 
$ cat kind.config
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30000
    hostPort: 80 protocol: TCP
  - containerPort: 30001
    hostPort: 443 protocol: TCP 
```

All config files are in the public GitHub
[repository](https://github.com/traefik-tech-blog/traefik-sre-logging.git) that
accompany this article, so you just have to clone it and create our local
cluster with this network configuration: 

```bash 
$ git clone https://github.com/traefik-tech-blog/traefik-sre-logging/ 
$ cd traefik-sre-logging 
$ kind create cluster --config=kind.config 
$ kubectl cluster-info
$ kubectl get nodes
```

## Set up Traefik

First, we will deploy Traefik using the helm chart and enable logging. We can
configure the Helm Chart deployment with a values file. We’ll start by just enabling
access.logs. By default, they will be on stdout of traefik pod.

```yaml 
$ cat traefik-values-01.yaml 
ports:
  web:
    nodePort: 30000
  websecure:
    nodePort: 30001
logs:
  access:
    enabled: true
ingressRoute:
  dashboard:
    enabled: true
    # You should protect your dashboard if it’s on the internet matchRule:
    matchRule: Host(`dashboard.docker.localhost`)
    entryPoints: ["web"]      
additionalArguments:
- "--ping"
```

The `ports` with `nodePort` is only needed with our kind setup.

Let's deploy it:

```bash
$ helm repo add traefik https://traefik.github.io/charts
$ helm install traefik -f traefik-values traefik/traefik
```

If _dashboard.docker.localhost_ does not resolve, you can check that you have
**myhostname** module enabled in `/etc/nsswitch.conf`:
```bash
$ cat /etc/nsswitch.conf
[...]
hosts:          files mdns4_minimal [NOTFOUND=return] dns myhostname
```

You should be able to access Traefik Dashboard on the configured Host:
![Traefik Dashboard Screenshot](./images/traefik-dashboard.png)

And to see logs on stdout of traefik pods:
```bash
kubectl logs -f -l app.kubernetes.io/name=traefik
```

## Deploy a sample application

We'll use a simple app like **whoami** for this post:

```bash
$ kubectl apply -f apps/
```

It can then be accessed with a browser or with curl:

```bash
$ curl http://whoami.docker.localhost
```

# Configure logging for a collector

If you want to collect logs, you'll need to put those logs into a file and
often you'll also need to convert it from raw to json.

For this, we'll use a local volume (emptyDir). 
One can change the format and set a path for access logs with those values:

```diff
--- traefik-values-01.yaml	2022-12-02 16:14:07.973877899 +0100
+++ traefik-values-02.yaml	2022-12-05 17:18:34.839234310 +0100
@@ -6,6 +6,14 @@
 logs:
   access:
     enabled: true
+    format: json
+    filePath: "/var/log/traefik/access.log"
+deployment:
+  additionalVolumes:
+  - name: logs
+additionalVolumeMounts:
+- name: logs
+  mountPath: /var/log/traefik
 ingressRoute:
   dashboard:
     enabled: true
```

To see the logs, we'll use a sidecar container to tail this file:
```diff
--- traefik-values-02.yaml	2022-12-05 17:18:34.839234310 +0100
+++ traefik-values-03.yaml	2022-12-05 17:21:47.387319123 +0100
@@ -9,6 +9,13 @@
     format: json
     filePath: "/var/log/traefik/access.log"
 deployment:
+  additionalContainers:
+  - name: tail-accesslogs
+    image: busybox
+    args: [ "/bin/sh", "-c", "tail -n+1 -F /var/log/traefik/access.log" ]
+    volumeMounts:
+    - name: logs
+      mountPath: /var/log/traefik
   additionalVolumes:
   - name: logs
 additionalVolumeMounts:
```

Instead of tail, one can of couse use a log collector.

After applying it, we can see logs in JSON format on the tail-accesslogs sidecar:
```bash
$ helm upgrade -i -f traefik-values-03.yaml traefik traefik/traefik
$ kubectl logs -f -l app.kubernetes.io/name=traefik -c tail-accesslogs
```

You should see all current field of ping requests:
```json
{
  "ClientAddr": "10.244.0.1:41388",
  "ClientHost": "10.244.0.1",
  "ClientPort": "41388",
  "ClientUsername": "-",
  "DownstreamContentSize": 2,
  "DownstreamStatus": 200,
  "Duration": 34633,
  "OriginContentSize": 2,
  "OriginDuration": 7325,
  "OriginStatus": 200,
  "Overhead": 27308,
  "RequestAddr": "10.244.0.13:9000",
  "RequestContentSize": 0,
  "RequestCount": 64,
  "RequestHost": "10.244.0.13",
  "RequestMethod": "GET",
  "RequestPath": "/ping",
  "RequestPort": "9000",
  "RequestProtocol": "HTTP/1.1",
  "RequestScheme": "http",
  "RetryAttempts": 0,
  "RouterName": "ping@internal",
  "StartLocal": "2022-12-05T16:25:29.38659907Z",
  "StartUTC": "2022-12-05T16:25:29.38659907Z",
  "entryPointName": "traefik",
  "level": "info",
  "msg": "",
  "time": "2022-12-05T16:25:29Z"
}
```

## Add headers to access.logs

As you can see and as it's
[documented](https://doc.traefik.io/traefik/observability/access-logs/#limiting-the-fieldsincluding-headers),
there is no header by default. 

Let's try to add HTTP Header to those access.logs:
```diff
--- traefik-values-03.yaml	2022-12-05 17:21:47.387319123 +0100
+++ traefik-values-04.yaml	2022-12-05 17:42:44.140244962 +0100
@@ -8,6 +8,9 @@
     enabled: true
     format: json
     filePath: "/var/log/traefik/access.log"
+    fields:
+      headers:
+        defaultmode: keep
 deployment:
   additionalContainers:
   - name: tail-accesslogs
```

We can apply this change:
```bash
$ helm upgrade -i -f traefik-values-04.yaml traefik traefik/traefik
$ kubectl logs -f -l app.kubernetes.io/name=traefik -c tail-accesslogs
```

And see header fields on the sidecar container, prefixed with `request`:
```json
{
  [...],
  "request_Accept": "*/*",
  "request_Connection": "close",
  "request_User-Agent": "kube-probe/1.25",
  "request_X-Forwarded-Host": "10.244.0.16:9000",
  "request_X-Forwarded-Port": "9000",
  "request_X-Forwarded-Proto": "http",
  "request_X-Forwarded-Server": "traefik-b6b999947-wb5lw",
  "request_X-Real-Ip": "10.244.0.1"
}
```

By default, Traefik won't trust X-Forwarded-XXX headers. If you have a Proxy
before your service, you may be interested to add this Proxy in your
[trustedIPs](https://doc.traefik.io/traefik/routing/entrypoints/#forwarded-headers)

Otherwise, the real IP coming from this Proxy won't be in your Traefik access logs.

## Protect sensitive header

Now let's say that we want to:
1. check that `Authorization` header with a token is here, but without writing this sensitive value.
2. Drop Accept and Connection fields

```diff
--- traefik-values-04.yaml	2022-12-05 17:42:44.140244962 +0100
+++ traefik-values-05.yaml	2022-12-05 17:56:34.397503223 +0100
@@ -11,6 +11,10 @@
     fields:
       headers:
         defaultmode: keep
+        names:
+          Accept: drop
+          Connection: drop
+          Authorization: redact
 deployment:
   additionalContainers:
   - name: tail-accesslogs
```

After applying it

```bash
$ helm upgrade -i -f traefik-values-05.yaml traefik traefik/traefik
$ kubectl logs -f -l app.kubernetes.io/name=traefik -c tail-accesslogs
```

And sending a request with Authorization header:
```bash
curl -X GET "http://whoami.docker.localhost" -H "Authorization: Bearer Thiswontbeintheaccesslogs"
```

We can see the new format, with sensitive Authorization header _redacted_:
```json
{
  [...],
  "origin_Accept-Ranges": "bytes",
  "origin_Content-Length": "3124",
  "origin_Content-Security-Policy": "frame-src self https://traefik.io https://*.traefik.io;",
  "origin_Content-Type": "text/html; charset=utf-8",
  "request_Authorization": "REDACTED",
  "request_User-Agent": "curl/7.74.0",
  "request_X-Forwarded-Host": "dashboard.docker.localhost",
  "request_X-Forwarded-Port": "80",
  "request_X-Forwarded-Proto": "http",
  "request_X-Forwarded-Server": "traefik-74d455649f-ndhcj",
  "request_X-Real-Ip": "10.244.0.1"
}
```

And there are also new _origin_ headers sent by Curl with our test.

## TimeZone and Buffering

Now, let's try to change the time zone and use the buffering feature, to gain some performance. 

```diff
--- traefik-values-05.yaml	2022-12-05 17:56:34.397503223 +0100
+++ traefik-values-06.yaml	2022-12-06 09:27:12.536753905 +0100
@@ -8,6 +8,7 @@
     enabled: true
     format: json
     filePath: "/var/log/traefik/access.log"
+    bufferingSize: 1000
     fields:
       headers:
         defaultmode: keep
@@ -15,6 +16,9 @@
           Accept: drop
           Connection: drop
           Authorization: redact
+env:
+- name: TZ
+  value: "Brazil/East"
 deployment:
   additionalContainers:
   - name: tail-accesslogs
```

We can apply it like the other changes

```bash
$ helm upgrade -i -f traefik-values-06.yaml traefik traefik/traefik
$ kubectl logs -f -l app.kubernetes.io/name=traefik -c tail-accesslogs
```

And see the difference on time zone:
```json
{
  [...],
  "StartLocal": "2022-12-06T05:29:15.353148769-03:00",
  "StartUTC": "2022-12-06T08:29:15.353148769Z",
  "time": "2022-12-06T05:29:15-03:00"
}
```

`StartLocal` and `time` are using the `Brazil/East` TimeZone, which is UTC-3. 

For the buffering, we can see on this simple load test that Traefik was able to
handle +16% more traffic.

```bash
$ # Before bufferingSize
$ ab -n 10000 -C 100 http://whoami.docker.localhost/
[...]
Requests per second:    1448.70 [#/sec] (mean)
$ # After applying bufferingSize to 1000
Requests per second:    1687.53 [#/sec] (mean)
```

Of course, it does not mean you will have the same gain on your environment.
It may depends on many aspect and you should test which value is better for
your environment. Don't forget too that it means that with this settings, some
logs can be lost. There's a trade-off between logs integrity and performance.

# Summary 

This simple blog post serves to demonstrate Traefik's comprehensive logging
capabilities.  You can really tailor Traefik logs before collecting them. It's
a powerful tool to understand the health and performance of services running on
Kubernetes clusters.

The other posts of this SRE series cover the other pillars of observability: 
Metrics and Tracing. They can be used with
[OpenTelemetry](https://opentelemetry.io/), with
[Prometheus](https://prometheus.io) and many others.

As usual, if you love Traefik and there are features you'd like to see in
future releases, open a feature request or get in touch on our [community
forums](https://community.traefik.io/). And if you'd like to dig deeper into
how your Traefik instances are operating, check out [Traefik
Hub](https://traefik.io/traefik-hub/), our Cloud Native networking platform.
