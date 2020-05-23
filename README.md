# do-k8s

Configuration-as-code and documentation for a little Kubernetes platform.

The "Kubernetes platform" consists of:

- A Kubernetes cluster, of course, running on [DigitalOcean Kubernetes](https://www.digitalocean.com/products/kubernetes/).
- An [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/) to handle host-based routing of Ingress resources, backed by a [DigitalOcean load balancer](https://www.digitalocean.com/products/load-balancer/).
- [ExternalDNS](https://github.com/kubernetes-sigs/external-dns) to allow managing DNS records from within Kubernetes.
- [cert-manager](https://cert-manager.io/) to automatically provision and renew certificates for Ingress TLS hosts.

The total cost for this setup, based on the included [cluster definition](deployment/cluster-args.json), is $30/month ($20/month for a 4GB, 2 vCPU droplet; $10/month for the load balancer).

## Usage

The following steps should ultimately be scripted.
In the mean time, it should be possible to follow the steps in order, copying and pasting snippets, in order to bring up a cluster with all the platform componentry running.

### Prerequisites

In order to follow these instructions you'll need the following installed and on your `PATH`:

- The [DigitalOcean CLI](https://github.com/digitalocean/doctl#installing-doctl).
- The [Kubernetes CLI](https://kubernetes.io/docs/tasks/tools/install-kubectl/).
- [Helm](https://helm.sh/docs/intro/install/).
- [`jq`](https://stedolan.github.io/jq/download/).

You will also need the following Helm repositories:

```sh
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add jetstack https://charts.jetstack.io
```

It's recommended to update your repositories using `helm repo update`.

### Choose a name

The cluster in Digital Ocean Kubernetes needs a name.
Furthermore, we need a domain name under which DNS entires should be created by ExternalDNS.

The snippets below will assume the following variables are set with the desired names:

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

The NGINX Ingress Controller project maintains a Helm chart that can be used to deploy the ingress controller:

```sh
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
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
- Only subdomains of the cluster domain will be managed.
  Deployed resources are named such that other instances of ExternalDNS could run in the same cluster in order to manage multiple domains.
- `TXT` records will be used to track ownership of DNS records, and the cluster's ID will be used as
  the 'owner ID'.
  `TXT` records will be prefixed with `_`.

ExternalDNS requires a DigitalOcean access token with write access in order to manage DNS entries.
The recommendation is to create one named "ExternalDNS – &lt;cluster domain&gt; (&lt;cluster ID&gt;)".
The following snippet will look up the cluster ID and request an access token:

```sh
cluster_id="$(doctl kubernetes clusters get "$cluster_name" --output json \
  | jq -r 'first | .id')"
read -rsp "Access token for ExternalDNS – $cluster_domain – ($cluster_id): " external_dns_access_token
```

We can now deploy ExernalDNS using Helm:

```sh
helm install external-dns-${cluster_domain//./-} bitnami/external-dns \
  --namespace external-dns \
  --create-namespace \
  --values deployment/external-dns.yaml \
  --set txtOwnerId=$cluster_id \
  --set domainFilters[0]=$cluster_domain \
  --set "digitalocean.apiToken=$external_dns_access_token"
```

This will create several Kubernetes resources.
You can check that the ExternalDNS agent started correctly with:

```sh
kubectl --namespace external-dns logs -l app.kubernetes.io/name=external-dns
```

### Deploy cert-manager

The final component of the platform is cert-manager, which allows the Kubernetes cluster to provision TLS certificates based on the demands of ingress resources.

Firstly, install cert-manager into the cluster using Helm:

```sh
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

Check the cert-manager deployment is running smoothly with:

```sh
kubectl --namespace cert-manager logs -l app.kubernetes.io/name=cert-manager
```

We are going to use DNS solvers for certificate request challenges, and as such need to supply a DigitalOcean access token with write access.
The recommendation is to create one named "cert-manager – &lt;cluster domain&gt; (&lt;cluster ID&gt;)".
The following snippet will request an access token:

```sh
read -rsp "Access token for cert-manager – $cluster_domain – ($cluster_id): " cert_manager_access_token
```

We can then create issuers for Lets Encrypt using the included Kubernetes manifest:

```sh
{
  read -p 'Email to use with Lets Encrypt: ' lets_encrypt_email
  echo >&2
  helm install cert-manager-issuers ./deployment/cert-manager-issuers \
    --namespace cert-manager \
    --set email=$lets_encrypt_email \
    --set dnsZones[0]=$cluster_domain \
    --set "accessToken=$cert_manager_access_token"
}
```

### Test the platform

Finally, we can test the platform!
Deploy the included Helm chart:

```sh
helm install test ./deployment/test \
  --namespace test \
  --create-namespace \
  --set clusterDomain=$cluster_domain
```

It may take a minute or two for everything to provision.
A good indication is to watch the `certificate` resource that cert-manager generates:

```sh
kubectl --namespace test get certificates
```

Once the certificate is `Ready`, you should be able to cURL `https://test.$cluster_domain` and see the text `OK`:

```sh
$ curl https://test.$cluster_domain
OK
```

Once satisfied, clean up the test:

```sh
helm delete test --namespace test
kubectl delete namespace test
```

And you're done.
