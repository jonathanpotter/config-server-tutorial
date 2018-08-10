# Config Server Tutorial

A tutorial for Spring Cloud Config. In Part 1 (optional), you will set up a Spring Cloud Config Server running locally on a workstation. In Part 2, you will set up a GitHub repo where your configuration will be managed. In Part 3, you will use the PCF marketplace to create a PCF service instance of a Config Server - I'll call this a PCF Config Server Instance. (If you did Part 1, the PCF Config Server Instance replaces the Config Server app that you built in Part 1 so you don't have to write and manage your own Config Server app. Let PCF do it for you.) In Part 4, you will write a client app that retrieves configuration from your PCF Config Server Instance. In Part 5, you will see advanced topics such as managing and upgrading a Config Server PCF service instance.

- [Part 1: Config Server Basics](#part-1-config-server-basics)
- [Part 2: GitHub Configuration Repo](#part-2-github-configuration-repo)
- [Part 3: Config Server on PCF](#part-3-config-server-on-pcf)
- [Part 4: Writing a Config Client App](#part-4-writing-a-config-client-app)
- [Part 5: Advanced Topics](#part-5-advanced-topics)

## Suggested Prerequisites

- Complete the [Spring Guide for Centralized Configuration](https://spring.io/guides/gs/centralized-configuration/)
- Review the Spring Cloud [documentation](https://cloud.spring.io/spring-cloud-static/spring-cloud.html#_spring_cloud_config)

`Config Server Architecture Diagram`
![Config Server Architecture Diagram](https://docs.pivotal.io/spring-cloud-services/1-5/common/config-server/images/config-server-fig1.png)

## Part 1: Config Server Basics

In Part 1, you will use Spring Initializr to write your own config server app and run it locally on your workstation. Completing this part will give you a deeper understanding of how config server works. However, this part is optional if you are planning to use the Config Server PCF service. One of the benefits of using the PCF service is that you do not have to build and manage your own config server app. You may choose to skip ahead to Part 2 if you'd like.

### Create a Local Git Repo

Config server requires a git repository where it can read configuration information. Create an empty git repo on your local workstation. Then create some configuration in a file called application.properties that we'll use later.

```bash
# Note: I'm using Git Bash for Windows
$ cd ~/workspace
$ mkdir config-repo
$ cd config-repo
$ git init
$ echo "info.foo: bar" > application.properties
$ git add -A .
$ git commit -m "add application.properties"
```

### Create Config Server App

- Create empty app project from [Spring Initializr](http://start.spring.io/).
  - A Gradle Project with Java and Spring Boot 2.0.4
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
spring.cloud.config.server.git.uri=file:///users/jpotte46/workspace/config-repo

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
$ cd ~/workspace/config-repo
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
$ git clone git@github.ford.com:JPOTTE46/config-repo.git
$ cd ./config-repo
$ echo "info.foo: bar" > application.properties
$ git add -A .
$ git commit -m "add application.properties"
$ git push
```

### Create public/private key pair for GitHub repo

The configuration information in your GitHub repo must be secure and protected. In order for configure server to access the repo, we will use a public/private key pair. Passphrase-encrypted private keys are not supported by config server, so do not set a passphrase when creating the key pair.

```bash
# Generate key pair with options of type, bits, comment, output_file location
$ ssh-keygen -t rsa -b 4096 -C "jpotte46/config-repo keys" \
  -f /c/users/jpotte46/.ssh/config-repo
```

The command above creates 2 key files. In my case, the files are placed in `C:\users\jpotte46\.ssh` and named `config-repo.pub` and `config-repo`. The public key will be added to the GitHub repo as a Deploy Key. The private key will be used by the config server.

### Add Public Key to GitHub repo
 
Now add the new public key as a read-only deploy key to the GitHub configuration repo. In a web browser, go to the GitHub repo > Settings > Deploy Keys > Add deploy key. Then paste the contents of your public key. In my case, my public key is at `C:\users\jpotte46\.ssh\config-repo.pub`.

### Convert Private Key to String
Config Server will not accept the original formatting of the private key with multiple lines. Before proceeding, we must convert the private key to a string replacing all newline characters with `\n`.

I have done this before in Notepad++ or using the sed utility, but your mileage may vary. Also, our team has a [PCF Dev Guide](https://github.ford.com/PCFDev-Reference/pcfdev-sample-config-repo) with a smooth bash script to do this with sed. Here is how I do it in Notepad++.

Here is how the original file looks.
`C:\users\jpotte46\.ssh\config-repo`

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
- Save it somewhere secure.

If it worked, you will be left with a single-line string where newlines have been replaced by `\n` characters. It should look something like below. Save this somewhere secure as this is the private key to your GitHub configuration repo. This key will be used by the config server to access the GitHub repo.
`C:\users\jpotte46\.ssh\config-repo-string`
```
-----BEGIN RSA PRIVATE KEY-----\nMIIJKQIBAAKCAgEAnMq/adADO5lLCRIg6cSKA8p+LCCFvgV7HjolgHkOTjnWQm/d\ncM0Ou/XQQqAW/O3MN/TzrQoCBmxG+5FF/F1akRtNRF4po3bUOfefmji6c4MRoLxr\nekCzqoOhIvund7if92fTF51ab450EFDmau5c8J88Vqf70e8b...
```

<!---
### Add Private Key to Config Server (Optional)

If you built a config server app in Part 1, you can update your config server app to use the new GitHub repo with the new private key. However, if you did not do Part 1, or you just want to jump ahead anyway, you can totally skip this section and go on to Part 3. In Part 3, we'll be creating a PCF Config Server Instance and we will configure it using this private key that we made into a string in the last step.

If you've decided to continue with this step, you will need to add the details of your GitHub repo to your config server app's application.properties file (or application.yml if you prefer).

`src/main/resources/application.properties`
```bash
# Replace with your GitHub configuration repo address and private key.
# Note that the properties are case sensitive!
spring.cloud.config.server.git.uri=git@github.ford.com:JPOTTE46/config-repo.git
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
$ cd ~/workspace/config-repo
$ echo "info.foo: far" > application.properties
$ git add -A .
$ git commit -m "make a change"
$ git push

# When config server app receives the request below, it will 
# git pull from the GitHub repo. The modified value of info.foo
# will then be displayed.
$ curl -v 127.0.0.1:8888/app/default/master     
```
--->
## Part 3: Config Server on PCF
PCF offers config server as a service in the PCF marketplace. This means that you do not have to write or manage your own Config Server as an app. Instead you will create a Config Server as a service instance from the PCF marketplace. Your PCF Config Server Instance will retrieve configuration from the GitHub config repo you created in Part 2.

### Create Config Server PCF Service Instance

Now create a config server instance from the PCF marketplace. This instance will need to be configured to use the private key in the string format that was created in Part 2. The create-service command needs this configuration in JSON format. The command below shows how to pass the configuration as JSON when creating a service instance. Replace the values below with your private key and the GitHub configuration repo you created in Part 2.

If you completed Part 1 of creating your own config server app, we defined some properties in an application.yml file. used the properties defined in application.yml. We take this configuration and transform it into a JSON file that will be used to configure our PCF service instance.

```
$ cf marketplace
$ PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----\nMIIJK...oncM=\n-----END RSA PRIVATE KEY-----"
$ cf create-service p-config-server standard my-sample-config-server -c "$(cat <<EOF
{
   "git": {
       "uri": "git@github.ford.com:JPOTTE46/config-repo.git",
       "privateKey": "$PRIVATE_KEY"
   }
}
EOF
)"
```

Check for errors from command line and in App Manager dashboard. Run the command below and get the Dashboard URL for the config service instance. Open the dashboard in a web browser.

```bash
$ cf service my-sample-config-server

Service instance: my-sample-config-server
Service: p-config-server
Bound apps:
Tags:
Plan: standard
Description: Config Server for Spring Cloud Applications
Documentation url: http://docs.pivotal.io/spring-cloud-services/
Dashboard: https://spring-cloud-broker.apps-pcf02v2i.cf.ford.com/dashboard/p-config-server/e13e3852-575b-4b89-a86a-493c78d38142

Last Operation
Status: create succeeded
Message:
Started: 2018-08-09T14:51:43Z
Updated: 2018-08-09T14:51:44Z
```

If you see a message banner in green that "Config server is online", then the config service instance has been created, and successfully connected to the GitHub configuration repo using the private key your created in Part 2. You can move on to the next section.

Often teams will see a red error message. If so, go back through Part 2 and double-check each step. Mistake usually occurs during these steps:
- Reformatting the private key into a string replacing the carriage returns and line feeds with a `\n` characters.
- Specifying the wrong GitHub uri. Ford's GitHub installation only supports SSH (not HTTPS).
- Perhaps your shell handles quotes during the `cf create-service` step. You may need some escape characters in the JSON during this step.

## Part 4: Writing a Config Client App

### Write a Config Client
In the previous examples, we've just curled the unsecured endpoint of the config server app to watch it return configuration information. But going forward, we need a client app that will retrieve configuration information from a secured config server instance. We will stand up a quick config client app.

- Create empty app project from [Spring Initializr](http://start.spring.io/).
  - A Gradle Project with Java and Spring Boot 2.0.4
  - Artifact: config-client
  - Dependencies: Actuator, Config Client, Web
- Download and unzip the app.
- Open the app in an IDE (like Eclipse or IntelliJ).
- You should now have an empty Spring Boot app.

Modify as below to set up an `/infofoo` endpoint that returns the value of `info.foo`.

`src/main/java/.../ConfigClientApplication.java`

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

Modify as below to set up basic authentication credentials on the client config app endpoints.

`src/main/resources/.../application.properties`

```
spring.security.user.name=admin
spring.security.user.password=thispasswordshouldbechangedbutyouprobablywont
```

Modify as below to include the necessary Pivotal dependencies (`io.pivotal.spring.cloud`). Full details are given in [Pivotal Docs](https://docs.pivotal.io/spring-cloud-services/1-5/common/config-server/writing-client-applications.html).

`build.gradle`

```groovy
dependencies {
	compile('org.springframework.boot:spring-boot-starter-actuator')
	compile('org.springframework.boot:spring-boot-starter-web')
	compile('org.springframework.cloud:spring-cloud-starter-config')
	compile('io.pivotal.spring.cloud:spring-cloud-services-starter-config-client')
	testCompile('org.springframework.boot:spring-boot-starter-test')
}

dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
        mavenBom "io.pivotal.spring.cloud:spring-cloud-services-dependencies:2.0.1.RELEASE"
    }
}
```


The config client app will expose an `/infofoo` endpoint that will return the value of the info.foo property. The default value is `some default`. However, if the config client app can reach the config server, this value will be updated with the value at the config server which in turn will be the value set in our GitHub configuration repo. In our case, this value is `bar` in the GitHub config repo.

Try running the config client locally with no config server running. Confirm that the `/infofoo` endpoint returns `some default`.
```bash
$ curl -s -u admin:thispasswordshouldbechangedbutyouprobablywont localhost:8080/infofoo
some default
```

Now, push the app to your PCF space.

```bash
$ gradlew clean build
$ cf target -o Your_Org -s Your_Space
$ cf push config-client --random-route -p build/libs/config-client-0.0.1-SNAPSHOT.jar
```

Your app should be running on PCF now with a unique route. Test the /infofoo endpoint again for the default response.

```bash
$ curl -s -u admin:thispasswordshouldbechangedbutyouprobablywont YOUR_APP_URL/infofoo
some default
```

Leave your app running in PCF and go on to the next section.

### Bind Config Client App to Service Instance

To recap, you have created a GitHub configuration repo, a PCF Config Server Instance that successfully connected to the config repo, and a client app that can read configuration from a PCF Config Server Instance after we set up the client app to do so. Now we need to connect the client app to the PCF Config Server Instance. This is done through PCF service binding.

PCF service instances are secured by PCF’s OAuth2 service. Binding the client-config app to the PCF Config Server Instance injects the necessary credentials for the app to authenticate with the PCF Config Server Instance.

```bash
$ cf bind-service config-client my-sample-config-server
```

If your config-client app was already running during the `bind-service` command, the app should still return the default value at the `infofoo` endpoint. Try it and see.

This is because by default, the configuration values are only read during the client app’s startup. You can observe this by restarting your app and seeing that it retrieves the updated value of `info.foo` as set in the config repo.

```
$ curl -s -u admin:thispasswordshouldbechangedbutyouprobablywont YOUR_APP_URL/infofoo
some default
$ cf restart config-client
$ curl -s -u admin:thispasswordshouldbechangedbutyouprobablywont YOUR_APP_URL/infofoo
bar
```

Success! You have built an app that pulls its configuration from the PCF Config Server Instance during app start up. Change the value of `info.foo` in GitHub and restart your app to see the changes take effect.

## Part 5: Advanced Topics

These are bonus exercises you can do on your own if you'd like to learn more about the advanced features of config server.

### Refreshing Configuration
Client apps of the config server, **DO NOT** regularly poll the config server. However, a refresh of configuration can be triggered by enabling a `/refresh` endpoint. The [Spring Guide on Centralized Configuration](https://spring.io/guides/gs/centralized-configuration/#_reading_configuration_from_the_config_server_using_the_config_client) shows how to do this by annotating an app controller with the Spring Cloud Config `@RefreshScope` and trigging a refresh event.

### Refresh Configuration Changes with Spring Cloud Bus
In the real world, apps run multiple instances. If you want to refresh your app's configuration, you'll have to refresh each app instance individually. Additionally, if you want to refresh multiple microservice apps, you will have to refresh each instance of each app. This is not practical at scale. Spring Cloud Bus uses a message broker so configuration changes can be triggered across multiple apps.

Feel free to work through these tutorials.
- https://sivalabs.in/2017/08/spring-cloud-tutorials-auto-refresh-config-changes-using-spring-cloud-bus/
- http://www.baeldung.com/spring-cloud-bus.

### High-Availability Mode for Config Server
By default, creating a config server from the PCF marketplace results in a single running instance of config server. However, you may want to run multiple instances of the service instance for HA. See https://docs.pivotal.io/spring-cloud-services/1-4/common/config-server/managing-service-instances.html.

### Upgrading Config Server Service Instances
Dev teams need to pay attend to client dependencies between their app and config server. When the platform support team upgrades config server in the PCF marketplace, dev teams need to update the dependencies in their app and perform upgrade steps for their config server instances. The [Pivotal Docs](https://docs.pivotal.io/spring-cloud-services/1-4/common/config-server/managing-service-instances.html) show how to update config server instances.

## References
Here are some additional completed examples and references.
- [Setting Up GitHub Config Repo and PCF Config Server Instance](https://github.ford.com/PCFDev-Reference/pcfdev-sample-config-repo)
- [Config Server Example](https://github.com/spring-cloud-samples/configserver)

A list of things to clean up after running this tutorial.
- `~/workspace/config-client`
- `~/workspace/config-repo`
- `~/.ssh/config-repo*`
- GitHub Repo `config-repo`
- Delete apps and service instances in PCF environment