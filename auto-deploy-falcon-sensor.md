# GCP Auto Deploy

```
Client ID
9c26b81db0b4xxxxxxxxc419a536801

Secret
ClcsU23xxxxxxxxxxxxxxxxxxoIu8Z1VH9m

Base URL
https://api.crowdstrike.com
```

quay.io/crowdstrike/detection-container

Startup Script
```
 #! /bin/bash
export FALCON_CLIENT_ID="9c26b81db0b4xxxxxxxxc419a536801"
export FALCON_CLIENT_SECRET="ClcsU23xxxxxxxxxxxxxxxxxxoIu8Z1VH9m"
export FALCON_BACKEND=bpf
curl -L https://raw.githubusercontent.com/crowdstrike/falcon-scripts/v1.3.3/bash/install/falcon-linux-install.sh | bash
``` 

gcloud script
```
INSTANCE_ID=

# Deploy
gcloud compute ssh --zone us-central1-a 3084xxxxxxxxxxxxx4319 -- 'export FALCON_CLIENT_ID="9c26b81db0b4xxxxxxxxc419a536801" && export FALCON_CLIENT_SECRET="ClcsU23xxxxxxxxxxxxxxxxxxoIu8Z1VH9m" && curl -L https://raw.githubusercontent.com/crowdstrike/falcon-scripts/v1.3.3/bash/install/falcon-linux-install.sh | sudo -E bash'

# Deploy w bpf
gcloud compute ssh --zone us-central1-a 3084xxxxxxxxxxxxx4319 -- 'export FALCON_CLIENT_ID="9c26b81db0b4xxxxxxxxc419a536801" && export FALCON_CLIENT_SECRET="ClcsU23xxxxxxxxxxxxxxxxxxoIu8Z1VH9m" && export FALCON_BACKEND=bpf && curl -L https://raw.githubusercontent.com/crowdstrike/falcon-scripts/v1.3.3/bash/install/falcon-linux-install.sh | sudo -E bash'

# Validate
gcloud compute ssh --zone us-central1-a 3084xxxxxxxxxxxxx4319 -- ' \
export FALCON_CLIENT_ID="9c26b81db0b4xxxxxxxxc419a536801" && export FALCON_CLIENT_SECRET="ClcsU23xxxxxxxxxxxxxxxxxxoIu8Z1VH9m" && curl -L https://raw.githubusercontent.com/crowdstrike/falcon-scripts/v1.3.3/bash/install/falcon-linux-install.sh \
| sudo -E bash'

# Check aid
gcloud compute ssh --zone us-central1-a 3084xxxxxxxxxxxxx4319 -- 'sudo /opt/CrowdStrike/falconctl -g --aid'

# Check RFM
gcloud compute ssh --zone us-central1-a 3084xxxxxxxxxxxxx4319 -- 'sudo /opt/CrowdStrike/falconctl -g --rfm-state'

# Check systemctl
gcloud compute ssh --zone us-central1-a 3084xxxxxxxxxxxxx4319 -- 'systemctl status falcon-sensor'
```


Command line to run in cloud shell
```
gcloud compute instance-templates create-with-container auto-install-demo-w-sensor --project=ykwan-421807 --machine-type=e2-micro --network-interface=network=default,network-tier=PREMIUM --no-restart-on-failure --maintenance-policy=TERMINATE --provisioning-model=SPOT --instance-termination-action=STOP --service-account=4184xxxxxxx1-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --container-image=quay.io/crowdstrike/detection-container --container-restart-policy=always --create-disk=auto-delete=yes,boot=yes,device-name=auto-install-demo,image=projects/cos-cloud/global/images/cos-stable-109-17800-147-60,mode=rw,size=10,type=pd-standard --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --labels=container-vm=cos-stable-109-17800-147-60
```

# AWS

