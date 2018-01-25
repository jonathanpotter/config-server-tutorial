# Config Server Tutorial
A tutorial for Spring Cloud Config.

## Prerequisites
- Complete the [Spring Guide for Centralized Configuration](https://spring.io/guides/gs/centralized-configuration/)
- Review the Spring Cloud [documentation](https://cloud.spring.io/spring-cloud-static/spring-cloud.html#_spring_cloud_config)

## Create Git Repo for Configuration Files
- On GitHub, create a new, personal, private repo called central-config on Ford GitHub. In my case, my new repo is at https://github.ford.com/JPOTTE46/central-config.
- Clone new repo locally
- Create an example configuration file.
- Commit and Git push to GitHub.

```bash
# Note: Using Git Bash for Windows
$ cd ~/workspace
$ git clone git@github.ford.com:JPOTTE46/central-config.git
$ cd ./central-config
$ echo "info.foo: bar" > application.properties
$ git add -A .
$ git commit -m "Add application.properties"
$ git push
```
