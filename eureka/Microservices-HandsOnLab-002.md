# Unit 2: Deploying the Inventory microservice 

This unit exercises walks you though the inventory microservices application installation and looking into how the processing works.

## Exercise 1: Deploying MSSQL database
The MSSQL is running in a stand-alone container. This container can also be considered to run in the Enterprise as an on-premises resources that must be connected through the Secure Gateway. However in this setup, you will connect the Inventory microservices directly to the MSSQL container. The following steps are modified from `https://github.com/ibm-cloud-architecture/refarch-cloudnative-mysql/README.md` to accommodate a SUFFIX to make sure the names are unique. It is assumed the environment variable SUFFIX is already set.

1. Look at the process to build the mysql container

        # cd refarch-cloudnative-mysql
        # vi Dockerfile
![mysql Dockerfile](exercises/020-mysql-Dockerfile.png)
The Dockerfile is a lot more complex than the Eureka and Zuul one. These are the summary on what it is doing:
   -  Using the mysql container from DockerHub
   -  Uses server\_id, master\_ip and master\_root\_password (which are used to create a replication cluster for resiliency; these values can be ignored in this single instance environment)
   -  Install curl and python 2.6 
   -  Install mysql python connector and mysql utilities
   -  Load scripts, set default path and set file permissions
   -  Specify to initialize the container using docker-entrypoint-wrapper.sh and run mysqld

2. Build docker image using the Dockerfile from repo.

        # docker build -t cloudnative/mysql .

4. Tag and push mysql database container to Bluemix, note that you will again use the SUFFIX variable to make your container names unique. 

        # docker tag cloudnative/mysql registry.ng.bluemix.net/$(cf ic namespace get)/mysql-${SUFFIX}
        # docker push registry.ng.bluemix.net/$(cf ic namespace get)/mysql-${SUFFIX}
    
5. Run MySQL container for database `inventorydb`. This database can be connected at `<docker-host-ipaddr/hostname>:3306` as `dbuser` using `Pass4dbUs3R`.
    
        # cf ic run -m 512 --name mysql-${SUFFIX} -p 3306:3306 \
         -e MYSQL_ROOT_PASSWORD=Pass4Admin123 \
         -e MYSQL_USER=dbuser -e MYSQL_PASSWORD=Pass4dbUs3R \
         -e MYSQL_DATABASE=inventorydb \
         registry.ng.bluemix.net/$(cf ic namespace get)/mysql-${SUFFIX}
    
6. Create `items` table and load sample data. You should see message _Data loaded to inventorydb.items._ Verify that there should be 12 rows in the table.
     
        # cf ic exec -it mysql-${SUFFIX} sh load-data.sh
	    # cf ic exec -it mysql-${SUFFIX} bash
	    root@instance-823719832> mysql -u ${MYSQL_USER} -p${MYSQL_PASSWORD} ${MYSQL_DATABASE}
	    mysql> select * from items;
	    mysql> quit
	    root@instance-823719832> exit
   
Inventory database is now ready in Bluemix Container.  

## Exercise 2: Deploying Inventory microservice
The inventory microservice is based on Spring framework and run in IBM Container. The instruction here is based on `https://github.com/ibm-cloud-architecture/refarch-cloudnative-micro-inventory/README.md`.  

1. Explore the application. First look into the configuration file, `src/main/resources/application.yml`. As any Spring framework based application, this is where the configuration is stored.

        # cd refarch-cloudnative-micro-inventory
        # vi src/main/resources/application.yml
![](exercises/022-inv-appl-yml.png)
   The important configuration options are:
   - eureka.client.serviceUrl.defaultZone
   - server.context-path
   - spring.datasource.url
   - spring.datasource.username
   - spring.datasource.password
   
    These options will be overriden when the container is started.

2. Build the application. The build result is in `build/libs/micro-inventory-0.0.1.jar`. This is the file that is used by the microservice. 

        # cd refarch-cloudnative-micro-inventory
        # ./gradlew build

3. Look at the Dockerfile.
 
        # vi docker/Dockerfile
![](exercises/024-inv-dockerfile.png)
   The Dockerfile is similar to the eureka or zuul with the addition of installation of the New Relic agent. New Relic is a Java performance monitoring with low overhead. 
 
4. Build the docker container. 

        # cp build/libs/micro-inventory-0.0.1.jar docker/app.jar
        # cd docker
        # docker build -t cloudnative/inventoryservice . 

3. Tag and push the local docker image to bluemix private registry.

        # docker tag cloudnative/inventoryservice registry.ng.bluemix.net/$(cf ic namespace get)/inventoryservice-${SUFFIX}
        # docker push registry.ng.bluemix.net/$(cf ic namespace get)/inventoryservice-${SUFFIX}

4. Get private IP address of the database container.

        # cf ic inspect mysql-${SUFFIX} | grep -i ipaddress
    
5. Start the application in IBM Bluemix container. Replace `{ipaddr-db-container}` with private IP address of the database container, `{dbuser}` with database user name and `{password}` with database user password.

        # cf ic group create -p 8080 -m 256 --min 1 --desired 1 \
         --auto --name micro-inventory-group-${SUFFIX} \
         -e "spring.datasource.url=jdbc:mysql://${ipaddr}:3306/inventorydb" \
         -e "eureka.client.serviceUrl.defaultZone=http://netflix-eureka-${SUFFIX}.mybluemix.net/eureka/" \
         -e "spring.datasource.username=dbuser" \
         -e "spring.datasource.password=Pass4dbUs3R" \
         -n inventoryservice-${SUFFIX} -d mybluemix.net \
         registry.ng.bluemix.net/$(cf ic namespace get)/inventoryservice-${SUFFIX}

4. Validate the inventory service.

        # curl http://inventoryservice-${SUFFIX}.mybluemix.net/micro/inventory/13402
![](exercises/028-inv-curl-1.png)

     
## Exercise 2: Looking at integration to Eureka

This exercise explores the integration of the microservices to Eureka and Zuul as the OSS. 

1. Open the Eureka interface for the registered endpoints. Go to `http://netflix-eureka-${SUFFIX}.mybluemix.net/` and check the registered application. 
![inventory eureka](exercises/023-inv-eureka.png)
2. Check access to inventory API from zuul. Now that we see that eureka recognize the application, and the application itself is active; you can try to access the application from zuul. Go to `http://netflix-zuul-${SUFFIX}.mybluemix.net/inventory-microservice/micro/inventory/13412` and check whether you get the same result as before.
![](exercises/029-inv-curl-2.png)


## Exercise 3: Understanding Spring framework Java program

1. Looking at the Java program using Spring framework. The source is located under src/main/java. The application uses Java package of inventory.mysql and has the structure similar to:
![applClasses](exercises/025-inv-applstructure.png)
   - Application: the main program that loads spring
   - InventoryController: the logic for URL mappings
   - models/IInventoryRepo: uses CrudController that allows encapsulation of data into API
   - models/Inventory: class that is used to map individual data item's fields
2. The main logic that controls the API is provided in InventoryController.java. 
![apilogic](exercises/026-inv-logic.png)
   The interface is quite simple. Based on the prefix in the application.yml (`/micro`) and he @RequestMapping directive, you can see that 
`http://inventoryservice-${SUFFIX}.mybluemix.net/micro/check` will give `It works!` as its reply.
![](exercises/027-inv-check.png) 