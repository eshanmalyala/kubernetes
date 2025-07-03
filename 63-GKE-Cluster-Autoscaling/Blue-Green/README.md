# kubectl commands 
```  kubectl apply -f blue-deployment.yaml && blue-service.yaml```
check for the deployment status 

``` kubectl apply -f gree-deployment.yaml  ``` deploy the green deployment 

``` kubectl get pods -i version=green ```  check for the green pod status 
# Switch traffic from Blue to Green
To switch traffic to the Green version, simply update the Service to point to the Green pods.

``` kubectl port-forward <green pod id>  8080:3000 ```   http://localhost:8080 to see if it works as expected.

 in the service deployment yaml modify the selector 
 
selector:
  version: blue  

  to 

  selector:
  version: green 


``` kubectl apply -f service.yaml```

# Roll Back
If something goes wrong you can easly roll back from green to blue 
selector:
  version: blue

  ``` kubectl apply -f service.yaml ```

# Automate with kubectl patch
To switch from Blue to Green via the command line (without editing YAML), you can use:

```kubectl patch service webapp-service -p '{"spec":{"selector":{"version":"green"}}}'```

```kubectl patch service webapp-service -p '{"spec":{"selector":{"version":"blue"}}}'```

Refere the URL : https://www.tutorialspoint.com/kubernetes/kubernetes_blue_green_deployments.htm
