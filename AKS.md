# Creating AKS Cluster

When creating the Kubernetes Cluster make sure you pick the VM-size of Standard_D11_v2 with a node pool of 1. This will make it available for the free credit. 


## Create Storage

Use the name `avibstorage2` for the resource group.

## Create File Share Service

Use the name `avib-file-share` for the resource group.

## Create Resource Group

Use the name `AVIB-RSRCG` for the resource group.

## Create Namespace

First we must get the credentials using `az aks get-credentials --resource-group AVIB-RSRCG --name AVIB-Cluster` 
You will have to have the console available in Azure. Make sure to pick Bash. Once you have this you can create the namespace with `kubectl create namespace avib-namespace`.

## Create Azure Container Registry 

Create a Container Registry on Azure. Make sure to use the standard plan since we will need more than 2 webhooks.

## Create the CD/CI from VSCode

Generate the a PAT (Personal Access Token) on GitHub. For the install the Azure Deploy on VSCode and then right click on the Azure Subscription on the Kubernetes extension.

avibregistry

## Installing Neo4j database

### Creating a public static IP

Create 3 public static ip, make sure to copy the IP from the output. If not you can find the IP created under the Resource Group in Azure. 

`az network public-ip create -g MC_AVIB-RSRCG_AVIB-Cluster_eastus -n core01 --sku standard --dns-name neo4jcore01 --allocation-method Static`


`az network public-ip create -g MC_AVIB-RSRCG_AVIB-Cluster_eastus -n core02 --sku standard --dns-name neo4jcore02 --allocation-method Static`

`az network public-ip create -g MC_AVIB-RSRCG_AVIB-Cluster_eastus -n core03 --sku standard --dns-name neo4jcore03 --allocation-method Static`



In my case the IP addresses were: 

1. 13.72.82.9
2. 13.72.82.129
3. 13.68.244.173

On the Azure CLI export those values using: 
```
export IP0=13.72.82.9
export IP1=13.72.82.129
export IP2=13.68.244.173
export ADDR0=$IP0
export ADDR1=$IP1
export ADDR2=$IP2
```

Next use the following: 

```
export DEPLOYMENT=avib-graph
export NAMESPACE=default
export ADDR0=13.72.82.9
export ADDR1=13.72.82.129
export ADDR2=13.68.244.173
```

Then we upload the `custom-core-configmap.yaml` to the Fileshare. After that we execute the following commands: 
```
cd clouddrive
cat tools/external-exposure/custom-core-configmap.yaml | envsubst | kubectl apply -f -
```



### Installing Neo4j using Helm Chart 

Run the following command `helm install avib-graph https://github.com/neo4j-contrib/neo4j-helm/releases/download/4.2.2-1/neo4j-4.2.2-1.tgz --set core.numberOfServers=3 --set readReplica.numberOfServers=0 --set core.configMap=avib-graph-neo4j-externally-addressable-config --set acceptLicenseAgreement=yes --set neo4jPassword=admin --set dbms.backup.enabled=true `
You will see an output like the following
```
NAME: avib-graph
LAST DEPLOYED: Wed Apr 28 04:26:31 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
Your cluster is now being deployed, and may take up to 5 minutes to become available.
If you'd like to track status and wait on your rollout to complete, run:

$ kubectl rollout status \
    --namespace default \
    StatefulSet/avib-graph-neo4j-core \
    --watch

You can inspect your logs containers like so:

We can see the content of the logs by running the following command:

$ kubectl logs --namespace default -l \
    "app.kubernetes.io/instance=avib-graph,app.kubernetes.io/name=neo4j,app.kubernetes.io/component=core"

We can now run a query to find the topology of the cluster.

export NEO4J_PASSWORD=$(kubectl get secrets avib-graph-neo4j-secrets --namespace default -o jsonpath='{.data.neo4j-password}' | base64 -d)
kubectl run -it --rm cypher-shell \
    --image=neo4j:4.2.2-enterprise \
    --restart=Never \
    --namespace default \
    --command -- ./bin/cypher-shell -u neo4j -p "$NEO4J_PASSWORD" -a neo4j://avib-graph-neo4j.default.svc.cluster.local "call dbms.routing.getRoutingTable({}, 'system');"

This will print out the addresses of the members of the cluster.

Note:
You'll need to substitute <password> with the password you set when installing the Helm package.
If you didn't set a password, one will be auto generated.
You can find the base64 encoded version of the password by running the following command:

kubectl get secrets avib-graph-neo4j-secrets -o yaml --namespace default
```

### External Exposure

We must now create the load balancing for each pod created. Use the load-balancer files provided. Make sure to update the IP address for each one according to the IP obtained previously. 

To create the Load Balancer we use the following commands: 
```
kubectl apply -f load-balancer.yaml
kubectl apply -f load-balancer2.yaml
kubectl apply -f load-balancer3.yaml
```

### Create secret key
We must create a file on Azure Cloud, this can be done using the Azure Cli. Use the `nano` commando to create a file. This file will be name `Azure-credentials.sh`, the content in this case is:
```
export ACCOUNT_NAME=avibstorage
export ACCOUNT_KEY=K96l0lpXJP9D+d/0aSzkj6Iwd0Q7otHZ237UHrljxujXGaR4IOPiEtgcJN++LFD4Thmsf2zOqu9G3k4ulvyHOg==
```
The account name is the storage account. To get the storage key use the following command: 
`ACCOUNT_KEY=$(az storage account keys list --resource-group "AVIB-RSRCG" --account-name "avibstorage" --query [0].value -o tsv)` get the key using `echo $ACCOUNT_KEY`

Once the file is created, the secret can be created using the following command:
`kubectl create secret generic neo4j-azure-credentials --from-file=credentials=Azure-credentials.sh`

## Loading data to the Neo4j pod
First make sure your backup file is accessible. You can upload a file easily making use of VSCode Extension for `Azure Storage`. Easily navigate the extension to upload a file.

Once the file is uploaded it will be stored in the cloudedrive folder. Use the Azure Cli to access the Neo4j pod and copy the file into the pod.

Run the following commands: 

`cd clouddrive`

`kubectl cp neo4j-20210323.dump default/avib-graph-neo4j-core-0:/var/lib/neo4j`

`kubectl exec --stdin --tty avib-graph-neo4j-core-0 -- /bin/bash`

Once the file is copied in the pod follow similar steps to the `Load-neo4j.md` instruction file. More detail on the commands can be found in that file. For now follow these steps.

1. `cypher-shell -u neo4j -p admin -d system 'STOP DATABASE neo4j'`
2. `cypher-shell -u neo4j -p admin -d system 'DROP DATABASE neo4j'`
3. `neo4j-admin load --verbose --from=/var/lib/neo4j/neo4j-20210323.dump --force --database=neo4j`
4. `cypher-shell -u neo4j -p admin -d system 'CREATE DATABASE neo4'`
5. `cypher-shell -u neo4j -p admin -d system 'START DATABASE neo4j'`


## Possible errors

If you encounter an error of connection when trying to acces the webpage similar to this one. 
```
Neo4jError: WebSocket connection failure. Due to security constraints in your web browser, the reason for the failure is not available to this Neo4j Driver. Please use your browsers development console to determine the root cause of the failure. Common reasons include the database being unavailable, using the wrong connection URL or temporary network problems. If you have enabled encryption, ensure your browser is configured to trust the certificate Neo4j is configured to use. WebSocket `readyState` is: 3
```

You might have to enable Azure to expose the ports being used by Neo4j. Another posible solution to this error is to access using the IP instead of the DNS name. 

