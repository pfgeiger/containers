# Unit 3: Deploying the Social Review microservice on Cloudant 

This unit exercises walk you on installing the social review exercises.
 
## Exercise 1: Defining a Cloudant database

The social review application runs on a container with a Cloudant database. 

1. Create a new Cloudant service for the Social Review microservice.

         # cf create-service cloudantNoSQLDB Lite socialreviewdb-${SUFFIX}
         # cf create-service-key socialreviewdb-${SUFFIX} Credential-1
         # cf service-key socialreviewdb-${SUFFIX} Credential-1

2. Safe the credentials: url, user, password and host. Use the URL to create database **socialreviewdb**:

         # curl -X PUT <cloudant-url>/socialreviewdb

![cloudant](exercises/0320-sr-cloudant-credential.png)

At this stage, your Cloudant database is ready. The Cloudant database does not need any information about the data structure. Now you can create the container that hosts the application. Again this application is based on the Spring framework.

## Exercise 2: Building the Social Review microservice

This exercise builds the social review application.

1. Look at the socialreview configuration in refarch-cloudnative-micro-socialreview/src/main/resources/application.yml

        # cd refarch-cloudnative-micro-socialreview
        # vi src/main/resources/application.yml
![appl yml](exercises/0325-sr-appl-yml.png)
   The important parameters here are (you should be able to know what their values should be by now):

   - eureka.client.serviceUrl.defaultZone
   - cloudant.username
   - cloudant.password 
   - cloudant.host

2. Build the application using Gradle. The result is stored in `build/libs/micro-soialreview-0.1.0.jar`. Note that the test step will try to run the application, which is not available locally.

        # ./gradlew build -x test

3. Copy the jar file and build docker container. You can look into the docker/Dockerfile. Its content is similar to the one from inventory microservice.

        # cp build/libs/micro-soialreview-0.1.0.jar docker/app.jar
        # cd docker
        # docker build -t cloudnative/socialreviewservice .

3. Load the socialreview microservice container to Bluemix; note that you again use the SUFFIX variable.

        # docker tag cloudnative/socialreviewservice registry.ng.bluemix.net/$(cf ic namespace get)/socialreviewservice-${SUFFIX}
        # docker push registry.ng.bluemix.net/$(cf ic namespace get)/socialreviewservice-${SUFFIX}

4. Run socialreview microservice as IBM Container group 

        # cf ic group create -p 8080 -m 256 \
          --min 1 --desired 1 --auto \
          --name micro-socialreview-group \
          -n socialreviewservice-${SUFFIX} \
          -d mybluemix.net \
          -e "eureka.client.serviceUrl.defaultZone=http://netflix-eureka-${SUFFIX}.mybluemix.net/eureka/" \
          -e "cloudant.username=<uuid>-bluemix" \
          -e "cloudant.password=<password> " \
          -e "cloudant.host=https://<uuid>-bluemix.cloudant.com" \ 
          registry.ng.bluemix.net/$(cf ic namespace get)/socialreviewservice-${SUFFIX}

5. Check the result from a Web browser or `curl` command 

        curl http://socialreviewservice-<suffix>.mybluemix.net/micro/review/



## Exercise 2: Verify from Eureka and Zuul
This exercise explores the integration of the microservices to Eureka and Zuul as the OSS. 

1. Open the Eureka interface for the registered endpoints. Go to `http://netflix-eureka-${SUFFIX}.mybluemix.net/` and check the registered application. 
![socialreview eureka](exercises/023-inv-eureka.png)
2. Check access to inventory API from zuul. Now that we see that eureka recognize the application, and the application itself is active; you can try to access the application from zuul. Go to `http://netflix-zuul-${SUFFIX}.mybluemix.net/socialreview-microservice/micro/review` and check whether you get the same result as before.
![](exercises/029-inv-curl-2.png)




 