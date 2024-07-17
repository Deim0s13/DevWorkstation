# DevSpace Developer Workstation Demo
This repository contains a short walkthrough demonstration of using Ansible to configure an OpenShift Dev Spaces instance and an OpenShift Virtualization machine.\

## Install dev spaces operator

To prepare for our demo we need to install the OpenShift Dev Spaces operator.

```bash
cat << 'EOF' | oc apply --filename -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: devspaces
  namespace: openshift-operators
spec:
  channel: stable
  installPlanApproval: Automatic
  name: devspaces
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: devspacesoperator.v3.14.0                                                                                                              
EOF
```

## Create checluster custom resource

Once the operator is installed we can create our instance of the `checluster` custom resource

```bash
cat << 'EOF' | oc apply --filename -
apiVersion: org.eclipse.che/v2
kind: CheCluster
metadata:
  name: devspaces
  namespace: openshift-operators
  finalizers:
    - checluster.che.eclipse.org
    - cluster-resources.finalizers.che.eclipse.org
    - cheGateway.clusterpermissions.finalizers.che.eclipse.org
    - oauthclients.finalizers.che.eclipse.org
    - dashboard.clusterpermissions.finalizers.che.eclipse.org
spec:
  components:
    cheServer:
      debug: false
      logLevel: INFO
    dashboard:
      logLevel: ERROR
    devWorkspace: {}
    devfileRegistry: {}
    imagePuller:
      enable: false
      spec: {}
    metrics:
      enable: true
    pluginRegistry: {}
  containerRegistry: {}
  devEnvironments:
    startTimeoutSeconds: 300
    security: {}
    secondsOfRunBeforeIdling: -1
    maxNumberOfWorkspacesPerUser: -1
    containerBuildConfiguration:
      openShiftSecurityContextConstraint: container-build
    maxNumberOfRunningWorkspacesPerUser: -1
    imagePullPolicy: Always
    defaultNamespace:
      autoProvision: true
      template: <username>-devspaces
    secondsOfInactivityBeforeIdling: -1
    storage:
      pvcStrategy: per-user
    persistUserHome:
      enabled: true
  gitServices: {}
  networking:
    auth:
      gateway:
        configLabels:
          app: che
          component: che-gateway-config
        kubeRbacProxy:
          logLevel: 0
        oAuthProxy:
          cookieExpireSeconds: 86400
        traefik:
          logLevel: INFO
EOF
```

## Create Secret

Before creating a demo VM and the subsequent Ansible components, we need to create a secret with a public key.

You can use a public key you have created on your local machine using ssh-keygen.

Replace the public key in the below command with your key.

```bash
cat << 'EOF' | oc apply --filename -
apiVersion: v1
kind: Secret
metadata:
  name: rhel-ssh-key
  namespace: default
data:
  key: c3NoLXJzYSBBQUFBQjNOemFDMXljMkVBQUFBREFRQUJBQUFDQVFEUllWOUNURmpNVGpNZXF1R29tazhSSWlFMFBsRmMxemRzdytwOHQ2Ynp4OG9rNmNCdmovMCswMGI2SEdIQ3cxRmFBR0YrbC9ZNEYreHpzN1hVanZyZWVTQ2NQVVVGVVBMbTBVcDFpcmlnWHo0a3F1R0prRnBzUTk0RjdtZ2laT3FFbTBUcll0SmUvSFp2aDkrdTlEcStEWnQ3R3BIYUFFSnAwemJpb0pxaG1Md3k4Ri9HTzBKWEpQQ3dJdkVuNC9naXVqdEJ0WUFPNE10STdXclBGMTlBNFlFMGtwdGhBNWx2UE9xNnFpWDg1SW5kVTVwSGNMeHdOSmtvWWJWYzRnMENMZ3FBckZNL250VEZpQ2RVS1ZaM1lDczROc1VKVzc0SE9mc3BYdUxnckhnMnUzQWhuRnlUQUJyR1paY0F6NW50VkQ1cWtYZVZ5NVRaRnJPSXViYWgxd25FZkVtZzViYkJRS0U5ZXBuMGxpcnE0L3dJVzEzWXVoeHZhYi9kb29kTjhiU05CV0xiSXBzYjNCYzVWSTdlaDlNOWN6WXFUNGp3R1VsVW1hYTR3YlVQc04zR091MFNEdHJHemxQT1B4SlRkZ3MwYWY0VmowNDRkZEhxUzFENVgwdjNJaXlZVW43U1FralFEbmVTUFI2T0U0b3BXTEtDNFBmMnoxempwaWZKTFJydzJhRXozaWpycXVYc1RGNlBlMnZyeHlEY0dOQlNQK1BpeHVVczRiRzNOaFNHV2hzaHQ3UWJJbGFobmx0b0ZEcW5paHVwdEhCd1pYc210SlZ6Y3dEOHNQUHVNSWZROWlDbUV4Y3lSbkhZRUl3aXp2UFVyNXdqMDZaUlRLMUNVUUpXTzJid1pkajRnbVFGcm1CZ0VBQldXdVBvT2taTm5WMEFMVUI3N3c9PSBwbGVhdGhlbkBwbGVhdGhlbi1tYWM=
type: Opaque
EOF
```

