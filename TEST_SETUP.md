# Test Setup

The following document describes the setup and execution of the SNO IBU Measurements.

## 1. Create a ZTP vDU Cluster for 4.14.1 and verify all Day 2 operators are present

We will need a baremetal SNO Cluster for 4.14.1.
To properly capture performance metrics, we will need to ensure that we run all performance relevant operators and configurations.
As of this point we will not run any measurements yet, but just create the Seed Image for a cluster.

Important Considerations are any deployment configuration environments like ZTP due to CRs like the PerformanceProfile.

If you have an existing guidance on preparing your metal cluster for any other OCP version (e.g. 4.15), please follow that guidance.
Make sure you patch the dnsmasq configuration and the container access directory for proper hostname changes so that the IBU works correctly.

## 2. Create an auditable API Server Configuration

By default, the API Server does not log any request bodies. We will need to enable this for the audit logs to be useful.
This is because we use the date of the first request to the API Server as the start of the measurement for a specific component.

You can simply enable this configuration with:

 ```yaml
 apiVersion: config.openshift.io/v1
 kind: APIServer
 metadata:
   name: cluster
 spec:
   audit:
     profile: WriteRequestBodies
 ```
## 3. (Optional based on ACM) Detach the Cluster from ACM Hub

If you are running in a managed environment, make sure to remove the cluster from ACM.
Otherwise during the IBU there will be leftover resources and certificates and you might not be able to recover.

    ```shell
    oc delete managedcluster sno-worker-00 
    ```
## 4. Create a seed image on the host either via CR (SeedGenerator) or via commandline on the SNO.

