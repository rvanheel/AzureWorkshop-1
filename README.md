You can run the **Azure Powershell** commands either in the **Azure Portal CLI**, or on your **computer**.

What you need or need to do for this workshop:
* An Azure Subscription
* Install [Git](https://git-scm.com/downloads) on your computer
* Optional, Install [NodeJs](https://nodejs.org/en/download) on your computer
* [Visual Studio 2015+ (Community, Pro or Enterprise)](https://visualstudio.microsoft.com/downloads/) or [Visual Studio Code](https://code.visualstudio.com/download)
* Optional, Install the [Azure Powershell CmdLets](https://docs.microsoft.com/en-us/powershell/azure/install-az-ps?view=azps-3.4.0) if you would like to run the Azure Powershell Commands from your computer

Run the following commands only from a terminal session on your **computer** ( NO NEED TO DO THIS IF YOU USE THE AZURE PORTAL):
```powershell
# Connect to Azure with a browser sign in token
Connect-AzAccount
# show the list of your active subscriptions
Get-AzSubscription
# select your subscription if necessary
Set-AzContext -SubscriptionId (Get-AzSubscription -SubscriptionName "***YOUR SUBSCRIPTION NAME***").SubscriptionId
```
# 1. Develop and deploy a Web App to Azure

# Set some variables first

$ResourceGroup = "devoteam-demo"
$AppServicePlanName = ("DevoTeamPlan_{0}" -f (Get-Date -format "yyyyMMddHHmmss"))
$WebAppName = ("WebApp-{0}" -f (Get-Date -format "yyyyMMddHHmmss"))
$KeyVaultName = ("KeyVault-{0}" -f (Get-Date -format "yyyyMMddHHmmss"))
$StorageAccount = ("storage{0}" -f (Get-Date -format "yyyyMMddHHmmss"))

1. Create a resource group in the Azure Portal
```powershell
New-AzResourceGroup -Location "westeurope" -Name $ResourceGroup
```
2. Create a empty Web App in the Azure Portal
```powershell
# create a serviceplan first
New-AzAppServicePlan -Location westeurope -ResourceGroupName $ResourceGroup -Name $AppServicePlanName -Tier "Standard"
# create the webapp
New-AzWebApp -Location westeurope -ResourceGroupName $ResourceGroup -AppServicePlan $AppServicePlanName -Name $WebAppName
```
3. Create a .NET Core MVC application
```sh
dotnet new mvc --name WebAppDemo
cd WebAppDemo/
# open with Visual Studio Code
code .

```
4. Initialize a local GIT repository and create a .gitignore file
```sh
git init
touch .gitignore
# make the content of .gitignore to lool like:
bin/
obj/
```
5. Enable GIT deployment on the Web App in the Azure Portal
6. Add Azure deployment to the local GIT repository
```sh
git remote add azure [git_url]
# let's push what we have so far
git add --all
git commit -m "Initial version"
git push azure master --force
# now do a npm package for versioned deployments, accept all defaults
npm init
# modify the scripts section inside package.json to:
```
```json
{
    "scripts": {
        "postversion": "git push azure master --force && git push azure master --tags",
        "version": "git add --all"
    }
}
```
7. Commit the application and publish to Azure
```sh
npm version patch -f
```
8. Create a deployment slot in the Azure Portal for the Web App and enable GIT deployment
```powershell
New-AzWebAppSlot -ResourceGroupName $ResourceGroup -AppServicePlan $AppServicePlanName -Name $WebAppName -Slot "Staging"
```
9. Update the local GIT repository with the deployment slot GIT repository
```sh
git remote set-url azure [deployment_slot_git_url]
```
10. Deploy to the new deployment slot in Azure
```sh
git push azure master --force
```
11. Enable AUTO swap in the Azure Portal for the Web App
```powershell
Set-AzWebAppSlot -ResourceGroupName $ResourceGroup -AppServicePlan $AppServicePlanName -Name $WebAppName -Slot "Staging" -AutoSwapSlotName "production"
```
# 2. Implement Azure Storage using Storage Tables
1. Create a storage account in the Azure Portal
```powershell
New-AzStorageAccount -ResourceGroupName $ResourceGroup -Location "westeurope" -Name $StorageAccount -SkuName "Standard_LRS" -Kind "StorageV2"
```
2. Create a .NET Core application
```sh
dotnet new console --name DemoApp
cd DemoApp
# open with Visual Studio Code
code . 
```
3. Add the Windows Azure Storage package
```sh
# Console
dotnet add package WindowsAzure.Storage --version 9.3.3 
# Visual Studio
Install-Package WindowsAzure.Storage -Version 9.3.3
```
4. create a class based on TableEntity
```C#
public class Customer : TableEntity
{
    public int Age { get; set; }
    public string Firstname { get; set; }
    public string Lastname { get; set; }
}
```
5. create a method to insert random data inside the Storage Table
```C#
private static async Task InsertRecords(int recordCount)
{
    var connectionString = "*** TO BE REPLACED ***";
    var storageAccount = CloudStorageAccount.Parse(connectionString);
    var tableClient = storageAccount.CreateCloudTableClient();
    var table = tableClient.GetTableReference("demo1234");
    await table.CreateIfNotExistsAsync();
    TableBatchOperation batch = new TableBatchOperation();

    var lastNames = new []{"Smith", "Johnson", "Williams", "Jones", "Brown", "Davis", "Miller", "Wilson", "Moore", "Taylor","Anderson"};
    var firstNames = new [] {"Clark", "Allen", "Scott", "Mitchell", "Sue","Sarah", "Jennifer", "Audrey"};

    var rnd = new Random();
    for(int a = 0, l = recordCount; a < l; a++)     { 
        batch.Add(TableOperation.Insert(new Customer{
            Age = rnd.Next(18,85),
            Firstname = firstNames[rnd.Next(firstNames.Length)],
            Lastname = lastNames[rnd.Next(lastNames.Length)],
            PartitionKey = "devoteam",
            RowKey = Guid.NewGuid().ToString()
        }));
    }
    await table.ExecuteBatchAsync(batch);            
}
```
6. Implement the method
```C#
static void Main(string[] args)
{
    Console.Write("Inserting records ... ");
    var sw = System.Diagnostics.Stopwatch.StartNew();
    // the maximum number of records per batch is 100
    InsertRecords(100).GetAwaiter().GetResult();
    sw.Stop();
    Console.WriteLine($"finished in {sw.Elapsed}.");            
}
```
7. Add the Windows Azure Storage package in the Web Application
```sh
# Console
dotnet add package WindowsAzure.Storage --version 9.3.3 
# Visual Studio
Install-Package WindowsAzure.Storage -Version 9.3.3
```
8. Implement the Storage Table in the constructor of the HomeController.cs
```C#
CloudTable table;
public HomeController()
{
    var connectionString = "*** TO BE REPLACED ***";            
    var storageAccount = CloudStorageAccount.Parse(connectionString);
    var tableClient = storageAccount.CreateCloudTableClient();
    table = tableClient.GetTableReference("demo1234");
}
```
9. create a class based on TableEntity
```C#
public class Customer : TableEntity
{
    public int Age { get; set; }
    public string Firstname { get; set; }
    public string Lastname { get; set; }
}
```
10. create a web method / endpoint to retrieve all records from the storage table inside the HomeController.cs
```C#
[HttpGet("/records")]
public async Task<IActionResult> Json()
{
    var partitionKey = "devoteam";
    var query = new TableQuery<Customer>();
    query.Where(TableQuery.GenerateFilterCondition("PartitionKey", QueryComparisons.Equal,  partitionKey));
    var records = await table.ExecuteQuerySegmentedAsync(query, null);
    return Json(records);
}
```
11. Consume the new endpoint, and show the output
```html
# add CSS in <head> section inside _Layout.cshtml
<link href="https://cdn.datatables.net/1.10.19/css/jquery.dataTables.min.css" rel="stylesheet" />

# add JS in <body> section inside _Layout.cshtml
<script src="https://cdn.datatables.net/1.10.19/js/jquery.dataTables.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/moment.js/2.24.0/moment.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/moment.js/2.24.0/locale/nl.js"></script>

# add jQuery datatables in Index.cshtml
<table id="customers" style="min-width: 100% !important; width:100% !important;"></table>
@section Scripts {
    <script>
        $("table#customers").DataTable({
            "ajax": {
                "url": "/records",
                "dataSrc": ""
            },
            "columns":[
                { 
                    data: "rowKey",
                    title: "RowKey"
                }, {
                    data: "firstname",
                    title: "FirstName"
                }, {
                    data: "lastname",
                    title: "LastName"
                }, {
                    data: "age",
                    title: "Age"
                }
                , {
                    data: "timestamp",
                    render: function(data) {
                        return moment(data).format('L LTS');
                    },
                    title: "Created At"
                }
            ],
            "deferRender": true,
            "lengthMenu": [ 10, 25, 50, 75, 100 ],
            "processing": true,
            "stateSave": true
        });
    </script>
}
```
12. Optimize by moving the connectionstring in the Environment Settings locally inside launch.json
```json
{
  "AZURE_CONNECTION_STRING": "** TO BE REPLACED ***"
}

```
13. Add the connectionString in the Application Settings in the Azure Portal
```powershell
# get the azure storage key
$storageKey=(Get-AzStorageAccountKey -ResourceGroupName $ResourceGroup -AccountName $StorageAccount).Value[0];
# assign it to the Web App
Set-AzWebApp -AppSettings @{ AZURE_CONNECTION_STRING = "DefaultEndpointsProtocol=https;AccountName=$StorageAccount;AccountKey=$StorageKey;" } -Name $WebAppName -ResourceGroupName $ResourceGroup
# mark the setting as a slot setting
Set-AzWebAppSlotConfigName -AppSettingNames "AZURE_CONNECTION_STRING" -Name $WebAppName -ResourceGroupName $ResourceGroup
# update our staging slot as well
Set-AzWebAppSlot -AppSettings @{ AZURE_CONNECTION_STRING = "DefaultEndpointsProtocol=https;AccountName=$StorageAccount;AccountKey=$StorageKey;" } -Name $WebAppName -ResourceGroupName $ResourceGroup -Slot "Staging"
```
14. Change the use of the connectionString in the HomeController.cs
```C#
var storageAccount = CloudStorageAccount.Parse(Environment.GetEnvironmentVariable("AZURE_CONNECTION_STRING"));
```
15. publish the new changes to the Azure Web App, by incrementing the version
```sh
npm version patch -f
```
# 3. Securing using Azure KeyVault
1. Create a service principal in the Azure Portal
```powershell

$Application = New-AzADApplication -DisplayName "DevoTeamDemoApp" -IdentifierUris "http://localhost:5000"
$password = ConvertTo-SecureString -String "MyVeryStrongPassword1234*" -AsPlainText -Force
$cred = New-AzADAppCredential -ObjectId $Application.ObjectId -Password $password -EndDate "2029-12-31T23:59:00"
$sp = New-AzADServicePrincipal -ApplicationId $Application.ApplicationId

```
2. Create a KeyVault in the Azure Portal
```powershell
New-AzKeyVault -ResourceGroupName $ResourceGroup -Location "westeurope" -Name $KeyVaultName -Sku "Standard"
```
3. Grant the permissions for the service principal in the Azure Portal
```powershell
# set access for this service principal
Set-AzKeyVaultAccessPolicy -ResourceGroupName $ResourceGroup -VaultName $KeyVaultName -ServicePrincipalName $Application.ApplicationId -PermissionsToSecrets get,list
# set access to the key vault from your account
$userId = (Get-AzADUser -First 1).Id
Set-AzureRmKeyVaultAccessPolicy –VaultName $KeyVaultName –Object $userId -PermissionsToSecrets get,list,set,delete,purge

```
4. Create a secret in the KeyVault with the ConnectionString
```powershell
$secretValue = ConvertTo-SecureString -String "DefaultEndpointsProtocol=https;AccountName=$StorageAccount;AccountKey=$StorageKey;" -AsPlainText -Force
Set-AzKeyVaultSecret -VaultName $KeyVaultName -Name "azcs" -SecretValue $secretValue
```
5. Implement the KeyVault package in the Web App
```sh
# console
dotnet add package Microsoft.Extensions.Configuration.AzureKeyVault --version 2.1.1
# Visual Studio
Install-Package Microsoft.Extensions.Configuration.AzureKeyVault -Version 2.1.1
```
6. Add the KeyVault settings in appsetttings.json
```json
{
  "AzureADApplicationId": "*** TO BE REPLACED ***",
  "AzureADPassword":"MyVeryStrongPassword1234*",
  "KeyVaultName": "*** PUT THE KEYVAULT NAME HERE ***"
}

```
```C#
// add registration inside program.cs
public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
            WebHost.CreateDefaultBuilder(args)
            .ConfigureAppConfiguration((context, config) =>
            {
                var buildConfig = config.Build();
                //Add Key Vault to configuration pipeline, we need the whole url
                config.AddAzureKeyVault(
                    $"https://{buildConfig["KeyVaultName"]}.vault.azure.net/",                     
                    buildConfig["AzureADApplicationId"],
                    buildConfig["AzureADPassword"],
                    new DefaultKeyVaultSecretManager());                
            })
            .UseStartup<Startup>();

// register as a Singleton inside start.cs for Dependency Injection
public void ConfigureServices(IServiceCollection services)
{
    var storageAccount = CloudStorageAccount.Parse(Configuration["azcs"]);
    var tableClient = storageAccount.CreateCloudTableClient();
    var table = tableClient.GetTableReference("demo1234");
    services.AddSingleton<CloudTable>(table);
}

// change constructor of HomeController.cs, by using Dependency Injection
private readonly CloudTable _table;
public HomeController(CloudTable table)
{
   _table = table;
}
```
7. Add Application Settings inside the Azure Portal
```powershell
Set-AzWebApp -AppSettings @{ KeyVaultName = $KeyVaultName; AzureADPassword = "MyVeryStrongPassword1234*"; AzureADApplicationId = $Application.ApplicationId.ToString(); } -Name $WebAppName -ResourceGroupName $ResourceGroup
# mark the setting as a slot setting
Set-AzWebAppSlotConfigName -AppSettingNames "KeyVaultName","AzureADPassword","AzureADApplicationId" -Name $WebAppName -ResourceGroupName $ResourceGroup
# update our staging slot as well
Set-AzWebAppSlot -AppSettings @{ KeyVaultName = $KeyVaultName; AzureADPassword = "MyVeryStrongPassword1234*"; AzureADApplicationId = $Application.ApplicationId.ToString(); } -Name $WebAppName -ResourceGroupName $ResourceGroup -Slot "Staging"

```
```sh
# publish a new version
npm version patch -f
```
8. Remove Previous ADApp and Enable Managed Identity on the Web App in the Azure Portal
9. Optimize the code inside program.cs
```C#
       public static IWebHostBuilder CreateWebHostBuilder(string[] args) => WebHost.CreateDefaultBuilder(args)
            .ConfigureAppConfiguration((context, config) =>
        {
            var buildConfig = config.Build();
            // Create Managed Service Identity token provider
            var tokenProvider = new AzureServiceTokenProvider();
            //Create the Key Vault client
            var kvClient = new KeyVaultClient((authority, resource, scope) => tokenProvider.KeyVaultTokenCallback(authority, resource, scope));
            config.AddAzureKeyVault(
                $"https://{buildConfig["KeyVaultName"]}.vault.azure.net/",                     
                kvClient, 
                 new DefaultKeyVaultSecretManager());            
            })
            .UseStartup<Startup>();
```
10. publish the new version to Azure
```sh
npm version major -f
```

10. Remove the resource group and all used resources at once without confirmation:
```powershell
Get-AzureRmResourceGroup -Name $ResourceGroup | Remove-AzureRmResourceGroup -Force -Verbose
```