## Create RHEL VM

Go ahead and create a simple RHEL VM using the following command. This will create a VM named rhel9-demo.

```bash
cat << 'EOF' | oc apply --filename -
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: rhel9-demo
  namespace: default
  finalizers:
    - kubevirt.io/virtualMachineControllerFinalize
  labels:
    app: rhel9-demo
    kubevirt.io/dynamic-credentials-support: 'true'
    vm.kubevirt.io/template: rhel9-server-small
    vm.kubevirt.io/template.namespace: openshift
    vm.kubevirt.io/template.revision: '1'
    vm.kubevirt.io/template.version: v0.27.0
spec:
  dataVolumeTemplates:
    - apiVersion: cdi.kubevirt.io/v1beta1
      kind: DataVolume
      metadata:
        creationTimestamp: null
        name: rhel9-demo
      spec:
        sourceRef:
          kind: DataSource
          name: rhel9
          namespace: openshift-virtualization-os-images
        storage:
          resources:
            requests:
              storage: 30Gi
  running: true
  template:
    metadata:
      annotations:
        vm.kubevirt.io/flavor: small
        vm.kubevirt.io/os: rhel9
        vm.kubevirt.io/workload: server
      creationTimestamp: null
      labels:
        kubevirt.io/domain: rhel9-demo
        kubevirt.io/size: small
    spec:
      accessCredentials:
        - sshPublicKey:
            propagationMethod:
              noCloud: {}
            source:
              secret:
                secretName: rhel-ssh-key
      architecture: amd64
      domain:
        cpu:
          cores: 1
          sockets: 1
          threads: 1
        devices:
          disks:
            - disk:
                bus: virtio
              name: rootdisk
            - disk:
                bus: virtio
              name: cloudinitdisk
          interfaces:
            - macAddress: '02:9b:54:00:00:05'
              masquerade: {}
              model: virtio
              name: default
          networkInterfaceMultiqueue: true
          rng: {}
        features:
          acpi: {}
          smm:
            enabled: true
        firmware:
          bootloader:
            efi: {}
        machine:
          type: pc-q35-rhel9.2.0
        memory:
          guest: 2Gi
        resources: {}
      networks:
        - name: default
          pod: {}
      terminationGracePeriodSeconds: 180
      volumes:
        - dataVolume:
            name: rhel9-demo
          name: rootdisk
        - cloudInitNoCloud:
            userData: |-
              #cloud-config
              user: cloud-user
              password: e8yf-fwwg-ce7p
              chpasswd: { expire: False }
          name: cloudinitdisk

EOF
```

## Configuring AAP

Before you can continue, you need to install the AWX CLI, as we will use these commands to create Ansible jobs, templates, etc.

