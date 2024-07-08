1# DevSpace Developer Workstation Demo
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
