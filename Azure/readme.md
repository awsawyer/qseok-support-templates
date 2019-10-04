# Install Qlik Sense Enterprise on Kubernetes with Azure With Azure Cloud Setup, SSL and Auth0 IDP
This is for a minimal or basic install and is not used for Growth or Enterprise use cases 

Some of the problems include: 
* Use of DEV Mongo DB
* Use of temporary certificates
* Not in multiple availability zones

## Prerequisites 

Create an account with Azure using a personal free account. 
* This will have $200 credit which is enough to follow the steps below.
* https://azure.microsoft.com/en-us/free/

Install Azure CLI on local machine using the following steps. 
* https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-windows

Create an Auth0 Account
* https://auth0.com/signup

Creation of Tiny Cert with DNS Alias
1.	https://www.tinycert.org/
2.	Create an account. 
3.	From with the account, create a New Certificate Authority.

Download VS Code
* https://code.visualstudio.com/ 

Have ```kubectl``` and ```helm``` in the env path of your Administrator machine. 

## Create the K8s environment
This section will go through the steps of creating the Kubernetes envionrment in Microsoft Azure. 

### Azure Login and registration

 Login to Azure
```
    az login
```
 This will throw you to a browser where you login using your Microsoft creditials. Make sure to login. 
  
Register Microsoft Container and Compute Services
```
    az provider register -n Microsoft.ContainerService
    az provider register -n Microsoft.Compute
```

### Create the Resource Group and Kubernetes Cluster

Create a Resource Group
```
    az group create --name qseok-aks --location eastus
```
 
 <details>
    <summary> This should give you a return like this one </summary>
 
```
    {
    "id": "/subscriptions/463ede92-6ea5-1111-801c-ce3f2ce18799/resourceGroups/qseok-aks",
    "location": "eastus",
    "managedBy": null,
    "name": "qseok-aks",
    "properties": {
        "provisioningState": "Succeeded"
    },
    "tags": null,
    "type": "Microsoft.Resources/resourceGroups"
    }
``` 

 </details>

Create a Azure Kubernetes Service Cluster
```
    az aks create --resource-group qseok-aks --name qseok-k8s --node-count 2 --generate-ssh-keys --node-vm-size Standard_DS11_v2
```
 Again, this is a basic instance. Either create with ```--node-count 1 ``` or ```--node-count 2 ```  


### Get Access to your create cluster from your local laptop

Get Cluster Credentials
```
    az aks get-credentials --resource-group qseok-aks --name qseok-k8s
```

<details>
    <summary> The Result should be like this </summary>

