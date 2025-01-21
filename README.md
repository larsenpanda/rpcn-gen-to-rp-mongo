# Redpanda Connect - Generate to Redpanda and Sink to MongoDB Atlas
This is an example working demo in EKS to show and play around with generating data to Repdanda and sending from 2 different Mongo Atlas collections

### Pre-req's

- AWS Account with ability to stand up an EKS cluster with `eksctl` (for best experience)
- Mongo Atlas account (free tier is fine)

### Install your EKS cluster

Follow the instructions here: https://docs.redpanda.com/current/deploy/deployment-option/self-hosted/kubernetes/eks-guide/

Note: You wont need to connect externally, it's probably more trouble than it's worth, everything will be able to operate between the pods internally.

### Prepare your MongoDB Atlas instance and collections

As mentioned above, there are 2 collections to create: 

1. logins
2. transactions

Make sure to create a **user account** with collection write access privs and **allow connections from anywhere (or the IP's for the EKS nodes)**

## Check to see if Redpanda is running properly

If you have installed Redpanda without issue into your EKS environment run the following to validate that it's up and running - 

`kubectl get pods -n redpanda`

### Forward local 8080 port to the internal Redpanda Console node and ensure running

Run the following in a new terminal window/tab, it will run in the background -

`kubectl --namespace redpanda port-forward svc/redpanda-console 8080:8080`

Visit http://localhost:8080 and see if Console is running ✅

# Redpanda Connect deployment

Refer to the official Redpanda Connect documentation here: https://docs.redpanda.com/redpanda-connect/get-started/quickstarts/helm-chart/

Walk through the documentation and ensure you can get to the **"Hello, Redpanda Connect!"** output to confirm there are no issues.

# Updating Redpanda helm release to include source and sink operations

_**The solution created in this demo / enablement content has this architecture:**_

<img width="872" alt="image" src="https://github.com/user-attachments/assets/fa3f7023-a7b8-44b3-bdb3-a66b49ec8c17" />

_**The infrastructure in EKS looks like this:**_

<img width="719" alt="image" src="https://github.com/user-attachments/assets/a9c899b5-88db-4725-baf4-e03719c985a5" />

### Deploy the solution

There are 3 primary yaml files that drive the integration -

- <a href="https://github.com/larsenpanda/rpcn-gen-to-rp-mongo/blob/main/pipeline-sourcing.yaml">pipeline-sourcing.yaml</a>
- <a href="https://github.com/larsenpanda/rpcn-gen-to-rp-mongo/blob/main/pipeline-sinking-logins.yaml">pipeline-sinking-logins.yaml</a>
- <a href="https://github.com/larsenpanda/rpcn-gen-to-rp-mongo/blob/main/pipeline-sinking-tx.yaml">pipeline-sinking-tx.yaml</a>

The "**sourcing**" yaml generates data and sends it to 2 different Redpanda topics.

The "**sinking-logins**" yaml consumes from the `logins` topic in Redpanda using the Franz-Go kafka client and maps it to MongoDB documents then sinks in the "logins" collection in Atlas.

The "**sinking-tx**" yaml consumes from the `transactions` topic in Redpanda using the Franz-Go kafka client and maps it to MongoDB documents then sinks in the "transactions" collection in Atlas.

There are a few different ways of approaching the deployment methodology-
1. Deploy individual yaml files for each component and name / manage the helm release independently
2. Deploy one helm release/pod that does the sourcing and another that does the sinking using a case statement to sink to the appropriate mongo collection dependent on the topic it's sourcing from
3. Merge the yaml files into a single logical helm release that runs as one single pod in "streams mode"

The approach that is the most simple to maintain in my opinion is option #3. 

- #1 is very straight forward, simply deploy each yaml with a different helm release name.
- #2 is represented by <a href="https://github.com/larsenpanda/rpcn-gen-to-rp-mongo/blob/main/connect-switch-output.yaml">this yaml file</a> example provided in the repo.
- #3 is accomplished by the following approach documented <a href="https://docs.redpanda.com/redpanda-connect/guides/streams_mode/about/">on this doc page</a>

#3 looks like this (for copy/pastability):

**First** create a configMap of the 3 yamls (`sourcing`, `sinking-logins`, `sinking-tx`) using the following command:

`kubectl create configmap connect-streams --from-file=pipeline-sourcing.yaml --from-file=pipeline-sinking-tx.yaml --from-file=pipeline-sinking-logins.yaml --namespace redpanda --dry-run=client -o yaml | kubectl apply -f -`

**Next**, uninstall the original Redpanda Connect release that outputted "Hello, Redpanda Connect" and install the following Redpanda Connect pod to instantiate streams mode

`helm upgrade --install connect-streams redpanda/connect --namespace redpanda --values connect-streams.yaml`

# Check logs

To see how everything is running (beyond checking the topics and mongo collections in the desired interface)- 

First, set the env variable for the generated pod name of connect-streams:

`export POD_NAME=$(kubectl get pods --namespace redpanda -l "app.kubernetes.io/name=redpanda-connect,app.kubernetes.io/instance=mo-connect-streams" -o jsonpath="{.items[0].metadata.name}")`

Next, run the kubectl logs command against the variable set:

`kubectl logs $POD_NAME --namespace redpanda`

# Conclusion

You should now have a fully running pipeline on an EKS cluster, go check out the <a href="https://docs.redpanda.com/redpanda-connect/home/">Redpanda Connect Documentation</a> to see what else you can do with other inputs and outputs!