```
# API keys
# Scope: 
CLIENT_ID=9c26b81db0b4xxxxxxxxc419a536801
SECRET=ClcsU23xxxxxxxxxxxxxxxxxxoIu8Z1VH9m
BASE_URL=https://api.crowdstrike.com

# Create SSM Parameters
aws ssm put-parameter \
    --name "/CrowdStrike/Falcon/ClientId" \
    --type "SecureString" \
    --description "CrowdStrike Falcon API Client ID for the distributor package" \
    --region "ap-southeast-1" \
    --value "$CLIENT_ID"
aws ssm put-parameter \
    --name "/CrowdStrike/Falcon/ClientSecret" \
    --type "SecureString" \
    --description "CrowdStrike Falcon API Secret for the distributor package" \
    --region "ap-southeast-1" \
    --value "$SECRET"
aws ssm put-parameter \
    --name "/CrowdStrike/Falcon/Cloud" \
    --type "SecureString" \
    --description "CrowdStrike Falcon API Base URL for the distributor package" \
    --region "ap-southeast-1" \
    --value "$BASE_URL"

# Create IAM role
aws iam create-role \
    --role-name "ykwan-crowdstrike-distributor-deploy-role" \
    --assume-role-policy-document '{
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "Service": "ssm.amazonaws.com"
                },
                "Action": "sts:AssumeRole"
            }
        ]
    }' \
    --description "Role for running SSM automation documents" \
    --max-session-duration 3600

aws iam attach-role-policy \
    --role-name ykwan-crowdstrike-distributor-deploy-role \
    --policy-arn arn:aws:iam::aws:policy/service-role/AmazonSSMAutomationRole    


aws ssm create-association \
    --name "CrowdStrike-FalconSensorDeploy" \
    --targets "Key=InstanceIds,Values=*" \
    --parameters "AutomationAssumeRole=arn:aws:iam::517716713836:role/ykwan-crowdstrike-distributor-deploy-role, SecretStorageMethod=ParameterStore, FalconCloud=/CrowdStrike/Falcon/Cloud, FalconClientId=/CrowdStrike/Falcon/ClientId, FalconClientSecret=/CrowdStrike/Falcon/ClientSecret, Action=Install" \
    --association-name "ykwan-crowdstrike-distributor-deploy" \
    --automation-target-parameter-name "InstanceIds" \
    --region "ap-southeast-1"

https://github.com/CrowdStrike/aws-ssm-distributor/blob/main/official-package/README.md
https://us-east-1.console.aws.amazon.com/iam/home?region=ap-southeast-1#/roles/details/ykwan-crowdstrike-distributor-deploy-role?section=permissions
https://ap-southeast-1.console.aws.amazon.com/ec2/home?region=ap-southeast-1#Instances:
https://ap-southeast-1.console.aws.amazon.com/systems-manager/state-manager/6cadd1a8-bb94-48c0-a473-cf8049611445/description?region=ap-southeast-1


```

# Azure

```
az sig gallery-application create \
    --application-name myApp \
    --gallery-name myGallery \
    --resource-group myResourceGroup \
    --os-type Linux \
    --location "East US"

az sig gallery-application version create \
   --version-name 1.0.0 \
   --application-name myApp \
   --gallery-name myGallery \
   --location "East US" \
   --resource-group myResourceGroup \
   --package-file-link "https://<storage account name>.blob.core.windows.net/<container name>/<filename>" \
   --install-command "mv myApp .\myApp\myApp" \
   --remove-command "rm .\myApp\myApp" \
   --update-command  "mv myApp .\myApp\myApp" \
   --default-configuration-file-link "https://<storage account name>.blob.core.windows.net/<container name>/<filename>"\

az vm application set \
	--resource-group myResourceGroup \
	--name myVM \
  	--app-version-ids /subscriptions/{subID}/resourceGroups/MyResourceGroup/providers/Microsoft.Compute/galleries/myGallery/applications/myApp/versions/1.0.0 \
  	--treat-deployment-as-failure true
```

VM App

Ensure the following API scopes are enabled:

Install:
Sensor Download [read]
Sensor update policies [read]
Uninstall:
Host [write]
Sensor update policies [write]

```
powershell.exe -command "Expand-Archive -Path .\falcon-install.zip .; .\falcon_windows_install.ps1 -FalconClientId 9c26b81db0b4xxxxxxxxc419a536801 -FalconClientSecret O0oPjIJKips56RAFgcXbd97hW32C8ZH14VnLxBQk

powershell.exe -command "Expand-Archive -Path .\falcon-install.zip .; .\falcon_windows_uninstall.ps1 -FalconClientId "9c26b81db0b4xxxxxxxxc419a536801" -FalconClientSecret "O0oPjIJKips56RAFgcXbd97hW32C8ZH14VnLxBQk" -RemoveHost
```

