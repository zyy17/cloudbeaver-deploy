## Cloudbeaver Helm chart for Kubernetes

### Use as GitHub Helm chart

After chart release is published, you can install directly from GitHub Pages:

```bash
helm repo add cloudbeaver https://zyy17.github.io/cloudbeaver-deploy
helm repo update
helm install cloudbeaver cloudbeaver/cloudbeaver --values values.yaml
```

To publish a new chart version:

1. Bump `version` in `charts/cloudbeaver/Chart.yaml`
2. Push a tag like `v0.3.1`
3. GitHub Action `Release Helm Chart` will publish the chart package and `index.yaml`
4. Workflow releases chart from `charts/cloudbeaver` (`charts_dir: charts`)

#### Minimum requirements:

* Kubernetes >= 1.23
* 2 CPUs
* 4Gb RAM
* Linux or macOS as deploy host
* `git` and `kubectl` installed
* An ingress controller (NGINX, HAProxy, ALB, or Traefik) and [Kubernetes Helm plugin](https://helm.sh/docs/topics/plugins/) added to your `k8s`

#### Supported Ingress Controllers:

* **nginx** - NGINX Ingress Controller (default)
* **haproxy** - HAProxy Ingress Controller  
* **alb** - AWS Application Load Balancer (for AWS EKS)
* **traefik** - Traefik Ingress Controller

For AWS EKS specific deployment instructions, see [AWS EKS deployment guide](../AWS/aws-eks/README.md).

### Traefik ingress notes

If you use Traefik, set `ingressController: traefik` in `values.yaml`.

You can provide Traefik-specific annotations through:

- `traefik.annotations` (for example, `cert-manager.io/cluster-issuer`)

When `httpScheme=https`, the chart configures TLS for host `cloudbeaverBaseDomain`.
It uses secret `<release-name>-ingress-tls`.

### Storage access mode update

The original PVC template `playground/cloudbeaver-deploy/k8s/templates/volume/cloudbeaver.yaml`
was changed from `ReadWriteMany` to `ReadWriteOnce`.

### Resource limits and requests

You can configure container resource constraints in `values.yaml`:

- `cloudbeaver.resources` for the CloudBeaver container
- `backend.dbResources` for the internal PostgreSQL container

Both support standard Kubernetes `requests` and `limits` for CPU and memory.


### User and permissions changes

Starting from CloudBeaver v25.0 process inside the container now runs as the ‘dbeaver’ user (‘UID=8978’), instead of ‘root’.  
If a user with ‘UID=8978’ already exists in your environment, permission conflicts may occur.  
Additionally, the default Docker volumes directory’s ownership has changed.  
Previously, the volumes were owned by the ‘root’ user, but now they are owned by the ‘dbeaver’ user (‘UID=8978’).  

### How to run services
- Clone this repo from GitHub: `git clone https://github.com/zyy17/cloudbeaver-deploy`
- `cd cloudbeaver-deploy`
- Edit chart values in `charts/cloudbeaver/values.yaml` (use any text editor)
- You must set the `cloudbeaver_db_password` variable before deploying the cluster. The database password is empty by default and the deployment will fail without it.
- Configure domain and SSL certificate (optional)
  - Add an A record in your DNS hosting for value `cloudbeaverBaseDomain` with your load balancer IP address.
  - If you set *HTTPS* endpoint scheme, create a valid TLS certificate for `cloudbeaverBaseDomain` and place it into `charts/cloudbeaver/ingressSsl`:  
    Certificate: `charts/cloudbeaver/ingressSsl/fullchain.pem`  
    Private Key: `charts/cloudbeaver/ingressSsl/privkey.pem`
- Deploy Cloudbeaver with Helm: `helm install cloudbeaver ./charts/cloudbeaver --values ./charts/cloudbeaver/values.yaml`

### Version update procedure.

- Change directory to `cloudbeaver-deploy`.
- Change value of `imageTag` in configuration file `charts/cloudbeaver/values.yaml` with a preferred version. Go to next step if tag `latest` set.
- Upgrade cluster: `helm upgrade cloudbeaver ./charts/cloudbeaver --values ./charts/cloudbeaver/values.yaml` 

### OpenShift deployment

You need additional configuration changes

- In `charts/cloudbeaver/values.yaml` change the `ingressController` value to `haproxy`
- Add security context  
  Uncomment the following lines in `cloudbeaver.yaml` files in [templates/deployment](charts/cloudbeaver/templates/deployment):
    ```yaml
          # securityContext:
          #     runAsUser: 1000
          #     runAsGroup: 1000
          #     fsGroup: 1000
          #     fsGroupChangePolicy: "Always"
    ```

### Digital Ocean proxy configuration

Edit ingress controller with:

- `kubectl edit service -n ingress-nginx ingress-nginx-controller`

and add two lines in the `metadata.annotations`

- `service.beta.kubernetes.io/do-loadbalancer-enable-proxy-protocol: "true"`
- `service.beta.kubernetes.io/do-loadbalancer-hostname: "cloudbeaverBaseDomain"`