```
    {
    "aadProfile": null,
    "addonProfiles": null,
    "agentPoolProfiles": [
        {
        "availabilityZones": null,
        "count": 3,
        "enableAutoScaling": null,
        "maxCount": null,
        "maxPods": 110,
        "minCount": null,
        "name": "nodepool1",
        "orchestratorVersion": "1.13.10",
        "osDiskSizeGb": 100,
        "osType": "Linux",
        "provisioningState": "Succeeded",
        "type": "AvailabilitySet",
        "vmSize": "Standard_DS11_v2",
        "vnetSubnetId": null
        }
    ],
    "apiServerAuthorizedIpRanges": null,
    "dnsPrefix": "qseok-k8s-qseok-aks-363ede",
    "enablePodSecurityPolicy": null,
    "enableRbac": true,
    "fqdn": "qseok-k8s-qseok-aks-363ede-070d86d8.hcp.eastus.azmk8s.io",
    "id": "/subscriptions/363ede92-6ea4-4796-801d-ce32ce18799/resourcegroups/qseok-aks/providers/Microsoft.ContainerService/managedClusters/qseok-k8s",
    "identity": null,
    "kubernetesVersion": "1.13.10",
    "linuxProfile": {
        "adminUsername": "azureuser",
        "ssh": {
        "publicKeys": [
            {
            "keyData": "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC5rA0mVpmy+3xw6XC3ZDLdO7qg30qyfyMM9fFHjsxSIojWYBOpMDSADpUEcEgRzb2O57TiAeg4FEUu6AQZS0oVJ0nBwdTR7HM39MIvG5vdNDc4VrfhRmvrHMeVcNem7x7t4uyht06m2dm80dZ32Waa14X1jDhpaV+DmPA/DtufiP1M0ZbNsVjGI2GW+lneGCE3V+XGTwRbvaRHJwnPZ9lBUB8DhX+tV78sip1nwCyuC/uaq1GM/TGyKXcZGU5rY2r1WSSyXNy1/DjBz3begykiaYYRBri+e2M2t9OOg0j/DQwSQpV8rZrUVKfTwBUTPog0YuL84uw5bEYKCJPxA+l"
            }
        ]
        }
    },
    "location": "eastus",
    "maxAgentPools": 1,
    "name": "qseok-k8s",
    "networkProfile": {
        "dnsServiceIp": "10.0.0.20",
        "dockerBridgeCidr": "176.19.0.1/16",
        "loadBalancerSku": "Basic",
        "networkPlugin": "kubenet",
        "networkPolicy": null,
        "podCidr": "10.144.0.0/16",
        "serviceCidr": "10.0.0.0/16"
    },
    "nodeResourceGroup": "MC_qseok-aks_qseok-k8s_eastus",
    "provisioningState": "Succeeded",
    "resourceGroup": "qseok-aks",
    "servicePrincipalProfile": {
        "clientId": "b604058c-a650-44ed-be6e-db6a972094b1",
        "secret": null
    },
    "tags": null,
    "type": "Microsoft.ContainerService/ManagedClusters",
    "windowsProfile": null
    }
```

</details>

Merge cluster creds into Local Kube setup
```
    az aks get-credentials --resource-group qseok-aks --name qseok-k8s
```

Confirm the working kubectl
```
    kubectl config current-context
```

Result: 
```
    qseok-k8s
```
## Create Configurations, Dependancies (RBAC, HELM, STORAGE)
Now we are going to create the configurations that will be used by Qlik Sense Enteprise

### RBAC and Helm Tiller Account 
Apply Role Based Access Controls (Special Thanks Justin Donnelly!)
```
    kubectl apply -f https://raw.githubusercontent.com/awsawyer/Azure/yaml/rbac-config.yaml
```

Initialize Helm w/Tiller Service Account
```
    helm init --service-account tiller
```
May encounter an issue with Helm not being installed despite being in your path. I ran ```choco install kubernetes-helm```

List AZ Groups w/Table Format
```
    az group list -o table
```

### Storage

Create your storage account. Remember this must be unique across ALL of Azure
```
    az storage account create -g MC_qseok-aks_qseok-k8s_eastus -n <STORAGE_ACCOUNT_NAME> --sku Standard_LRS
```

In this, I normally use this format ```qseokk8s<your first name><4 random digits>```

In this case, ```qseokk8sadam1689```

Also, the ```-g``` should be from your result resource group, like ```"nodeResourceGroup": "MC_qseok-aks_qseok-k8s_eastus",``` Everything should lowercase. 


<details>
    <summary> The Result should be like this </summary>

