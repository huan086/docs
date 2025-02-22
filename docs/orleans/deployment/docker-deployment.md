---
title: Docker deployment
description: Learn how to deploy Orleans apps with Docker.
ms.date: 03/09/2022
---

# Docker deployment

> [!TIP]
> Even if you are very familiar with Docker and/or Orleans, as any other Orleans documentation, it's recommended you to read it to the end to avoid problems you may face that we already worked around.

This article and its sample are a work in progress. Any feedback, PR, or suggestion is very welcome.

## Deploy Orleans solutions to Docker

Deploying Orleans to Docker can be tricky given the way Docker orchestrators and clustering stacks were designed. The most complicated thing is to understand the concept of _Overlay Network_ from Docker Swarm and Kubernetes networking model.

Docker containers and networking models were designed to run mostly stateless and immutable containers. So, spinning up a cluster running node.js or Nginx applications is pretty easy. However, if you try to use something more elaborate, like a real clustered or distributed application (like Orleans-based ones) you will eventually have trouble setting it up. It is possible, but not as simple as web-based applications.

Docker clustering consists of putting together multiple hosts to work as a single pool of resources, managed using a _Container Orchestrator_. _Docker Inc._ provide **Swarm** as their option for Container Orchestration while _Google_ has **Kubernetes** (aka **K8s**). There are other Orchestrators like **DC/OS**, **Mesos**, but in this document, we will talk about Swarm and K8s as they are more widely used.

The same grain interfaces and implementation which run anywhere Orleans is already supported will run on Docker containers as well, so no special considerations are needed to be able to run your application in Docker containers.

