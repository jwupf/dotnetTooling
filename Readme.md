# using this example

## tool chain

### build the tool chain

You need to build the final container with your own user to have the GIS/UID match inside the container with yours outside of it. This ensures that you can use the created files afterwards. 

If you are lazy, and you only have one user on your system and if that user/group has a ID of 1000 or if you are on windows(not inside a WSL container), you can run:

````pwsh
docker build -t dotnet_toolchain -f Dockerfile .
````

If you run the command inside a linux environment I highly suggest to run it like this:

````pwsh
docker build -f Dockerfile --build-arg UID=$(id -u) --build-arg GID=$(id -g) -t dotnet_toolchain  .
````

### using the tool chain

The examples expect you to be in the (project) directory you want to work with.

The general usage is like this:

````pwsh
docker run --rm -v "$(pwd):/src" -w /src dotnet_toolchain <cmd> [arg1 arg2]
````

To create a new project in the current folder, run:

````pwsh
docker run --rm -v "$(pwd):/src" -w /src dotnet_toolchain dotnet new webapp -o myWebApp
````

To build the project, go into the project folder and run:

````pwsh
docker run --rm -v "$(pwd):/src" -w /src dotnet_toolchain dotnet build
````

To run a simple console application, run:

````pwsh
docker run --rm -v "$(pwd):/src" -w /src dotnet_toolchain dotnet run
````

To run a application that exposes network endpoints(non https) you have to do the following:

````pwsh
docker run -it --rm -v "$(pwd):/src" -w /src -p 8000:80 -e ASPNETCORE_URLS=http://+:80 -e ASPNETCORE_ENVIRONMENT=Development dotnet_toolchain dotnet run --no-launch-profile
````

CRTL + C (STRG + C for the germans) should kill it. If you forgot the ``-it`` parameter, the you will have a non interactive shell, e.g. your button presses don not reach the server process. In that case run ``docker ps`` and find the ID of your container. The kill it with:

````pwsh
docker kill <ID>
````

You could also name the container by slightly modifying the call(only really need for non-interactive shells):

````pwsh
docker run --rm -v "$(pwd):/src" -w /src -p 8000:80 -e ASPNETCORE_URLS=http://+:80 -e ASPNETCORE_ENVIRONMENT=Development --name myServerApp dotnet_toolchain dotnet run --no-launch-profile
````

The you can kill it with:

````pwsh
docker kill myServerApp
````
