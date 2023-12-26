---
draft: true
title: Wildcard Let's Encrypt certificates with Traefik and k3s
date: 2021-12-15T00:07:25+01:00
summary: >
  **tl;dr.** Learn how to use wildcard certificates issued with cert-manager and Let's Encrypt with K3s and its standard ingress controller Traefik v2 with as little effort as possible.
tags:
- traefik
- k3s
- lets encrypt
---

{{< alert >}}
**tl;dr.** Learn how to use wildcard certificates issued with cert-manager and Let's Encrypt with K3s and its standard ingress controller Traefik v2 with as little effort as possible.
{{< /alert >}}

## A brief preface …

For a few years now, I have been running a number of different applications on a Raspberry Pi. From the beginning, Docker-Compose has been my tool of choice to organize that construct. Starting on a Raspberry Pi 2, a Raspberry Pi 4 is now working - and getting bored, while serving web apps in my home network.

Since I work mostly with Kubernetes in my current job, the idea of replacing [Docker-Compose](https://docs.docker.com/compose/) with a Kubernetes "cluster" was natural. Cluster in quotes, since I'm going to leave it at a single Raspberry Pi 4. Whether the effort and complexity of setting up Kubernetes is justified for this simple use case is up to you - thus hereby be warned!

After some playing around, [K3s](https://k3s.io) has become my Kubernetes distribution of choice for use on the Raspberry. Rancher, the developers behind it, promise a distribution with extensive features and the goal of significantly reducing memory requirements so that K3s can also be used on Edge, IoT or ARM hardware in general.

Enough preliminaries, on with the ...

## Problem

A subset of the running applications should also be accessible from the Internet on subdomains of `k3s.nobbs.eu`. To avoid the internal detour over the internet, my home network DNS answers for subdomains of this address with the intranet IP of the Raspberry Pi. Due to the external accessibility, HTTPS is used - both internally and externally - accomplished by using [cert-manager](https://cert-manager.io), [Let's Encrypt](https://letsencrypt.org) and the [DNS01-Challenge](https://letsencrypt.org/docs/challenge-types/#dns-01-challenge) with the domain service of my choice.

The conventional approach would be to add annotations to each `Ingress` object, so that cert-manager gets its own certificate for each of the subdomains - unfortunately, from a privacy point of view, this is not the optimal solution for my use case, since publicly available Certificate Transparency Logs, such as [Crt.sh](https://crt.sh/?q=k3s.nobbs.eu), "leak" all subdomains with Let's Encrypt certificates - whether used internally or externally. Not a major problem, but then again, not everyone needs to be able to see which services I try out or run internally.

A wildcard certificate for `*.k3s.nobbs.eu` remedies the problem, but how do you configure K3s and [Traefik](https://traefik.io/) to make this work? *Here I would like to refer to the post by [Lachlan Mitchell](https://lachlan.io/blog/using-wildcard-certificates-with-traefik-and-k3s) - who describes therein the process for Traefik v1 and first made me think of the implementation in Traefik v2.

## Solution

For simplicity, I will assume that cert-manager has already been installed in the cluster -- if not, the project's [documentation](https://cert-manager.io/docs/installation/) may help. Let's further assume that a Let's Encrypt `ClusterIssuer` object named `letsencrypt-prod` has been configured and is ready to be used to generate certificates.

### Generate the certificate

We will create a `Certificate` object in the `kube-system` namespace - we choose this one because the `Deployment` of Traefik also runs in this namespace by default. The `Certificate` object for the wildcard domain `*.k3s.nobbs.eu` looks like this:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-k3s-nobbs-eu
  namespace: kube-system
spec:
  secretName: wildcard-k3s-nobbs-eu-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - "*.k3s.nobbs.eu"
```

Now cert-manager takes over and does its magic. After a few minutes, a valid wildcard certificate for the specified domain should be found in the `secret` named `wildcard-k3s-nobbs-eu-tls`. In order to see the current status of the process, `kubectl get certificate -n kube-system -o wide -w` may be used.

### Create Dynamic Config for Traefik

Next, we need to prepare a `ConfigMap` with a configuration snippet for Traefik that sets the default certificates. For this we create the following `ConfigMap`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: traefik-dynamic-config
  namespace: kube-system
data:
  dynamic.yaml: |
    tls:
      stores:
        default:
          defaultCertificate:
            certFile: /certs/tls.crt
            keyFile: /certs/tls.key
```

### Configure Traefik

To make Traefik use this certificate, we need to modify the Traefik `Deployment` accordingly. However, we will not do this directly - such changes would be immediately undone, since Traefik is managed by the `HelmChart` manifest file `traefik.yaml` in the path `/var/lib/rancher/k3s/server/manifests`. Modifications to this file are automatically reflected to the cluster.

{{< alert >}}
The [K3s documentation](https://rancher.com/docs/k3s/latest/en/networking/#traefik-ingress-controller) recommends not to edit the `traefik.yaml` directly either, but to create another `HelmChartConfig` manifest file in the same path.
{{< /alert >}}

Thus, we create another file `traefik-config.yaml` and put the following content into this file:

```yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    volumes:
      - name: wildcard-k3s-nobbs-eu-tls
        mountPath: /certs
        type: secret
      - name: traefik-dynamic-config
        mountPath: /config
        type: configMap

    additionalArguments:
      - --providers.file.filename=/config/dynamic.yaml
```

Once this file is saved, the pre-installed [Helm Controller](https://github.com/k3s-io/helm-controller) will start and update the Traefik `Deployment`.

### Using the wildcard certificate

Once the Traefik `Pod` is regenerated after the update and is available again, we can now use the wildcard certificate. To do this, we need to configure the `spec.tls` section on an `Ingress` object, e.g. as follows.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pihole-ingress
  namespace: dns
spec:
  rules:
    - host: pihole.k3s.nobbs.eu
      http:
        paths:
          - pathType: Prefix
            path: "/"
            backend:
              service:
                name: http
                port:
                  name: http
  tls:
    - hosts:
        - pihole.k3s.nobbs.eu
```

{{< alert >}}

The `spec.tls.secretName` attribute must not be set, otherwise Traefik will try to load the certificate from the referenced `secret`.

{{< /alert >}}

That's all - by the way, the configured wildcard certificate is used for all HTTPS connections, especially also for 404 error pages for non-matched hosts.

This solution, even if it initially requires some adjustments to Traefik, offers the advantages that cert-manager continues to take care of the lifecycle of the certificates and renews them when necessary. In addition, the certificate `secret` does not need to be copied to other namespaces.
