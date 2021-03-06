<properties
   pageTitle="Deploy an existing application in Azure Service Fabric | Microsoft Azure"
   description="Walkthrough on how to package an existing application so it can be deployed on an Azure Service Fabric cluster"
   services="service-fabric"
   documentationCenter=".net"
   authors="bmscholl"
   manager="timlt"
   editor=""/>

<tags
   ms.service="service-fabric"
   ms.devlang="dotnet"
   ms.topic="article"
   ms.tgt_pltfrm="NA"
   ms.workload="NA"
   ms.date="11/09/2015"
   ms.author="bscholl"/>

# Deploy an existing application to Service Fabric

You can run any type of existing application, such as Node.js, Java or native applications in Service Fabric. Service Fabric treats those applications like stateless services and places them on nodes in a cluster based on availability and other metrics. This article describes how to package and deploy an existing application to a Service Fabric cluster.

## Benefits of running an existing application in Service Fabric

There are a couple of advantages that come with running the application in Service Fabric Cluster:

- High availability: Applications that are run in Service Fabric are highly available out of the box. Service Fabric makes sure that always one instance of an application is up and running
- Health monitoring: Out of the box Service Fabric health monitoring detects if the application is up and running and provides diagnostics information the case of a failure   
- Application Life cycle management: Besides no downtime upgrades Service Fabric also allows to roll back to the previous version if there is an issue during upgrade.    
- Density: You can run multiple applications in cluster which eliminates the need for each application to run on its own hardware

In this article we cover the basic steps to package an existing application and deploy it to Service Fabric.  


## Quick overview of Application and Service manifest files

Before getting into the details of deploying an existing application, it is useful to understand the Service Fabric packaging and deployment model. The Service Fabric packaging deployment model relies mainly on two files:


* **Application Manifest**

  The application manifest is used to describe the application and lists the services that compose it plus other parameters, such as the number of instances, that are used to define how the service(s) should be deployed. In the Service Fabric world, an application is the 'upgradable unit'. An application can be upgraded as a single unit where potential failures (and potential rollbacks) are managed by the platform to guarantee that the upgrade process either completely success or, if it fails, it does not leave the application is an unknown/unstable state.


* **Service Manifest**

  The service manifest describes the components of a service. It includes data such as name and type of the service (information that Service Fabric uses to manage the service), its code, configuration and data components plus some additional parameters that can be used to configure the service once it is deployed. We are not going into the details of all the different parameters available in the service manifest, we will go through the subset that is required to make an existing application to run on Service Fabric

For detailed information on the Service Fabric packaging format read [this](service-fabric-develop-your-service-index.md).

## Application package file structure
In order to deploy an application to Service Fabric, the application needs to follow a predefined directory structure. Below is an example of that structure.

```
|-- AppplicationPackage
	|-- code
		|-- existingapp.exe
	|-- config
		|--Settings.xml
    |--data    
    |-- ServiceManifest.xml
|-- ApplicationManifest.xml
```

The root contains the ApplicationManifest.xml file that defines the application. A subdirectory for each service included in the application is used to contain all the artifacts that the service requires: The ServiceManifest.xml and, typically 3 directories:

- *code*: contains the service code
- *config*: contains a settings.xml file (and other files if necessary) that the service can access at runtime to retrieve specific configuration settings.
- *data*: an additional directory to store additional local data that service may need. Note: Data should be used to store only ephymeral data, Service Fabric does not copy/replicate changes to the data directory if the service needs to be relocated, for instance, during failover.

Note: You don't have to create the `config` and `data` directories in case you don't need them.

## The process of packaging an existing app

The process of packaging an existing application is based on the following steps:

- create the package directory structure
- add application's code and configuration files
- update the service manifest file
- update the application manifest