You can install this on your local machine and use the oc login command to access the AAP environment. Otherwise, you can start an Ansible DevSpace and use the terminal within VSCode.

### Install Ansible Tower CLI

Use the following pip command to install the AWX CLI

```bash
pip install ansible-tower-cli --user
```

### Create Credential

We decided to create a Credential manually for this environment as it involves using a private key, and it was a much simpler option than trying to automate the process.

Use the following steps to create your Credential.

- Name the Credential 'RHEL SSH Credential'
- Make sure to include the username, e.g. cloud-user
- You will need to copy the private key information that you created earlier
- Make sure the 'Privileged Escalation Method' is set to 'sudo'

### Create AAP Project - Register RHEL Post Install Configuration

The following command will create an AAP Project that we will use to run post-installation activities on the RHEL 9 demo VM.

```bash
awx-cli project create -h $(oc get route -n aap -o jsonpath='{.spec.host}' aap) -u admin -p $(oc get secret -n aap aap-admin-password -o jsonpath='{.data.password}' | base64 --decode) -n "RHEL Post Install Configuration" --organization Default --scm-type git --scm-url https://github.com/Deim0s13/DevWorkstation.git --scm-update-on-launch true
```

### Create AAP Project - Developer Workstation

The following command will create another Ansible Project that we will use to configure the Developer Workstation.

```bash
awx-cli project create -h $(oc get route -n aap -o jsonpath='{.spec.host}' aap) -u admin -p $(oc get secret -n aap aap-admin-password -o jsonpath='{.data.password}' | base64 --decode) -n "Developer Workstation" --organization Default --scm-type git --scm-url https://github.com/Deim0s13/DevWorkstation.git --scm-update-on-launch true
```

### Create a Smart Inventory

We have created a Smart Inventory to target the RHEL 9 demo VM. For this to work successfully, it is dependent on the existing inventory.

```bash
awx-cli inventory create -h $(oc get route -n aap -o jsonpath='{.spec.host}' aap) -u admin -p $(oc get secret -n aap aap-admin-password -o jsonpath='{.data.password}' | base64 --decode) -n "RHEL VMs" --organization Default --kind smart --host-filter name__icontains=rhel
```

### Sync Inventory

The Smart Inventory is dependent on the Virtualisation Inventory. It would help if you synced this for the Smart Inventory to pick up the RHEL 9 Demo VM.

Future update: Add a command to run both needed sync jobs.

### Create AAP Job Template - Post VM Install Configuration

Create a Job Template to run the post-install-configuration playbook to register the RHEL VM and enable the VSCode repository.

Note: Remember to add variables
org_id: <Insert Org ID>
activationkey: <Insert activationkey>

```bash
awx-cli job_template create -h $(oc get route -n aap -o jsonpath='{.spec.host}' aap) -u admin -p $(oc get secret -n aap aap-admin-password -o jsonpath='{.data.password}' | base64 --decode) -n "RHEL Configuration" --job-type run -i "RHEL VMs" --project "RHEL Post Install Configuration" --playbook post-install-configuration.yml --credential "RHEL SSH Credential"
```

### Create AAP Job Template - Apply Developer Workstation Configuration

Create a Job Template to run the configure-dev-workstation playbook, which will configure and install tools like VSCode.

```bash
awx-cli job_template create -h $(oc get route -n aap -o jsonpath='{.spec.host}' aap) -u admin -p $(oc get secret -n aap aap-admin-password -o jsonpath='{.data.password}' | base64 --decode) -n "Configure Developer Workstation" --job-type run -i "RHEL VMs" --project "Developer Workstation" --playbook configure-dev-workstation.yml --credential "RHEL SSH Credential"
```
### Run both Ansible Jobs

The next step is to run the two jobs. We recommend running these within the Ansible Platform.

- Run the RHEL Configuration job first.
- Run the Configure Developer Workstation job upon successful completion of the first job.
- Once completed, you can log in to the RHEL 9 demo VM to validate that you successfully installed Podman and VSCode.

Future improvement: Add command to run jobs
