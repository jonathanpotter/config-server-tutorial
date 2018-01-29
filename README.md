# Config Server Tutorial

A tutorial for Spring Cloud Config. In part one, you will set up a Spring Cloud Config server running locally on a workstation. In part two, you will set up a GitHub repo where configuration will be managed and set your config server to use this repo. In part three, you will use the config server PCF service instead of managing your own config server app.

## Suggested Prerequisites
- Complete the [Spring Guide for Centralized Configuration](https://spring.io/guides/gs/centralized-configuration/)
- Review the Spring Cloud [documentation](https://cloud.spring.io/spring-cloud-static/spring-cloud.html#_spring_cloud_config)

## Part 1: Config Server Basics
In Part 1, you will use Spring Initializr to write your own config server app and run it locally on your workstation. Completing this part will give you a deeper understanding of how config server works. However, this part is optional if you are planning to use the Config Server PCF service. You may choose to skip ahead to Part 2 if you'd like.

### Create Config Server App
- Create empty app project from [Spring Initializr](http://start.spring.io/).
- - A Gradle Project with Java and Spring Boot 1.5.9
- - Artifact: config-server
- - Dependencies: Actuator, Config Server
- Unzip the app (into a Git repo if you want).
- Open the app in IDE (like IntelliJ).
- You should have an empty Spring Boot app. Test it in IDE or from terminal.

### Test Initial App
```
$ ./gradlew clean build
$ ./gradlew bootRun
$ curl -v 127.0.0.1:8080/health
```

### Make App a Config Server
- Add import for Spring Cloud Config Server and @EnableConfigServer in .java file.
- Configure app to use port 8888 and use the local copy of the git config repo in application.yml.
- Turn off security (for now) in application.yml.
- Test again.
```
$ ./gradlew bootRun
$ curl -v 127.0.0.1:8888/health                 # NOTE: we changed the port to 8888
$ curl -v 127.0.0.1:8888/app/default/master     # The contents of application.properties are here
```

### Test Config Changes
The client apps using config server will be hitting endpoints like `/app/default/master` as above. When the config server receives a request on these endpoints, it will get the latest configuration from the config repo. This can be seen by making changes in the repo.
```
$ ./gradlew bootRun
$ curl -v 127.0.0.1:8888/app/default/master     # Original value of info.foo is bar.
$ cd ~/workspace/central-config
$ echo "info.foo: car" > application.properties # Update the file in the local repo.
$ git add -A .
$ git commit -m "modify info.foo value"
$ curl -v 127.0.0.1:8888/app/default/master     # Modified value of info.foo is fetched from repo and returned.
$ git reset --hard HEAD^                        # Discard the commit reseting info.foo to bar.
```

## Part 2: GitHub Configuration Repo

### Create Git Repo for Configuration Files
- On GitHub, create a new, personal, private repo called central-config on Ford GitHub. In my case, my new repo is at https://github.ford.com/JPOTTE46/central-config.
- Clone new repo locally.
- Create an example configuration file.
- Commit and Git push to GitHub.

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
- Create public/private key pair for GitHub repo.
```
# Generate key pair with options of type, bits, comment, output_file
$ ssh-keygen -t rsa -b 4096 -C "jpotte46/central-config keys" -f /c/users/jpotte46/.ssh/central-config
```
- Add the new public key as a read-only deploy key to the GitHub configuration repo you created earlier. In my case, my config repo is at https://github.ford.com/JPOTTE46/central-config. In a web browser, go to the GitHub repo > Settings > Deploy Keys > Add deploy key. Then paste the contents of the public key. In my case, public key is at /c/users/jpotte46/.ssh/central-config.pub.
- Update the Config Server's application.yml with the GitHub repo SSH address and private key. An [example](http://cloud.spring.io/spring-cloud-static/Dalston.SR5/single/spring-cloud.html#_git_ssh_configuration_using_properties).
- Start app, make config changes on GitHub and test the app.
```
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