>[AZURE.NOTE]: We do provide a packaging tool allowing you to create the ApplicationPackage automatically. The tool is currently in preview. You can find more information [here](http://aka.ms/servicefabricpacktool).

### Create the package directory structure
You can start by creating the directory structure as described before.

### Add the application's code and configuration files
After you have created the directory structure, you can add the application's code and configuration files under the code and config directory. You can also create additional directories or sub directories under the code or config directories. Service Fabric does an xcopy of the content of the application root directory so there is no predefined structure to use other than creating two top directories code and settings (but you can pick different names if you want, more details in the next section).

>[AZURE.NOTE]: Make sure that you include all the files/dependencies that the application needs. Service Fabric will copy the content of the application package on all nodes in the cluster where the application's services are going to be deployed. The package should contain all the code that the application needs in order to run. It is not recommended to assume that the dependencies are already installed.

### Edit the Service Manifest file
The next step is to edit the Service Manifest file to include the following information:

- The name of the service type. This is an 'Id' that Service Fabric uses in order to identify a service
- The command to use to launch the application (ExeHost)
- Any script that needs to be run in order to setup/configure the application (SetupEntrypoint)

Below is an example of a `ServiceManifest.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<ServiceManifest xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" Name="NodeApp" Version="1.0.0.0" xmlns="http://schemas.microsoft.com/2011/01/fabric">
   <ServiceTypes>
      <StatelessServiceType ServiceTypeName="NodeApp" UseImplicitHost="true"/>
   </ServiceTypes>
   <CodePackage Name="code" Version="1.0.0.0">
      <SetupEntryPoint>
         <ExeHost>
             <Program>scripts\launchConfig.cmd</Program>
         </ExeHost>
      </SetupEntryPoint>
      <EntryPoint>
         <ExeHost>
            <Program>node.exe</Program>
            <Arguments>bin/www</Arguments>
            <WorkingFolder>CodePackage</WorkingFolder>
         </ExeHost>
      </EntryPoint>
   </CodePackage>
   <Resources>
      <Endpoints>
         <Endpoint Name="NodeAppTypeEndpoint" Protocol="http" Port="3000" Type="Input" />
      </Endpoints>
   </Resources>
</ServiceManifest>
```

Let's go over the different part of the file that you need to update:

### ServiceTypes

```xml
<ServiceTypes>
  <StatelessServiceType ServiceTypeName="NodeApp" UseImplicitHost="true" />
</ServiceTypes>
```

- You can pick any name that you want for `ServiceTypeName`, the value is used in the `ApplicationManifest.xml` to identify the service.
- You need to specify `UseImplicitHost="true"`. This attribute tells Service Fabric that the service is based on a self-contained app so that it needs to do is just to launch it as a process and monitor its health.

### CodePackage
The CodePackage specifies the location (and version) of the service's code.

```xml
<CodePackage Name="Code" Version="1.0.0.0">
```

The `Name` element is used to specify the name of the directory in the Application Package that contains the service's code. `CodePackage` also has the `version` attribute that can be used to specify the version of the code and potentially be used to upgrade the service's code using Service Fabric's ALM infrastructure.
### SetupEntrypoint

```xml
<SetupEntryPoint>
   <ExeHost>
       <Program>scripts\launchConfig.cmd</Program>
   </ExeHost>
</SetupEntryPoint>
```
The SetupEntrypoint is used to specify any executable or batch file that should be executed before the service's code is launched. It is an optional element so it does not need to be included if there is no intialization/setup that is required. The SetupEntryPoint is executed every time the service is restarted. There is only one SetupEntrypoint so setup/config scripts needs to be bundled on a single batch file if the application's setup/config requires multiple scripts. Like the Entrypoint element, SetupEntrypoint can execute any type of file: executable, batch fiules, powershell cmdlet. In the example above, the SetupEntrypoint is based on a batch file launchConfig.cmd that is located in the `scripts` subdirectory of the Code directory (assuming the WorkingDirectory element is set to Code).

### Entrypoint

```xml
<EntryPoint>
  <ExeHost>
    <Program>node.exe</Program>
    <Arguments>bin/www</Arguments>
    <WorkingFolder>CodeBase</WorkingFolder>
  </ExeHost>
</EntryPoint>
```

The `Entrypoint` element in the service manifest file is used to specify how to launch the service. The `ExeHost` element specifies the executable (and arguments) that should be used to launch the service.

- `Program`:specifies the name of the executable that should be executed in order to start the service.
- `Arguments`: it specifies the arguments that should be passed to the executable. it can be a list of parameters with arguments.
- `WorkingFolder`: it specifies the working directory for the process that is going to be started. You can specify two values:
	- `CodeBase`: the working directory is going to be set to the Code directory in the application package (`Code` directory in the structure shown below)
	- `CodePackage`: the working directory will be set to the root of the application package	(`MyServicePkg`)
- `WorkingDirectory` element is useful to set the correct working directory so relative paths can be used by either the application or initialization scripts.

### Endpoints

```xml
<Endpoints>
   <Endpoint Name="NodeAppTypeEndpoint" Protocol="http" Port="3000" Type="Input" />
</Endpoints>

```
The `Endpoint` element specifies the endpoints the application can listen on. In this exmpale the Node.js application listens on port 3000.

## Application Manifest file

Once you have configured the `servicemanifest.xml` file you need to make some changes to the `ApplicationManifest.xml` file to ensure the correct Service type and name are used.

```xml
<?xml version="1.0" encoding="utf-8"?>
<ApplicationManifest xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" ApplicationTypeName="NodeAppType" ApplicationTypeVersion="1.0" xmlns="http://schemas.microsoft.com/2011/01/fabric">
   <ServiceManifestImport>
      <ServiceManifestRef ServiceManifestName="NodeApp" ServiceManifestVersion="1.0.0.0" />
   </ServiceManifestImport>
</ApplicationManifest>
```

### ServiceManifestImport

In the `ServiceManifestImport` you can specify one or more services that you want to include in the app. Services are referenced with `ServiceManifestName` that specifies the name of the directory where the `ServiceManifest.xml` file is located.

```xml
<ServiceManifestImport>
  <ServiceManifestRef ServiceManifestName="NodeApp" ServiceManifestVersion="1.0.0.0" />
</ServiceManifestImport>
```

### Setting up logging
For existing application it is very useful to be able to see console logs to find out if the application and configuration scripts don't show any error.
Console redirection can be configured in the `ServiceManifest.xml` file using the `ConsoleRedirection` element

```xml
<EntryPoint>
  <ExeHost>
    <Program>node.exe</Program>
    <Arguments>bin/www</Arguments>
    <WorkingFolder>CodeBase</WorkingFolder>
    <ConsoleRedirection FileRetentionCount="5" FileMaxSizeInKb="2048"/>
  </ExeHost>
</EntryPoint>
```

* `ConsoleRedirection` can be used to redirect console output (both stdout and stderr) to a working directory so they can be used to verify that there are no errors during the setup or execution of the application in the Service Fabric cluster.

	* `FileRetentionCount` determines how many files are saved in the working directory. A value of, for instance, 5 means that the log files for the previous 5 executions are stored in the working directory.
	* `FileMaxSizeInKb` specifies the max size of the log files.

Log files are saved on one of the service's working directories, in order to determine where the files are located, you need to use the Service Fabric Explorer to determine in which node the service is running and which is the working directory that is currently used. How to do this is covered later in this article.

### Deployment
The last step is to deploy your application. The PowerShell script below shows how to deploy you application to the local development cluster and start a new Service Fabric service.

```Powershell

Connect-ServiceFabricCluster localhost:19000

Write-Host 'Copying application package...'
Copy-ServiceFabricApplicationPackage -ApplicationPackagePath 'C:\Dev\MulitpleApplications' -ImageStoreConnectionString 'file:C:\SfDevCluster\Data\ImageStoreShare' -ApplicationPackagePathInImageStore 'Store\nodeapp'

Write-Host 'Registering application type...'
Register-ServiceFabricApplicationType -ApplicationPathInImageStore 'Store\nodeapp'

New-ServiceFabricApplication -ApplicationName 'fabric:/nodeapp' -ApplicationTypeName 'NodeAppType' -ApplicationTypeVersion 1.0

New-ServiceFabricService -ApplicationName 'fabric:/nodeapp' -ServiceName 'fabric:/nodeapp/nodeappservice' -ServiceTypeName 'NodeApp' -Stateless -PartitionSchemeSingleton -InstanceCount 1

```
A Service Fabric service can be deployed in various 'configurations', for instance it can be deployed as a single or multiple instances or it can be deployed in such a way that there is one instance of the service on each node of the Service Fabric cluster.

The `InstanceCount` parameter of `New-ServiceFabricService` cmdlet is used to specify how many instances of the service should be launched in the Service Fabric cluster. You can set the `InstanceCount` value depending on the type of application that you are deploying. The two most common scenarios are:
* `InstanCount = "1"`: in this case only one instance of the service will be deployed on the cluster. Service Fabric's scheduler determines on which node the service is going to be deployed.

* `InstanceCount ="-1"`: in this case one instance of the service will be deployed on every node in the Service Fabric cluster. The end result will be having one (and only one) instance of the service for each node in the cluster. This is a useful configuration for front-end applications (ex. a REST endpoint) because client applications just need to 'connect' to any of the node in the cluster in order to use the endpoint. This configuration can also be used when, for instance, all nodes of the Service Fabric cluster are connected to a load balancer so client traffic can be distributed across the service running on all nodes in the cluster.

### Check your running application

In Service Fabric Explorer, identify the node where the service is running. In this example it runs on Node1:

![running app](./media/service-fabric-deploy-existing-app/runningapplication.png)

If you navigate to the node and browse to the application, you will see the essential node information including its location on disk.

![location on disk](./media/service-fabric-deploy-existing-app/locationondisk.png)

If you browse to the directory using Server Explorer you can find working directory and the service's logs folder as shown below.

![location on disk](./media/service-fabric-deploy-existing-app/loglocation.png)


## Next steps
In this article you have learned how to package an existing application and deploy it to Service Fabric. As a next step you can check out additional content for this topic.

- Sample for packaging and deploying an existing application on [Github](https://github.com/bmscholl/servicefabric-samples/tree/comingsoon/samples/RealWorld/Hosting/SimpleApplication), including the pre-release of the packaging tool
- Sample for packaging multiple applications on [Github](https://github.com/bmscholl/servicefabric-samples/tree/comingsoon/samples/RealWorld/Hosting/SimpleApplication)
- How to get started with [creating your first Service Fabric application using Visual Studio](service-fabric-create-your-first-application-in-visual-studio.md)
