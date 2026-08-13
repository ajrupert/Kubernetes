# Cilium Gateway API with cert-manager and Let's Encrypt

## Overview

This guide builds on **Cilium Gateway API with LB-IPAM and L2 Announcement**.

The existing Cilium Gateway, LoadBalancer IP, L2 Announcement configuration and applications remain unchanged.

This guide adds:

- cert-manager
- Let's Encrypt (production)
- HTTPS on the existing Gateway
- Automatic TLS certificate renewal

### Traffic Flow Diagram

```
flowchart LR
    Client -->|HTTPS| Firewall["Firewall / NAT"]
    Firewall --> IP["192.168.1.151"]
    IP --> Gateway["Cilium Gateway"]

    Gateway -->|TLS termination| Route["HTTPRoute"]
    Route --> App1["hello-app-1 <br/>app1.example.com"]
    Route --> App2["hello-app-2 <br/>app2.example.com"]

subgraph cert-manager
    LE["Let's Encrypt"] --> CM["cert-manager"]
    CM --> Secret["TLS Secret"]
end

    Secret --> Gateway
```

---

## Prerequisites

This guide assumes the previous guide has already been completed successfully. You should already have:

- A working Kubernetes cluster
- Cilium with Gateway API, LB-IPAM and L2 Announcement
- A working `GatewayClass` (`cilium`)
- Gateway `demo-gateway`
- Applications `hello-app-1` and `hello-app-2` with their HTTPRoutes

Verify the Gateway:

```
kubectl get gateway
```

Example:

```
NAME            CLASS    ADDRESS          PROGRAMMED   AGE
demo-gateway    cilium   192.168.1.151    True         ...
```

Verify the HTTPRoutes:

```
kubectl get httproute
```

Example:

```
NAME                HOSTNAMES
hello-app-1-route   app1.example.com
hello-app-2-route   app2.example.com
```

---

## 1. Configure DNS

Let's Encrypt must be able to reach the applications over the public internet.

Make sure the DNS records for the hostnames point to your public IP address:

```
app1.example.com    A    <PUBLIC-IP>
app2.example.com    A    <PUBLIC-IP>
```

Your firewall must forward HTTP and HTTPS traffic to the Gateway LoadBalancer IP:

```
TCP/80    -> 192.168.1.151:80
TCP/443   -> 192.168.1.151:443
```

HTTP is required because this guide uses the ACME HTTP-01 challenge. During certificate issuance, cert-manager creates a temporary HTTPRoute for the challenge. The existing Gateway therefore needs to keep an HTTP listener on port 80.

---

## 2. Install cert-manager

cert-manager is responsible for requesting and renewing the certificates.

```
helm install \
  cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --version v1.20.3 \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true \
  --set config.gatewayAPI.enabled=true
```

Check the installation:

```
kubectl get pods -n cert-manager
```

Expected:

```
NAME                          READY   STATUS
cert-manager-...              1/1     Running
cert-manager-cainjector-...   1/1     Running
cert-manager-webhook-...      1/1     Running
```

The Gateway API integration must be enabled because cert-manager will create temporary HTTPRoute resources for the ACME HTTP-01 challenge.

---

## 3. Create the Let's Encrypt ClusterIssuer

Create: `letsencrypt-production.yaml`

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    email: admin@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-production
    solvers:
    - http01:
        gatewayHTTPRoute:
          parentRefs:
          - name: demo-gateway
            namespace: default
            kind: Gateway
```

Replace `admin@example.com` with a valid email address.

Apply:

```
kubectl apply -f letsencrypt-production.yaml
```

Check:

```
kubectl get clusterissuer
```

Expected:

```
NAME                      READY
letsencrypt-production    True
```

The `gatewayHTTPRoute.parentRefs` configuration tells cert-manager to use the existing `demo-gateway` for the HTTP-01 challenge. cert-manager will create a temporary HTTPRoute that points to its ACME challenge solver. After the certificate has been issued, the temporary HTTPRoute is removed.

---

## 4. Create the TLS Certificate

We will request one certificate containing both application hostnames.

Create: `gateway-certificate.yaml`

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: demo-gateway-tls
spec:
  secretName: demo-gateway-tls
  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer
  dnsNames:
  - app1.example.com
  - app2.example.com
```

