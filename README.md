# do-k8s

Configuration-as-code and documentation for a little Kubernetes platform.

The "Kubernetes platform" consists of:

- A Kubernetes cluster, of course, running on [DigitalOcean Kubernetes].
- An [NGINX Ingress Controller] to handle host-based routing of Ingress resources, backed by a [DigitalOcean load balancer].
- [ExternalDNS] to allow managing DNS records from within Kubernetes.
- [cert-manager] to automatically provision and renew certificates for Ingress TLS hosts.

The total cost for this setup, based on the included [cluster definition], is $30/month ($20/month for a 4GB, 2 vCPU droplet; $10/month for the load balancer).

## Usage

The following steps should ultimately be scripted.
In the mean time, it should be possible to follow the steps in order, copying and pasting snippets, in order to bring up a cluster with all the platform componentry running.

### Prerequisites

In order to follow these instructions you'll need the following installed and on your `PATH`:

- The [DigitalOcean CLI].
- The [Kubernetes CLI].
- [`jq`].

### Choose a name

The cluster in Digital Ocean Kubernetes needs a name.
Furthermore, we need a domain name under which DNS entires should be created by ExternalDNS.

The snippets below will assume the following variables are set with the desired names (they do not need to be exported):

- **`cluster_name`**: The name for the DigitalOcean Kubernetes cluster.
- **`cluster_domain`**: The DigitalOcean Domain to use for external DNS.

For example:

```sh
cluster_name='default'
cluster_domain='connec.co.uk'
```

### Create a cluster

Now we can create the cluster.

```sh
readarray -t cluster_args < <(cat deployment/cluster-args.json | jq -r '.[]')
doctl kubernetes clusters create "$cluster_name" "${cluster_args[@]}"
```

This will take some time.
Once it has completed, your kubeconfig will be updated and you should be able to list everything running on the cluster by default:

```sh
kubectl get pods --all-namespaces
```

They should all be `Running`.

### Deploy the NGINX ingress controller

The NGINX Ingress Controller project maintains Kubernetes manifests that can be used to deploy the ingress controller:

```sh
version=0.32.0
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-$version/deploy/static/provider/do/deploy.yaml
```

This will crate a bunch of Kubernetes resources, including a `LoadBalancer` service.
The cluster will provision a DigitalOcean Load Balancer for this service, which may take some time.
You can monitor progress by watching:

```sh
kubectl --namespace ingress-nginx get services ingress-nginx-controller
```

### Deploy ExternalDNS

ExternalDNS allows us to manage DNS records from within Kubernetes by annotating services or ingresses.
There are a lot of configuration options for ExternalDNS, and the following have been chosen for simplicity and consistency:

- Only ingress resources will be used as sources for DNS names.
- Ingress resources that want external DNS entries should be annotated with `external-dns: ''`.
- Only subdomains of the cluster domain will be managed.
  Deployed resources are named such that other instances of ExternalDNS could run in the same cluster in order to manage multiple domains.
- `TXT` records will be used to track ownership of DNS records, and the cluster's ID will be used as
  the 'owner ID'.
  `TXT` records will be prefixed with `_`.
- The functional DNS records will all be `CNAME`s to the ingress domain, due to the workaround required for cert-manager.

ExternalDNS requires a DigitalOcean access token with write access in order to manage DNS entries.
The recommendation is to create one named "ExternalDNS – &lt;cluster domain&gt; (&lt;cluster ID&gt;)".
The following snippet will look up the cluster ID and request an access token:

```sh
cluster_id="$(doctl kubernetes clusters get "$cluster_name" --output json \
  | jq -r 'first | .id')"
{
  read -rsp "Access token for ExternalDNS – $cluster_domain – ($cluster_id): " external_dns_access_token
  echo >&2
  external_dns_access_token_base64="$(printf '%s' "$external_dns_access_token" | base64)"
}
```

We can now deploy the ExernalDNS Kubernetes manifest:

```sh
cluster_domain_dns_label="${cluster_domain//./-}"
external_dns="$(cat deployment/external-dns.yaml)"
for var in cluster_domain_dns_label cluster_domain cluster_id external_dns_access_token_base64; do
  external_dns="$(echo "$external_dns" | sed "s/\$$var/${!var}/")"
done
echo "$external_dns" | kubectl apply -f -
```

This will create several Kubernetes resources.
You can check that the ExternalDNS agent started correctly with:

```sh
kubectl -n external-dns logs -l app=external-dns
```

### Deploy cert-manager

The final component of the platform is cert-manager, which allows the Kubernetes cluster to provision TLS certificates based on the demands of ingress resources.

Firstly, install cert-manager into the cluster using the official manifest:

```sh
version=v0.14.3
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/$version/cert-manager.yaml
```

This will create a slew of Kubernetes custom resource definitions and resources.
Check the cert-manager deployment is running smoothly with:

```sh
kubectl --namespace cert-manager logs -l app=cert-manager
```

We are going to use DNS solvers for certificate request challenges, and as such need to supply a DigitalOcean access token with write access.
The recommendation is to create one named "cert-manager – &lt;cluster domain&gt; (&lt;cluster ID&gt;)".
The following snippet will request an access token:

```sh
{
  read -rsp "Access token for cert-manager – $cluster_domain – ($cluster_id): " cert_manager_access_token
  echo >&2
  cert_manager_access_token_base64="$(printf '%s' "$cert_manager_access_token" | base64)"
}
```

We can then create issuers for Lets Encrypt using the included Kubernetes manifest:

```sh
{
  read -p 'Email to use with Lets Encrypt: ' lets_encrypt_email
  cat deployment/cluster-issuers.yaml \
    | sed "s/\$cluster_domain/$cluster_domain/" \
    | sed "s/\$lets_encrypt_email/$lets_encrypt_email/" \
    | sed "s/\$cert_manager_access_token_base64/$cert_manager_access_token_base64/" \
    | kubectl apply -f -
}
```

### Test the platform

Finally, we can test the platform!
Deploy the included test Kubernetes manifest:

```sh
cat deployment/test.yaml \
  | sed "s/\$cluster_domain/$cluster_domain/" \
  | kubectl apply -f -
```

It may take a minute or two for everything to provision.
A good indication is to watch the `certificate` resource that cert-manager generates:

```sh
kubectl get certificates
```

Once the certificate is `Ready`, you should be able to cURL `https://test.$cluster_domain` and see the text `OK`:

```sh
$ curl https://test.$cluster_domain
OK
```

Once satisfied, clean up the test:

```sh
kubectl delete -f deployment/test.yaml
```

And you're done.

[DigitalOcean Kubernetes]: https://www.digitalocean.com/products/kubernetes/
[NGINX Ingress Controller]: https://kubernetes.github.io/ingress-nginx/
[DigitalOcean load balancer]: https://www.digitalocean.com/products/load-balancer/
[ExternalDNS]: https://github.com/kubernetes-sigs/external-dns
[cert-manager]: https://cert-manager.io/
[cluster definition]: deployment/cluster-args.json
[DigitalOcean CLI]: https://github.com/digitalocean/doctl#installing-doctl
[Kubernetes CLI]: https://kubernetes.io/docs/tasks/tools/install-kubectl/
[`jq`]: https://stedolan.github.io/jq/download/
[compatibility issue]: https://github.com/jetstack/cert-manager/issues/863#issuecomment-567062996
