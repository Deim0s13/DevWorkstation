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
