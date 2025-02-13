# Set up a multi-zone deployment

For a description of how multi-zone deployments work in Kuma, see [about multi-zone deployments](../how-multi-zone-works). This page explains how to configure and deploy Kuma in a multi-zone environment:

- Set up the global control plane
- Set up the zone control planes
- Verify control plane connectivity
- Set up cross-zone communication between data plane proxies

## Set up the global control plane

The global control plane must run on a dedicated cluster, and cannot be assigned to a zone.

:::: tabs :options="{ useUrlFragment: false }"
::: tab "Kubernetes"

The global control plane on Kubernetes must reside on its own Kubernetes cluster, to keep its resources separate from the resources the remote control planes create during synchronization.

1.  Run:

    ```bash
    kumactl install control-plane --mode=global | kubectl apply -f -
    ```

1.  Find the external IP and port of the `global-remote-sync` service in the `kuma-system` namespace:

    ```bash
    kubectl get services -n kuma-system
    NAMESPACE     NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                                                                  AGE
    kuma-system   global-remote-sync     LoadBalancer   10.105.9.10     35.226.196.103   5685:30685/TCP                                                           89s
    kuma-system   kuma-control-plane     ClusterIP      10.105.12.133   <none>           5681/TCP,443/TCP,5676/TCP,5677/TCP,5678/TCP,5679/TCP,5682/TCP,5653/UDP   90s
    ```

    In this example the value is `35.226.196.103:5685`. You pass this as the value of `<global-kds-address>` when you set up the remote control planes.

:::
::: tab "Helm"

1.  Set the `controlPlane.mode` value to `global` in the chart (`values.yaml`), then install. On the command line, run:

    ```sh
    helm install kuma --namespace kuma-system --set controlPlane.mode=global kuma/kuma
    ```

    Or you can edit the chart and pass the file to the `helm install kuma` command. To get the default values, run:

    ```sh
    helm show values kuma/kuma
    ```
1.  Find the external IP and port of the `global-remote-sync` service in the `kuma-system` namespace:

    ```bash
    kubectl get services -n kuma-system
    NAMESPACE     NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                                                                  AGE
    kuma-system   global-remote-sync     LoadBalancer   10.105.9.10     35.226.196.103   5685:30685/TCP                                                           89s
    kuma-system   kuma-control-plane     ClusterIP      10.105.12.133   <none>           5681/TCP,443/TCP,5676/TCP,5677/TCP,5678/TCP,5679/TCP,5682/TCP,5653/UDP   90s
    ```

    By default, it's exposed on [port 5685](../networking/networking.md). In this example the value is `35.226.196.103:5685`. You pass this as the value of `<global-kds-address>` when you set up the remote control planes.

:::
::: tab "Universal"

1.  Set up the global control plane, and add the `global` environment variable:

    ```sh
    KUMA_MODE=global kuma-cp run
    ```

:::
::::

## Set up the remote control planes

You need the following values to pass to each remote control plane setup:

- `zone` -- the zone name. An arbitrary string. This value registers the remote control plane with the global control plane.
- `kds-global-address` -- the external IP and port of the global control plane.

:::: tabs :options="{ useUrlFragment: false }"
::: tab "Kubernetes"

1.  On each remote control plane, run:

    ```sh
    kumactl install control-plane \
    --mode=remote \
    --zone=<zone name> \
    --ingress-enabled \
    --kds-global-address grpcs://`<global-kds-address>` | kubectl apply -f -
    ```

    where `zone` is the same value for all remote control planes in the same zone.

:::
::: tab "Helm"

1.  On each remote control plane, run:

    ```bash
    helm install kuma \
    --namespace kuma-system \
    --set controlPlane.mode=remote \
    --set controlPlane.zone=<zone-name> \
    --set ingress.enabled=true \
    --set controlPlane.kdsGlobalAddress=grpcs://<global-kds-address> kuma/kuma
    ```

    where `controlPlane.zone` is the same value for all remote control planes in the same zone.

:::
::: tab "Universal"

1. On each remote control plane, run:

    ```sh
    KUMA_MODE=remote \
    KUMA_MULTIZONE_REMOTE_ZONE=<zone-name> \
    KUMA_MULTIZONE_REMOTE_GLOBAL_ADDRESS=grpcs://<global-kds-address> \
    ./kuma-cp run
    ```

   where `KUMA_MULTIZONE_REMOTE_ZONE` is the same value for all remote control planes in the same zone.

2. Generate the zone ingress token:

   To register the zone ingress with the remote control plane, we need to generate a zone ingress token first

    ```sh
    kumactl generate zone-ingress-token --zone=<zone-name> > /tmp/ingress-token
    ```

   You can also generate the token [with the REST API](../security/zone-ingress-auth.md).

3. Create an `ingress` data plane proxy configuration to allow `kuma-cp` services to be exposed for cross-zone communication:

    ```bash
    echo "type: ZoneIngress
    name: ingress-01
    networking:
      address: 127.0.0.1 # address that is routable within the zone
      port: 10000
      advertisedAddress: 10.0.0.1 # an address which other zones can use to consume this zone-ingress
      advertisedPort: 10000 # a port which other zones can use to consume this zone-ingress" > ingress-dp.yaml
    ```

