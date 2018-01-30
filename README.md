# Config Server Tutorial

A tutorial for Spring Cloud Config. In Part 1, you will set up a Spring Cloud Config server running locally on a workstation. In Part 2, you will set up a GitHub repo where configuration will be managed. In Part 3, you will use the config server PCF service instead of managing your own config server app. In Part 4, you will see advanced topics such as managing and upgrading a Config Server PCF service instance.

- [Part 1: Config Server Basics](#Part-1:-Config-Server-Basics)
- [Part 2: GitHub Configuration Repo](#Part-2:-GitHub-Configuration-Repo)
- [Part 3: Config Server on PCF](#Part-3:-Config-Server-on-PCF)
- [Part 4: Advanced Topics](#Part-4:-Advanced-Topics)

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

In Part 1, you used a local file repository to store configuration. Beyond the local workstation environment, you will be using a git repo on the [Ford GitHub](https://github.ford.com) instance. In Part 2, you will create a new repo in GitHub where configuration can be managed and protect the repo with a public/private key pair.  

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

### Create public/private key pair for GitHub repo

The configuration information in your GitHub repo must be secure and protected. In order for configure server to access the repo, we will use a public/private key pair. Passphrase-encrypted private keys are not supported by config server, so do not set a passphrase when creating the key pair.

```bash
# Generate key pair with options of type, bits, comment, output_file location
$ ssh-keygen -t rsa -b 4096 -C "jpotte46/central-config keys" \
  -f /c/users/jpotte46/.ssh/central-config
```

The command above creates 2 key files. In my case, the files are placed in `C:\users\jpotte46\.ssh` and named `central-config.pub` and `central-config`. The public key will be added to the GitHub repo as a Deploy Key. The private key will be used by the config server.

### Add Public Key to GitHub repo
 
Now add the new public key as a read-only deploy key to the GitHub configuration repo. In a web browser, go to the GitHub repo > Settings > Deploy Keys > Add deploy key. Then paste the contents of your public key. In my case, my public key is at `C:\users\jpotte46\.ssh\central-config.pub`.

### Convert Private Key to String
Before proceeding, we must convert the private key to a string replacing all newline characters with `\n`. The config server PCF service will not accept the original formatting with multiple lines. I have done this before in Notepad++ and using the sed utility before, but your mileage may vary. Let me know if you have a better suggestion. Here is how I do it in Notepad++.

Here is the original `C:\users\jpotte46\.ssh\central-config`
```
-----BEGIN RSA PRIVATE KEY-----
MIIJKQIBAAKCAgEAnMq/adADO5lLCRIg6cSKA8p+LCCFvgV7HjolgHkOTjnWQm/d
cM0Ou/XQQqAW/O3MN/TzrQoCBmxG+5FF/F1akRtNRF4po3bUOfefmji6c4MRoLxr
ekCzqoOhIvund7if92fTF51ab450EFDmau5c8J88Vqf70e8b...
```


- Open the private key in Notepad++.
- Open the Find/Replace dialog and configure it like this.
```
Match whole word only: false
Find what:             \r\n
Replace with:          \\n
Search Mode:           Extended
```
- Then click Replace All.


If it worked, you will be left with a single-line string where newlines have been replaced by `\n` characters. It should look something like below. Save this somewhere secure as this is the private key to your GitHub configuration repo. This key will allow the config server to access the GitHub repo.
```
-----BEGIN RSA PRIVATE KEY-----\nMIIJKQIBAAKCAgEAnMq/adADO5lLCRIg6cSKA8p+LCCFvgV7HjolgHkOTjnWQm/d\ncM0Ou/XQQqAW/O3MN/TzrQoCBmxG+5FF/F1akRtNRF4po3bUOfefmji6c4MRoLxr\nekCzqoOhIvund7if92fTF51ab450EFDmau5c8J88Vqf70e8b...
```

### Add Private Key to Config Server (Optional)

If you built a config server app in Part 1, you can update your config server app to use the new GitHub repo with the new private key. However, if you did not do Part 1, or you just want to jump ahead anyway, you can totally skip this section and go on to Part 3. In Part 3, we'll be creating a config server service instance and we will configure it using this private key that we made into a string in the last step.

If you've decided to continue with this step, you will need to add the details of your GitHub repo to your config server app's application.properties file (or application.yml if you prefer).

`src/main/resources/application.properties`
```bash
# Replace with your GitHub configuration repo address and private key.
# Note that the properties are case sensitive!
spring.cloud.config.server.git.uri=git@github.ford.com:JPOTTE46/central-config.git
spring.cloud.config.server.git.ignoreLocalSshSettings=true
spring.cloud.config.server.git.privateKey=-----BEGIN RSA PRIVATE KEY-----\nMIIJKQIBA...
```
Some more [documentation](http://cloud.spring.io/spring-cloud-static/Dalston.SR5/single/spring-cloud.html#_git_ssh_configuration_using_properties) on these properties if you need it.

Now start the config server app, make changes in GitHub, and observe as those changes propagate to the config server.
```bash
$ ./gradlew bootRun
$ curl -v 127.0.0.1:8080/health                 # Note config server is using the GitHub repo
$ curl -v 127.0.0.1:8888/app/default/master     # The initial value of info.foo is fetched

# Now make changes to info.foo value on GitHub.
$ cd ~/workspace/central-config
$ echo "info.foo: far" > application.properties
$ git add -A .
$ git commit -m "make a change"
$ git push

# When config server app receives the request below, it will 
# git pull from the GitHub repo. The modified value of info.foo
# will then be displayed.
$ curl -v 127.0.0.1:8888/app/default/master     
```

## Part 3: Config Server on PCF
PCF offers config server as a service in the PCF marketplace. This means that you do not have to write or manage your own config server. You will still need to have a GitHub repo where your config server service instance will read your configuration.

### Write a Config Client
In the previous examples, we've just curled the unsecured endpoint of the config server app to watch it return configuration information. But going forward, we need a client app that will do this. We will stand up a quick one.

- Create empty app project from [Spring Initializr](http://start.spring.io/).
  - A Gradle Project with Java and Spring Boot 1.5.9
  - Artifact: config-client
  - Dependencies: Actuator, Config Client, Web
- Download and unzip the app.
- Open the app in an IDE (like Eclipse or IntelliJ).
- You should now have an empty Spring Boot app.

`ConfigClientApplication.java`

```java
// Add these imports
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.beans.factory.annotation.Value;

@SpringBootApplication
public class ConfigClientApplication {

	public static void main(String[] args) {
		SpringApplication.run(ConfigClientApplication.class, args);
	}
}

// Add this endpoint
@RefreshScope
@RestController
class InfoFooRestController {

    @Value("${info.foo: some default}")
    private String infoFoo;

    @RequestMapping("/infofoo")
    String getInfoFoo() {
        return this.infoFoo;
    }
}
```

The config client app will expose an `/infofoo` endpoint that will return the value of the info.foo property. The default value is `some default`. However, if the config client app can reach the config server, this value will be updated with the value at the config server which in turn will be the value in our GitHub configuration repo. In our case, this value is `bar`.

Try running the config client locally with no config server running. Confirm that the `/infofoo` endpoint returns `some default`.

Now, push the app to your PCF space.

```bash
$ gradlew clean build
$ cf target -o Your_Org -s Your_Space
$ cf push config-client123456 --random-route -p build/libs/config-client-0.0.1-SNAPSHOT.jar
```

<!-- DRAFT TEXT

Your app should be running on PCF now with a unique route. Test the /infofoo endpoint again for the default response.

-->

### Create Config Server PCF Service Instance

<!-- In Part 1, the config server app we wrote used the properties defined in application.yml. We take this configuration and transform it into a JSON file that will be used to configure our PCF service instance.

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
-->

> TODO: Perform cf create-service p-config-server explaining the config.json file creation using the GitHub private key string from Part 2

### Bind Config Client App to Service Instance 
> TODO: Bind app to service, and restage app explaining that config server instance is secured by PCF’s OAuth2 service, and binding the client app injects the necessary credentials for authentication with config server.

### Refreshing Configuration
> TODO: Then explain the /refresh endpoint on the client app, make a change in GitHub, show value at client app, then hit the /refresh endpoint and show value at client app.

## Part 4: Advanced Topics

### Refresh Configuration Changes with Spring Cloud Bus
> TODO: Set up the Spring Cloud Bus for running multiple client app instances. See https://sivalabs.in/2017/08/spring-cloud-tutorials-auto-refresh-config-changes-using-spring-cloud-bus/ and http://www.baeldung.com/spring-cloud-bus.

#### High-Availability Mode for Config Server
> TODO: Running multiple instances of the service instance for HA. See https://docs.pivotal.io/spring-cloud-services/1-4/common/config-server/managing-service-instances.html.

#### Upgrading Config Server Service Instances
> TODO: How upgrades affect Config Server and how to upgrade. See https://docs.pivotal.io/spring-cloud-services/1-4/common/config-server/managing-service-instances.html.

## References
Here are some additional completed examples and references.
- Config Server [example](https://github.com/spring-cloud-samples/configserver).