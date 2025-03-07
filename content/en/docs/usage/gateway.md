---
title: "Securing Gateway Resources"
linkTitle: "Securing Gateway Resources"
weight: 105
type: "docs"
---

**FEATURE STATE**: cert-manager 1.5 [alpha]

{{% pageinfo color="info" %}}

📌  This page focuses on automatically creating Certificate resources by
annotating Gateway resource. If you are looking for using an ACME Issuer along
with HTTP-01 challenges using the Gateway API, see [ACME
HTTP-01](/docs/configuration/acme/http01/).

{{% /pageinfo %}}

cert-manager can generate TLS certificates for Gateway resources. This is
configured by adding annotations to a Gateway and is similar to the process for
[Securing Ingress Resources](/docs/usage/ingress/).

The Gateway resource is part of the [Gateway API][gwapi], a set of CRDs that you
install on your Kubernetes cluster and which provide various improvements over
the Ingress API.

[gwapi]: https://gateway-api.sigs.k8s.io

The Gateway resource holds the TLS configuration, as illustrated in the
following diagram (source: https://gateway-api.sigs.k8s.io):

![Gateway vs. HTTPRoute](/images/gateway-roles.png)

Note that cert-manager only supports setting up the TLS configuration on the
Gateway resource when the Gateway is configured to terminate the TLS connection.
There is still uncertainty around how the HTTPRoute TLS configuration should be
used (see [the discussion about TLS interaction between HTTPRoute and
Gateway][gateway-api#577]) and no existing implementation supports the TLS
pass-through mode in which the HTTPRoute contains the TLS configuration and
terminates the connection (see the [feature request for supporting `spec.tls` in
HTTPRoute for Istio][istio#31747]).

[istio#31747]: https://github.com/istio/istio/issues/31747
[gateway-api#577]: https://github.com/kubernetes-sigs/gateway-api/issues/577


{{% pageinfo color="info" %}}

📌  This feature requires the installation of the Gateway API CRDs and passing a
feature flag to the cert-manager controller.

To install the Gateway API CRDs, run the following command:

```sh
kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v0.3.0" | kubectl apply -f -
```

To enable the feature in cert-manager, turn on the `GatewayAPI` feature gate:

- If you are using Helm:

  ```sh
  helm upgrade --install cert-manager jetstack/cert-manager --namespace cert-manager \
    --set "extraArgs={--feature-gates=ExperimentalGatewayAPISupport=true}"
  ```

- If you are using the raw cert-manager manifests, add the following flag to the
  cert-manager controller Deployment:

  ```yaml
  args:
    - --feature-gates=ExperimentalGatewayAPISupport=true
  ```

The Gateway API CRDs should either be installed before cert-manager starts or
the cert-manager Deployment should be restarted after installing the Gateway API
CRDs. This is important because some of the cert-manager components only perform
the Gateway API check on startup. You can restart cert-manager with the
following command:

```sh
kubectl rollout restart deployment cert-manager -n cert-manager
```

{{% /pageinfo %}}

The annotations `cert-manager.io/issuer` or `cert-manager.io/cluster-issuer`
tell cert-manager  to create a Certificate for a Gateway. For example, the
following Gateway will trigger the creation of a Certificate with the name
`example-com-tls`:

```yaml
apiVersion: networking.x-k8s.io/v1alpha1
kind: Gateway
metadata:
  name: example
  annotations:
    cert-manager.io/issuer: foo
spec:
  gatewayClassName: foo
  listeners:
    - hostname: example.com
      port: 443
      protocol: HTTPS
      routes:
        kind: HTTPRoute
        selector:
          matchLabels:
            app: foo
      tls:
        mode: Terminate
        certificateRef:
          name: example-com-tls
          kind: Secret
          group: core
```

A few moments later, cert-manager will create a Certificate. The Certificate is
named after the Secret name `example-com-tls`. The `dnsNames` field is set with
the `hostname` field from the Gateway spec.

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com-tls
spec:
  issuerRef:
    name: my-issuer
    kind: Issuer
    group: cert-manager.io
  dnsNames:
    - example.com # ✅ Copied from the `hostname` field.
  secretName: example-com-tls
```

## Use cases

### Generate TLS certs for selected TLS blocks

cert-manager skips any listener block that cannot be used for generating a
Certificate. For a listener block to be used for creating a Certificate, it must
meet the following requirements:

|           Field            |                         Requirement                         |
|----------------------------|-------------------------------------------------------------|
| `tls.hostname`             | Must not be empty.                                          |
| `tls.mode`                 | Must be set to `Terminate`. `Passthrough` is not supported. |
| `tls.certificateRef.name`  | Cannot be left empty.                                       |
| `tls.certificateRef.kind`  | Must be set to `Secret`.                                    |
| `tls.certificateRef.group` | Must be set to `core`.                                      |

In the following example, the first three listener blocks will not be used to
generate Certificate resources:

```yaml
apiVersion: networking.x-k8s.io/v1alpha1
kind: Gateway
metadata:
  annotations:
    cert-manager.io/issuer: my-issuer
spec:
  listeners:
    # ❌  Missing "tls" block, the following listener is skipped.
    - hostname: example.com

    # ❌  Missing "hostname", the following listener is skipped.
    - tls:
        certificateRef:
          name: example-com-tls
          kind: Secret"
          group: core

    # ❌  "mode: Passthrough" is not supported, the following listener is skipped.
    - hostname: example.com
      tls:
        mode: Passthrough
        certificateRef:
          name: example-com-tls
          kind: Secret
          group: core

    # ✅  The following listener is valid.
    - hostname: foo.example.com # ✅ Required.
      port: 443
      protocol: HTTPS
      routes:
        kind: HTTPRoute
        selector:
          matchLabels:
            app: foo
      tls:
        mode: Terminate # ✅ Required. "Terminate" is the only supported mode.
        certificateRef:
          name: example-com-tls # ✅ Required.
          kind: Secret  # ✅ Required. "Secret" is the only valid value.
          group: core # ✅ Required. "core" is the only valid value.
```

cert-manager has skipped over the first three listener blocks and has created a
single Certificate named `example-com-tls` for the last listener block:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com-tls
spec:
  issuerRef:
    name: my-issuer
    kind: Issuer
    group: cert-manager.io
  dnsNames:
    - foo.example.com
  secretName: example-com-tls
```

### Two listeners with the same Secret name

The same Secret name can be re-used in multiple TLS blocks, regardless of the
hostname. Let us imagine that you have these two listeners:

```yaml
apiVersion: networking.x-k8s.io/v1alpha1
kind: Gateway
metadata:
  name: example
  annotations:
    cert-manager.io/issuer: my-issuer
spec:
  gatewayClassName: foo
  listeners:
    # Listener 1.
    - hostname: example.com
      port: 443
      protocol: HTTPS
      routes:
        kind: HTTPRoute
        selector:
          matchLabels:
            app: foo
      tls:
        mode: Terminate
        certificateRef:
          name: example-com-tls
          kind: Secret
          group: core

    # Listener 2: Same Secret name as Listener 1, with a different hostname.
    - hostname: *.example.com
      port: 443
      protocol: HTTPS
      routes:
        kind: HTTPRoute
        selector:
          matchLabels:
            app: foo
      tls:
        mode: Terminate
        certificateRef:
          name: example-com-tls
          kind: Secret
          group: core

    # Listener 3: also same Secret name, except the hostname is also the same.
    - hostname: *.example.com
      port: 8443
      protocol: HTTPS
      routes:
        kind: HTTPRoute
        selector:
          matchLabels:
            app: foo
      tls:
        mode: Terminate
        certificateRef:
          name: example-com-tls
          kind: Secret
          group: core

   # Listener 4: different Secret name.
    - hostname: site.org
      port: 443
      protocol: HTTPS
      routes:
        kind: HTTPRoute
        selector:
          matchLabels:
            app: foo
      tls:
        mode: Terminate
        certificateRef:
          name: site-org-tls
          kind: Secret
          group: core
```

cert-manager will create two Certificates since two Secret names are used:
`example-com-tls` and `site-org-tls`. Note the Certificate's `dnsNames` contains
a single occurrence of `*.example.com ` for both listener 2 and 3 (the
`hostname` values are de-duplicated).

The two Certificates look like this:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com-tls
spec:
  issuerRef:
    name: my-issuer
    kind: Issuer
    group: cert-manager.io
  dnsNames:
    - example.com # From listener 1.
    - *.example.com # From listener 2 and 3.
  secretName: example-com-tls
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: site-org-tls
spec:
  issuerRef:
    name: my-issuer
    kind: Issuer
    group: cert-manager.io
  dnsNames:
    - site.org # From listener 4.
  secretName: site-org-tls
```

## Supported Annotations

If you are migrating to Gateway resources from Ingress resources, be aware that
there are some differences between [the annotations for Ingress
resources](https://cert-manager.io/docs/usage/ingress/#supported-annotations)
versus the annotations for Gateway resources.

The Gateway resource supports the following annotations for generating
Certificate resources:

- `cert-manager.io/issuer`: the name of an Issuer to acquire the certificate
  required for this Gateway. The Issuer _must_ be in the same namespace as the
  Gateway resource.

- `cert-manager.io/cluster-issuer`: the name of a ClusterIssuer to acquire the
  Certificate required for this Gateway. It does not matter which namespace your
  Gateway resides, as `ClusterIssuers` are non-namespaced resources.

- `cert-manager.io/issuer-kind`: the kind of the external issuer resource, for
  example `AWSPCACIssuer`. This is only necessary for out-of-tree issuers.

- `cert-manager.io/issuer-group`: the API group of the external issuer
  controller, for example `awspca.cert-manager.io`. This is only necessary for
  out-of-tree issuers.

- `cert-manager.io/common-name`: (optional) this annotation allows you to
  configure `spec.commonName` for the Certificate to be generated.

- ` cert-manager.io/duration`: (optional) this annotation allows you to
  configure `spec.duration` field for the Certificate to be generated.

- `cert-manager.io/renew-before`: (optional) this annotation allows you to
  configure `spec.renewBefore` field for the Certificate to be generated.

- `cert-manager.io/usages`: (optional) this annotation allows you to configure
  `spec.usages` field for the Certificate to be generated. Pass a string with
  comma-separated values i.e "key agreement,digital signature, server auth"
