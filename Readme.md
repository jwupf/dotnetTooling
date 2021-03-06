# Docker based toolchain for .net 6

This repository sums up stuff I learned about creating toolchains for .net 6.0 unsing Docker and needed infrastructure. The code inside this repository are examples to highlight some things, but are not meant to be used for anything usefull. I might add more complex examples later , if they are needed to bring a point across. 

If not specified otherwise, commands are executed in the root directory of the repository.

Right now I will focus on setting infrastructure up, so the code will be bare bones.

**Once everything is working I'll have to create a script that wrapps the horrible docker calls. Do not get scarred by them, they are just long.**

## Why to use Docker(or general container) for hosting a toolchain

Why Docker in general:
* product bundled inside a image with all required packages/configuration

Why toolchain:
It allows to describe:

* the required tools
* required processes

It gives you the ability to run a standardised toolchain with all the required packages in an isolated environment. The environment is described in a Dockerfile and realized by providing a image.  

Why the clean world view is not always working and how to make it still work, see the following chapters as well.

## Creating a tool chain

To be able to use files, created by the toolchain, we have to make sure that the file system permission of created files are correct. Those permission are given to a user, a group and others. User and group are identified by an ID. On a local system group ID's usually start at '1' and user ID's at 1000. Sometimes they are different. To make sure that that ID on your PC and ID's used inside containers match you have to create images that are adapted to your local circumstances.

If you are lazy, and you only have one user on your system and if that user/group has a ID of 1000 or if you are on windows(not inside a WSL container), you can run:

````pwsh
docker image build -t dotnet_toolchain --target base -f _toolchain/Dockerfile _toolchain/
````

If you run the command inside a linux environment I highly suggest to run it like this:

````pwsh
docker image build --build-arg UID=$(id -u) --build-arg GID=$(id -g) -t dotnet_toolchain --target base -f _toolchain/Dockerfile _toolchain/
````

This command will ensure that the user running programs inside a container will have the correct ID's and thus sets file permissions in a usable way.

### Debugging failing builds

If you are changing the Dockerfile to add new features to the created images you might encounter errors durring the build  process. Durring the build process docker goes through the Dockerfile and creates multiple intermediate images for each statement. If one of the statements fail and you want to investigate, the following might help.

To reproduce the scenario, you need access to the intermediate image. For that you need it's ID. Per default buildKit is used to create images and it does not show images durring build. To see the ID's you have to disable it by setting the environment variable DOCKER_BUILDKIT to zero. In bash this can be achived in a oneliner for one execution like so:

````bash
DOCKER_BUILDKIT=0 docker image build --build-arg UID=$(id -u) --build-arg GID=$(id -g) -t dotnet_toolchain --target base -f _toolchain/Dockerfile _toolchain/
````

Or in powershell set the environment variable first and optionally delete later:

````pwsh
$env:DOCKER_BUILDKIT=0 
docker image build --build-arg UID=$(id -u) --build-arg GID=$(id -g) -t dotnet_toolchain --target base -f _toolchain/Dockerfile _toolchain/
````

Now you have to pick the ID of the last succesfull build image. Use that ID to create a container that you can then use interactivly:

````pwsh
docker run --rm -it <ID> bash -il
````

This will open a interactive shell inside a container based oin that image. Now you can execute the failing command and have a look inside the container what went wrong.

## Using the tool chain for local results

Precondition: you need a image name **dotnet_toolchain** according to the previous chapter.

The general usage is like this:

````pwsh
docker run --rm -v "$(pwd):/work" -w /work dotnet_toolchain <cmd> [arg1 arg2]
````

This will:

* start a container that is removed after work is done (**``--rm``**)
* map your working directory into the container into the container local directory ``/work``(**``-v "$(pwd):/work"``**)
* runs within the working directory ``/work``(**``-w /work``**)
* use the image tagged with **dotnet_toolchain**
* uses the command **\<cmd>**
* gives **\<cmd>** the arguments **[arg1 arg2]**

### Example: A console based program

To create a new project, run:

````pwsh
docker run --rm -v "$(pwd):/work" -w /work dotnet_toolchain dotnet new console -o src/myConsoleApp2
````

This will create a folder **myConsoleApp2** in the local ``src`` folder containing code from a template for a simple console application.

To build the project, go into the project folder and run:

````pwsh
docker run --rm -v "$(pwd):/work" -w /work dotnet_toolchain dotnet build src/myConsoleApp2
````

To execute it, run:

````pwsh
docker run --rm -v "$(pwd):/work" -w /work dotnet_toolchain dotnet run --project src/myConsoleApp2
````

### Example: a simple web app

To create a new project, run:

````pwsh
docker run --rm -v "$(pwd):/work" -w /work dotnet_toolchain dotnet new webApp -o src/myWebApp2
````

