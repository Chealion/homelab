Let's set up a basic echo app with Cilium's Gateway API using both Node IPAM and L2!

1. Create your L2 configuration

https://docs.cilium.io/en/stable/network/l2-announcements

Configure Cilium (`l2announcements.enabled=true`), and then see l2.yaml to create the l2 resources.

You now have IPs available for a load balancer to leverage. Because we have no selectors set up as seen in the upstream docs, it will be the greedy default provider.

2. Create your Node IPAM configuration

https://docs.cilium.io/en/stable/network/node-ipam/

Configure Cilium (`nodeIPAM.enabled=true`).

To leverage this one, we will need to make sure our LoadBalancer service is using the loadBalancerClass of `io.cilium/node`.

3. Create our deployments

To make things easy - we'll use an available HTTP echo server available on GitHub: https://ealenn.github.io/Echo-Server/ 

We'll create one version for the l2 deployment, and another for the node IPAM deployment.

    kubectl apply -f deployments.yaml

You can check out the pods with `kubectl get pods` at this point to ensure they are running.

4. Deploy our gateways and HTTP routes

Next we'll deploy the respective gateways, and HTTP routes for each deployment. We need to create a second gateway class to use the node ipam config instead of the default.

    kubectl apply -f gateways.yaml
    kubectl apply -f routes.yaml

We can look to confirm that they're working using:

    kubectl get gateway
    kubectl get httproute

We should see 1 attached route to each gateway, and each gateway should have an external IP. The Node IPAM will list all of the available nodes.

    kubectl get gateway -o yaml | grep ttach 

5. Deploy our services

Next we'll deploy our services

    kubectl apply -f services.yaml

6. Confirm connectivity  

Grab the external gateway IPs and then we'll curl to echo back the info we want:

    L2GATEWAY=$(kubectl get gateway my-l2-gateway -o json | jq -r '.status.addresses[0].value')
    NODEGATEWAY=$(kubectl get gateway my-node-gateway -o json | jq -r '.status.addresses[0].value')
    curl -v http://${L2GATEWAY}/ | jq '.environment.HOSTNAME'
    curl -v http://${NODEGATEWAY}/ | jq '.environment.HOSTNAME'


NOTE: As of 1.18, it's not possible to enable access logs in Envoy which makes debugging a little harder to note that your HTTPRoute is not pointed to a service that exists.

## Resources
https://ealenn.github.io/Echo-Server/pages/quick-start/kubernetes.html