```
{
  "accessTier": null,
  "azureFilesIdentityBasedAuthentication": null,
  "creationTime": "2019-10-03T21:06:35.208943+00:00",
  "customDomain": null,
  "enableHttpsTrafficOnly": true,
  "encryption": {
    "keySource": "Microsoft.Storage",
    "keyVaultProperties": null,
    "services": {
      "blob": {
        "enabled": true,
        "lastEnabledTime": "2019-10-03T21:06:35.255791+00:00"
      },
      "file": {
        "enabled": true,
        "lastEnabledTime": "2019-10-03T21:06:35.255791+00:00"
      },
      "queue": null,
      "table": null
    }
  },
  "failoverInProgress": null,
  "geoReplicationStats": null,
  "id": "/subscriptions/363ede92-6ea4-4796-801d-ce3f2ce18799/resourceGroups/MC_qseok-aks_qseok-k8s_eastus/providers/Microsoft.Storage/storageAccounts/qseokk8sadam1689",
  "identity": null,
  "isHnsEnabled": null,
  "kind": "Storage",
  "lastGeoFailoverTime": null,
  "location": "eastus",
  "name": "qseokk8sadam1689",
  "networkRuleSet": {
    "bypass": "AzureServices",
    "defaultAction": "Allow",
    "ipRules": [],
    "virtualNetworkRules": []
  },
  "primaryEndpoints": {
    "blob": "https://qseokk8sadam1689.blob.core.windows.net/",
    "dfs": null,
    "file": "https://qseokk8sadam1689.file.core.windows.net/",
    "queue": "https://qseokk8sadam1689.queue.core.windows.net/",
    "table": "https://qseokk8sadam1689.table.core.windows.net/",
    "web": null
  },
  "primaryLocation": "eastus",
  "provisioningState": "Succeeded",
  "resourceGroup": "MC_qseok-aks_qseok-k8s_eastus",
  "secondaryEndpoints": null,
  "secondaryLocation": null,
  "sku": {
    "capabilities": null,
    "kind": null,
    "locations": null,
    "name": "Standard_LRS",
    "resourceType": null,
    "restrictions": null,
    "tier": "Standard"
  },
  "statusOfPrimary": "available",
  "statusOfSecondary": null,
  "tags": {},
  "type": "Microsoft.Storage/storageAccounts"
}

```
</details>

Apply AzureFile Storage to the Cluster Role

```
    kubectl create clusterrolebinding system:azure-cloud-provider --clusterrole=system:azure-cloud-provider --serviceaccount=kube-system:persistent-volume-binder
```

## Install Qlik Sense
Now, we will start installing the Qlik Sense Components. 

Add QSEoK "Stable" Helm Repo

```
    helm repo add qlik https://qlik.bintray.com/stable
```

Deploy Qliksense Init Chart
```
    helm install -n qseok-init qlik/qliksense-init
```

Deploy QlikSense from your edited values.yaml file
```
    helm install -n qseok qlik/qliksense -f ./qseok-values.yaml
```

### Check Deployment

Watch for all pods to get started
```
    kubectl get pods -w
```

Confirm that Services have started
```
    kubectl get svc
```

Make sure persistent volume claim
```
    kubectl get pvc
```

### Add Public IP to Cluster and Local Development Environment

