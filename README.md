# DevSpace Developer Workstation Demo
This repository contains a short walkthrough demonstration of using Ansible to configure an OpenShift Dev Spaces instance and an OpenShift Virtualization machine.


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


## Create RHEL VM

```bash
cat << 'EOF' | oc apply --filename -
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: rhel9-vm #Change VM name if necessary
  namespace: default
  finalizers:
    - kubevirt.io/virtualMachineControllerFinalize
  labels:
    app: rhel9-lime-rat-17
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
        name: rhel9-vm
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
        kubevirt.io/domain: rhel9-vm
        kubevirt.io/size: small
    spec:
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
            - macAddress: '02:61:92:00:00:03'
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
            name: rhel9-vm
          name: rootdisk
        - cloudInitNoCloud:
            userData: |-
              #cloud-config
              user: cloud-user
              password: g0h5-ckn2-u7xd
              chpasswd: { expire: False }
          name: cloudinitdisk
EOF
```


## Configuring AAP

Install AWX CLI to automation the configuration required in AAP - AWX CLI documentation: https://docs.ansible.com/ansible-tower/latest/html/towercli/usage.html#getting-started

### Install Ansible Tower CLI

Command to install Ansible Tower CLI

```bash
pip install ansible-tower-cli --user
```

### Create AAP Project

Command that will create a new Project in AAP using Git as the source

```bash
awx-cli project create -h $(oc get route -n aap -o jsonpath='{.spec.host}' aap) -u admin -p $(oc get secret -n aap aap-admin-password -o jsonpath='{.data.password}' | base64 --decode) -n devspaces --organization Default --scm-type git --scm-url https://github.com/Deim0s13/DevWorkstation.git
```

### Create a Smart Inventory

Command to create a Smart Inventory that filters based on rhel9

```bash
awx-cli inventory create -h $(oc get route -n aap -o jsonpath='{.spec.host}' aap) -u admin -p $(oc get secret -n aap aap-admin-password -o jsonpath='{.data.password}' | base64 --decode) -n rhel9 --organization Default --kind smart --host-filter name__icontains=rhel
```

### Create AAP Job Template

```bash
awx-cli job_template create -h $(oc get route -n aap -o jsonpath='{.spec.host}' aap) -u admin -p $(oc get secret -n aap aap-admin-password -o jsonpath='{.data.password}' | base64 --decode) -n devworkstaion-playbook --job-type run -i rhel9 --project devspaces --playbook setup_dev_workstation.yml
```