The cluster will need to have [LCA](https://github.com/openshift-kni/lifecycle-agent) installed.

With the prepared naked SNO cluster, we can now run the seed generation. If you are aware of how to use the SeedGenerator CR,
you can use that. Otherwise you can also run the seed generation manually on the SNO node:

 ```shell
 export MY_USER=jmoller
 export AUTHFILE=/tmp/my-auth.json
 podman login --authfile ${AUTHFILE} -u ${MY_USER} quay.io/${MY_USER}
 
 export SEED_AUTH=$(base64 -w 0 ${AUTHFILE} ; echo)
 export HUB_KUBECONFIG=$(base64 -w 0 /home/jmoller/hubkubeconfig; echo)
 ```
 ```yaml
 apiVersion: v1
 kind: Secret
 metadata:
   name: seedgen
   namespace: openshift-lifecycle-agent
 type: Opaque
 data:
   seedAuth: ${SEED_AUTH}
   hubKubeconfig: ${HUB_KUBECONFIG}
 ```
 ```yaml
 apiVersion: lca.openshift.io/v1alpha1
 kind: SeedGenerator
 metadata:
   name: seedimage
   namespace: openshift-lifecycle-agent
 spec:
   # the seed image should be 
   seedImage: quay.io/${MY_USER}/ibu-seed-sno0:v4.14.1
 ```
## 5. once the Seed image has been created, we can switch to the recipient cluster.

Now, install and prepare a cluster with the same metal hardware on a different (and lower) version.
This will be the cluster to upgrade from and the image we created above will be used above.

## 6. On the recipient, execute the regular workflow for the IBU. 

The cluster will need to have [LCA](https://github.com/openshift-kni/lifecycle-agent) installed, even if it is the recipient.

## 7. After this, initiate the IBU upgrade process and wait until the node reboots

Because the Seed Image was created with enabled audit logs, the APIServer will track all requests from the bootup.
If you want to measure time from BIOS on your hardware, you will have to manually trace this as it is not part of the tooling.

## 8. After the node reboots and becomes available, verify cluster operators and optional testing operators

First make sure that you have connectivity to the cluster after the node reboots.
Since the certificates change you must ssh into the node (ideally via preconfigured machineconfiguration public key).
Then, you can download the new `lb-ext` kubeconfig as described by [this solution](https://access.redhat.com/solutions/4845381).

To make sure that the cluster is available, make sure to wait until the `ClusterVersion` object is ready and available again. Also for any optional Operators, make sure that they are up and running as well.
This makes sure that you collect all relevant Audit Logs.

Once we confirmed the cluster is available again, we have a list of audit logs, but not yet on our machine.

## 9. Switch to [fastsno](https://github.com/omertuc/fastsno/tree/main) and collect the audit logs

The audit logs still lie on the SNO node and we need to download them and interpret them graphically.
We can do this by using the scripts in fastsno. However, since fastsno does not know about the NODENAME, we will need to adjust it before collecting the logs.

Adjust `https://github.com/omertuc/fastsno/blob/main/download_audit.sh` to the following based on your node (run `oc get nodes` to observe your node name if unsure):

 ```shell
#!/bin/bash
set -euxo pipefail

NODENAME=snonode.sno-worker-00.e2e.bos.redhat.com
SCRIPT_DIR=$(dirname "$(readlink -f "$0")")

mv audit "audit_old_$(date -u +%Y-%m-%dT%H:%M:%S%Z)" || true
mkdir -p "$SCRIPT_DIR/audit"
oc adm node-logs "node/${NODENAME}" --path=kube-apiserver/ | grep audit | while read -r log; do oc adm node-logs ${NODENAME} "--path=kube-apiserver/$log" > "audit/$log"; done
 ```

After this step succeeded, the node logs can be downloaded while running `./download_audit.sh` in the fastsno repository root.

## 10. Use the visualization script from fast-sno to get a timeline graph of created core resources.

Now we can simply follow the [Timeline instructions from the fastsno repository](https://github.com/omertuc/fastsno/blob/main/README.md#timeline) to get a timeline graph of the created resources.

## 11. Collect various other audit information before the startup of the API Server:

Now we have detailed audit information about the components inside kubernetes. However, we still lack the resources to analyze the startup times before the API Server comes up.
To do this, we will need to collect various other information from the node before the API Server comes up.
Since RHCOS is mostly systemd based, we can use the `systemd-analyze` tool to get a lot of information about the startup process.

To do this, we will need to collect the following information by running the following commands via SSH on the node:

```shell
mkdir /tmp/logs
systemd-analyze blame > /tmp/logs/systemd-analyze-blame.log
systemd-analyze critical-chain > /tmp/logs/systemd-analyze-critical-chain.log
systemd-analyze dump > /tmp/logs/systemd-analyze-dump.log
systemd-analyze plot > /tmp/logs/systemd-analyze-plot.html
systemctl status -u dnsmasq > /tmp/logs/dnsmasq.service.log
systemctl status -u kubelet > /tmp/logs/kubelet.service.log
systemctl status -u NetworkManager-wait-online > /tmp/logs/NetworkManager-wait-online.service.log
```

After this, run the following command to download the logs to a path of your choosing.
```shell
scp core@${NODE_DNS_OR_IP}:"/tmp/logs" "${PATH_TO_LOGS}/next-measurement" 
```

## 12. Analysis Instructions

Now that we have the data available for analysis, we can start manually verifiyng and analysing the data.
There are two main parts to this:

1. Verify that the data is correct and complete
2. Analyze the data and create a summary.

### Verify Data Correctness
Make sure that `./display_timeline.sh` in fastsno works and that the timeline is correctly displayed. 
Remember that there maybe a timezone difference between the node and your local machine.

The timeline should should be filled with most core operators and also have a legend with which you can identify whether the cluster is ready or not.
In the end of the logs there should be a consistent set of operators that are ready and available.

Additionally verify the systemd-analyze logs and make sure that the critical chain is correct and that the blame graph is correct.
The html file should show a similar graph to the timeline from the API Server and will allow to identify bottlenecks.
Anything that is not in the critical chain is not relevant for the startup time as we need to wait for it to finish anyway.

### Analyze the Data and Create a Summary
Identify the most relevant bottlenecks and summarize them. Then reach out to the relevant teams to discuss the bottlenecks and how to improve them.
Also verify that previous bottlenecks have been resolved and that the startup time has improved.
Create bugs for any issues that you find and link them to the relevant teams.