Once all services are up, you should see this
```
qseok-audit-584f8fd6c8-rbwg6                      1/1     Running   2          3m53s
qseok-chronos-7b94cf4499-kbtj2                    2/2     Running   0          3m53s
qseok-chronos-worker-59b5848b-hkgnb               1/1     Running   0          3m53s
qseok-collections-58fcb95747-ld2z8                1/1     Running   0          3m53s
qseok-data-connector-odbc-cmd-7dc469df94-n5ln8    1/1     Running   0          3m53s
qseok-data-connector-odbc-rld-7dfc7546d8-zhslz    1/1     Running   0          3m53s
qseok-data-connector-qwc-cmd-65f99466ff-4kcff     1/1     Running   4          3m52s
qseok-data-connector-qwc-rld-669fb479bd-skj97     1/1     Running   3          3m52s
qseok-data-connector-rest-cmd-d456bf96d-l9qh5     1/1     Running   0          3m52s
qseok-data-connector-rest-rld-6b5d575dc6-tcjlb    1/1     Running   0          3m52s
qseok-data-connector-sfdc-cmd-54d8696b88-862qd    1/1     Running   0          3m52s
qseok-data-connector-sfdc-rld-86b5766cb6-qzq8c    1/1     Running   0          3m52s
qseok-data-prep-7f94cf45b5-x85wk                  1/1     Running   0          3m51s
qseok-data-rest-source-799b4976f5-cpzbs           1/1     Running   0          3m51s
qseok-dcaas-6896475689-bncrw                      1/1     Running   0          3m51s
qseok-dcaas-redis-master-0                        1/1     Running   0          3m52s
qseok-dcaas-web-57f94b4bb8-s5d6k                  1/1     Running   0          3m51s
qseok-edge-auth-8484688885-2nqm4                  2/2     Running   2          3m51s
qseok-encryption-fbbcd45d7-95slb                  1/1     Running   0          3m50s
qseok-engine-default-6596f5d86b-8bx9n             1/1     Running   3          3m50s
qseok-eventing-7d5b9dfcb5-vltc5                   1/1     Running   0          3m50s
qseok-feature-flags-786d76865d-7pxp4              1/1     Running   0          3m50s
qseok-geo-operations-654d564fdf-mblc2             1/1     Running   0          3m49s
qseok-groups-9b5f4999f-j94j8                      1/1     Running   0          3m49s
qseok-hub-bc54878f5-r9cb4                         1/1     Running   0          3m49s
qseok-identity-providers-bd8d76846-4c6p5          1/1     Running   0          3m49s
qseok-insights-bcd949468-nfc58                    1/1     Running   1          3m49s
qseok-keys-844df5787d-58vwm                       1/1     Running   0          3m48s
qseok-licenses-c5bf9df4c-7dtdq                    2/2     Running   3          3m48s
qseok-locale-5d7b7d68b8-q8f2z                     1/1     Running   0          3m48s
qseok-management-console-74dd4f4cd9-95qzw         1/1     Running   0          3m48s
qseok-mongodb-7f5c9977fb-j8rf2                    1/1     Running   0          3m48s
qseok-nats-0                                      2/2     Running   0          3m51s
qseok-nats-streaming-0                            2/2     Running   3          3m52s
qseok-nats-streaming-1                            2/2     Running   0          2m15s
qseok-nats-streaming-2                            2/2     Running   0          94s
qseok-nginx-ingress-controller-75d9f8f585-pcrt9   1/1     Running   0          3m50s
qseok-odag-5b7bfb7c54-9skzw                       2/2     Running   1          3m48s
qseok-policy-decisions-7bcff576b9-r9cq6           1/1     Running   0          3m47s
qseok-precedents-76d6f95c87-wln2z                 1/1     Running   1          3m47s
qseok-qix-data-connection-8647784b5-4xdf2         1/1     Running   2          3m47s
qseok-qix-datafiles-76564b8d84-59rnt              1/1     Running   4          3m47s
qseok-qix-sessions-6d549846b-dm8mm                1/1     Running   0          3m47s
qseok-quotas-688d875bc6-7t9dp                     1/1     Running   0          3m46s
qseok-redis-master-0                              1/1     Running   0          3m51s
qseok-redis-slave-7d85f6b697-fdv72                1/1     Running   2          3m46s
qseok-reload-tasks-647f8f4c54-xf4cq               1/1     Running   1          3m46s
qseok-reloads-78dc69986c-85frb                    1/1     Running   3          3m46s
qseok-reporting-9f557484-bhmpc                    4/4     Running   0          3m46s
qseok-resource-library-79d8446669-ndngh           1/1     Running   2          3m45s
qseok-sense-client-7bf449c869-bd5hn               1/1     Running   0          3m45s
qseok-spaces-79686588d5-bvqw7                     1/1     Running   0          3m45s
qseok-temporary-contents-648d8bfd56-n5qw4         1/1     Running   0          3m45s
qseok-tenants-5f455f94c7-vkvb7                    1/1     Running   2          3m45s
qseok-users-864665f47f-tr9kj                      1/1     Running   2          3m44s
```

However, you haven't made the Ingress controller public yet. From ```kubectl get svc```

```
    qseok-nginx-ingress-controller           LoadBalancer   10.0.72.121    <pending>     80:31097/TCP,443:30539/TCP       3m42s
    qseok-nginx-ingress-controller-metrics   ClusterIP      10.0.30.197    <none>        9913/TCP                         3m42s
```

Use ```kubectl describe pods qseok-nginx-ingress-controller``` to get the IP of the configured pod.

Edit the values ```qseok-values.yaml``` with the new ip as Hostname and then run the upgrade command

```
    helm upgrade --install qseok qlik/qliksense -f ./qseok-values.yaml
```

## OR 

you can use the same hostname ```elastic.example``` and create a local host entry on your computer. 

Windows
```
    Open cmd.exe as administrator 
    cd C:\windows\System32\drivers\etc
    notepad hosts
```