The [Orleans-Docker](https://github.com/dotnet/orleans/tree/main/Samples/Orleans-Docker) sample provides a working example of how to run two Console applications. One as Orleans Client and another as Silo, and the details are described below.

The concepts discussed here can be used on both .NET Core and .NET 4.6.1 flavors of Orleans but to illustrate the cross-platform nature of Docker and .NET Core, we are going to focus on the example considering you are using .NET Core. Platform-specific (Windows/Linux/OSX) details may be provided in this article.

## Prerequisites

This article assumes that you have the following prerequisites installed:

* [Docker](https://www.docker.com/community-edition#/download) - Docker4X has an easy-to-use installer for the major supported platforms. It contains Docker engine and also Docker Swarm.
* [Kubernetes (K8s)](https://kubernetes.io/docs/tutorials/stateless-application/hello-minikube/) - Google's offer for Container Orchestration. It contains guidance to install _Minikube_ (a local deployment of K8s) and _kubectl_ along with all its dependencies.
* [.NET Core](https://dot.net) - Cross-platform flavor of .NET
* [Visual Studio Code (VSCode)](https://code.visualstudio.com/) - You can use whatever IDE you want. VSCode is cross-platform so we are using it to ensure it works on all platforms. Once you installed VSCode, install the [C# extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode.csharp).

> [!IMPORTANT]
> You are not required to have Kubernetes installed if you are not going to use it. Docker4X installer already includes Swarm so no extra installation is required to use it.

> [!NOTE]
> On Windows, Docker installer will enable Hyper-V at the installation process. As this article and its examples are using .NET Core, the container images used are based on **Windows Server NanoServer**. If you don't plan to use .NET Core and will target .NET 4.6.1 full framework, the image used should be **Windows Server Core** and the 1.4+ version of Orleans (which supports only .net full framework).

## Create Orleans solution

The following instructions show how to create a regular Orleans solution using the new `dotnet` tooling.

Please adapt the commands to whatever is appropriate in your platform. Also, the directory structure is just a suggestion. Please adapt it to your needs.

```bash
mkdir Orleans-Docker
cd Orleans-Docker
dotnet new sln
mkdir -p src/OrleansSilo
mkdir -p src/OrleansClient
mkdir -p src/OrleansGrains
mkdir -p src/OrleansGrainInterfaces
dotnet new console -o src/OrleansSilo --framework netcoreapp1.1
dotnet new console -o src/OrleansClient --framework netcoreapp1.1
dotnet new classlib -o src/OrleansGrains --framework netstandard1.5
dotnet new classlib -o src/OrleansGrainInterfaces --framework netstandard1.5
dotnet sln add src/OrleansSilo/OrleansSilo.csproj
dotnet sln add src/OrleansClient/OrleansClient.csproj
dotnet sln add src/OrleansGrains/OrleansGrains.csproj
dotnet sln add src/OrleansGrainInterfaces/OrleansGrainInterfaces.csproj
dotnet add src/OrleansClient/OrleansClient.csproj reference src/OrleansGrainInterfaces/OrleansGrainInterfaces.csproj
dotnet add src/OrleansSilo/OrleansSilo.csproj reference src/OrleansGrainInterfaces/OrleansGrainInterfaces.csproj
dotnet add src/OrleansGrains/OrleansGrains.csproj reference src/OrleansGrainInterfaces/OrleansGrainInterfaces.csproj
dotnet add src/OrleansSilo/OrleansSilo.csproj reference src/OrleansGrains/OrleansGrains.csproj
```

What we did so far was just boilerplate code to create the solution structure, projects, and add references between projects. Nothing different than a regular Orleans project.

By the time this article was written, Orleans 2.0 (which is the only version that supports .NET Core and cross-platform) is in Technology Preview so its NuGet packages are hosted in a MyGet feed and not published to Nuget.org official feed. To install the preview NuGet packages, we will use `dotnet` CLI forcing the source feed and version from MyGet:

```bash
dotnet add src/OrleansClient/OrleansClient.csproj package Microsoft.Orleans.Core -s https://dotnet.myget.org/F/orleans-prerelease/api/v3/index.json -v 2.0.0-preview2-201705020000
dotnet add src/OrleansGrainInterfaces/OrleansGrainInterfaces.csproj package Microsoft.Orleans.Core -s https://dotnet.myget.org/F/orleans-prerelease/api/v3/index.json -v 2.0.0-preview2-201705020000
dotnet add src/OrleansGrains/OrleansGrains.csproj package Microsoft.Orleans.Core -s https://dotnet.myget.org/F/orleans-prerelease/api/v3/index.json -v 2.0.0-preview2-201705020000
dotnet add src/OrleansSilo/OrleansSilo.csproj package Microsoft.Orleans.Core -s https://dotnet.myget.org/F/orleans-prerelease/api/v3/index.json -v 2.0.0-preview2-201705020000
dotnet add src/OrleansSilo/OrleansSilo.csproj package Microsoft.Orleans.OrleansRuntime -s https://dotnet.myget.org/F/orleans-prerelease/api/v3/index.json -v 2.0.0-preview2-201705020000
dotnet restore
```

Ok, now you have all the basic dependencies to run a simple Orleans application. Note that so far, nothing changed from your regular Orleans application. Now, let's add some code so we can do something with it.

## Implement your Orleans application

Assuming that you are using **VSCode**, from the solution directory, run `code .`. That will open the directory in **VSCode** and load the solution.

This is the solution structure we just created previously.

![Visual Studio Code: Explorer with Program.cs selected.](docker-orleans-solution.png)

We also added _Program.cs_, _OrleansHostWrapper.cs_, _IGreetingGrain.cs_ and _GreetingGrain.cs_ files to the interfaces and grain projects respectively, and here is the code for those files:

_IGreetingGrain.cs_:

```csharp
using System;
using System.Threading.Tasks;
using Orleans;

namespace OrleansGrainInterfaces
{
    public interface IGreetingGrain : IGrainWithGuidKey
    {
        Task<string> SayHello(string name);
    }
}
```

_GreetingGrain.cs_:

```csharp
using System;
using System.Threading.Tasks;
using OrleansGrainInterfaces;

namespace OrleansGrains
{
    public class GreetingGrain : Grain, IGreetingGrain
    {
        public Task<string> SayHello(string name)
        {
            return Task.FromResult($"Hello from Orleans, {name}");
        }
    }
}
```

_OrleansHostWrapper.cs_:

```csharp
using System;
using System.NET;
using Orleans.Runtime;
using Orleans.Runtime.Configuration;
using Orleans.Runtime.Host;

namespace OrleansSilo;

public class OrleansHostWrapper
{
    private readonly SiloHost _siloHost;

    public OrleansHostWrapper(ClusterConfiguration config)
    {
        _siloHost = new SiloHost(Dns.GetHostName(), config);
        _siloHost.LoadOrleansConfig();
    }

    public int Run()
    {
        if (_siloHost is null)
        {
            return 1;
        }

        try
        {
            _siloHost.InitializeOrleansSilo();

            if (_siloHost.StartOrleansSilo())
            {
                Console.WriteLine(
                    $"Successfully started Orleans silo '{_siloHost.Name}' as a {_siloHost.Type} node.");
                return 0;
            }
            else
            {
                throw new OrleansException(
                    $"Failed to start Orleans silo '{_siloHost.Name}' as a {_siloHost.Type} node.");
            }
        }
        catch (Exception exc)
        {
            _siloHost.ReportStartupError(exc);
            Console.Error.WriteLine(exc);

            return 1;
        }
    }

    public int Stop()
    {
        if (_siloHost is not null)
        {
            try
            {
                _siloHost.StopOrleansSilo();
                _siloHost.Dispose();
                Console.WriteLine($"Orleans silo '{_siloHost.Name}' shutdown.");
            }
            catch (Exception exc)
            {
                siloHost.ReportStartupError(exc);
                Console.Error.WriteLine(exc);

                return 1;
            }
        }
        return 0;
    }
}
```

_Program.cs_ (Silo):

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.NET;
using System.Threading.Tasks;
using Orleans.Runtime.Configuration;

namespace OrleansSilo
{
    public class Program
    {
        private static OrleansHostWrapper s_hostWrapper;

        static async Task<int> Main(string[] args)
        {
            int exitCode = await InitializeOrleansAsync();

            Console.WriteLine("Press Enter to terminate...");
            Console.ReadLine();

            exitCode += ShutdownSilo();

            return exitCode;
        }

        private static int InitializeOrleansAsync()
        {
            var config = new ClusterConfiguration();
            config.Globals.DataConnectionString =
                "[AZURE STORAGE CONNECTION STRING HERE]";
            config.Globals.DeploymentId = "Orleans-Docker";
            config.Globals.LivenessType =
                GlobalConfiguration.LivenessProviderType.AzureTable;
            config.Globals.ReminderServiceType =
                GlobalConfiguration.ReminderServiceProviderType.AzureTable;
            config.Defaults.PropagateActivityId = true;
            config.Defaults.ProxyGatewayEndpoint =
                new IPEndPoint(IPAddress.Any, 10400);
            config.Defaults.Port = 10300;
            var ips = await Dns.GetHostAddressesAsync(Dns.GetHostName());
            config.Defaults.HostNameOrIPAddress =
                ips.FirstOrDefault()?.ToString();

            s_hostWrapper = new OrleansHostWrapper(config);
            return hostWrapper.Run();
        }

        static int ShutdownSilo() =>
            s_hostWrapper?.Stop() ?? 0;
    }
}
```

_Program.cs_ (client):

```csharp
using System;
using System.NET;
using System.Threading;
using System.Threading.Tasks;
using Orleans;
using Orleans.Runtime.Configuration;
using OrleansGrainInterfaces;

namespace OrleansClient
{
    class Program
    {
        private static IClusterClient s_client;
        private static bool s_running;

        static async Task Main(string[] args)
        {
            await InitializeOrleansAsync();

            Console.ReadLine();

            s_running = false;
        }

        static async Task InitializeOrleansAsync()
        {
            var config = new ClientConfiguration
            {
                DeploymentId = "Orleans-Docker";
                PropagateActivityId = true;
            };
            var hostEntry =
                await Dns.GetHostEntryAsync("orleans-silo");
            var ip = hostEntry.AddressList[0];
            config.Gateways.Add(new IPEndPoint(ip, 10400));

            Console.WriteLine("Initializing...");

            using client = new ClientBuilder().UseConfiguration(config).Build();
            await client.Connect();
            s_running = true;
            Console.WriteLine("Initialized!");

            var grain = client.GetGrain<IGreetingGrain>(Guid.Empty);

            while (s_running)
            {
                var response = await grain.SayHello("Gutemberg");
                Console.WriteLine($"[{DateTime.UtcNow}] - {response}");

                await Task.Delay(1000);
            }
        }
    }
}
```

We are not going into details about the grain implementation here since it is out of the scope of this article. Please check other documents related to it. Those files are essentially a minimal Orleans application and we will start from it to move forward with the remaining of this article.

In this article, we are using `OrleansAzureUtils` membership provider but you can use any other already supported by Orleans.

## The Dockerfile

To create your container, Docker uses images. For more details on how to create your own, you can check [Docker documentation](https://docs.docker.com/engine/userguide/). In this article, we are going to use official [Microsoft images](https://hub.docker.com/r/microsoft/dotnet/). Based on the target and development platforms, you need to pick the appropriate image. In this article, we are using `microsoft/dotnet:1.1.2-sdk` which is a Linux-based image. You can use `microsoft/dotnet:1.1.2-sdk-nanoserver` for Windows for example. Pick one that suits your needs.

> **Note for Windows users**: As previously mentioned, to be cross-platform, we are using .NET Core and Orleans Technical preview 2.0 in this article. If you want to use Docker on Windows with the fully released Orleans 1.4+, you need to use the images that are based on Windows Server Core since NanoServer and Linux-based images, only support .NET Core.

_Dockerfile.debug_:

```dockerfile
FROM microsoft/dotnet:1.1.2-sdk
ENV NUGET_XMLDOC_MODE skip
WORKDIR /vsdbg
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        unzip \
    && rm -rf /var/lib/apt/lists/* \
    && curl -sSL https://aka.ms/getvsdbgsh | bash /dev/stdin -v latest -l /vsdbg
WORKDIR /app
ENTRYPOINT ["tail", "-f", "/dev/null"]
```

This _Dockerfile_ essentially downloads and installs the VSdbg debugger and starts an empty container, keeping it alive forever so we don't need to tear down/up while debugging.

Now, for production, the image is smaller since it contains only the .NET Core runtime and not the whole SDK, and the dockerfile is a bit simpler:

_Dockerfile_:

```dockerfile
FROM microsoft/dotnet:1.1.2-runtime
WORKDIR /app
ENTRYPOINT ["dotnet", "OrleansSilo.dll"]
COPY . /app
```

## The docker-compose file

The `docker-compose.yml` file, essentially defines (within a project) a set of services and its dependencies at the service level. Each service contains one or more instances of a given container, which is based on the images you selected on your Dockerfile. More details on the `docker-compose` can be found on [docker-compose documentation](https://docs.docker.com/compose/).

For an Orleans deployment, a common use case is to have a `docker-compose.yml` which contains two services. One for Orleans Silo, and the other for Orleans Client. The Client would have a dependency on the Silo and that means, it will only start after the Silo service is up. Another case is to add a storage/database service/container, like for example SQL Server, which should start first before the client and the silo, so both services should take a dependency on it.

> [!NOTE]
> Before you read further (and eventually get crazy with it), please note that _indentation_ **matters** in `docker-compose` files. So pay attention to it if you have any problems.

Here is how we will describe our services for this article:

_docker-compose.override.yml_ (Debug):

```yml
version: '3.1'

services:
  orleans-client:
    image: orleans-client:debug
    build:
      context: ./src/OrleansClient/bin/PublishOutput/
      dockerfile: Dockerfile.Debug
    volumes:
      - ./src/OrleansClient/bin/PublishOutput/:/app
      - ~/.nuget/packages:/root/.nuget/packages:ro
    depends_on:
      - orleans-silo
  orleans-silo:
    image: orleans-silo:debug
    build:
      context: ./src/OrleansSilo/bin/PublishOutput/
      dockerfile: Dockerfile.Debug
    volumes:
      - ./src/OrleansSilo/bin/PublishOutput/:/app
      - ~/.nuget/packages:/root/.nuget/packages:ro
```

_docker-compose.yml_ (production):

```yml
version: '3.1'

services:
  orleans-client:
    image: orleans-client
    depends_on:
      - orleans-silo
  orleans-silo:
    image: orleans-silo
```

In production, we don't map the local directory, and neither we have the `build:` action. The reason is that in production, the images should be built and pushed to your own Docker Registry.

## Put everything together

Now we have all the moving parts required to run your Orleans Application, we are going to put it together so we can run our Orleans solution inside Docker (Finally!).

> [!IMPORTANT]
> The following commands should be performed from the solution directory.

First, let's make sure we restore all NuGet packages from our solution. You only need to do it once. You are only required to do it again if you change any package dependency on your project.

```dotnetcli
dotnet restore
```

Now, let's build our solution using `dotnet` CLI as usual and publish it to an output directory:

```dotnetcli
dotnet publish -o ./bin/PublishOutput
```

> [!TIP]
> We are using `publish` here instead of build, to avoid problems with our dynamically loaded assemblies in Orleans. We are still looking for a better solution for it.

With the application built and published, you need to build your Dockerfile images. This step is only required to be performed once per project and should be only performed again if you change the Dockerfile, docker-compose, or for any reason you cleaned up your local image registry.

```shell
docker-compose build
```

All the images used in both `Dockerfile` and `docker-compose.yml` are pulled from the registry and cached on your development machine. Your images are built, and you are all set to run.

Now, let's run it!

```shell
# docker-compose up -d
Creating network "orleansdocker_default" with the default driver
Creating orleansdocker_orleans-silo_1 ...
Creating orleansdocker_orleans-silo_1 ... done
Creating orleansdocker_orleans-client_1 ...
Creating orleansdocker_orleans-client_1 ... done
#
```

Now if you run a `docker-compose ps`, you will see 2 containers running for the `orleansdocker` project:

```shell
# docker-compose ps
             Name                     Command        State   Ports
------------------------------------------------------------------
orleansdocker_orleans-client_1   tail -f /dev/null   Up
orleansdocker_orleans-silo_1     tail -f /dev/null   Up
```

> [!NOTE]
> If you are on Windows, and your container is using a Windows image as a base, the **Command** column will show you the PowerShell relative command to a `tail` on *NIX systems so the container will keep up the same way.

Now that you have your containers up, you don't need to stop it every time you want to start your Orleans application. All you need is to integrate your IDE to debug the application inside the container which was previously mapped in your `docker-compose.yml`.

## Scaling

Once you have your compose project running, you can easily scale up or down your application using `docker-compose scale` command:

```shell
# docker-compose scale orleans-silo=15
Starting orleansdocker_orleans-silo_1 ... done
Creating orleansdocker_orleans-silo_2 ...
Creating orleansdocker_orleans-silo_3 ...
Creating orleansdocker_orleans-silo_4 ...
Creating orleansdocker_orleans-silo_5 ...
Creating orleansdocker_orleans-silo_6 ...
Creating orleansdocker_orleans-silo_7 ...
Creating orleansdocker_orleans-silo_8 ...
Creating orleansdocker_orleans-silo_9 ...
Creating orleansdocker_orleans-silo_10 ...
Creating orleansdocker_orleans-silo_11 ...
Creating orleansdocker_orleans-silo_12 ...
Creating orleansdocker_orleans-silo_13 ...
Creating orleansdocker_orleans-silo_14 ...
Creating orleansdocker_orleans-silo_15 ...
Creating orleansdocker_orleans-silo_6
Creating orleansdocker_orleans-silo_5
Creating orleansdocker_orleans-silo_3
Creating orleansdocker_orleans-silo_2
Creating orleansdocker_orleans-silo_4
Creating orleansdocker_orleans-silo_9
Creating orleansdocker_orleans-silo_7
Creating orleansdocker_orleans-silo_8
Creating orleansdocker_orleans-silo_10
Creating orleansdocker_orleans-silo_11
Creating orleansdocker_orleans-silo_15
Creating orleansdocker_orleans-silo_12
Creating orleansdocker_orleans-silo_14
Creating orleansdocker_orleans-silo_13
```

After a few seconds, you will see the services scaled to the specific number of instances you requested.

```shell
# docker-compose ps
             Name                     Command        State   Ports
------------------------------------------------------------------
orleansdocker_orleans-client_1   tail -f /dev/null   Up
orleansdocker_orleans-silo_1     tail -f /dev/null   Up
orleansdocker_orleans-silo_10    tail -f /dev/null   Up
orleansdocker_orleans-silo_11    tail -f /dev/null   Up
orleansdocker_orleans-silo_12    tail -f /dev/null   Up
orleansdocker_orleans-silo_13    tail -f /dev/null   Up
orleansdocker_orleans-silo_14    tail -f /dev/null   Up
orleansdocker_orleans-silo_15    tail -f /dev/null   Up
orleansdocker_orleans-silo_2     tail -f /dev/null   Up
orleansdocker_orleans-silo_3     tail -f /dev/null   Up
orleansdocker_orleans-silo_4     tail -f /dev/null   Up
orleansdocker_orleans-silo_5     tail -f /dev/null   Up
orleansdocker_orleans-silo_6     tail -f /dev/null   Up
orleansdocker_orleans-silo_7     tail -f /dev/null   Up
orleansdocker_orleans-silo_8     tail -f /dev/null   Up
orleansdocker_orleans-silo_9     tail -f /dev/null   Up
```

> [!IMPORTANT]
> The `Command` column on those examples are showing the `tail` command just because we are using the debugger container. If we were in production, it would be showing `dotnet OrleansSilo.dll` for example.

## Docker swarm

Docker clustering stack is called **Swarm**, for more information see, [Docker Swarm](https://docs.docker.com/engine/swarm).

To run this article in a `Swarm` cluster, you don't have any extra work. When you run `docker-compose up -d` in a `Swarm` node, it will schedule containers based on the configured rules. The same applies to other Swarm-based services like [Docker Datacenter](https://hub.docker.com/bundles/docker-datacenter), [Azure ACS](/azure/aks) (in Swarm mode), and [AWS ECS Container Service](https://aws.amazon.com/ecs/). All you need to do is to deploy your `Swarm` cluster before deploying your **dockerized** Orleans application.

> [!NOTE]
> If you are using a Docker engine with the Swarm mode that already has support for `stack`, `deploy`, and `compose` v3, a better approach to deploy your solution would be `docker stack deploy -c docker-compose.yml <name>`. Just keep in mind that it requires v3 compose file to support your Docker engine and the majority of hosted services like Azure and AWS still use v2 and older engines.

## Google Kubernetes (K8s)

If you plan to use [Kubernetes](https://kubernetes.io/) to host Orleans, there is a community-maintained clustering provider available at [OrleansContrib\Orleans.Clustering.Kubernetes](https://github.com/OrleansContrib/Orleans.Clustering.Kubernetes) and there you can find documentation and samples on how to host Orleans in Kubernetes seamlessly using the provider.

## Debug Orleans inside containers

Well, now that you know how to run Orleans in a container from scratch, would be good to leverage one of the most important principles in Docker. Containers are immutable. And they should have (almost) the same image, dependencies, and runtime in development as in production. This ensures the good old statement _"It works on my machine!"_ never happens again. To make that possible, you need to have a way to develop _inside_ the container and that includes having a debugger attached to your application inside the container.

There are multiple ways to achieve that using multiple tools. After evaluating several, by the time I wrote this article, I ended up choosing one that looks more simple and is less intrusive in the application.

As mentioned earlier in this article, we are using `VSCode` to develop the sample, so here is how to get the debugger attached to your Orleans Application inside the container.

First, change two files inside your `.vscode` directory in your solution:

_tasks.json_:

```json
{
    "version": "0.1.0",
    "command": "dotnet",
    "isShellCommand": true,
    "args": [],
    "tasks": [
        {
            "taskName": "publish",
            "args": [
                "${workspaceRoot}/Orleans-Docker.sln", "-c", "Debug", "-o", "./bin/PublishOutput"
            ],
            "isBuildCommand": true,
            "problemMatcher": "$msCompile"
        }
    ]
}
```

This file essentially tells `VSCode` that whenever you build the project, it will execute the `publish` command as we manually did earlier.

_launch.json_:

```json
{
   "version": "0.2.0",
   "configurations": [
        {
            "name": "Silo",
            "type": "coreclr",
            "request": "launch",
            "cwd": "/app",
            "program": "/app/OrleansSilo.dll",
            "sourceFileMap": {
                "/app": "${workspaceRoot}/src/OrleansSilo"
            },
            "pipeTransport": {
                "debuggerPath": "/vsdbg/vsdbg",
                "pipeProgram": "/bin/bash",
                "pipeCwd": "${workspaceRoot}",
                "pipeArgs": [
                    "-c",
                    "docker exec -i orleansdocker_orleans-silo_1 /vsdbg/vsdbg --interpreter=vscode"
                ]
            }
        },
        {
            "name": "Client",
            "type": "coreclr",
            "request": "launch",
            "cwd": "/app",
            "program": "/app/OrleansClient.dll",
            "sourceFileMap": {
                "/app": "${workspaceRoot}/src/OrleansClient"
            },
            "pipeTransport": {
                "debuggerPath": "/vsdbg/vsdbg",
                "pipeProgram": "/bin/bash",
                "pipeCwd": "${workspaceRoot}",
                "pipeArgs": [
                    "-c",
                    "docker exec -i orleansdocker_orleans-client_1 /vsdbg/vsdbg --interpreter=vscode"
                ]
            }
        }
    ]
}
```

Now you can just build the solution from `VSCode` (which will publish) and start both the Silo and the Client. It will send a `docker exec` command to the running `docker-compose` service instance/container to start the debugger to the application and that's it. You have the debugger attached to the container and use it as if it was a locally running Orleans application. The difference now is that it is inside the container, and once you are done, you can just publish the container to your registry and pull it on your Docker hosts in production.
