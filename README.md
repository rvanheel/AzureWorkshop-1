# 1. Develop and deploy a Web App to Azure
1. Create a resource group in the Azure Portal
```powershell
New-AzResourceGroup -Location "westeurope" -Name "devoteam-demo"
```
2. Create a empty Web App in the Azure Portal
```powershell
# create a serviceplan first
New-AzAppServicePlan -Location westeurope -ResourceGroupName "devoteam-demo" -Name "DevoTeam" -Tier "Standard"
# create the webapp
New-AzWebApp -Location westeurope -ResourceGroupName "devoteam-demo" -AppServicePlan "DevoTeam" -Name "DevoteamDemoWebApp"
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
```
7. Commit the application and publish to Azure
```sh
git add --all
git commit -m "version 1"
git push azure master --force
```
8. Create a deployment slot in the Azure Portal for the Web App and enable GIT deployment
```powershell
New-AzWebAppSlot -ResourceGroupName "devoteam-demo" -AppServicePlan "DevoTeam" -Name "DevoteamDemoWebApp" -Slot "Staging"
```
9. Update the local GIT repository with the deployment slot GIT repository
```sh
git remote set-url [deployment_slot_git_url]
```
10. Deploy to the new deployment slot in Azure
```sh
git push azure master --force
```
11. Enable AUTO swap in the Azure Portal for the Web App
```powershell
Set-AzWebAppSlot -ResourceGroupName "devoteam-demo" -AppServicePlan "DevoTeam" -Name "DevoteamDemoWebApp" -Slot "Staging" -AutoSwapSlotName "production"
```
# 2. Implement Azure Storage using Storage Tables
1. Create a storage account in the Azure Portal
```powershell
New-AzStorageAccount -ResourceGroupName "devoteam-demo" -Location "westeurope" -Name "devoteamdemowebapp" -SkuName "Standard_LRS" -Kind "StorageV2"
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
"AZURE_CONNECTION_STRING": "** TO BE REPLACED ***"
```
13. Add the connectionString in the Application Settings in the Azure Portal
```powershell
# get the azure storage key
$storageKey=(Get-AzStorageAccountKey -ResourceGroupName "devoteam-demo" -AccountName "devoteamdemowebapp").Value[0];
# assign it to the Web App
Set-AzWebApp -AppSettings @{ AZURE_CONNECTION_STRING = "DefaultEndpointsProtocol=https;AccountName=devoteamdemowebapp;AccountKey=$StorageKey;" } -Name "devoteamdemowebapp" -ResourceGroupName "devoteam-demo"
# mark the setting as a slot setting
Set-AzWebAppSlotConfigName -AppSettingNames "AZURE_CONNECTION_STRING" -Name "devoteamdemowebapp" -ResourceGroupName "devoteam-demo"
14. Change the use of the connectionString in the HomeController.cs
# update our staging slot as well
Set-AzWebAppSlot -AppSettings @{ AZURE_CONNECTION_STRING = "DefaultEndpointsProtocol=https;AccountName=devoteamdemowebapp;AccountKey=$StorageKey;" } -Name "devoteamdemowebapp" -ResourceGroupName "devoteam-demo" -Slot "Staging"
```C#
var storageAccount = CloudStorageAccount.Parse(Environment.GetEnvironmentVariable("AZURE_CONNECTION_STRING"));
```
15. publish the new changes to the Azure Web App
# 3. Securing using Azure KeyVault
1. Create a service principal in the Azure Portal
```powershell
New-AzADApplication -DisplayName "DevoTeamDemoApp" -IdentifierUris "http://localhost:5000"
```
2. Create a KeyVault in the Azure Portal
```powershell
New-AzKeyVault -ResourceGroupName "devoteam-demo" -Location "westeurope" -Name "devoteam-demo" -Sku "Standard"
```
3. Grant the permissions for the service principal in the Azure Portal
```powershell
# set access for this service principal
$ObjectId = (Get-AzADApplication -DisplayName "DevoTeamDemoApp").ObjectId
Set-AzKeyVaultAccessPolicy -ResourceGroupName "devoteam-demo" -VaultName "devoteam-demo" -ObjectId  "$ApplicationId" -PermissionsToSecrets get,list
```
4. Create a secret in the KeyVault with the ConnectionString
```powershell
$secretValue = ConvertTo-SecureString -String "DefaultEndpointsProtocol=https;AccountName=devoteamdemowebapp;AccountKey=$StorageKey;" -AsPlainText -Force
Set-AzKeyVaultSecret -VaultName "devoteam-demo" -Name "azcs" -SecretValue $secretValue
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
"AzureADApplicationId": "*** TO BE REPLACED ***",
"AzureADPassword":"*** TO BE REPLACED ***",
"KeyVaultName": "*** TO BE REPLACED ***"
```
```C#
// add registration inside program.cs
public static IWebHostBuilder CreateWebHostBuilder(string[] args) => WebHost.CreateDefaultBuilder(args)
    .ConfigureAppConfiguration((context, config) =>
{
    var buildConfig = config.Build();
    //Add Key Vault to configuration pipeline, we need the whole url
    config.AddAzureKeyVault(
        $"https://{buildConfig["KeyVaultName"]}.vault.azure.net/",                     
        buildConfig["AzureADApplicationId"],
        buildConfig["AzureADPassword"],
        new DefaultKeyVaultSecretManager());

// register as a Singleton inside start.cs for Dependency Injection
public void ConfigureServices(IServiceCollection services)
{
    var storageAccount = CloudStorageAccount.Parse(Configuration["*** KeyVault Secret Name ***"]);
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
8. Enable Managed Identity on the Web App in the Azure Portal
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
```
