# Unit 1: Working with Operation Framework #

In this example, the Netflix OSS implementation of Eureka and Zuul are performed in IBM Containers. Eureka provides the Service Registry and Service Discovery capabilities and Zuul provides the Service Proxy and Load Balancing capabilities.  

## Exercise 1: Deploying Eureka for BlueCompute

The steps in this exercise are acquired from `https://github.com/ibm-cloud-architecture/refarch-cloudnative-netflix-eureka`. While it is faster to use the provided scripts to deploy the component, you will use individual commands to deploy the component in order to make you aware of the actual process to deploy it.  

<ol>
<li> Explore the eureka application. The application settings are stored in `src/main/resources/application.yml` file. </li>
<ul>
<li> cd refarch-cloudnative-netflix-eureka
<li> vi src/main/resources/application.yml
![eureka-appl](exercises/010-eureka-appl-yml.png)
 <p>For a spring application, the contents of the  application.yml file represents environment variables that can be configured when it is invoked. </p>
<p>These values can be overridden at execution, which is what you will be doing when the Container group is created.  </p>
   - The separation of configuration and code is in-line with 12 factor apps (Factor #3 Config).

2. Build the eureka jar file. There is an option to use Maven or Gradle. In this exercise, you use Gradle as some other components do not have the maven configuration. The build result will be at `build/libs/eureka-0.0.1-SNAPSHOT.jar`, once the processing ends, check that the file exists.

        # ./gradlew clean build

2. The generated JAR file is then used to run the Java application. The application would run in a docker container. The process is to build the container locally and then upload that to Bluemix. Docker building is based on the Dockerfile commands. Look on this file first.

        # cd docker
        # vi Dockerfile
![eureka Dockerfile](exercises/0110-eureka-Dockerfile.png)
In the Dockerfile, the container is built as follows:
  - From the standard java container from DockerHub. 
  - Adding a /tmp filesystem
  - Add app.jar (you will create this app.jar from the build result later)
  - Expose port 8761
  - Define an main process for the container (which is running the Spring application) 

3. Copy the jar file and build docker container locally for eureka

        # cp ../build/libs/eureka-0.0.1-SNAPSHOT.jar app.jar
        # docker build -t netflix-eureka .

2. Deploy eureka as IBM Container. You will use an environment variable called SUFFIX; this is to make your instances unique for the class. If you follow the README.md guide, this suffix is your container namespace string. Note that the group create defines environment variables that represents the entries in the application.yml.

        # export SUFFIX=<your suffix>
        # docker tag netflix-eureka registry.ng.bluemix.net/$(cf ic namespace get)/netflix-eureka-${SUFFIX}
        # docker push registry.ng.bluemix.net/$(cf ic namespace get)/netflix-eureka-${SUFFIX}
        # cf ic group create --name eureka_cluster --publish 8761 --memory 256 --auto \
          --min 1 --max 3 --desired 1 \
          --hostname netflix-eureka-${SUFFIX} \
          --domain mybluemix.net \
          --env eureka.client.fetchRegistry=true \
          --env eureka.client.registerWithEureka=true \
          --env eureka.client.serviceUrl.defaultZone=http://netflix-eureka-${SUFFIX}.mybluemix.net/eureka/ \
          --env eureka.instance.hostname=netflix-eureka-${SUFFIX}.mybluemix.net \
          --env eureka.instance.nonSecurePort=80 \
          --env eureka.port=80 \
           registry.ng.bluemix.net/$(cf ic namespace get)/netflix-eureka-${SUFFIX}


3. Wait a while to let the container initialized, and then verify eureka:

   - Open a Web browser and connect to http://netflix-eureka-${SUFFIX}.mybluemix.net
   - The Eureka interface is shown below with only itself registered as a component
![](exercises/011-eureka-web-1.png)

Now that Eureka is deployed, other components that uses the OSS can be deployed. Next step is to deploy the service proxy, Zuul. 

## Exercise 2: Deploying zuul 
         
The steps in this exercise is acquired from `https://github.com/ibm-cloud-architecture/refarch-cloudnative-netflix-zuul`. Again here, you will use individual commands to deploy it.  

1. Explore zuul application. The application settings are stored in `src/main/resources/application.yml` file.

        # cd refarch-cloudnative-netflix-zuul
        # vi src/main/resources/application.yml
![zuul-appl](exercises/012-zuul-appl-yml.png)
One of the most important parameter here is the eureka.client.serviceUrl.defaultZone. This defines the API endpoint for Eureka to register the service. This value must be retrieved from the Eureka container that you deploy in the previous exercise. 

3. Build zuul, you are again using Gradle. The result will be at `build/libs/zuul-proxy-0.0.1-SNAPSHOT.jar`, once the processing ended, check whether the file exists.

        # ./gradlew clean build 

4. Evaluate the Dockerfile to build zuul.

        # cd docker
        # vi Dockerfile
![zuul Dockerfile](exercises/0130-zuul-Dockerfile.png)
The zuul Dockerfile is quite similar to the eureka one. The difference is that it uses a different jar file that is build in the previous step.

4. Build docker container locally for zuul

        # cp ../build/libs/zuul-proxy-0.0.1-SNAPSHOT.jar app.jar
        # docker build -t netflix-zuul .
5. Deploy zuul as IBM Container; this step assumes SUFFIX is already set. 

        # docker tag netflix-zuul registry.ng.bluemix.net/$(cf ic namespace get)/netflix-zuul-${SUFFIX}
        # docker push registry.ng.bluemix.net/$(cf ic namespace get)/netflix-zuul-${SUFFIX}
        # cf ic group create --name zuul_cluster \
          --publish 8080 --memory 256 --auto --min 1 --max 3 --desired 1 \
          --hostname netflix-zuul-${SUFFIX} \
          --domain mybluemix.net \
          --env eureka.client.serviceUrl.defaultZone="http://netflix-eureka-${SUFFIX}.mybluemix.net/eureka" \
          --env eureka.instance.hostname=netflix-zuul-${SUFFIX}.mybluemix.net \
          --env eureka.instance.nonSecurePort=80 \
          --env eureka.instance.preferIpAddress=false \
          --env spring.cloud.client.hostname=netflix-zuul-${SUFFIX}.mybluemix.net \
          registry.ng.bluemix.net/$(cf ic namespace get)/netflix-zuul-${SUFFIX}

1. Verify that zuul proxy is registered to eureka, open `http://netflix-eureka-${SUFFIX}.mybluemix.net` and verify, it should show 2 registered applications.
![](exercises/013-eureka-web-2.png)

Now, the Netflix OSS components have been deployed. You can proceed to deploy the microservices.