Azure Policy
```
{
  "properties": {
    "displayName": "ykwan-falcon-sensor-windows-installation",
    "policyType": "Custom",
    "mode": "All",
    "metadata": {
      "category": "",
      "createdBy": "ea37856f-b025-47e3-98b4-22240e4efa69",
      "createdOn": "2024-05-04T19:20:51.9976029Z",
      "updatedBy": "ea37856f-b025-47e3-98b4-22240e4efa69",
      "updatedOn": "2024-05-04T19:50:55.3390861Z"
    },
    "version": "1.0.0",
    "parameters": {},
    "policyRule": {
      "if": {
        "allOf": [
          {
            "field": "type",
            "equals": "Microsoft.Compute/virtualMachines"
          },
          {
            "field": "Microsoft.Compute/virtualMachines/storageProfile.osDisk.osType",
            "equals": "Windows"
          }
        ]
      },
      "then": {
        "effect": "deployIfNotExists",
        "details": {
          "type": "Microsoft.Compute/virtualMachines",
          "existenceCondition": {
            "allOf": [
              {
                "count": {
                  "field": "Microsoft.Compute/virtualMachines/applicationProfile.galleryApplications[*]",
                  "where": {
                    "field": "Microsoft.Compute/virtualMachines/applicationProfile.galleryApplications[*].packageReferenceId",
                    "equals": "/subscriptions/5a84cb53-b383-44db-bd58-c65ca3dfcb8c/resourceGroups/ykwan-lab/providers/Microsoft.Compute/galleries/ykwanComputeGallery/applications/crowdstrike-falcon-sensor-windows/versions/0.0.18"
                  }
                },
                "greater": 0
              }
            ]
          },
          "roleDefinitionIds": [
            "/providers/Microsoft.Authorization/roleDefinitions/8e3af657-a8ff-443c-a75c-2fe8c4bcb635"
          ],
          "deployment": {
            "properties": {
              "mode": "incremental",
              "template": {
                "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json",
                "contentVersion": "1.0.0.0",
                "parameters": {
                  "vmName": {
                    "type": "string"
                  },
                  "location": {
                    "type": "string"
                  }
                },
                "resources": [
                  {
                    "apiVersion": "2021-07-01",
                    "type": "Microsoft.Compute/virtualMachines/VMapplications",
                    "name": "[concat(parameters('vmName'),'/crowdstrike-falcon-sensor-windows')]",
                    "location": "[parameters('location')]",
                    "properties": {
                      "packageReferenceId": "/subscriptions/5a84cb53-b383-44db-bd58-c65ca3dfcb8c/resourceGroups/ykwan-lab/providers/Microsoft.Compute/galleries/ykwanComputeGallery/applications/crowdstrike-falcon-sensor-windows/versions/0.0.18"
                    }
                  }
                ]
              },
              "parameters": {
                "vmName": {
                  "value": "[field('name')]"
                },
                "location": {
                  "value": "[field('location')]"
                }
              }
            }
          }
        }
      }
    },
    "versions": [
      "1.0.0"
    ]
  },
  "id": "/subscriptions/5a84cb53-b383-44db-bd58-c65ca3dfcb8c/providers/Microsoft.Authorization/policyDefinitions/d3ea8c47-64f9-4a70-9299-cb8060ed6a2b",
  "type": "Microsoft.Authorization/policyDefinitions",
  "name": "d3ea8c47-64f9-4a70-9299-cb8060ed6a2b",
  "systemData": {
    "createdBy": "ykwan@crowdstrike-azure-lab.io",
    "createdByType": "User",
    "createdAt": "2024-05-04T19:20:51.979624Z",
    "lastModifiedBy": "ykwan@crowdstrike-azure-lab.io",
    "lastModifiedByType": "User",
    "lastModifiedAt": "2024-05-04T19:50:55.3148904Z"
  }
}
```

Demo Flow
- VM App Overview: https://learn.microsoft.com/en-us/azure/virtual-machines/vm-applications?tabs=ubuntu
- VM App: https://portal.azure.com/#@crowdstrikeintegrationsgmai.onmicrosoft.com/resource/subscriptions/5a84cb53-b383-44db-bd58-c65ca3dfcb8c/resourceGroups/ykwan-lab/providers/Microsoft.Compute/galleries/ykwancomputegallery/applications/crowdstrike-falcon-sensor-windows/versions/0.0.18/overview
- Assign to VM 
- Compliance: https://portal.azure.com/#view/Microsoft_Azure_Policy/PolicyComplianceDetail.ReactView/assignmentId/%2Fsubscriptions%2F5a84cb53-b383-44db-bd58-c65ca3dfcb8c%2Fresourcegroups%2Fykwan-lab%2Fproviders%2Fmicrosoft.authorization%2Fpolicyassignments%2F55e8516d293d4169b01580f4/scopes~/%5B%22%2Fsubscriptions%2F5a84cb53-b383-44db-bd58-c65ca3dfcb8c%22%5D/policyDefinitionId/%2Fsubscriptions%2F5a84cb53-b383-44db-bd58-c65ca3dfcb8c%2Fproviders%2Fmicrosoft.authorization%2Fpolicydefinitions%2Fd3ea8c47-64f9-4a70-9299-cb8060ed6a2b

- Scaling: https://portal.azure.com/#@crowdstrikeintegrationsgmai.onmicrosoft.com/resource/subscriptions/5a84cb53-b383-44db-bd58-c65ca3dfcb8c/resourceGroups/ykwan-lab/providers/Microsoft.Compute/virtualMachineScaleSets/ykwan-hcl-auto-deploy-demo/scaling

- Falcon Console: https://falcon.crowdstrike.com/host-management/hosts?sort=first_seen.desc


Other Docs
- https://www.georgeollis.com/using-vm-applications-and-azure-policy/
- https://learn.microsoft.com/en-us/azure/virtual-machines/vm-applications-how-to?tabs=portal
- https://luke.geek.nz/azure/azure-vm-application-deployment/
- https://devblogs.microsoft.com/azure-vm-runtime/managing-vm-applications-with-azure-policies/
- https://learn.microsoft.com/en-us/azure/governance/policy/how-to/get-compliance-data#evaluation-triggers
- https://msftplayground.com/2019/06/on-demand-azure-policy-scan/


The evaluation of the policies take place on the following events:

* New policy assignment. ( ~30 minutes).
* Updated assignment of a existing policy. (~ 30 minutes)
* Deployment of a resources. (~15 minutes)
* Every 24 hours
* Update of the resource provider
* On-demand (~3 minutes)


- https://learn.microsoft.com/en-us/azure/governance/policy/how-to/remediate-resources?source=recommendations&tabs=azure-portal


Force Check
```
az policy state trigger-scan --resource-group "ykwan-lab"
```