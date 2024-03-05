# Harvester Hardening Guide

**THIS GUIDE IS A WIP**

This guide will walk you through how to harden your Rancher Harvester clusters. There are 4 main components to configure to harden Harvester: Elemental SLEMicro (OS), RKE2 (Kubernetes), and Longhorn (Storage Application). This guide will walk you through each.

Harvester runs atop Elemental, an immutable OS. This means that any changes you make within the mounted disk (it is running within a chroot) will be wiped upon reboot. There are two methods to addressing this and will depend on the method you have used to install Harvester.

What will be described below is the 'easier' method which involves installing the Harvester ISO onto your bare metal nodes manually. It is done in a post-install state, meaning after you have installed Harvester. The other method is via PXE-boot and can be done as part of the install. Technically this method is more secure but also requires more infrastructure setup to support (DHCP and PXE server, etc). For PoCs and first tests it is suggested that the post-install method is the way to go.

## Troubleshooting Changes

As this is a beta, and making changes to core OS layer components will mandate a reboot, sometimes it is possible to lock yourself out of the node and it never successfully boot given how Elemental works. Being exact with yaml configuration matters as names and indentations being incorrect can cause a boot failure. However, there is always a way to get back in and it depends on where the boot failure occurred. Nearly all troubleshooting will involve you getting into a terminal session on the physical node itself. SSH can be an option most of the time but if there is an early boot failure, even the SSH daemon will not load.

The suggested route to take to get back to a termainal if there is a failure is as follows:
1) SSH to `rancher@node_ip`
2) Use console to login directly to the node using the `rancher` username and password you provided during installation
3) Use an alternative terminal at the console (ctrl-alt-F2) to login directly to the node using the `rancher` username and password you provided during installation
4) Use an alternative terminal at the console (ctrl-alt-F2) to login directly to the node using the `rancher` username and `rancher` as the password. (This is when the init stage has entirely failed before the OS has started)

## Elemental SLEMicro

TODO

## RKE2

Much of this ties into the RKE2 STIG.

