# Airbroke Helm Chart

This Helm chart deploys Airbroke, a modern, React-based open source error catcher web application.

## Database

The `database.url` and `database.migrations_url` values must be set to the connection string of your PostgreSQL database.

## Configuration

The following table lists the configurable parameters of the Airbroke chart and their default values.

| Parameter | Description | Default |
| --------- | ----------- | ------- |
| `nameOverride` | String to partially override airbroke.fullname | `""` |
| `fullnameOverride` | String to fully override airbroke.fullname | `""` |
| `database.url` | PostgreSQL connection string | `""` |
| `database.migrations_url` | PostgreSQL connection string for migrations | `""` |
| `web.image` | Docker image for the web application | `ghcr.io/icoretech/airbroke:1.1.2` |
| `web.replicaCount` | Number of replicas to run | `1` |
| `web.updateStrategy` | Update strategy to use | `{type: RollingUpdate, rollingUpdate: {maxUnavailable: 0, maxSurge: 1}}` |
| `web.hpa.enabled` | Enables the Horizontal Pod Autoscaler | `false` |
| `web.ingress.enabled` | Enables Ingress | `false` |
| `pgbouncer.enabled` | Enables Pgbouncer, a lightweight connection pooler for PostgreSQL | `false` |

Please note, this is a simplified version of the parameters for the purpose of this README. For full configuration options, please refer to the `values.yaml` file.

You can specify additional environment variables using the `extraEnvs` parameter.

### Pgbouncer

Pgbouncer, a lightweight connection pooler for PostgreSQL, can be enabled using the `pgbouncer.enabled` parameter. You can customize the Pgbouncer configuration under `pgbouncer.config`.

## Example using Flux

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: airbroke
  namespace: airbroke
spec:
  releaseName: airbroke
  chart:
    spec:
      chart: ./charts/airbroke
      sourceRef:
        kind: GitRepository
        name: flux-system
        namespace: flux-system
  interval: 10m0s
  install:
    remediation:
      retries: 4
  upgrade:
    remediation:
      retries: 4
  values:
    database:
      url: 'postgresql://xxxx:xxxx@pgbouncer.default.svc.cluster.local:5432/airbroke_production?pgbouncer=true&connection_limit=100&pool_timeout=10&application_name=airbroke&schema=public'
      migrations_url: 'postgresql://xxxx:xxxx@postgres-postgresql.postgres.svc.cluster.local:5432/airbroke_production?schema=public'
    web:
      image: ghcr.io/icoretech/airbroke:main-c5d425f-1683936478 # {"$imagepolicy": "flux-system:airbroke"}
      replicaCount: 2
      hpa:
        enabled: true
        maxReplicas: 5
        cpu: 100
      resources:
        requests:
          cpu: 1500m
          memory: 500M
        limits:
          cpu: 1500m
          memory: 500M
      ingress:
        enabled: true
        ingressClassName: nginx
        annotations:
          cert-manager.io/cluster-issuer: letsencrypt
          external-dns.alpha.kubernetes.io/cloudflare-proxied: 'false'
          nginx.ingress.kubernetes.io/configuration-snippet: |
            real_ip_header proxy_protocol;
        hosts:
          - host: airbroke.mydomain.com
            paths:
              - '/'
        tls:
          - hosts:
              - airbroke.mydomain.com
```

Please be aware that the previous example serves as a basic template, and you'll likely need to adjust it to suit your specific requirements. For a comprehensive list of configurable options, please consult the values.yaml file. It's important to note that the image value included above is just a placeholder, and you should substitute it with your desired [image tag](https://github.com/icoretech/airbroke/pkgs/container/airbroke).

For automated image updates, consider utilizing [Flux Image Automation](https://fluxcd.io/docs/guides/image-update/).
Our images are deliberately tagged to facilitate Flux's automatic deployment updates whenever a new image becomes available.

```yaml
# example production semver image policy
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: airbroke
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: airbroke
  policy:
    semver:
      range: '>=1.1.3 <2.0.0'
```

```yaml
# example development image policy
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: airbroke
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: airbroke
  filterTags:
    pattern: '^main-[a-fA-F0-9]+-(?P<ts>.*)'
    extract: '$ts'
  policy:
    numerical:
      order: asc
```