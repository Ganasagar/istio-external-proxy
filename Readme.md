# Istio-mtls-install

# Setup Istio with MKE


The Configure an Egress Gateway example shows how to direct traffic to external services from your mesh via an Istio edge component called Egress Gateway. However, some cases require an external, legacy (non-Istio) HTTPS proxy to access external services. For example, your company may already have such a proxy in place and all the applications within the organization may be required to direct their traffic through it.

- This task asssumes that you are have istio 1.12.2 up and running with istio-enalbed for default namespace


### Test access to https site through Corporate proxy and non-istio-namespace
1. Create an external namespace and a sleep pods in it so that we can validate traffic external to istio can access world using Corporate proxy  
```
kubectl create namespace external

kubectl apply -n external -f samples/sleep/sleep.yaml

```

2. Create vars for proxy IP and proxy Port of the corporate proxy
```bash
export PROXY_IP=<IP_ADDRESS_OF_PROXY>

export PROXY_PORT=<PROXY_PORT>
```

3. Make a call to the outside world using the corporate proxy from a non-istio enabled namespace.
Note: we use proxy info as flags for curl command.
```bash
kubectl exec -it $(kubectl get pod -n external -l app=sleep -o jsonpath={.items..metadata.name}) -n external -- sh -c "HTTPS_PROXY=$PROXY_IP:$PROXY_PORT curl https://en.wikipedia.org/wiki/Main_Page" | grep -o "<title>.*</title>"

#output should be something like below
<title>Wikipedia, the free encyclopedia</title>

```
Now we know that access to our outside world works using our corporate proxy no issues. Now lets validate the same using Istio. 

### External https access through Istio's mesh 

- istio is not aware of level 7 traffic in this scenario 
- istio just creates a tcp tunnel and forwards the traffic to corporate traffic.
- Since we disabled global egress to REGISTER_ONLY direct access like previous sections wont work even if we add them in pods like previous section
- we would add the proxy to the mesh as a SE and include the proxy-details of the SE in the application pods and env's PROXY_IP=<> and PROXY_PORT=<>
- This method only captures SNI and Source pod in k8's world
- Note that in HTTPS all the HTTP-related information like method, URL path, response code, is encrypted so Istio cannot see and cannot monitor that information for HTTPS

1. Switch to default ns and Run the sleep pod in default which we would use for poking around from the istio mes
```bash
kubectl apply -f samples/sleep/sleep.yaml
```

2. Once we know the pod is up and running, lets export some vars so that we reduce the hassle to exec command using sleep pod.
```bash
export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})

```

3. Let's check what's the global policy for outbound traffic in the mesh.
To demonstrate the controlled way of enabling access to external services, you need to change the global.outboundTrafficPolicy.mode option from the ALLOW_ANY mode to the REGISTRY_ONLY mode.
```
#Command to test is global rule is allow any 
kubectl get configmap istio -n istio-system -o yaml | grep -o "mode: ALLOW_ANY"

#Command to set rules to allow only registered service in the mesh 
kubectl get configmap istio -n istio-system -o yaml | sed 's/mode: ALLOW_ANY/mode: REGISTRY_ONLY/g' | kubectl replace -n istio-system -f -

#Flag to during insatallation if desired
--set global.outboundTrafficPolicy.mode=REGISTRY_ONLY
```
4. Test to see if you can make calls to outside world without a registered service entry 
```
kubectl exec -it $SOURCE_POD -c sleep -- sh -c "HTTPS_PROXY=$PROXY_IP:$PROXY_PORT curl https://en.wikipedia.org/wiki/Main_Page" | grep -o "<title>.*</title>"
```

5. Define a TCP (not HTTP!) Service Entry for the HTTPS proxy. Although applications use the HTTP CONNECT method to establish connections with HTTPS proxies, you must configure the proxy for TCP traffic, instead of HTTP. Once the connection is established, the proxy simply acts as a TCP tunnel.
```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: proxy
spec:
  hosts:
  - my-company-proxy.com # ignored
  addresses:
  - $PROXY_IP/32
  ports:
  - number: $PROXY_PORT
    name: tcp
    protocol: TCP
  location: MESH_EXTERNAL
EOF
```
6. Now lets make calls to the outside world using service entry that we defined which refernces the corporate proxy. Send a request from the sleep pod in the default namespace. Because the sleep pod has a sidecar, Istio controls its traffic.
```
kubectl exec -it $SOURCE_POD -c sleep -- sh -c "HTTPS_PROXY=$PROXY_IP:$PROXY_PORT curl https://en.wikipedia.org/wiki/Main_Page" | grep -o "<title>.*</title>"
```
7. Check the Istio sidecar proxy’s logs of the sleep pod for your request
```
kubectl logs $SOURCE_POD -c istio-proxy

#You should see something like below
[2018-12-07T10:38:02.841Z] "- - -" 0 - 702 87599 92 - "-" "-" "-" "-" "172.30.109.95:3128" outbound|3128||my-company-proxy.com 172.30.230.52:44478 172.30.109.95:3128 172.30.230.52:44476 -
```

8. Check the access log of the proxy for your request
```
kubectl exec -it $(kubectl get pod -n external -l app=squid -o jsonpath={.items..metadata.name}) -n external -- tail -f /var/log/squid/access.log

#You should see something like this. 
1544160065.248    228 172.30.109.89 TCP_TUNNEL/200 87633 CONNECT en.wikipedia.org:443 - HIER_DIRECT/91.198.174.192 -
```
####NOTE
- You must not create service entries for the external services you access through the external proxy, like wikipedia.org
- This is because from Istio’s point of view the requests are sent to the external proxy only
- Istio is not aware of the fact that the external proxy forwards the requests further.
- You should add k8s network policies to influence non-istio traffic in the cluster