The main configuration settings to harden RKE2 will be located in the `/oem/99_custom.yaml` file. For more information, check the (docs)[https://docs.harvesterhci.io/v1.2/install/update-harvester-configuration/].

1. SSH to each Harvester node in your cluster and perform the following.

2. In `/oem/99-custom.yaml`, under the `stages.initramfs[0].files` section, add a new element to the list to configure sysctl options:
    
    ```bash
        - path: /etc/sysctl.d/99-sysctl.conf
          permissions: 600
          owner: 0
          group: 0
          content: |
            vm.panic_on_oom=0
            vm.overcommit_memory=1
            kernel.panic=10
            kernel.panic_on_oops=1
    ```

3. In `/oem/99-custom.yaml`, under the `stages.initramfs[0].files` section, add a new element to the list to configure RKE2 hardening options:
    
    ```bash
        - path: /etc/rancher/rke2/config.yaml.d/52-stig.yaml
          permissions: 600
          owner: 0
          group: 0
          content: |
              profile: cis-1.23 # Change this to cis-1.6 for Harvester 1.1.x or lower.
              use-service-account-credentials: true
              kube-controller-manager-arg:
              - "cert-dir=/var/lib/rancher/rke2/server/tls/kube-controller-manager"
              - "secure-port=10257"
              - "tls-min-version=VersionTLS12"
              - "tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384"
              kube-scheduler-arg:
              - "tls-min-version=VersionTLS12"
              - "tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384"
              kube-apiserver-arg:
              - "tls-min-version=VersionTLS12"
              - "tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384"
              - "authorization-mode=RBAC,Node"
              - "anonymous-auth=false"
              - "audit-policy-file=/etc/rancher/rke2/config.yaml.d/92-harvester-kube-audit-policy.yaml"
              - "audit-log-mode=blocking-strict"
              kubelet-arg:
              - "protect-kernel-defaults=true"
              - "streaming-connection-idle-timeout=5m"
              pod-security-admission-config-file: /etc/rancher/rke2/config.yaml.d/51-rke2-pss-custom.yaml
    ```

4. In `/oem/99-custom.yaml`, under the `stages.initramfs[0].files` section, add a new element to the list to configure Pod Security Admission (PSA) configurations.
    
    **NOTE**: For any namespaces you utilize within Harvester (for instances, namespaces for dedicated networking spaces or VMs), you will also need to add them to the below list of namespaces.
    
    ```bash
        - path: /etc/rancher/rke2/config.yaml.d/51-rke2-pss-custom.yaml
          permissions: 600
          owner: 0
          group: 0
          content: |
              apiVersion: apiserver.config.k8s.io/v1
              kind: AdmissionConfiguration
              plugins:
                - name: PodSecurity
                  configuration:
                    apiVersion: pod-security.admission.config.k8s.io/v1
                    kind: PodSecurityConfiguration
                    defaults:
                      enforce: "restricted"
                      enforce-version: "latest"
                      audit: "restricted"
                      audit-version: "latest"
                      warn: "restricted"
                      warn-version: "latest"
                    exemptions:
                      usernames: []
                      runtimeClasses: []
                      namespaces: [calico-apiserver,
                                  calico-system,
                                  cattle-alerting,
                                  cattle-csp-adapter-system,
                                  cattle-elemental-system,
                                  cattle-epinio-system,
                                  cattle-externalip-system,
                                  cattle-fleet-local-system,
                                  cattle-fleet-system,
                                  cattle-gatekeeper-system,
                                  cattle-global-data,
                                  cattle-global-nt,
                                  cattle-impersonation-system,
                                  cattle-istio,
                                  cattle-istio-system,
                                  cattle-logging,
                                  cattle-logging-system,
                                  cattle-monitoring-system,
                                  cattle-neuvector-system,
                                  cattle-prometheus,
                                  cattle-provisioning-capi-system,
                                  cattle-resources-system,
                                  cattle-sriov-system,
                                  cattle-system,
                                  cattle-ui-plugin-system,
                                  cattle-windows-gmsa-system,
                                  cert-manager,
                                  cis-operator-system,
                                  fleet-default,
                                  harvester-system,
                                  harvester-public,
                                  ingress-nginx,
                                  istio-system,
                                  kube-node-lease,
                                  kube-public,
                                  kube-system,
                                  longhorn-system,
                                  rancher-alerting-drivers,
                                  security-scan,
                                  tigera-operator]
    ```

5. In `/oem/99-custom.yaml`, under the `stages.initramfs[0].files` section, find the element that has `path: /etc/rancher/rke2/config.yaml.d/92-harvester-kube-audit-policy.yaml` and update the `contents` section to this:
    
    ```bash
          apiVersion: audit.k8s.io/v1
          kind: Policy
          omitStages:
            - "ResponseStarted"
            - "ResponseComplete"
          rules:
          - level: Metadata
            resources:
            - group: ""
              resources: ["secrets"]
          - level: RequestResponse
            resources:
            - group: ""
              resources: ["*"]
    ```

5. To manage race conditions, create the `etcd` user/group by running the following commands:

   ```bash
   groupadd etcd
   useradd -r -c "etcd user" -G etcd -s /sbin/nologin -M etcd
   ```

7. In `/oem/99-custom.yaml`, under the `stages.initramfs[0].commands` section, add the following commands to the end of the list to ensure these persist in the future:
    
    ```bash
        - groupadd etcd
        - useradd -r -c "etcd user" -G etcd -s /sbin/nologin -M etcd
        - sysctl -p
    ```

8. Restart the Harvester node.

9. After reboot, SSH into your Harvester node and re-run your sysctl command to account for any race conditions between files and commands: `sysctl -p`

## Longhorn

Securing Longhorn is in early work but the key need here is going to be the enabling of Encryption at Rest. To do so, an ecryption key is going to need to be created. Longhorn provides the ability to do encryption for all of its volumes and can allow for a very granular approach to managing encrypted volumes. One can either encrypt all volmes with the same key globally or slice up different encrypted volumes with different keys. For a security minimum, it is recommended at least one global key is defined. 

To do this, you'll need to access the Harvester cluster using `kubectl`. 