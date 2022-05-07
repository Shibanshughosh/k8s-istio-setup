# k8s-istio-setup
Istio setup instructions for demo profile

#####Download Istio

curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.13.3 sh -
cd istio-1.13.3
export PATH=$PWD/bin:$PATH

#####Install Istio

	istioctl install --set profile=demo -y
	kubectl label namespace default istio-injection=enabled
	kubectl label namespace service-mesh-demo istio-injection=enabled # For any custom namespace 

##Verify label is updated to add sidecars in the selected namespaces

	kubectl get ns --show-labels

	---------For already installed apps, we need to do a restart of the resource (Deployment) for the sidecar injection to happen
  
  	kubectl -n service-mesh-demo rollout restart deploy node-app
  	kubectl -n default rollout restart deploy devsecops
  	kubectl -n default rollout restart deploy node-app

  ---------Understanding namespaces and DNS
  When you create a Service, it creates a corresponding DNS entry. This entry is of the form <service-name>.<namespace-name>.svc.cluster.local, which means that if 
  a container uses <service-name> it will resolve to the service which is local to a namespace. This is useful for using the same configuration across multiple    
  namespaces such as Development, Staging and Production. If you want to reach across namespaces, you need to use the fully qualified domain name (FQDN).


#####Deploy Sample application
  
	kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
	kubectl get services

	kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"


#####Open application to outside traffic

	  kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
	  istioctl analyze
  
#####Determining the ingress IP and ports

  kubectl get svc istio-ingressgateway -n istio-system

#If the EXTERNAL-IP value is set, your environment has an external load balancer that you can use for the ingress gateway. If the EXTERNAL-IP value is <none> (or 
perpetually #<pending>), your environment does not provide an external load balancer for the ingress gateway. In this case, you can access the gateway using the 
service’s node port.
  
  
#####Set the ingress IP and ports

	  export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
	  export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
	  export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')



-----------Follow these instructions if your environment does not have an external load balancer and choose a node port instead. 
      
	export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
      	export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')

#####Set GATEWAY_URL: environment variable

  export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT  
  
#####Ensure an IP and port were successfully assigned to the env variable

  echo "$GATEWAY_URL"  
  
  
#####Verify external access

  echo "http://$GATEWAY_URL/productpage"

  Paste the output from the previous command into your web browser and confirm that the Bookinfo product page is displayed.  
  
  
#####View the dashboard - intall the addons 
  
    Install Kiali and the other addons and wait for them to be deployed.

    kubectl apply -f samples/addons
    kubectl rollout status deployment/kiali -n istio-system

    #Check the deployments in istio-system ns
    kubectl get all -n istio-system  

      -----In demo environment change the svc type for kiali and other tools to NodePort (Only if Load Balancer setup is not there in the environment)
      -----#This should not be done in production environment

     kubectl -n istio-system edit svc kiali  # Change type from ClusterIP to NodePort

      -----#Kiali has 3 purpose----------
       -Identify services that are part of Istio Service Mesh
       -Display connections between the services
       -Identify how the services are performing (higlight bottlenecks)
	   
#####Access the Kiali dashboard

  istioctl dashboard kiali

  In the left navigation menu, select Graph and in the Namespace drop down, select default.

  To see trace data, you must send requests to your service. The number of requests depends on Istio’s sampling rate. You set this rate when you install Istio. The 
  default sampling rate is 1%. You need to send at least 100 requests before the first trace is visible. To send a 100 requests to the productpage service, use the 
  following command:

  for i in $(seq 1 100); do curl -s -o /dev/null "http://$GATEWAY_URL/productpage"; done
     -----------If above does not work, try below-------------------
      while true; do curl -s -o /dev/null "http://$GATEWAY_URL/productpage"; echo ;sleep 1; done
      while true; do curl -s -o /dev/null "http://143.244.142.148:32750/productpage"; echo ;sleep 1; done   ###host IP and nodeport

  The Kiali dashboard shows an overview of your mesh with the relationships between the services in the Bookinfo sample application. It also provides filters to 
  visualize the traffic flow.  

  
  
#####Istio Peer Authentication (enable/disable mTLS)

vi peerauth.yaml
---------------------------
	apiVersion: security.istio.io/v1beta1
	kind: PeerAuthentication
	metadata:
	  name: default
	  namespace: istio-system
	spec:
	  mtls:
		mode: DISABLE                       
		

    -----Other modes-----------
        DISABLE - No mTLS
        PERMISSIVE - mTLS for Pods under the mesh, plain text for others		
        STRICT - mTLS for all		
    ---------------------------

    kubectl apply -f peerauth.yaml
    kubectl get pa -n istio-system
    kubectl edit pa -n istio-system      # change mode as desired STRICT


  Once mode set to STRICT, request cannot be sent using curl and clusterIP since it soes not send the required certificates
  So we need to create the Ingress gateway (IG) to take care of this issue. The request can be sent from Browser and IG will take care of the certs

  ###Ksniff - kubectl plugin that utilize tcpdump and wireshark to start a remote capture on any pod in K8s cluster #######
      ----Ksniff Not production ready yet------  
  
