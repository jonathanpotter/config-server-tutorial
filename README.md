# Config Server Tutorial

A tutorial for Spring Cloud Config. In part one, you will set up a Spring Cloud Config server running locally on a workstation. In part two, you will set up a GitHub repo where configuration will be managed and set your config server to use this repo. In part three, you will use the config server PCF service instead of managing your own config server app.

## Suggested Prerequisites

- Complete the [Spring Guide for Centralized Configuration](https://spring.io/guides/gs/centralized-configuration/)
- Review the Spring Cloud [documentation](https://cloud.spring.io/spring-cloud-static/spring-cloud.html#_spring_cloud_config)

## Part 1: Config Server Basics

In Part 1, you will use Spring Initializr to write your own config server app and run it locally on your workstation. Completing this part will give you a deeper understanding of how config server works. However, this part is optional if you are planning to use the Config Server PCF service. You may choose to skip ahead to Part 2 if you'd like.

### Create a Local Git Repo

Config server requires a git repository where it can read configuration information. Create an empty git repo on your local workstation. Then create some configuration in a file called application.properties that we'll use later.

```bash
# Note: I'm using Git Bash for Windows
$ cd ~/workspace
$ mkdir central-config
$ cd central-config
$ git init
$ echo "info.foo: bar" > application.properties
$ git add -A .
$ git commit -m "add application.properties"
```

### Create Config Server App

- Create empty app project from [Spring Initializr](http://start.spring.io/).
  - A Gradle Project with Java and Spring Boot 1.5.9
  - Artifact: config-server
  - Dependencies: Actuator, Config Server
- Download and unzip the app.
- Open the app in an IDE (like Eclipse or IntelliJ).
- You should have an empty Spring Boot app. Test it in IDE or from terminal.

```bash
$ ./gradlew clean build
$ ./gradlew bootRun
# If everything is working, the /health endpoint provided by Spring Boot Actuator, 
# should return a 200 OK message and some positive-sounding JSON.
$ curl -v 127.0.0.1:8080/health
```

Now, turn the Spring Boot app into a config server and configure it to use the local git repo as its configuration repository.

- Edit .java file adding import for Spring Cloud Config Server and the @EnableConfigServer annotation.
- Bam! Your app is now a config server!
- Edit application.properties to configure the config server. See example below.

`src/main/resources/application.properties`

```
# The standard config server port is 8888.
server.port=8888

# My local configuration repository.
spring.cloud.config.server.git.uri=file:///users/jpotte46/workspace/central-config

# Don't do this in production! But for now, turn off authentication.
management.security.enabled=false
```

Now, test your working config server.

```bash
$ ./gradlew bootRun
$ curl -v 127.0.0.1:8888/health                 # NOTE: we changed the port to 8888
$ curl -v 127.0.0.1:8888/app/default/master     # The contents of application.properties are here
```

To recap, you've created a local Git repo to hold configuration information, and you've written and deployed a config server that reads configuration information from this repo and makes it available to client apps via http on port 8888. Note that configuration endpoints, such as `/app/default/master` above, will be protected in production, but we have temporarily left them unprotected for this tutorial.

### Test Making Config Changes in Repo

The client apps using config server will be hitting endpoints like `/app/default/master` as above. When the config server receives a request on these endpoints, it will get the latest configuration from the config repo. This propagation of configuration can be seen by making changes in the repo and curling the config server again.

```bash
$ ./gradlew bootRun
$ curl -v 127.0.0.1:8888/app/default/master     # Original value of info.foo is bar.
$ cd ~/workspace/central-config
$ echo "info.foo: car" > application.properties # Update the file in the local repo.
$ curl -v 127.0.0.1:8888/app/default/master     # Modified value of info.foo is fetched from repo and returned.
```

## Part 2: GitHub Configuration Repo

In Part 1, you used a local file repository to store configuration. Beyond the local workstation environment, you will be using a git repo on the [Ford GitHub](https://github.ford.com) instance. In Part 2, you will create a new repo in GitHub and reconfigure your config server app to use this repo.  

### Create Repo on GitHub for Configuration Files
- On [Ford GitHub](https://github.ford.com), create a new, private repo.
- Clone the new repo locally.
- Create an example configuration file and push to GitHub.

```bash
# Note: Using Git Bash for Windows
$ cd ~/workspace
$ git clone git@github.ford.com:JPOTTE46/central-config.git
$ cd ./central-config
$ echo "info.foo: bar" > application.properties
$ git add -A .
$ git commit -m "add application.properties"
$ git push
```

### Make Config Server Use GitHub Repo

The configuration information in your GitHub repo must be secure and protected. In order for configure server to access the repo, we will use a public/private key pair.

#### Create public/private key pair for GitHub repo

```bash
# Generate key pair with options of type, bits, comment, output_file location
$ ssh-keygen -t rsa -b 4096 -C "jpotte46/central-config keys" \
  -f /c/users/jpotte46/.ssh/central-config
```

The command above creates 2 key files. In my case, the files are placed in `C:\users\jpotte46\.ssh` and named `central-config.pub` and `central-config`. The public key will be added to the GitHub repo as a Deploy Key. The private key will be used by the config server.

#### Add Public Key to GitHub repo
 
Now add the new public key as a read-only deploy key to the GitHub configuration repo. In a web browser, go to the GitHub repo > Settings > Deploy Keys > Add deploy key. Then paste the contents of your public key. In my case, my public key is at `C:\users\jpotte46\.ssh\central-config.pub`.

#### Add Private Key to Config Server
- Update the Config Server's application.yml with the GitHub repo SSH address and private key. An [example](http://cloud.spring.io/spring-cloud-static/Dalston.SR5/single/spring-cloud.html#_git_ssh_configuration_using_properties).
- Start app, make config changes on GitHub and test the app.
```bash
$ ./gradlew bootRun
$ curl -v 127.0.0.1:8080/health                 # Note the GitHub repo address as the config source
$ curl -v 127.0.0.1:8888/app/default/master
# Make changes to info.foo value on GitHub.
$ curl -v 127.0.0.1:8888/app/default/master     # Modified value of info.foo is fetched from GitHub and returned.
```

## Part 3: Config Server on PCF
PCF offers config server as a service in the PCF marketplace. This means that you do not have to write or manage your own config server. You do need to have a GitHub repo where your config server service instance will read your configuration.

### Create Config Server PCF Service Instance
In the previous section, our config server used the properties defined in application.yml. We take this configuration and transform it into a JSON file that will be used to configure our PCF service instance.

```
// convert application.yml to config-server.json
cf marketplace
cf create-service p-config-server standard my-config-server -c config-server.json
```
Now push the config-client app to PCF and bind the app to the config-server instance.
```
cf push
// cf bind-service
// test the app
```

## Some Completed Examples
- Config Server [example](https://github.com/spring-cloud-samples/configserver).