This will create a folder **myWebApp2** in the local ``src`` folder containing code from a template for a simple web application.

To run a application that exposes network endpoints(non https) you have to do the following:

````pwsh
docker run -it --rm -v "$(pwd):/work" -w /work -p 8000:80 -e ASPNETCORE_URLS=http://+:80 -e ASPNETCORE_ENVIRONMENT=Development dotnet_toolchain dotnet run --no-launch-profile --project src/myWebApp2
````

The command will start the web application inside a container. Inside the container the server listens to port **80**, which is being mapped by docker to port **8000** on the local host. E.g. you can have a look at it in your local browser by visiting [http://localhost:8000](http://localhost:8000)

To be able to easily kill the server that serves the app, we have to start the container with a interactive shell. Thats we the ``-it`` argument got added. So CRTL + C (STRG + C for the germans) should kill it. 

If you forgot the ``-it`` parameter, the you will have a non interactive shell, e.g. your button presses don not reach the server process. In that case run ``docker ps`` and find the ID of your container. The kill it with:

````pwsh
docker kill <ID>
````

You could also name the container(*myServerApp*) by slightly modifying the call(only really need for non-interactive shells):

````pwsh
docker run --rm -v "$(pwd):/work" -w /work -p 8000:80 -e ASPNETCORE_URLS=http://+:80 -e ASPNETCORE_ENVIRONMENT=Development --name myServerApp dotnet_toolchain dotnet run --no-launch-profile
````

The you can kill that container with:

````pwsh
docker kill myServerApp
````

 

### Example: a web app with https

#### Development

By todays standards, using http without encryption is folish. Good for learning the basics in a personal lab environment. But as soon as other people use it more care has to be taken. For that http is wrapped in a secure channel and together they form https. 

To not store the certificate in plain text on a hard drive, it is encrypted using a passphrase. And now, every time you want to use the certificate to encrypt your connection, you need that password. Usually you need it only on program start, thus a user prompt or temporary environment variable can be used. But for development that is impractical.

For development purpose you create a developer certificate, encrypt it and use that. To help automate this, you need a ID that can be used by the automation to link certificate and password. Thus first you have to create a ID in your project:

````pwsh
docker run --rm -v "$(pwd):/work" -w /work dotnet_toolchain dotnet user-secrets init
````

Now you can create a certificate for local development(for now it will be in the root folder of the project, not good for any kind of deployment):

````pwsh
docker run --rm -v "$(pwd):/work" -w /work dotnet_toolchain dotnet dev-certs https -ep aspnetapp.pfx -p "scarrypassword"
````

Running the RestAPI server:

````pwsh
docker run -it --rm -v "$(pwd):/src" -w /src -p 8000:80 -p 8001:443 -e ASPNETCORE_URLS="https://+;http://+" -e ASPNETCORE_HTTP_PORT=8000 -e ASPNETCORE_HTTPS_PORT=8001 -e ASPNETCORE_ENVIRONMENT=Development -e ASPNETCORE_Kestrel__Certificates__Default__Password="scarrypassword" -e ASPNETCORE_Kestrel__Certificates__Default__Path=/src/aspnetapp.pfx dotnet_toolchain dotnet run --no-launch-profile
````

Set environment variables:

* ASPNETCORE_URLS - configures the urls the server listens to
* ASPNETCORE_HTTP_PORT - configures the HTTP port
* ASPNETCORE_HTTPS_PORT - configures the HTTPS port
* ASPNETCORE_ENVIRONMENT - The environment to use(our example has non configured ...?)
* ASPNETCORE_Kestrel__Certificates__Default__Path - path for Kestrel(http[s] server) to find our certificate
* ASPNETCORE_Kestrel__Certificates__Default__Password - password of the used certificate

Command line options for ``dotnet run``:

* --no-launch-profile - make sure to start without any profile and use the settings from the environment variables

## Using the tool chain to create a product image in developer mode

Without specifying the product you want to build, a image for the app 'myConsoleApp' wich is tagged with the name **product** will be build with the following command line:

````pwsh
docker image build -f _toolchain/Dockerfile --build-arg UID=$(id -u) --build-arg GID=$(id -g) -t product .
````

You can now start a container based on that image via:

````pwsh
docker run product /app/myConsoleApp
````

If the app is based on asp.net, you do only want to have HTTP and are running a developer build, the run it as follows:

````pwsh
docker run -it --rm -p 8000:80 -e ASPNETCORE_URLS=http://+:80 -e ASPNETCORE_ENVIRONMENT=Development mywebapp /app/myWebApp --no-launch-profile
````

t.b.d.: certifacte handling for images

## how to cross compile for a different target architectures
