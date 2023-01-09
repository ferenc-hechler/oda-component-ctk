# following the video and description


## make node js code executable

```
npm install
```

fails with the following message:

```
found 52 vulnerabilities (4 low, 14 moderate, 30 high, 4 critical)
  run `npm audit fix` to fix them, or `npm audit` for details
```

```
npm audit fix

npm audit fix --force

npm fund
```

## static validation

```
npm run L1-static .\v1alpha3.component.yaml
```

success, 23 passing   :-)

## dynamic validation

### deploy component


set default namespace to components

```
kubectl config set-context --current --namespace=components
```

```
$ kubectl apply -n components -f .\v1alpha3.component.yaml

persistentvolumeclaim/r1-mongodb-pv-claim created
service/r1-mongodb created
service/r1-partyroleapi created
service/r1-prodcatapi created
deployment.apps/r1-mongodb created
deployment.apps/r1-partyroleapi created
deployment.apps/r1-prodcatapi created
job.batch/r1-roleinitialization created
component.oda.tmforum.org/r1-productcatalog created
```

wait until deployment is finished

```
kubectl get components -n components
NAME                EXPOSED_APIS   DEVELOPER_UI   DEPLOYMENT_STATUS
r1-productcatalog                                 In-Progress-CompCon
```

status should go into "Complete".

looks like mongodb is not comming up:

```
$ kubectl get pods -n components

NAME                               READY   STATUS    RESTARTS   AGE
r1-mongodb-5d489dcb55-c9t4r        0/1     Pending   0          112s
r1-partyroleapi-5cf5748d4c-4ccx6   0/1     Running   0          112s
r1-prodcatapi-548fb76fcb-nkptj     1/1     Running   0          112s
r1-roleinitialization-xdzwf        1/1     Running   0          112s
```

```
Warning  FailedScheduling  3m56s (x3 over 4m)  default-scheduler  
  0/1 nodes are available: 
    pod has unbound immediate PersistentVolumeClaims. 
      preemption: 0/1 nodes are available: 
        1 No preemption victims found for incoming pod..
```

Storageclass is not empty, it is set to default. So, the claim is pending:

```
kubectl get pvc -n components r1-mongodb-pv-claim
NAME                  STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
r1-mongodb-pv-claim   Pending                                      default        5m56s
```
  
Storageclass should be ommitted or set to "openebs-hostpath".
patched yaml file:

```
...
---
# Source: productcatalog/templates/persistentVolumeClaim-mongodb.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: r1-mongodb-pv-claim
  labels:
    oda.tmforum.org/componentName: r1-productcatalog
spec:
  #storageClassName: default
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

Now the DB pod starts:

```
$ kubectl get pods

NAME                               READY   STATUS      RESTARTS   AGE
r1-mongodb-5d489dcb55-vxprr        1/1     Running     0          87s
r1-partyroleapi-5cf5748d4c-tzhbv   1/1     Running     0          87s
r1-prodcatapi-548fb76fcb-7prbs     1/1     Running     0          87s
r1-roleinitialization-qgccz        0/1     Completed   0          87s
```

Looking at the events there is an error creating VirtualServices

```
5m54s       Error     Logging                 api/r1-productcatalog-productcatalogmanagement   Handler 'apiStatus' failed temporarily: Exception creating virtualService.
```

Errorlog:

```
[2023-01-09 19:23:44,885] APIOperator          [ERROR   ] [updateImplementationStatus/components/r1-prodcatapi] ApiException when calling DiscoveryV1beta1Api->list_namespaced_endpoint_slice: (404)
Reason: Not Found
HTTP response headers: HTTPHeaderDict({'Audit-Id': '3909d8c8-df51-405b-b82c-05a75e33d831', 'Cache-Control': 'no-cache, private', 'Content-Type': 'application/json', 'X-Kubernetes-Pf-Flowschema-Uid': '0db040dc-07a2-4994-b7e4-edb138118973', 'X-Kubernetes-Pf-Prioritylevel-Uid': 'f431a49e-7508-4992-a5b0-95b0cc105987', 'Date': 'Mon, 09 Jan 2023 19:23:44 GMT', 'Content-Length': '174'})
HTTP response body: {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"the server could not find the requested resource","reason":"NotFound","details":{},"code":404}



[2023-01-09 19:23:44,910] APIOperator          [ERROR   ] [updateImplementationStatus/components/r1-partyroleapi] ApiException when calling DiscoveryV1beta1Api->list_namespaced_endpoint_slice: (404)
Reason: Not Found
HTTP response headers: HTTPHeaderDict({'Audit-Id': '63a5dd92-e791-404d-9fea-753fe2788e0e', 'Cache-Control': 'no-cache, private', 'Content-Type': 'application/json', 'X-Kubernetes-Pf-Flowschema-Uid': '0db040dc-07a2-4994-b7e4-edb138118973', 'X-Kubernetes-Pf-Prioritylevel-Uid': 'f431a49e-7508-4992-a5b0-95b0cc105987', 'Date': 'Mon, 09 Jan 2023 19:23:44 GMT', 'Content-Length': '174'})
HTTP response body: {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"the server could not find the requested resource","reason":"NotFound","details":{},"code":404}



[2023-01-09 19:23:44,911] APIOperator          [WARNING ] Exception when calling CoreApi->read_namespaced_service: (404)
Reason: Not Found
HTTP response headers: HTTPHeaderDict({'Audit-Id': '12c78e47-1055-4051-906f-fb109e2f320c', 'Cache-Control': 'no-cache, private', 'Content-Type': 'application/json', 'X-Kubernetes-Pf-Flowschema-Uid': '0db040dc-07a2-4994-b7e4-edb138118973', 'X-Kubernetes-Pf-Prioritylevel-Uid': 'f431a49e-7508-4992-a5b0-95b0cc105987', 'Date': 'Mon, 09 Jan 2023 19:23:44 GMT', 'Content-Length': '216'})
HTTP response body: {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"services \"istio-ingressgateway\" not found","reason":"NotFound","details":{"name":"istio-ingressgateway","kind":"services"},"code":404}



[2023-01-09 19:23:44,916] kopf.objects         [ERROR   ] [components/r1-productcatalog-productcatalogmanagement] Handler 'apiStatus' failed temporarily: Exception getting IstioIngressStatus.
[2023-01-09 19:23:44,957] APIOperator          [WARNING ] Exception when calling CoreApi->read_namespaced_service: (404)
Reason: Not Found
HTTP response headers: HTTPHeaderDict({'Audit-Id': '92a217e8-e9c2-4c6e-891e-f7fd7fadae19', 'Cache-Control': 'no-cache, private', 'Content-Type': 'application/json', 'X-Kubernetes-Pf-Flowschema-Uid': '0db040dc-07a2-4994-b7e4-edb138118973', 'X-Kubernetes-Pf-Prioritylevel-Uid': 'f431a49e-7508-4992-a5b0-95b0cc105987', 'Date': 'Mon, 09 Jan 2023 19:23:44 GMT', 'Content-Length': '216'})
HTTP response body: {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"services \"istio-ingressgateway\" not found","reason":"NotFound","details":{"name":"istio-ingressgateway","kind":"services"},"code":404}
```

So, the problem is, that "DiscoveryV1beta1Api" is now replaced with V1.

```
$ kubectl api-resources | findstr discovery

endpointslices                                 discovery.k8s.io/v1                    true         EndpointSlice 
```

in dockerhub we can see, that the last update for this component was one year ago and 
there is only one version 0.1.5.
So, it looks like no update is available.

