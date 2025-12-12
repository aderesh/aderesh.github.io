+++
date = '2025-12-11T20:02:59-05:00'
draft = false
title = 'Exposing Traefik dashboard in k3s after upgrading to v3'
+++

Once I updated my k3s cluster from v1.31.14+k3s1 (https://docs.k3s.io/release-notes/v1.31.X) to v1.32.10+k3s1 (https://docs.k3s.io/release-notes/v1.32.X), I noticed that my Traefik dashboard stopped working. I spent some time trying to find a way to expose the dashboard but couldn’t find quick guides for k3s. ChatGPT wasn’t helpful either, so I decided to write this short article.

This article/guide is helpful if you’re looking to update a default k3s Traefik v3 configuration.

Note: I’ll describe how to expose what’s called an insecure dashboard — it is intended only for trusted/internal networks and should not be exposed to the public internet.

My first point of investigation was the Traefik v3 dashboard documentation: https://doc.traefik.io/traefik/reference/install-configuration/api-dashboard/. Based on the docs, a few flags must be set to true:
- `api.dashboard=true` enables the dashboard UI in Traefik v3.
- `api.insecure=true` exposes that dashboard without authentication on the `traefik` entrypoint (port 8080).

There are a few ways to configure Traefik v3. You can pass configuration through command-line arguments or use a `traefik.yml` file.

Let’s start with the Traefik deployment command-line arguments:
```bash
kubectl -n kube-system describe deploy traefik | grep -A15 -B5 Args:
```

This is what I saw in v1.32.10+k3s1:
```bash
  Containers:
   traefik:
    Image:       rancher/mirrored-library-traefik:3.5.1
    Ports:       9100/TCP (metrics), 8080/TCP (traefik), 8000/TCP (web), 8443/TCP (websecure)
    Host Ports:  0/TCP (metrics), 0/TCP (traefik), 0/TCP (web), 0/TCP (websecure)
    Args:
      --entryPoints.metrics.address=:9100/tcp
      --entryPoints.traefik.address=:8080/tcp
      --entryPoints.web.address=:8000/tcp
      --entryPoints.websecure.address=:8443/tcp
      --api.dashboard=true
      --ping=true
      --metrics.prometheus=true
      --metrics.prometheus.entrypoint=metrics
      --providers.kubernetescrd
      --providers.kubernetescrd.allowEmptyServices=true
      --providers.kubernetesingress
      --providers.kubernetesingress.allowEmptyServices=true
      --providers.kubernetesingress.ingressendpoint.publishedservice=kube-system/traefik
      --entryPoints.websecure.http.tls=true
      --log.level=INFO
```

Alright, `api.dashboard` is set, but `api.insecure` is missing. To expose the dashboard without authentication, we need to add the `--api.insecure=true` argument.

Now, how do we update the Traefik deployment in k3s? According to the [official documentation](https://docs.k3s.io/networking/networking-services#traefik-ingress-controller), Traefik is deployed using a Helm chart. The Traefik manifest is located at `/var/lib/rancher/k3s/server/manifests/traefik.yaml`, and the actual chart is maintained [here](https://github.com/k3s-io/k3s-charts/blob/main/charts/traefik/). The documentation explains how to customize the default packaged components (e.g., Traefik) using [HelmChartConfig](https://docs.k3s.io/add-ons/helm#customizing-packaged-components-with-helmchartconfig). That’s the approach we’ll use.

After reviewing the [documentation](https://docs.k3s.io/add-ons/helm#customizing-packaged-components-with-helmchartconfig) and the Traefik chart [values.yaml](https://github.com/k3s-io/k3s-charts/blob/main/charts/traefik/37.1.1%2Bup37.1.0/values.yaml), I created the following manifest called `traefik-customize.yaml`:
```yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    additionalArguments:
      - "--api.insecure=true"
```
Looks pretty simple, eh? According to the documentation, one way to apply it is to save this manifest to `/var/lib/rancher/k3s/server/manifests/` on your server node. However, it also works to apply it directly with `kubectl`, which is what I did — just personal preference. I haven’t experienced any disadvantages to this approach; most importantly, it survives cluster reboots.
Once you apply it, the Helm controller will replace the old pod and create a new one.

`kubectl apply -f traefik-customize.yaml`

The next step is to expose the dashboard. I used the following Service and Ingress configuration:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: traefik-dashboard
  namespace: kube-system
spec:
  type: ClusterIP
  ports:
  - name: traefik
    port: 8080
    targetPort: traefik
    protocol: TCP
  selector:
    app.kubernetes.io/instance: traefik-kube-system
    app.kubernetes.io/name: traefik
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: traefik-ingress
  namespace: kube-system
spec:
  rules:
    - host: traefik.lan
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: traefik-dashboard
                port:
                  number: 8080
```
Make sure the `traefik.lan` DNS record is configured to point to the server node. If you don’t have a DNS server, you can update your local `hosts` file.

And voila — the dashboard should be available at `http://traefik.lan/dashboard/` but `http://traefik.lan` also works (Traefik v3 upgrade?). 

Happy coding!