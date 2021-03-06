# Deploying the Social Review microservice

This unit exercises walk you on installing the social review exercises. The social review microservice runs in a container and uses the Cloudant database that you created in an earlier exercise.

<em>Expected outcome:</em> You have the socialreview microservice running in a Bluemix container and comminicating with a mySQL database. Success is measured by successfully using the application to query the database.

![appl yml](images/socialarchitecture.png)



## Exercise 1: Building the Social Review microservice

This exercise builds the social review application. Like the inventory application, it uses the Spring framework.



1. If you didn't run the clonePeers.sh previously, you first need to get the code. Enter the following from the base directory:

	  # git clone https://github.com/pfgeiger/refarch-cloudnative-micro-socialreview.git

 	Look at the socialreview configuration in refarch-cloudnative-micro-socialreview/src/main/resources/application.yml

        # cd refarch-cloudnative-micro-socialreview
        # vi src/main/resources/application.yml
![appl yml](images/appl-yml.png)
  	 The important parameters here are:

	- cloudant.username
	- cloudant.password
	- cloudant.host

  The values for theses parameters can be retrieved from Bluemix.
    -Open your Bluemix Console at console.ng.bluemix.net and login if necessary.
    -Open the Navigation bar and click **Dashboard**
    ![appl yml](images/OpenDashboard.png)
    -Scroll to the **All Services** section and locate your CloudantNoSQL DB service. It has the name socialreviewdb-*<suffix>*.
    -Click the database name to open the Manage page, and then click **Service Credentials**.
    appl yml](images/CloudantCredentials.png)
    -There is one Key that is named **Credential-1**. Click **View Credentials** to display the database credentials.  
    -You see the required information for the **usename, password, and host.**

2. Build the application using Gradle. The result is stored in `build/libs/micro-soialreview-0.1.0.jar`. Note that the test step will try to run the application, which is not available locally.

        # ./gradlew build -x test

        Note: the -x option indicates to exclude that directory from the build.

3. Copy the jar file and build docker container. You can look into the docker/Dockerfile. Its content is similar to the one from inventory microservice.

        # cp build/libs/micro-socialreview-0.1.0.jar docker/app.jar
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
          -e "cloudant.username=*<uuid>-bluemix*" \
          -e "cloudant.password=*<password>* " \
          -e "cloudant.host=*https://<uuid>-bluemix.cloudant.com*" \
          registry.ng.bluemix.net/$(cf ic namespace get)/socialreviewservice-${SUFFIX}

          You will need to replace the cloudant.username, password, and host with the values from the Cloudant credentials that you retrieved in **Step !**

5. Check the result from a Web browser or `curl` command

        curl http://socialreviewservice-<suffix>.mybluemix.net/micro/review/
