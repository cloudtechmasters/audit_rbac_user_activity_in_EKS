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

