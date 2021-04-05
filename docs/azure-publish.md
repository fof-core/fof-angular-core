# Github to Azure blob storage

High levels steps to publish an Angular app from github to an Azure blob storage configured as a static website.  
- It assumes an Angular project already exists into github.  
- You must have, at least, an [Azure free tiers account](https://azure.microsoft.com/en-gb/free/free-account-faq/).

## Create an azure blob storage and configure it as a static website

First, [sign in to the Azure portal](https://portal.azure.com/)

Search for `Storage accounts` and click `+ Add`
- Subscription: free tiers
- Ressource group: `Create new` -> name-of_your_project_rg (everything in this tuto will be under that ressource group)
- Storage account name: name-of_your_project_test (the storage for our test environment)
- Location: Best for your needs. eg for Europe: (Europe) North Europe (cheapest)
- Performance: standard
- Account kind: Storage v2
- Replication: RA-GRS (cheapeast)
- Review to create tab -> create (all other options: default)

Wait for `Deployment is in progress` is done and go to the ressource name-of_your_project_test

Go to Static website on the left menu (or search for it)
 - Enable it 
 - Index document name: index.html
 - Error document path: index.html. All non managed routes will be redirect to index.html: perfect for a SPA
 - Save

You should see
- An Azure Storage container has been created to host your static website. $web
- Primary endpoint: https://name-of_your_project_test.[random].web.core.windows.net/

> IMPORTANT: Keep the `Primary endpoint` and the `name of the created storage account` for further needs

## Configure Azure deployment credentials 

You can find detailled documentation [on the azure-login marketplace for github](https://github.com/marketplace/actions/azure-login#configure-deployment-credentials).

in short: you need to create a service principal whose will be used to deploy to azure


- go to azure portal
- Open the Azure [Cloud Shell](https://shell.azure.com). You can alternately use the Azure CLI if you've installed it locally.

````bash 
az ad sp create-for-rbac --name "{sp-name}" --sdk-auth --role contributor \
    --scopes /subscriptions/{subscription-id}/resourceGroups/{resource-group}
````

Replace the following
- {sp-name} with a suitable name for your service principal, such as the name of the project itself.
- {subscription-id} with the subscription ID you want to use (found in Subscriptions in portal)
- {resource-group} the resource group you created at the beggining.

In return, you should have a json object like the following one 
> IMPORTANT: Keep it for further needs 

````json
{
  "clientId": "<GUID>",
  "clientSecret": "<GUID>",
  "subscriptionId": "<GUID>",
  "tenantId": "<GUID>",
  (...)
}
````

## Configure github secrets

We will store the azure credentials in github vars we can use to deploy

First, [sign in to the github](https://github.com/)

Go to organisation, or project, or repo settings (highest level to simplify this tuto) 
and select `secrets`
- new [organization/project/repo] secret
- name: AZURE_CREDENTIALS
- value: the entire json object returned by the `az ad sp create-for-rbac...` command line


## Configure github action to build & deploy to azure

We will create the first workflow with the github interface because it will create for us the `.github/workflows` folder and initlize everything for us.

Go to your angular repo and select `Actions`
- new workflows
- set up a workflow yourself 
- copy/past the following and change `{your_storage_account_name}` with the `name of the created storage account`

````yaml
name: Publish Static Web App to Azure Blob Storage

on:
  push:
    branches:
      - master
    paths-ignore:
      - '**.md'
      - 'docs/**'
  pull_request:
    paths-ignore:
      - '**.md'
      - 'docs/**'

jobs:
  build_and_publish:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout the repo
      uses: actions/checkout@v1

    - name: Login to Azure
      uses: Azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: Install npm packages
      shell: bash
      run: |        
        npm install
    - name: Build app
      shell: bash
      run: |        
        npm run build
#     - name: Test app
#       shell: bash
#       run: |        
#         npm run test:unit
    - name: Publish app
      uses: Azure/cli@v1.0.0
      with:
        azcliversion: latest
        inlineScript: |
          az storage blob upload-batch -s $GITHUB_WORKSPACE/dist/fof-angular-core -d \$web --account-name {your_storage_account_name}
````

- Save with `Start commit` -> `Commit new file`  

Because we use `on: push` in our yaml config file for github actions and we commit it on the master branch, the action should be fired directly.  
If everything was going fine, have a look on the `Primary endpoint` url you saved at the beggining of this tuto: your Angular app should be up there.











 



