# audit_rbac_user_activity_in_EKS

Audit Role Base Access Control (Service Account based) user activity in EKS
============================================================================

Assumption:

You already have EKS cluster setup done.

steps:
1. Create test deployment and service on EKS.
2. Create SA for test user with read only view
3. Enable Audit log from AWS console
4. Monitor ser activities in cloudwatch ingsights 


1. Create test deployment and service on EKS.

      # kubectl apply -f springboot-deployment.yaml 
      deployment.apps/springboot created

      # kubectl apply -f springboot-service.yaml 
      service/springboot created

      # kubectl get all  <br /> 
      NAME                             READY   STATUS    RESTARTS   AGE <br /> 
      pod/springboot-db6684d7b-2p54w   1/1     Running   0          73m <br />
      pod/springboot-db6684d7b-pw5fk   1/1     Running   0          73m <br />
      pod/springboot-db6684d7b-znfd4   1/1     Running   0          73m <br />

      NAME                 TYPE           CLUSTER-IP       EXTERNAL-IP  <br />                                                              PORT(S)           AGE <br/>
      service/kubernetes   ClusterIP      10.100.0.1       <none>                                                                    443/TCP           100m  <br />
      service/springboot   LoadBalancer   10.100.148.110   a09c07ff01c804cfe9aa5c996541f6d6-1367374000.us-east-2.elb.amazonaws.com   33333:31563/TCP   72m   <br />

      NAME                         READY   UP-TO-DATE   AVAILABLE   AGE <br /> 
      deployment.apps/springboot   3/3     3            3           73m

      NAME                                   DESIRED   CURRENT   READY   AGE <br />
      replicaset.apps/springboot-db6684d7b   3         3         3       73m
     
2. Create SA for test user with read only view

    # kubectl create sa test
    
    service_account=test <br />
    namespace=default <br />
    eks_cluster_name=eksdemo <br />
    secret_token_name=$(kubectl get serviceAccounts ${service_account} --namespace "${namespace_name}" -ojsonpath='{.secrets[0].name}') <br />
    account_token=$(kubectl get secrets ${secret_token_name} --namespace "${namespace_name}" -ojsonpath='{.data.token}' | base64 -d) <br />
    server=$(kubectl config view --flatten --minify -ojsonpath='{.clusters[0].cluster.server}') <br />
    certificate_authority_data=$(kubectl config view --flatten --minify -ojsonpath='{.clusters[0].cluster.certificate-authority-data}') <br />


    cat <<EOF > /tmp/eks-${service_account}.yaml
    apiVersion: v1
    clusters:
    - cluster:
        certificate-authority-data: ${certificate_authority_data}
        server: ${server}
      name: ${eks_cluster_name}
    contexts:
    - context:
        cluster: ${eks_cluster_name}
        user: ${eks_cluster_name}-user
        namespace: ${namespace_name}
      name: ${eks_cluster_name}-${namespace_name}-cluster
    current-context: ${eks_cluster_name}-${namespace_name}-cluster
    kind: Config
    preferences: {}
    users:
    - name: ${eks_cluster_name}-user
      user:
        token: ${account_token}
    EOF

    
    
    # cat /tmp/eks-test.yaml 
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMvakNDQWVhZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJeU1EZ3dOREUwTkRZek1Gb1hEVE15TURnd01URTBORFl6TUZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTmtMCkxzZ0VXU0poOXE4MmtWQ2xxci9DSDh2eXhlTDVZY2dTQSt6QVpnbXYzVWJBS3ZYZ3ExL1I1dlQ2b0lUWEs1SjgKMlVyYTNuR05vQjc2TTI2My9tUlF3eEgvUDVOUkhqZGxHeWptcGt0SW1NekM4RmlOZWZoU3J6NnJ2UGd5NVRPSQpiRWt6cTNyZzVaakppVnNzU2U4NE83bHNjZnNFOFdHWUIyZWtLNTY4Z2R6Yzgwam52YVNWK2Rsc3BjaWtYVDczCm5KdVQ4Y0FrRDk5bWpsUi9MQmt1RVZRREFEY1V5UlVBaDlzYTdtajlwK1ZPVEM4S3pXeENYZFM5VTl4aFcwTTAKZ3ZmTlRUdXRYMEdIdW5mOVJGajAxTG5CVytqMC9NUFZqRHdYWW9UVlRpRUJDK01BNVprRGFNSlRhOGZoWlpIYwpIZGMwMEdzZ2xYa29PUDV4dk5FQ0F3RUFBYU5aTUZjd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZLa0FtWGlvWWNlV3BLNVBUdmpYUVdRZE5VTjZNQlVHQTFVZEVRUU8KTUF5Q0NtdDFZbVZ5Ym1WMFpYTXdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBQ0wyZ3F6TmhCWWlRanpudzVVRApRY1F6bzBHdTh5Z1o2VW8vd0k5d3UzUk8zeFVBUEVLdkxZd1V2UDJUMXQwc1J0V3A0QURlM2lGV2RpZllxOHFPCnVEUGZ1YWhqVEFtM0l5elBGL2JuTjF1ajlsWU4yVFo0bFh3MG9xTXhMeVBsSElxMlFBUFd2cmgrSmJ6U2hRUFYKL0VrZWNWTTFENFQ4WXJ5R2JadWppaXhkK1hxOEJQZ014OU1GY3Blc3BkbTNIdHpEMHZhY3FxTytndXRpV2RmaQp2dVE3MGZBZWRnZDVtZEMyL1cxRFNJckd6a0YyY2pjQWw4RDNQbGNabmJLNk10ck0rbkJrYTRldE50YU9DTzl6CnQ0SzljcElmeUYwcjR5S1JmSUtVRTloNldvY1d0eVZ4UllOb2d0bUQrQ3Z0ajkyNStNMjNyblJITHBLMnNDdjAKTUIwPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://3CB848FBCC241E283D5A2E66844C7B33.gr7.us-east-2.eks.amazonaws.com
  name: eksdemo
