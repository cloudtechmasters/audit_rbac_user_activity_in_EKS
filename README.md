**Audit Role Base Access Control (Service Account based) user activity in EKS**


Assumption:

You already have EKS cluster setup done.

**Steps:**
       
 1.  Create test deployment and service on EKS.
 2.  Create SA for test user with read only view
 3.  Test queries with test user kubeconfig
 4.  Enable Audit log from AWS console
 5.  Monitor service account activities in cloudwatch ingsights
 


**1. Create test deployment and service on EKS**

       # kubectl apply -f springboot-deployment.yaml 
       deployment.apps/springboot created
      
       # kubectl apply -f springboot-service.yaml 
       service/springboot created
      
       # kubectl get all  <br /> 
       NAME                             READY   STATUS    RESTARTS   AGE 
       pod/springboot-db6684d7b-2p54w   1/1     Running   0          73m
       pod/springboot-db6684d7b-pw5fk   1/1     Running   0          73m
       pod/springboot-db6684d7b-znfd4   1/1     Running   0          73m
      
       NAME                 TYPE           CLUSTER-IP       EXTERNAL-IP       PORT(S)           AGE 
       service/kubernetes   ClusterIP      10.100.0.1       <none>            443/TCP           100m
       service/springboot   LoadBalancer   10.100.148.110   a09c07ff01c804cfe9aa.us-east-2.elb.amazonaws.com   33333:31563/TCP   72m 
      
       NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
       deployment.apps/springboot   3/3     3            3           73m
      
       NAME                                   DESIRED   CURRENT   READY   AGE
       replicaset.apps/springboot-db6684d7b   3         3         3       73m
     
2. Create SA for test user with read only view

 We are creating a clusterrole which will have only read permission on obejcts in default namespace. (cluster view role)

       # kubectl create sa test
        serviceaccount/test created

        service_account=test
        namespace=default
        eks_cluster_name=eksdemo
        secret_token_name=$(kubectl get serviceAccounts ${service_account} --namespace "${namespace_name}" -ojsonpath='{.secrets[0].name}')
        account_token=$(kubectl get secrets ${secret_token_name} --namespace "${namespace_name}" -ojsonpath='{.data.token}' | base64 -d)
        server=$(kubectl config view --flatten --minify -ojsonpath='{.clusters[0].cluster.server}')
        certificate_authority_data=$(kubectl config view --flatten --minify -ojsonpath='{.clusters[0].cluster.certificate-authority-data}')


Generate kubeconfig for Test user

            cat <<EOF > /tmp/eks-${service_account}-config.yaml
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
    
    
**# cat /tmp/eks-test.yaml** 

              apiVersion: v1
              clusters:
              - cluster:
                  certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCZJQ0FURS0tLS0tCg==
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
                  token: eyJhbGciOiJSUzI1NiIsImtpZCI6IIpRyVSQIcGmA


3. Test queries with test user kubeconfig

              # k config get-contexts
                 CURRENT   NAME                                              CLUSTER                       AUTHINFO                                          NAMESPACE
                 *         i-067e49e18150ff9fd@eksdemo.us-east-2.eksctl.io   eksdemo.us-east-2.eksctl.io   i-067e49e18150ff9fd@eksdemo.us-east-2.eksctl.io   


              # export KUBECONFIG=/tmp/eks-test.yaml
              
              # k config get-contexts
              CURRENT   NAME               CLUSTER   AUTHINFO       NAMESPACE
              *         eksdemo--cluster   eksdemo   eksdemo-user 

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


4. Enable Audit log from AWS console 

Goto EKS cluster --> Logging --> Manage Logging --> Enable Audit

<img width="1481" alt="image" src="https://user-images.githubusercontent.com/68885738/182895565-82608e13-7932-4b20-82f3-eb3da194d558.png">

5. Monitor service account activities in cloudwatch ingsights

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