#####Setup Ingress Gateway

-------Check istio CRDs -samle output---------
     
      Kubectl get crd
     
      NAME                                       CREATED AT
      authorizationpolicies.security.istio.io    2022-04-24T07:48:57Z
      destinationrules.networking.istio.io       2022-04-24T07:48:57Z
      envoyfilters.networking.istio.io           2022-04-24T07:48:57Z
      gateways.networking.istio.io               2022-04-24T07:48:57Z
      istiooperators.install.istio.io            2022-04-24T07:48:57Z
      peerauthentications.security.istio.io      2022-04-24T07:48:57Z
      proxyconfigs.networking.istio.io           2022-04-24T07:48:57Z
      requestauthentications.security.istio.io   2022-04-24T07:48:57Z
      serviceentries.networking.istio.io         2022-04-24T07:48:57Z
      sidecars.networking.istio.io               2022-04-24T07:48:57Z
      telemetries.telemetry.istio.io             2022-04-24T07:48:57Z
      virtualservices.networking.istio.io        2022-04-24T07:48:57Z
      wasmplugins.extensions.istio.io            2022-04-24T07:48:57Z
      workloadentries.networking.istio.io        2022-04-24T07:48:57Z
      workloadgroups.networking.istio.io         2022-04-24T07:48:57Z

------ISTIO GW and VS (istio-gw-vs.yaml) [To be setp in the iindividual namespaces]
  
  The Gateway specification describes the L4-L6 properties of a load balancer. 
  A VirtualService can then be bound to a gateway to control the forwarding of traffic arriving at a particular host or gateway port.
  
  
  
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: devsecops-gateway
  namespace: default
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"

---

apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: devsecops-numeric
  namespace: default
spec:
  hosts:
  - "*"
  gateways:
  - devsecops-gateway
  http:
  - match:
    - uri:
        prefix: /increment
    - uri:
        exact: /
    route:
    - destination:
        host: devsecops-svc
        port:
          number: 8080




   -----------------If the gateway is in different namespace, then update VS accordingly 
   -----------------The same principle can be applied for multiple appications on different namespaces (multiple VS mapped to one GW

		apiVersion: networking.istio.io/v1alpha3
		kind: VirtualService
		metadata:
		  name: helloworld-virtualservice
		  namespace: helloworld
		spec:
		  gateways:
		  - default/ingressgateway



	----------------------So the above VS for devsecops becomes
	
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: devsecops-numeric
  namespace: default
spec:
  hosts:
  - "*"
  gateways:
  - service-mesh-demo/bookinfo-gateway
  http:
  - match:
    - uri:
        prefix: /increment
    - uri:
        exact: /
    route:
    - destination:
        host: devsecops-svc
        port:
          number: 8080

--------------------------------------------------------------------------------

    kubectl appply -f istio-gw-vs.yaml # creates in the default ns
    kubectl get vs,gateway -n default\
  
  
#####To use Ingress Gateway created above - Need to use Istio GW svc

  check the Ingresss GW svc

  ------------example--------------------
    root@master:~# k get svc -n istio-system | grep  -i gateway
    istio-egressgateway    ClusterIP      10.106.31.152    <none>        80/TCP,443/TCP                                                               25h
    istio-ingressgateway   LoadBalancer   10.97.55.104     <pending>     15021:30816/TCP,80:32750/TCP,443:32623/TCP,31400:32008/TCP,15443:31909/TCP   25h

  ------Since I dont have a LB, will make use of NodePort. The ingress GW is setup on port 80, we can use the 32750 NodePort to access    

    curl localhost:32750/
    root@master:~# curl localhost:32750/
    Kubernetes DevSecOps

    root@master:~# curl localhost:32750/increment/23
    24


 -----remove the below temporarily from Vs to test the routing----------
  
    - uri:
          exact: /
 -------------------------------------------------------------------------		
		
    root@master:~# curl localhost:32750/ -v      # -v verbose option 		
    This will not able to connect and provide the output 

    restore the uri now
  
  
  
######Now check the traffic with PA - STRICT and Gateway setup done


##Generate traffic - change the Cluster IP and Port  
while true; do  curl 10.107.137.202:8080/increment/99; echo ;sleep 1; done  

##After changing the above traffic generator for aligning with Istio GW, below is the updated command
while true; do curl localhost:32750/increment/99; echo ;sleep 1; done

#check the PA mode 
kubectl get pa -n istio-system  OR kubectl get pa -A
  
  
  
  
  
  
  
  
  
  
  
