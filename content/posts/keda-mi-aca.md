---
title: "Simplifying Event-Driven Architectures: KEDA Scaling with Managed Identity in Azure Container Apps"
date: 2024-07-09T13:58:40-05:00
tags: ["Azure", "KEDA", "microservices", "managedidentity", "cloudcomputing", "kubernetes"]
toc: true
---
## Introduction
I've been using [Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/overview) (ACA) for some time, It has been a perfect fit for some use cases where you want to avoid managing a whole Kubernetes environment or don't want to use a shared one. But, not everything can be perfect; one limitation that some of us found, was that it wasn't possible to have [KEDA](https://keda.sh/) authenticate using a [Managed Identities](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview) (MI).

There were ways to get around this limitation, as I described on this [GitHub issue](https://github.com/microsoft/azure-container-apps/issues/592#issuecomment-1950668036) and one of my previous [blog posts](https://jemrpo.com/posts/az-ca-202402/); but you ended up having to write a lot of glue code to build a small microservice and maintaining this will add more operational overhead and will make the architecture more complex. This is fun to assemble, but most of us are busy, so a simpler architecture or solution would be ideal.

## The Microsoft Azure Team to the Rescue
After a long wait, the team in charge of this service [announced](https://github.com/microsoft/azure-container-apps/issues/592#issuecomment-1960110969) they were working on this feature. Finally, we were seeing the light at the end of the tunnel. Then, on 20240625 we finally got the [news we were waiting for](https://github.com/microsoft/azure-container-apps/issues/592#issuecomment-2150783618): the feature was being rolled out with the API version `2024-02-02-preview`.

## Testing the New Feature
Although there are many use cases and Azure services to which this can be applied, I'm going to focus on the [azure-pipelines](https://keda.sh/docs/2.13/scalers/azure-pipelines/) use case. This allows you to scale your [self-hosted Azure DevOps agents](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/docker?view=azure-devops#linux) based on the pool queue.

Before this feature, the `rules` section of the Azure Container Apps [ARM template](https://learn.microsoft.com/en-us/azure/templates/microsoft.app/containerapps?pivots=deployment-language-arm-template) would look like this.
```json
"rules": [
    {
        "name": "azure-pipelines",
        "type": "azure-pipelines",
        "metadata": {
            "organizationURLFromEnv": "AZP_URL",
            "personalAccessTokenFromEnv": "AZP_TOKEN",
            "poolID": "${adoPoolId}",
            "poolName": "${azPool}",
        },
        "auth": [
            {
                "secretRef": "organizationurl",
                "triggerParameter": "organizationURL"
            },
            {
                "secretRef": "azptoken",
                "triggerParameter": "personalAccessToken"
            }
        ]
    }
]
```
As you might have guessed, KEDA authenticates with [Azure DevOps](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/docker?view=azure-devops#linux) using the Organization URL, the Personal Access Token or Bearer Token (depending on your implementation), and the poolID or poolName.

Now, with the new feature on API version `2024-02-02-preview`, we need to specify the Managed Identity we want to use, the Organization URL, and either the poolID or poolName and under the hood, it will obtain the Bearer Token to authenticate with Azure DevOps, as shown in the code below.
```json
"rules": [
    {
        "name": "azure-pipelines",
        "type": "azure-pipelines",
        "identity": "${managedIdentity}",
        "metadata": {
            "poolID": "${adoPoolId}",
            "poolName": "${azPool}",
        },
        "auth": [
            {
                "secretRef": "organizationurl",
                "triggerParameter": "organizationURL"
            }
        ]
    }
]
```

After applying these changes (I used [OpenTofu](https://opentofu.org/) and the [AzAPI](https://registry.terraform.io/providers/Azure/azapi/latest/docs) provider), we can test. Since I'm lazy, I created 10 jobs using the following command.
```bash
for i in {1..10}; do az pipelines run  --organization "https://dev.azure.com/AwesomeORG" \
    --branch "main" --name "adoagents-test" --project "AwesomeProject" ; sleep 3; done
```

Since I'm using Azure Container Apps Jobs, we have to go to the **Monitoring** > **Execution History** section to see if the jobs started.
![alt text](aca-job.png "ACA Job on Azure Portal")

And if we go to the **Agent Pool** section in **Azure DevOps**, we can see that the jobs executed successfully.
![alt text](ado-job.png "Job on ADO").

## Conclusion
As you just saw, it works after making a few modifications. One of the main benefits of this new feature is that the overall architecture is simplified as you no longer need glue code or additional microservices to renew Bearer Tokens and update the secrets, or to rely on on [Personal Access Tokens](https://learn.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&tabs=Windows) which are insecure and cumbersome to manage.

I used the [azure-pipelines](https://keda.sh/docs/2.14/scalers/azure-pipelines/) integration as an example, but this feature has been proven to work with other services such as [Service Bus](https://keda.sh/docs/2.14/scalers/azure-service-bus/), [Azure Storage Queues](https://keda.sh/docs/2.14/scalers/azure-storage-queue/) and more, this is great news because this feature will help to simplify your [Event Driven Architectures](https://en.wikipedia.org/wiki/Event-driven_architecture).

For additional information, check the following documentation.
- [Managed identities in Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/managed-identity?tabs=portal%2Cdotnet#use-managed-identity-for-scale-rules).
- [Set scaling rules in Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/scale-app?pivots=azure-cli#authentication-1).
- KEDA [Azure Pipelines](https://keda.sh/docs/2.14/scalers/azure-pipelines/).
- KEDA [Azure Service Bus](https://keda.sh/docs/2.14/scalers/azure-service-bus/).
- KEDA [Azure Storage Queue](https://keda.sh/docs/2.14/scalers/azure-storage-queue/)

In case you have any comments or questions, just let me know. You can always find me on [LinkedIn](https://www.linkedin.com/in/juanestebanmrpo/).