# ARM & Bicep Deployment for Tailscale subnet router in ACI

These Bicep & ARM templates deploy a Tailscale [subnet router][1] as an [Azure Container Instance][2]. The subnet router ACI instance is deployed into an existing Azure Virtual Network and advertises to your Tailnet the CIDR block for the Azure VNet.

<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fcocallaw%2Fazure-tailscale-aci-deploy%2Fmain%2FARM%2Fazuredeploy.json" target="_blank">
    <img src="https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/1-CONTRIBUTION-GUIDE/images/deploytoazure.svg?sanitize=true"/> 
</a>

## Deployment 
When deploying the ARM or Bicep templates, the value of the `containerRegistry` parameter will determine where the deployment pulls the container image from. 
- If `DockerHub` is selected, the image will be pulled from [cocallaw/tailscale-sr on Docker Hub][3], the parameters `tailscaleImageRepository` and `tailscaleImageRepository` are not used and can be left to their default values or null.
- If `ACR` is selected, the image will be pulled from Azure Container Registry using the values of the `tailscaleImageRepository` and `tailscaleImageRepository` parameters.
## Docker Container
The `docker/Dockerfile` file extends the `tailscale/tailscale`
[image][4] with an entrypoint script that starts the Tailscale daemon and runs
`tailscale up` using an [auth key][5] and the relevant advertised [CIDR block][6].

The Docker container must be built and pushed to an ACR if the parameter `containerRegistry` is set to `ACR` so that it can be referenced during deployment. If the parameter `containerRegistry` is set to `DockerHub`, the container does not need to be built as it will be pulled from Docker Hub.

### Build locally with Docker and [push image to ACR][7]
```bash
docker build \
  --tag tailscale-subnet-router:v1 \
  --file ./docker/Dockerfile \
  .

# Optionally override the tag for the base `tailscale/tailscale` image
docker build \
  --build-arg TAILSCALE_TAG=v1.29.18 \
  --tag tailscale-subnet-router:v1 \
  --file ./docker/Dockerfile \
  .
```

### Build remotely using [Azure Container Registry Tasks][8] with Azure CLI
```bash
ACR_NAME=<registry-name>
az acr build --registry $ACR_NAME --image tailscale:v1 .

# Optionally override the tag for the base `tailscale/tailscale` image
ACR_NAME=<registry-name>
az acr build --registry $ACR_NAME --build-arg TAILSCALE_TAG=v1.29.18 --image tailscale:v1 .
```

## Subnet Delegation 
To assist with the deployment of the ACI container group in the Azure VNet, the subnet being used should be [delegated][9] to the `Microsoft.ContainerInstance/containerGroups`.
```bash
# Update the subnet with a delegation for Microsoft.ContainerInstance/containerGroups
az network vnet subnet update \
  --resource-group myResourceGroup \
  --name mySubnet \
  --vnet-name myVnet \
  --delegations Microsoft.ContainerInstance/containerGroups

# Verify that the subnet is now delegated to the ACI instance
  az network vnet subnet show \
  --resource-group myResourceGroup \
  --name mySubnet \
  --vnet-name myVnet \
  --query delegations
```  

## Notes
- The Tailscale state (`/var/lib/tailscale`) is stored in a [Azure File Share][10] in a Storage Account so that the subnet router only needs to be [authorized][11] once.

## Improvements Needed
### Container Registry Authentication
Currently the templates only support using a username and password to authenticate to the ACR repository, and the server URL is derived from the ACR repository name.
- Validation testing needed for use with Docker Hub
- Add Option to use [anonymous pull][12] with ACR
- Investigate using a service principal to authenticate to the ACR repository

### Container Size Selection
When the Tailscale container is deployed, the size is set to 1 CPU core and 1 GiB of memory. Currently there is no option to adjust this size, unless the template file is edited.
- Add Variable Option to adjust the size of the ACI container. Possible Small/Med/Large options that are available for deployment but easily defined by the user. 


[1]: https://tailscale.com/kb/1019/subnets/
[2]: https://docs.microsoft.com/azure/container-instances/container-instances-overview
[3]: https://hub.docker.com/r/cocallaw/tailscale-sr
[4]: https://hub.docker.com/r/tailscale/tailscale
[5]: https://tailscale.com/kb/1085/auth-keys/
[6]: https://tailscale.com/kb/1019/subnets/
[7]: https://docs.microsoft.com/azure/container-registry/container-registry-get-started-docker-cli?tabs=azure-cli
[8]: https://docs.microsoft.com/azure/container-registry/container-registry-tutorial-quick-task
[9]: https://docs.microsoft.com/azure/virtual-network/subnet-delegation-overview
[10]: https://docs.microsoft.com/azure/storage/files/storage-files-introduction
[11]: https://tailscale.com/kb/1099/device-authorization/
[12]: https://docs.microsoft.com/azure/container-registry/anonymous-pull-access