contexts:
- context:
    cluster: eksdemo
    user: eksdemo-user
    namespace: 
  name: eksdemo--cluster
current-context: eksdemo--cluster
kind: Config
preferences: {}
users:
- name: eksdemo-user
  user:
    token: eyJhbGciOiJSUzI1NiIsImtpZCI6IlZkQzlpODljZ19UWUZMeEVGa0ZhQXByRWtqWGVxSVVwNFUzRDdRdF9QdDAifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6InRlc3QtdG9rZW4tN25sYmgiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoidGVzdCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImExYTEwOWYzLTUzNjEtNDU0Yi05YzQ2LTg4YTk3YjlmNjM4MiIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OnRlc3QifQ.TRtkDLV49oYV-DDvdCTcYvq1acX-dpIQpGN3dXde1cIgcKXL5gOfChwlhGIv8GeNhBrFowbXzm1TAaWZgAoK7-woegj2xBq-2PbK4nue5P6JwQCO0Y_UB06h37hQmeOoFVY4WyilSGHCKOXEU4slaBjzMSHaYPJgSoRJ6cD61hCAxPibTPWzuoPT2phHguqRZixuX7pTXgoT4Wiklbb4xhGvjyUMyzOHONithlIn750-2QdIRDB4vWn433YmxABgbaU3EbKc4EatgNq5HquxBjYSjVSn22nmBjZb3JLYsorYCBQcDj-4aL8hjSPm-JNmAeNoGGk4LgIpRyVSQIcGmA


3. Enable Audit log from AWS console

      # kubectl get all
      NAME                             READY   STATUS    RESTARTS   AGE
      pod/springboot-db6684d7b-2p54w   1/1     Running   0          72m
      pod/springboot-db6684d7b-pw5fk   1/1     Running   0          72m
      pod/springboot-db6684d7b-znfd4   1/1     Running   0          72m

      NAME                 TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)           AGE
      service/kubernetes   ClusterIP      10.100.0.1       <none>                                                                    443/TCP           100m
      service/springboot   LoadBalancer   10.100.148.110   a09c07ff01c804cfe9aa5c996541f6d6-1367374000.us-east-2.elb.amazonaws.com   33333:31563/TCP   72m

      NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
      deployment.apps/springboot   3/3     3            3           72m
      Error from server (Forbidden): replicasets.apps is forbidden: User "system:serviceaccount:default:test" cannot list resource "replicasets" in API group "apps" in the namespace "default"
      Error from server (Forbidden): statefulsets.apps is forbidden: User "system:serviceaccount:default:test" cannot list resource "statefulsets" in API group "apps" in the namespace "default"
      Error from server (Forbidden): horizontalpodautoscalers.autoscaling is forbidden: User "system:serviceaccount:default:test" cannot list resource "horizontalpodautoscalers" in API group "autoscaling" in the namespace "default"
      Error from server (Forbidden): cronjobs.batch is forbidden: User "system:serviceaccount:default:test" cannot list resource "cronjobs" in API group "batch" in the namespace "default"
      Error from server (Forbidden): jobs.batch is forbidden: User "system:serviceaccount:default:test" cannot list resource "jobs" in API group "batch" in the namespace "default"


4. Monitor ser activities in cloudwatch ingsights 

Goto EKS cluster --> Logging --> Manage Logging --> Enable Audit

<img width="1481" alt="image" src="https://user-images.githubusercontent.com/68885738/182895565-82608e13-7932-4b20-82f3-eb3da194d558.png">

Once you enable the Audit, it will create the Log Group "/aws/eks/cluster-name/cluster" (replace cluster name with your cluster name.) 

ex. /aws/eks/eksdemo/cluster

<img width="1049" alt="image" src="https://user-images.githubusercontent.com/68885738/182897376-0e43b090-de44-4343-9ee8-714fe0dfdf36.png">

**sample query:**

fields @logStream, @timestamp, user.username, responseStatus.code, responseStatus.reason, responseStatus.status, responseObject.message
| filter @logStream like /^kube-apiserver-audit/
| filter user.username like "system:serviceaccount:default:test"
| sort count desc

<img width="1460" alt="image" src="https://user-images.githubusercontent.com/68885738/182898786-17beb725-70dc-49ae-b69e-f843dff7507b.png">


Reference : https://aws.amazon.com/premiumsupport/knowledge-center/eks-get-control-plane-logs/