4. Apply the ingress config, passing the IP address of the remote control plane to `cp-address`:

    ```
    kuma-dp run \
    --proxy-type=ingress \
    --cp-address=https://<kuma-cp-address>:5678 \
    --dataplane-token-file=/tmp/ingress-token \
    --dataplane-file=ingress-dp.yaml
    ```
:::
::::

## Verify control plane connectivity

You can run `kumactl get zones`, or check the list of zones in the web UI for the global control plane, to verify remote control plane connections.

When a remote control plane connects to the global control plane, the `Zone` resource is created automatically in the global control plane.

The Ingress tab of the web UI also lists remote control planes that you deployed with Ingress.

## Set up cross-zone communication

### Enable mTLS

You must [enable mTLS](../policies/mutual-tls.md) for cross-zone communication between services.

Kuma uses the Server Name Indication field, part of the TLS protocol, as a way to pass routing information cross zones. Thus, the mTLS is mandatory to enable cross-zone service communication.

### Zone Ingress requirements

Cross-zone communication between services is available only if Zone Ingress has a public address and public port.

On Kubernetes, Kuma automatically tries to pick up the public address and port. Depending on your load balancing implementation, you might need to wait a few minutes for Kuma to get the address.

### Cross-communication details

:::: tabs :options="{ useUrlFragment: false }"
::: tab "Kubernetes"

To view the list of service names available for cross-zone communication, run:

```bash
kubectl get dataplanes -n echo-example -o yaml | grep kuma.io/service
           kuma.io/service: echo-server_echo-example_svc_1010
```

To consume the example service only within the same Kuma zone, you can run:

```
<kuma-enabled-pod>$ curl http://echo-server:1010
```

To consume the example service across all zones in your Kuma deployment (that is, from endpoints ultimately connecting to the same global control plane), you can run either of:

```
<kuma-enabled-pod>$ curl http://echo-server_echo-example_svc_1010.mesh:80
<kuma-enabled-pod>$ curl http://echo-server.echo-example.svc.1010.mesh:80
```

And if your HTTP clients take the standard default port 80, you can the port value and run either of:

```
<kuma-enabled-pod>$ curl http://echo-server_echo-example_svc_1010.mesh
<kuma-enabled-pod>$ curl http://echo-server.echo-example.svc.1010.mesh
```

Because Kuma on Kubernetes relies on transparent proxy, `kuma-dp` listens on port 80 for all virtual IPs that are assigned to services in the `.mesh` DNS zone. The DNS names are rendered RFC compatible by replacing underscores with dots.
We can configure more flexible setup of hostnames and ports using [Virtual Outbound](../policies/virtual-outbound.md).

:::
::: tab "Universal"

With a hybrid deployment, running in both Kubernetes and Universal mode, the service tag should be the same in both environments (e.g `echo-server_echo-example_svc_1010`):

```yaml
type: Dataplane
mesh: default
name: backend-02 
networking:
  address: 127.0.0.1
  inbound:
  - port: 2010
    servicePort: 1010
    tags:
      kuma.io/service: echo-server_echo-example_svc_1010
```

If the service is only meant to be run Universal, `kuma.io/service` does not have to follow `{name}_{namespace}_svc_{port}` convention.

To consume a distributed service in a Universal deployment, where the application address is `http://localhost:20012`:

```yaml
type: Dataplane
mesh: default
name: web-02 
networking:
  address: 127.0.0.1
  inbound:
  - port: 10000
    servicePort: 10001
    tags:
      kuma.io/service: web
  outbound:
  - port: 20012
    tags:
      kuma.io/service: echo-server_echo-example_svc_1010
```

Alternatively, you can just call `echo-server_echo-example_svc_1010.mesh` without defining `outbound` section if you configure [transparent proxy](../networking/transparent-proxying.md).

:::
::::

The Kuma DNS service format (e.g. `echo-server_kuma-test_svc_1010.mesh`) is a composition of Kubernetes Service Name (`echo-server`),
Namespace (`kuma-test`), a fixed string (`svc`), the service port (`1010`). The service is resolvable in the DNS zone `.mesh` where
the Kuma DNS service is hooked.

### Delete a zone

To delete a `Zone` we must first shut down the corresponding Kuma remote control plane instances. As long as the Remote CP is running this will not be possible, and Kuma returns a validation error like:

```
zone: unable to delete Zone, Remote CP is still connected, please shut it down first
```

When the Remote CP is fully disconnected and shut down, then the `Zone` can be deleted. All corresponding resources (like `Dataplane` and `DataplaneInsight`) will be deleted automatically as well.

:::: tabs :options="{ useUrlFragment: false }"
::: tab "Kubernetes"
```sh
kubectl delete zone zone-1
```
:::
::: tab "Unviersal"
```sh
kumactl delete zone zone-1
```
:::
::::

### Disable a zone

Change the `enabled` property value to `false`:

:::: tabs :options="{ useUrlFragment: false }"
::: tab "Kubernetes"
```yaml
apiVersion: kuma.io/v1alpha1
kind: Zone
metadata:
  name: zone-1
spec:
  enabled: false
```
:::
::: tab "Universal"
```yaml
type: Zone
name: zone-1
spec:
  enabled: false
```
:::
::::

This global control plane to exchange configuration with this zone, the zone's ingress from zone-1 will be deleted from other zone. As a result, the traffic won't be routed to this zone. The zone displays as **Offline** in the GUI and CLI.
