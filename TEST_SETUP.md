# Test Setup

1. Create a ZTP vDU Cluster for 4.14.1 and verify all Day 2 operators are present
2. Make sure you patch the dnsmasq configuration and the container access directory for proper hostname changes
3. Create an auditable API Server Configuration with
    ```yaml
    apiVersion: config.openshift.io/v1
    kind: APIServer
    metadata:
      name: cluster
    spec:
      audit:
        profile: WriteRequestBodies
    ```
4. Detach the Cluster from the ACM Hub
    ```shell
    oc delete managedcluster sno-worker-00 
    ```
5. Create a seed image on the host either via CR (SeedGenerator) or via commandline on the SNO.
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
6. once the Seed image has been created, we can switch to the recipient image
7. On the recipient, execute the regular workflow for the IBU. The cluster will need to have LCA installed inbefore
8. After this, initiate the IBU upgrade process and wait until the node reboots
9. After the node reboots and becomes available, switch to [fastsno](https://github.com/omertuc/fastsno/tree/main) and collect the audit logs by running
    ```shell
   #!/bin/bash
   set -euxo pipefail
   
   NODENAME=snonode.sno-worker-00.e2e.bos.redhat.com
   SCRIPT_DIR=$(dirname "$(readlink -f "$0")")
   
   mv audit "audit_old_$(date -u +%Y-%m-%dT%H:%M:%S%Z)" || true
   mkdir -p "$SCRIPT_DIR/audit"
   oc adm node-logs "node/${NODENAME}" --path=kube-apiserver/ | grep audit | while read -r log; do oc adm node-logs ${NODENAME} "--path=kube-apiserver/$log" > "audit/$log"; done
    ```
10. Use the visualization script from fast-sno to get a timeline graph of created core resources.
11. Login to the node via ssh and run the following commands to get various other audit information before the startup of the API Server:
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
12. Download the logs to your machine by running 
   ```shell
   scp core@${NODE_DNS_OR_IP}:"/tmp/logs" "${PATH_TO_LOGS}/next-measurement" 
   ```