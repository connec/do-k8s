# do-k8s

Configuration-as-code and documentation for a little Kubernetes platform.

The "Kubernetes platform" consists of:

- A Kubernetes cluster, of course, running on [DigitalOcean Kubernetes](https://www.digitalocean.com/products/kubernetes/).
- An [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/) to handle host-based routing of Ingress resources, backed by a [DigitalOcean load balancer](https://www.digitalocean.com/products/load-balancer/).
- [ExternalDNS](https://github.com/kubernetes-sigs/external-dns) to allow managing DNS records from within Kubernetes.
- [cert-manager](https://cert-manager.io/) to automatically provision and renew certificates for Ingress TLS hosts.

This is a similar idea to [bitnami/kube-prod-runtime](https://github.com/bitnami/kube-prod-runtime), except this project exclusively supports DigitalOcean and aims to be deployable via Helm.

The total cost for this setup, based on the settings suggested in the guide, is $30/month:

- $20/month for a 4GB, 2 vCPU droplet
- $10/month for the NGINX Ingress Controller's load balancer

## Setup guide

### Prerequisites

In order to follow these instructions you'll need the following installed and on your `PATH`:

- The [DigitalOcean CLI](https://github.com/digitalocean/doctl#installing-doctl).
- The [Kubernetes CLI](https://kubernetes.io/docs/tasks/tools/install-kubectl/).
- [Helm](https://helm.sh/docs/intro/install/).
- [`jq`](https://stedolan.github.io/jq/download/).

### Create a cluster

First, store the cluster name in an environment variable so we can refer to it later:

```sh
read -p 'Cluster name: ' cluster_name
```

Now we can create a cluster using the DigitalOcean CLI.
As noted above, this guide opts for a single 2 vCPU, 4GB memory droplet ($20/Month).

**Note:** You can find available regions, sizes, and versions using `doctl kubernetes options regions|sizes|versions`.

```sh
doctl kubernetes clusters create "$cluster_name" \
  --region lon1 \
  --version 1.17.5-do.0 \
  --node-pool 'name=default;size=s-2vcpu-4gb;count=1'
```

This will take some time.
Once it has completed, your kubeconfig will be updated and you should be able to list everything running on the cluster by default:

```sh
kubectl get pods --all-namespaces
```

Everything should be `Running`.

### Configuration

A few pieces of configuration need to be set when installing the Helm chart.
We will continue to use environment variables to store configuration, which can then be used in the `helm install` command later.

#### Get the cluster's ID

The cluster ID will be used when a unique-to-the-cluster value is needed in configuration.
It can be obtained using the DigitalOcean CLI:

```sh
cluster_id="$(doctl kubernetes clusters get "$cluster_name" --output json \
  | jq -r 'first | .id')"
```

#### Choose a domain name

We need a domain name under which DNS entries will be managed by ExternalDNS and cert-manager.

```sh
read -p 'Cluster domain: ' cluster_domain
```

#### Set an email for Let's Encrypt

We need to specify an email address in order to generate certificates using Let's Encrypt.

```sh
read -p "Let's Encrypt email: " lets_encrypt_email
```

#### Create an access token

ExternalDNS and cert-manager both need write access to the DigitalOcean API in order to write DNS records.
ExternalDNS for obvious reasons, and cert-manager in order to satisfy [DNS01 challenges](https://cert-manager.io/docs/configuration/acme/dns01/).
We recommend creating one called `do-k8s ($cluster_id)`.

```sh
read -rsp "Access token for 'do-k8s ($cluster_id)': " digitalocean_api_token \
  && echo
```

### Installation

Now that we have configured things, we can deploy everything using the included Helm chart.

```sh
helm dependency update ./chart
helm install do-k8s ./chart \
  --namespace do-k8s \
  --create-namespace \
  --set certIssuers.acme.email=$lets_encrypt_email \
  --set certIssuers.acme.dnsZones[0]=$cluster_domain \
  --set digitalocean.apiToken=$digitalocean_api_token \
  --set external-dns.txtOwnerId=$cluster_id \
  --set external-dns.domainFilters[0]=$cluster_domain \
  --set "external-dns.digitalocean.secretName=do-k8s"
```

This will deploy a functional do-k8s setup into a `do-k8s` namespace based on the configuration set in environment variables above.
You can override the following values:

- `certIssuers.acme.email`: The email to for the do-k8s Let's Encrypt cluster issuers.
- `certIssuers.acme.dnsZones`: The DNS zones the do-k8s cluster issuers should support.
  The value in the command above uses the `$cluster_domain`.
- `digitalocean.apiToken`: A DigitalOcean API token with write access.
- `cert-manager.*`: See the [cert-manager chart documentation](https://hub.helm.sh/charts/jetstack/cert-manager).
- `external-dns.*`: See the [external-dns chart documentation](https://hub.helm.sh/charts/bitnami/external-dns).
- `ingress-nginx.*`: See the [ingress-nginx chart documentation](https://github.com/kubernetes/ingress-nginx/tree/master/charts/ingress-nginx).

**Note:** due to limitations when installing CRDs with Helm, the release name and namespace must be `do-k8s` (or updated manually in `chart/crds/cert-manager.yaml`).

### Test

You can check that everything has settled using:

```sh
kubectl --namespace do-k8s get all
```

In particular, it may take a while for the `do-k8s-ingress-nginx-controller` service's `EXTERNAL-IP` to resolve from `PENDING`.
Once it has, you can launch a trivial deployment that uses ingress with DNS and TLS:

```sh
cat examples/test.yaml \
  | sed "s/\$cluster_domain/$cluster_domain/" \
  | kubectl apply -f -
```

Wait for the certificate to become ready by checking:

```sh
kubectl get certificates --namespace test
```

Once it is, you should be able to hit the test endpoint:

```sh
$ curl https://test.connec.co.uk/
OK
```

Once you're satisfied, delete the test with:

```sh
cat examples/test.yaml \
  | sed "s/\$cluster_domain/$cluster_domain/" \
  | kubectl delete -f -
```

And you're done.
