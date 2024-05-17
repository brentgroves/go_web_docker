# **[Go web app in Docker](https://semaphoreci.com/community/tutorials/how-to-deploy-a-go-web-application-with-docker)**

## reference

**[Semaphore CI/CD](https://brentgroves.semaphoreci.com/)**
github.com/astaxie/beego

## Goals

By the end of this article, you will:

- Have a basic understanding of Docker,
- Find out how Docker can help you while developing a Go application,
- Learn how to create a Docker container for a Go application for production, and
- Know how to use Continuous Integration and Delivery (CI/CD) to automatically build a Docker image.

## Prerequisites

For this tutorial, you will need:

- Docker installed on your machine.
- A free Docker Hub account.
- A Semaphore account.

You can find all the code for this tutorial in the **[golang-mathapp](https://github.com/TomFern/golang-mathapp)** repository.

## Advantages over Virtual Machines

Containers offer similar resource allocation and isolation benefits as virtual machines. However, the similarity ends there.

A virtual machine needs its own guest operating system while a container shares the kernel of the host operating system. This means that containers are much lighter and need fewer resources. A virtual machine is, in essence, an operating system within an operating system. Containers, on the other hand, are just like any other application in the system. Basically, containers need fewer resources (memory, disk space, etc.) than virtual machines, and have much faster start-up times than virtual machines.

## Benefits of Docker During Development

Some of the benefits of using Docker in development include:

A standard development environment used by all team members,
Updating dependencies centrally and using the same container everywhere,
An identical environment in development to that of production, and
Fixing potential problems that might appear only in production.

## Why Use Docker with a Go Web Application?

Most Go applications are simple binaries. This begs the question—why use Docker with a Go application? Some of the reasons to use Docker with Go include:

Web applications typically have templates and configuration files. Docker helps keep these files in sync with the binary.
Docker ensures identical setups in development and production. There are times when an application works in development, but not in production. Using Docker frees you from having to worry about problems like these.
Machines, operating systems, and installed software can vary significantly across a large team. Docker provides a mechanism to ensure a consistent development setup. This makes teams more productive and reduces friction and avoidable issues during development.

## Creating a Simple Go Web Application

We’ll create a simple web application in Go for demonstration in this article. This application, which we’ll call MathApp, will:

- Expose routes for different mathematical operations,
- Use HTML templates for views,
- Use a configuration file to customize the application, and
- Include tests for selected functions.

Visiting /sum/3/6 will show a page with the result of adding 3 and 6. Likewise, visiting /product/3/6 will show a page with the product of 3 and 6.

In this article, we used the **[Beego](https://beego.vip/)** framework. Note that you can use any framework (or none at all) for your application.

## Final Directory Structure

Upon completion, the directory structure of MathApp will look like:

```bash
MathApp
    ├── Dockerfile
    ├── Dockerfile.production
    └── src
        ├── conf
        │   └── app.conf    
        ├── go.mod    
        ├── go.src
        ├── main.go
        ├── main_test.go    
        ├── vendor
        └── views
            ├── invalid-route.html
            └── result.html

```

The main application file is main.go, located at the src directory. This file contains all the functionality of the app. Some of the functionality from main.go is tested using main_test.go.

The views folder contains the view files invalid-route.html and result.html. The configuration file app.conf is placed in the conf folder. Beego uses this file to customize the application.

Create the GitHub Repository
We’ll use **[Go mod](https://blog.golang.org/using-go-modules)**, the official module manager, to handle Go modules in a portable way without having to worry about GOPATH.

A module is a collection of Go packages stored in a file tree with a go.mod file at its root. The go.mod file defines the module’s module path, which is also the import path used for the root directory, and its dependency requirements, which are the other modules needed for a successful build. Each dependency requirement is written as a module path and a specific semantic version.

We’ll start by creating a GitHub repository and cloning it to your machine.

![](https://semaphoreci.com/wp-content/uploads/2020/03/Screenshot103.webp)

Use the repository name to initialize the project:

```bash
pushd .
# Install bee cli globally
go install github.com/beego/bee/v2@latest
#To check whether the `bee` CLI is successfully installed in your system, give this command: `bee version` 🐝

bee version
2024/05/16 14:48:31 INFO     ▶ 0001 Getting bee latest version...
2024/05/16 14:48:31 INFO     ▶ 0002 Your bee are up to date
______
| ___ \
| |_/ /  ___   ___
| ___ \ / _ \ / _ \
| |_/ /|  __/|  __/
\____/  \___| \___| v2.1.0

├── GoVersion : go1.22.0
├── GOOS      : linux
├── GOARCH    : amd64
├── NumCPU    : 4
├── GOPATH    : 
├── GOROOT    : /home/brent/sdk/go1.22.0
├── Compiler  : gc
└── Date      : Thursday, 16 May 2024

mkdir ~/src/go_web_docker/src
cd ~/src/go_web_docker/src
export GOFLAGS=-mod=vendor
export GO111MODULE=on

go mod init github.com/brentgroves/go_web_docker.git 
# (example: go mod init github.com/tomfern/go-web-docker)
```

GOFLAGS
this is nothing but an environment variable(if you don't understand what an environment variable is, think of it like a value that can be accessed by any process in your current environment. These values are maintained by the OS).
So this GOFLAGS variable has a space separated list of flags that will automatically be passed to the appropriate go commands.
The flag we are setting here is mod, this flag is applicable to the go build command and may not be applicable to other go commands.

what does setting -mod=mod flag, actually do during go build?
The -mod flag controls whether go.mod may be automatically updated and whether the vendor directory is used.
-mod=mod tells the go command to ignore the vendor directory and to automatically update go.mod, for example, when an imported package is not provided by any known module.

**[Refer](https://go.dev/ref/mod#build-commands)**

Therefore

GOFLAGS="-mod=mod" go build main.go
is equivalent to

go build -mod=mod main.go

## Vendoring

When using modules, the go command typically satisfies dependencies by downloading modules from their sources into the module cache, then loading packages from those downloaded copies. Vendoring may be used to allow interoperation with older versions of Go, or to ensure that all files used for a build are stored in a single file tree.

The go mod vendor command constructs a directory named vendor in the main module’s root directory containing copies of all packages needed to build and test packages in the main module. Packages that are only imported by tests of packages outside the main module are not included. As with go mod tidy and other module commands, build constraints except for ignore are not considered when constructing the vendor directory.

## GO111MODULE

GO111MODULE is a setting in Go programming language that controls the behavior of Go modules, which is a feature added in Go version 1.11 to help manage dependencies in Go projects.

Before the introduction of Go modules, Go relied on the GOPATH environment variable to manage dependencies. In this model, all Go packages and their dependencies were stored in a single directory called GOPATH. This approach was simple, but it led to several problems, including version conflicts, the need to manually manage dependencies, and difficulties in sharing code.

From now on, we can use these commands:

```bash
# if you dont run tidy first go mod download won't do anything
go mod tidy
go mod download
go mod vendor
go mod verify
all modules verified
```

To download the required dependencies in the vendor/ folder (instead of downloading them in the GOROOT, this will come in handy later).

Application File Contents
Before continuing, let’s create the file structure:

```bash
mkdir conf views
```

The main application file (main.go) contains all the application logic. The contents of this file are as follows:

In your application, this might be split across several files. However, for the purpose of this tutorial, I like to have everything in one place.

## Test File Contents

The main.go file has some functions which need to be tested. The tests for these functions can be found in main_test.go. The contents of this file are as follows:

Testing your application is particularly useful if you want to do **[Continuous Deployment](https://semaphoreci.com/cicd)**. If you have **[adequate testing in place](https://semaphoreci.com/blog/automated-testing-cicd)**, then you can make stress-free deployments anytime, any day of the week.

## View Files Contents

The view files are HTML templates; these are used by the application to display the response to a request. The content of views/result.html is as follows:

```html
<!-- views/result.html -->
<!doctype html>
<html>
    <head>
        <title>MathApp - {{.operation}}</title>
    </head>
    <body>
        The {{.operation}} of {{.num1}} and {{.num2}} is {{.result}}
    </body>
</html>
```

The content of views/invalid-route.html is as follows:

```html
<!-- invalid-route.html -->
<!doctype html>
<html>
    <head>
        <title>MathApp</title>
        <meta name="viewport" content="width=device-width, initial-scale=1">
<!-- The width=device-width part sets the width of the page to follow the screen-width of the device (which will vary depending on the device).

The initial-scale=1.0 part sets the initial zoom level when the page is first loaded by the browser. -->


        <meta charset="UTF-8">
    </head>

    <body>
        Invalid operation
    </body>
</html>
```

What is The Viewport?
The viewport is the user's visible area of a web page.

The viewport varies with the device, and will be smaller on a mobile phone than on a computer screen.

Before tablets and mobile phones, web pages were designed only for computer screens, and it was common for web pages to have a static design and a fixed size.

Then, when we started surfing the internet using tablets and mobile phones, fixed size web pages were too large to fit the viewport. To fix this, browsers on those devices scaled down the entire web page to fit the screen.

This was not perfect!! But a quick fix.

## Configuration File Contents

The conf/app.conf file is read by **[Beego](https://medium.com/@vijeshomen/exploring-golang-and-beego-a-beginners-guide-with-examples-part-1-79619f0db1ac)** to configure the application. Its content is as follows:

Let’s import the necessary packages 😊

```bash
```bash
pushd .

cd ~/src/go_web_docker/src
export GOFLAGS=-mod=vendor
export GO111MODULE=on
# if you dont run tidy first go mod download won't do anything
go mod tidy
go mod download
go mod vendor
go mod verify
all modules verified

```

```bash
appname = mathapp
runmode = "dev"
httpport = 8010
```

In this file:

appname: is the name of the process that the application will run as,
httpport: is the port on which the application will be served, and
runmode: specifies which mode the application should run in. Valid values include dev for development and prod for production.
Finally, if you haven’t yet done so, install the Go modules with:

```bash
# if you dont run tidy first go mod download won't do anything
go mod tidy
go mod download
go mod vendor
go mod verify
all modules verified
```

go mod download is downloading all of the modules in the dependency graph, which it can determine from reading only the go.mod files. It doesn't know which of those modules are actually needed to satisfy a build, so it doesn't pull in the checksums for those modules (because they may not be relevant).

On the other hand, go mod tidy has to walk the package graph in order to ensure that all imports are satisfied. So it knows, for a fact, without doing any extra work, that the modules it walks through are needed for a go build in some configuration.

External libraries is where Go modules come in. You use go mod tidy and go get to download dependencies from the internet when you do not already have them on your computer.

Once you have the libraries, you can use go mod vendor to copy them from your system's Go cache directory to the actual repository that uses them. You check in these dependencies into source control. This way you have total control over the code you depend upon. These dependencies are now part of your code, you own them now. You actually own them, even if you do not vendor them, but you lack control over them, which is a situation that is to be avoided if you like your code future-proof.

## Using Docker During Development

This section will explain the benefits of using Docker during development, and walk you through the steps required to use Docker in development.

## Configuring Docker for Development

We’ll use a Dockerfile to configure Docker for development. The setup should satisfy the following requirements for the development environment:

- We will use the application mentioned in the previous section,
- The files should be accessible both from inside and outside of the container,
- We will use the bee tool, this will be used to live-reload the app (inside the Docker container) during development,
- Docker will expose the application on port 8010,
- In the Docker container, the application is located at /home/app,
- The name of the Docker image we’ll create for development will be mathapp.

Step 1 – Creating the Dockerfile

Go back to the top level of your project:

```bash
cd ..
```

The following Dockerfile should satisfy the above requirements.

```dockerfile
FROM golang:1.18-bullseye

RUN go install github.com/beego/bee/v2@latest

ENV GO111MODULE=on
ENV GOFLAGS=-mod=vendor

ENV APP_HOME /go/src/mathapp
RUN mkdir -p "$APP_HOME"

WORKDIR "$APP_HOME"
EXPOSE 8010
CMD ["bee", "run"]
```

The first line:

```dockerfile
# FROM golang:1.18-bullseye
FROM golang:1.22-bullseye
```

References the **[official image for Go](https://hub.docker.com/_/golang)** as the base image. This image comes with Go 1.18 pre-installed.

Note I changed the Dockerfile to pull golang:1.22 instead of 1.18 to match the go.mod file on my dev system.

The second line:

```dockerfile
RUN go install github.com/beego/bee/v2@latest
```

Installs the bee tool globally (Docker commands run as root by default), which will be used to live-reload our code during development.

Next, we configure the environment variables for Go modules:

```dockerfile
ENV GO111MODULE=on
ENV GOFLAGS=-mod=vendor
```

The next lines:

```dockerfile
ENV APP_HOME /go/src/mathapp
RUN mkdir -p "$APP_HOME"
WORKDIR "$APP_HOME"
```

Creates a folder for the code and makes it active.

The next to last line tells Docker that port 8010 is of interest.

```dockerfile
EXPOSE 8010
```

The final line:

```dockerfile
CMD ["bee", "run"]
```

Uses the bee command to start our application.

Step 2 – Building the Image

Once the Docker file is created, run the following command to create the image:

```bash
pushd .
cd ~/src/go_web_docker
docker build -t mathapp-development .
```

Executing the above command will create an image named mathapp:

-t mathapp: sets the tag name for the new image, we can reference the image later as mathapp:latest
Don’t forget to type the last dot (.) in the command, otherwise you’ll get an error.

This command can be used by everyone working on this application. This will ensure that an identical development environment is used across the team.

To see the list of images on your system, run the following command:

```bash
docker images
```

Note that the exact names and number of images might vary. However, you should see at least the golang and mathapp images in the list:

```bash
REPOSITORY               TAG            IMAGE ID            CREATED                 SIZE
golang                   1.18           25c4671a1478        2 weeks ago             809MB
mathapp-development      latest         8ae092824585        60 seconds ago 
```

Did not see golang image?

## Step 3 – Running the Container

Once you have mathapp, you can start a container with:

```bash
pushd .
cd ~/src/go_web_docker
docker run -it --rm -p 8010:8010 -v $PWD/src:/go/src/mathapp mathapp-development
...
github.com/brentgroves/go_web_docker.git
2024/05/16 19:53:00 SUCCESS  ▶ 0005 Built Successfully!
2024/05/16 19:53:00 INFO     ▶ 0006 Restarting 'mathapp'...
2024/05/16 19:53:00 SUCCESS  ▶ 0007 './mathapp' is running...
2024/05/16 19:53:00.689 [I] [asm_amd64.s:1695]  http server Running on http://:8010
```

Let’s break down the above command to see what it does.

- The docker run command is used to run a container from an image,
- The -it flag starts the container in an interactive mode (tie it to the current shell),
- The --rm flag cleans out the container after it shuts down,
- The --name mathapp-instance names the container mathapp-instance,
- The -p 8010:8010 flag allows the container to be accessed at port 8010,
- The -v $PWD/src:/go/src/mathapp is more involved. It maps the src/ directory from the machine to /go/src/mathapp in the container. This makes the development files available inside and outside the container, and
- The mathapp part specifies the image name to use in the container.

Executing the above command starts the Docker container. This container exposes your application on port 8010. It also rebuilds your application automatically whenever you make a change. You should see the following output in your console:

```bash
______
| ___ \
| |_/ /  ___   ___
| ___ \ / _ \ / _ \
| |_/ /|  __/|  __/
\____/  \___| \___| v2.0.2
2022/05/10 13:39:29 INFO     ▶ 0003 Using 'mathapp' as 'appname'
2022/05/10 13:39:29 INFO     ▶ 0004 Initializing watcher...
2020/03/17 14:43:24.912 [I] [asm_amd64.s:1373]  http server Running on http://:8010
```

To check the setup, visit <http://localhost:8010/sum/4/5> in your browser. You should see something similar to the following:

![](https://semaphoreci.com/wp-content/uploads/2020/03/Screenshot_20200317_105403.webp)

From another terminal list containers:

```bash
docker container ls -a
CONTAINER ID   IMAGE                                                                              COMMAND                  CREATED              STATUS                      PORTS                                       NAMES
...
3e6a202a19e9   mathapp-development                                                                "bee run"                About a minute ago   Up About a minute           0.0.0.0:8010->8010/tcp, :::8010->8010/tcp   youthful_jemison
```

Note: This assumes that you’re working on your local machine.

To try the live-reload feature, make a modification in any of the source files. For instance, edit src/main.go, replace this line:

```go
c.Data["operation"] = operation
To something like this:

c.Data["operation"] = "real " + operation
```

Bee should pick up the change, even inside the container, and reload the application seamlessly:

```bash
______
| ___ \
| |_/ /  ___   ___
| ___ \ / _ \ / _ \
| |_/ /|  __/|  __/
\____/  \___| \___| v2.0.2
2022/05/10 13:39:29 INFO     ▶ 0003 Using 'mathapp' as 'appname'
2022/05/10 13:39:29 INFO     ▶ 0004 Initializing watcher...
2022/05/10 13:39:29 INFO.   [asm_amd64.s:1373]  http server Running on http://:8010
```

Now reload the page on the browser to see the modified message:

![](https://semaphoreci.com/wp-content/uploads/2020/03/Screenshot_20200317_160332.webp)

## Using Docker in Production

This section will explain how to deploy a Go application in a Docker container. We will use Semaphore to do the following:

- **[Automatically build](https://semaphoreci.com/blog/build-stage)** after changes are pushed to the git repository,
- **[Automatically run tests,](https://semaphoreci.com/blog/20-types-of-testing-developers-should-know)**
- Create a Docker image if the build is successful and if the tests pass, and
- Push the Docker image to Docker Hub.

## Creating a Dockerfile for Production

We’ll write a new Dockerfile to create a complete, self-contained image; without external dependencies.

Enter the following contents in a new file called Dockerfile.production:

```dockerfile
# Dockerfile.production

FROM registry.semaphoreci.com/golang:1.22 as builder

ENV APP_HOME /go/src/mathapp

WORKDIR "$APP_HOME"
COPY src/ .

RUN go mod download
RUN go mod verify
RUN go build -o mathapp

FROM registry.semaphoreci.com/golang:1.22

ENV APP_HOME /go/src/mathapp
RUN mkdir -p "$APP_HOME"
WORKDIR "$APP_HOME"

COPY src/conf/ conf/
COPY src/views/ views/
COPY --from=builder "$APP_HOME"/mathapp $APP_HOME

EXPOSE 8010
CMD ["./mathapp"]
```

Let’s take a detailed look at what each of these commands does. The first command:

```dockerfile
FROM registry.semaphoreci.com/golang:1.22 as builder
```

Tells us this is a **[multi-stage build;](https://docs.docker.com/develop/develop-images/multistage-build/)** it defines an intermediate image that will only have one job: compile the Go binary.

You might have noticed that we’re not pulling the image from Docker Hub, the default image registry. Instead, we’re using the Semaphore Docker Registry, which is more convenient, faster, and pulls don’t count against your Docker Hub **[rate limits.](https://docs.docker.com/docker-hub/download-rate-limit/)**

The following commands:

```dockerfile
ENV APP_HOME /go/src/mathapp

WORKDIR "$APP_HOME"
COPY src/ .
```

Creates the application folder for the app and copies the source code.

The last commands in the intermediate image download the modules and build the executable:

```dockerfile
RUN go mod download
RUN go mod verify
RUN go build -o mathapp
```

Next comes the final and definitive container, where we will run the services.

```dockerfile
FROM registry.semaphoreci.com/golang:1.22
```

We use the COPY command to copy files into the image, the --from argument let us copy the generated binary from the builder stage.

```dockerfile
COPY src/conf/ conf/
COPY src/views/ views/
COPY --from=builder $APP_HOME/mathapp $APP_HOME
```

We finalize by exposing the port and starting the binary:

```dockerfile
EXPOSE 8010
CMD ["./mathapp"]
```

To build the deployment image:

```bash
docker build -t mathapp-production -f Dockerfile.production .
```

You can run it with:

```bash
docker run -it --rm -p 8010:8010 mathapp-production
```

Notice that we don’t need to map any directories, as all the source files are included in the container.

To check the setup, visit <http://localhost:8010/sum/4/5> in your browser. You should see something similar to the following:

![](https://semaphoreci.com/wp-content/uploads/2020/03/Screenshot_20200317_105403.webp)

## Continuous Integration with Semaphore

Docker is a great solution to package and deploy Go applications. The only downside is the additional steps required to build and test the image. This hurdle is easily is best dealt with **[Continuous Integration and Continuous Delivery (CI/CD)](https://semaphoreci.com/cicd)**.

A **[Continuous Integration (CI) platform](https://semaphoreci.com/continuous-integration)** can test our code on every iteration, on every push and every merge. Developers adopting CI no longer have to fear of merging branches, nor be anxious about release day. In fact, CI lets developers merge all the time and make safe releases any day of the week. A good CI setup will run a series of comprehensive tests, like the ones we prepared so far, to weed out any bugs.

Once the code is ready, we can extend our CI setup with **[Continuous Delivery (CD)](https://semaphoreci.com/cicd)**. CD can prepare and build the Docker images, leaving them ready to deploy at any time.

## **[NEXT](https://semaphoreci.com/community/tutorials/how-to-deploy-a-go-web-application-with-docker)**

**[github repo](https://medium.com/@joeponzio/how-to-use-a-private-github-repo-as-a-go-module-442fbedc80c9)**