Apply:

```
kubectl apply -f gateway-certificate.yaml
```

Check:

```
kubectl get certificate
```

Wait for cert-manager to complete the ACME challenge. Expected:

```
NAME                READY   SECRET              AGE
demo-gateway-tls    True    demo-gateway-tls    ...
```

Verify the TLS Secret:

```
kubectl get secret demo-gateway-tls
```

Expected:

```
NAME                TYPE                DATA
demo-gateway-tls    kubernetes.io/tls   2
```

---

## 5. Update the Gateway

The existing Gateway currently listens for HTTP traffic on port 80. We will add an HTTPS listener on port 443.

Update: `gateway.yaml`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: demo-gateway
spec:
  gatewayClassName: cilium
  infrastructure:
    labels:
      advertise: "true"
    annotations:
      kube-vip.io/ignore: "true"
  listeners:
  - name: http
    protocol: HTTP
    port: 80
  - name: https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
      - name: demo-gateway-tls
```

Apply:

```
kubectl apply -f gateway.yaml
```

Check:

```
kubectl get gateway demo-gateway
```

`PROGRAMMED` should remain `True`. The TLS connection is terminated at the Cilium Gateway; the connection from the Gateway to the backend remains plain HTTP.

---

## 6. Verify the HTTPRoutes

The existing HTTPRoutes do not need to be changed, for example:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: hello-app-1-route
spec:
  parentRefs:
  - name: demo-gateway
  hostnames:
  - app1.example.com
  rules:
  - backendRefs:
    - name: hello-app-1
      port: 80
```

The same HTTPRoute can now also be accessed through the HTTPS listener.

```
kubectl get httproute
```

The routes should still be accepted by the Gateway.

---

## 7. Test HTTPS

```
curl https://app1.example.com
curl https://app2.example.com
```

Both applications are now accessible over HTTPS with a valid, browser-trusted Let's Encrypt certificate.

---

## 8. Automatic Renewal

cert-manager automatically manages the certificate lifecycle.

```
kubectl get certificate demo-gateway-tls -o wide
kubectl describe certificate demo-gateway-tls
```

`Ready` should be `True`. cert-manager will automatically renew the certificate before it expires — no manual certificate replacement is required.

---

## Troubleshooting

**Certificate is not ready**

```
kubectl describe certificate demo-gateway-tls
kubectl get certificaterequest
kubectl get order
kubectl get challenge
```

These resources can be used to determine where the ACME process is failing.

**HTTP-01 challenge is failing**

```
kubectl get httproute
```

During certificate issuance an additional `cm-acme-http-solver-*` route should appear temporarily. Verify that it references `demo-gateway`, and that the Gateway has a listener on port 80 — do not remove it while the HTTP-01 challenge is in use.

**Let's Encrypt cannot reach the challenge**

Make sure incoming HTTP traffic on TCP/80 is actually forwarded to `192.168.1.151:80`.

**HTTPS listener is not programmed**

```
kubectl describe gateway demo-gateway
kubectl get secret demo-gateway-tls
```

If the Secret does not exist, check the status of the Certificate.

---

## Cleanup

```
kubectl delete certificate demo-gateway-tls
kubectl delete clusterissuer letsencrypt-production
kubectl delete secret demo-gateway-tls
helm uninstall cert-manager -n cert-manager
```

The existing Cilium Gateway, LB-IPAM, L2 Announcement configuration, applications and HTTPRoutes are not removed.

---

## Conclusion

The existing Cilium Gateway now provides HTTPS with automatically managed Let's Encrypt certificates. Cilium remains responsible for the Gateway, the LoadBalancer IP and traffic routing, while cert-manager manages the TLS certificate lifecycle.
