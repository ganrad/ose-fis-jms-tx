# OpenShift FIS Microservice *ose-fis-jms-tx*
**Important Note:** This project assumes the readers have a basic working knowledge of *Red Hat OpenShift Enterprise v3.1/v3.2* (or upstream project -> OpenShift Origin) & are familiar with the underlying framework components such as Docker & Kubernetes.  Readers are also advised to familiarize themselves with the *Kubernetes* API object model (high level) before beginning to work on this *microservice* implementation.  For quick reference, links to a couple of useful on-line resources are listed below.

1.  [OpenShift Enterprise Documentation](https://docs.openshift.com/)
2.  [Kubernetes Documentation](http://kubernetes.io/docs/user-guide/pods/)

This project uses OpenShift FIS (Fuse Integration Services) tools and explains how to develop, build and deploy Apache Camel based microservices in OpenShift Enterprise v3.1/v3.2.

<div id="deploy"/>
For building Apache Camel applications within Docker containers and then deploying the resulting container images onto OpenShift, developers can take two different approaches or paths.  The steps outlined here use approach # 1 (see below) in order to build and deploy this microservice application.

1.  **S2I (Source to Image) Workflow** : Using this path, a user generates a template object definition (TOD) using the fabric8 Maven plug-in which is included in the OpenShift FIS tools package.  The TOD contains a list of kubernetes objects and also includes info. on the S2I image (builder image) which will be used to build the container image containing the camel application binaries along with the respective run-time (Fuse or Camel).  To learn more about FIS for OpenShift or types of runtimes an Apache Camel application can be deployed to, refer to this [blog] (http://blog.christianposta.com/cloud-native-camel-riding-with-jboss-fuse-and-openshift/) 
2.  **Apache Maven Workflow** : Using this path, the developer uses fabric8 Maven plug-in(s) to build the Apache Camel application, generate the docker image containing both the compiled application binary & the run-time, push the docker image to the registry & lastly generate the TOD containing the list of kubernetes objects necessary to deploy the application to OpenShift.  For more detailed info. on this workflow & steps for deploying a sample application using this workflow, please refer to this GitHub project <https://github.com/RedHatWorkshops/rider-auto-openshift>

## Description
This project buids upon the OpenShift concepts discussed in the GitHub project titled [ose-fis-auto-dealer](https://github.com/ganrad/ose-fis-auto-dealer).  Additionally, this project presumes the readers have gone thru the *ose-fis-auto-dealer* project and successfully deployed the microservice to OpenShift. 

This project examines and demonstrates the following OpenShift Enterprise / FIS features that are essential for building a highly performant, reliable and scalable integration application.

1.  **S2I Workflow**: Accelerate the development, testing and deployment of integration applications with OpenShift xPaaS.
2.  **Reliability**: Use of a transaction manager (Spring PTM) for guaranteed delivery of messages between source and target systems.  Data integrity and consistency are key requirements for almost all enterprise applications.  To ensure data integrity and consistency, the application has to perform all read/write operations within a *transaction* under the control of a transaction manager (TM). The TM will ensure all reads and writes within a given transaction are executed atomically and the transaction is either committed or rolled back.  When the messages cannot be delivered to the destination systems, the TM will roll back the transaction to allow re-delivery of the messages at a later time.  This ensures data doesn't get lost in-flight between the source and target systems and the data is always consistent across all systems participating in the transaction. 
3.  **Stability**: Use of proven, tried and true enterprise integration patterns (EIPs) for building sophisticated integration applications using Apache Camel (included in FIS).
4.  **High Performance**: Use of Spring-Boot to instantiate & run the Camel application natively in the JVM.  With Spring-Boot an application server or run-time is not required to run the application.  As a result, the application is extremely light weight and delivers better throughput and run-time performance.  
5.  **Scalability**: Build stateless microservices and deploy them on OpenShift.  Leverage OpenShift's built-in container scaling feature to dynamically scale the application in order to efficiently handle the incoming data processing workload (scale up or down).
6.  **High Availability**: Leverage OpenShift's built-in high availability features to ensure the application is always available and ready to process transactions resulting in zero application downtime.

This microservice is implemented using Apache Camel routes.  At a high level, the Camel routes execute the following sequence of steps (see description and diagram below):

1. Read XML messages from a JMS Queue using a local transaction manager.  See screenshot below for the structure of the XML message.

  ```
  <?xml version="1.0" encoding="UTF-8"?>
  <msgEnvelope>
    <header>
      <batchId>1001</batchId>
      <contType>List</contType>
      <msgType>Vehicle</msgType>
      <msgVersion>001</msgVersion>
    </header>
    <body>
      <items>
        <vehicle>
          <vehicleId>001</vehicleId>
          <make>Honda</make>
          <model>Civic</model>
          <type>LX</type>
          <year>2016</year>
          <price>18999</price>
          <inventoryCount>2</inventoryCount>
          <opType>Create</opType>
        </vehicle>
        <vehicle>
          <vehicleId>001</vehicleId>
          <make>Honda</make>
          <model>Civic</model>
          <type>LX</type>
          <year>2016</year>
          <price>19099</price>
          <inventoryCount>2</inventoryCount>
          <opType>Update</opType>
        </vehicle>
        ......
      </items>
    </body>
  </msgEnvelope>
  ```
2. Uses [content based router](http://camel.apache.org/content-based-router.html) EIP to determine the type of the message (‘Vehicle’) and send it to its intended destination.
3. Uses a [splitter](http://camel.apache.org/splitter.html) EIP to split the inbound message into multiple outbound messages.
4. Uses an XSLT stylesheet to transform the inbound message format to the outbound message format as expected by the target microservice.
5. Makes REST (Http) API calls to transmit the XML messages to the corresponding target service end-point.

![alt tag](https://raw.githubusercontent.com/ganrad/ose-fis-jms-tx/master/images/description.png)

## Steps for deploying **ose-fis-jms-tx** microservice on OpenShift Enterprise v3.1/v3.2

### A] Start Apache ActiveMQ JMS Provider
This project uses the Apache ActiveMQ JMS provider (Broker) for providing reliable messaging & guaranteed delivery of messages between source and target systems.  The instructions outlined here assume the messaging provider is running on a server/node which resides outside the OpenShift Cluster.  This solution can also be easily tailored to use a JMS provider  that is running within the OpenShift cluster ([JBoss A-MQ](http://www.jboss.org/products/amq/overview/) within a Docker container). 

Refer to the Apache ActiveMQ website [documentation] (http://activemq.apache.org/getting-started.html) for installing and configuring an ActiveMQ messaging server / broker.  Note down the server hostname (or IP Address) and listener port number (default: 61616) as we will need to provide these values while configuring the microservice in Step B.

1. Specify correct values for properties *jms.user* and *jms.password* in the file *src/main/resources/jms.properties*.  These properties are used by the Camel application (microservice) to authenticate against the Apache ActiveMQ server.  The default values for these properties in the *jms.properties* file is *admin*. 

### B] Deploy *ose-fis-jms-tx* microservice
The steps listed below for building and deploying the microservice application follows approach <a href="#deploy"># 1</a> described above, the S2I workflow.

1.  Fork this repository so that it gets added to your GitHub account.
2.  Download the template file (template object definition) into your OpenShift master node.
  * On the GitHub project (browser) click on *kube-template.json*, then click on *Raw*.  Copy the http URL and use the CURL command to download the template file to your OpenShift master node (OR to the server where you have installed OpenShift client tools). Use the command below to download the template file.
  
  ```
  $ curl https://raw.githubusercontent.com/<your GIT account name>/ose-fis-auto-dealer/master/kube-template.json > kube-template.json
  ```
3.  Import the template file into your project 
  * Alternatively, you can import the template to the 'openshift' project using '-n openshift' option.  This would give all OpenShift users access to this template definition.
  
  ```
  $ oc create -f kube-template.json
  ```
  * To view all application templates in your current project
  ```
  $ oc get templates
  ```
4.  Using a web browser, login to OpenShift using the Web UI - 

  ```
  https://<Host name or IP address of OSE master node>:8443/
  ```

  Or Alternatively use the command line interface (CLI).

  ```
  $ oc login -u user -p password  
  ```
  
5.  Select the *fis-apps* OpenShift project which you created earlier in this [GitHub project] (https://github.com/ganrad/ose-fis-auto-dealer).  Alternatively, if you are using the CLI, use the command shown below to switch to *fis-apps* project.

  ```
  $ oc project fis-apps
  ```

6.  Create the microservice application
  * This command kicks off the S2I build process in OpenShift.
  * Use the OpenShift Web UI to create the application.  Click 'Add to Project' button at the top.  Next, enter 'fis' in the 'Select Image or Template' text field.  Then select the *'fis-jms-tx-template'*.  See the screenshot below.
  
  ![alt tag](https://raw.githubusercontent.com/ganrad/ose-fis-jms-tx/master/images/sel-template.png)
  * On the next screen, specify values for GIT_REPO, JMS_HOST and JMS_PORT application parameters as they apply to your environment. See screenshot below.
  
  ![alt tag](https://raw.githubusercontent.com/ganrad/ose-fis-jms-tx/master/images/app-param.png)

  **GIT_REPO:** The GIT Repository URL.  Remember to substitute your GIT account *user name* in the URL.  
  **JMS_HOST:** The JMS provider host name or IP address.  
  **JMS_PORT:** The JMS provider listener port number.
  * Alternatively, if you are using the CLI, use the command below to create the application.
  ```
  $ oc new-app --template=fis-jms-tx-template --param=GIT_REPO=https://github.com/<your GIT account username>/ose-fis-jms-tx.git,JMS_HOST=<JMS Provider Host>,JMS_PORT=<JMS Provider Port>
  ```

7.  Use the commands below to check the status of the build and deployment 
  * The build (Maven) might take a while (approx. 10-20 mins) to download all dependencies, build the code and then push the image into the integrated Docker registry.
  * Once the build completes and the image is pushed into the registry, the deployment process would start.
  * Check the builds
  ```
  $ oc get builds
  ```
  * Stream/view the build logs. Substitute the name of the build in the command below.
  ```
  $ oc logs -f <build name | build pod name>
  ```
  * Check the deployment
  ```
  $ oc get dc
  ```
  * Check the pods.  After the deployment pod completes, the application pod should show a status of *running*.
  ```
  $ oc get pods
  ```
  * At this point, you should have successfully built an Apache Camel based JMS microservice using OpenShift FIS tooling and deployed the same to OpenShift PaaS!
  
  ![alt tag](https://raw.githubusercontent.com/ganrad/ose-fis-jms-tx/master/images/pod-status.png)
8.  Open separate command line windows and tail the output from the both the application (microservice) Pods.
   
   ```
   $ oc get pods
   $ oc log pod -f <pod name>
   ```
   Substitute the name of the Pods in the command above (as applicable).  

### B] Test *ose-fis-jms-tx* microservice
**NOTE:** The microservice [ose-fis-auto-dealer](https://github.com/ganrad/ose-fis-auto-dealer) should have been deployed to OpenShift and the corresponding Pods for the microservice and backend datastore (MongoDB) should be running.  The *ose-fis-auto-dealer* microservice exposes REST API service end-points which will be consumed by this microservice.

1.  Create and save a few XML Vehicle data files (*'batch update'*) into a working directory preferably on the server/machine where your Apache ActiveMQ server is deployed/running.  A sample XML data file is provided in the *'data'* directory of this project. 
2.  Use the Apache ActiveMQ Admin web application and login using admin credentials.  See Admin URL below.  Replace *'<host>'* with the hostname/IP address of the ActiveMQ server.

  ```
  ActiveMQ Admin URL : https://<host>:8161/admin
  ```
3.  On the admin web page, click on the link *'Send'*.   In the next page, specify **'vehiQ'** as the *'Queue'* name in the *Destination* field.  Copy the XML message you saved in Step 1 into the *'Message body'* text box below.  Then click on *'Send'*.  See screenshot below.

  ![alt tag](https://raw.githubusercontent.com/ganrad/ose-fis-jms-tx/master/images/amq-send.png)
  
4.  The Vehicle (*Batch* update) XML message should be immediately read from the queue by this microservice.  This message would then be split into individual XML messages and each message would be sent to the respective REST service API end-point.
  * You should be able to view Http response code returned by the REST service API end-points in the Pod command window.  Output from the *ose-fis-jms-tx* microservice (Pod) is shown below.

   ```
   2016-06-11 15:29:27 INFO  postMessages:96 - HTTP response code: 200
   2016-06-11 15:29:27 INFO  postMessages:96 - End
   2016-06-11 15:29:27 INFO  postMessages:96 - Begin:
   Operation: Delete
   XML message:
   <?xml version="1.0" encoding="UTF-8"?><vehicle>
        <vehicleId>020</vehicleId>
        <make>Toyota</make>
        <model>Tacoma</model>
        <type>4x4 LE</type>
        <year>2016</year>
        <price>32000</price>
        <inventoryCount>2</inventoryCount>
        
      </vehicle>
   2016-06-11 15:29:27 INFO  postMessages:96 - HTTP response code: 200
   2016-06-11 15:29:27 INFO  postMessages:96 - End
   2016-06-11 15:29:27 INFO  processMessages:96 - End
   2016-06-11 15:29:27 INFO  getVehicleDataMsgs:96 - End

   ```
   * Output from the *ose-fis-auto-dealer* microservice (Pod) is shown below.
   ```
   2016-06-11 15:29:27,351 [tp1620112330-25] INFO  storeVehicle                   - HTTP request body:
   <?xml version="1.0" encoding="UTF-8"?><vehicle>
        <vehicleId>008</vehicleId>
        <make>Toyota</make>
        <model>Tacoma</model>
        <type>4x4 LE</type>
        <year>2016</year>
        <price>32000</price>
        <inventoryCount>2</inventoryCount>
        
      </vehicle>
   2016-06-11 15:29:27,351 [tp1620112330-25] INFO  storeVehicle                   - Database:test, Collection:ose
   2016-06-11 15:29:27,352 [tp1620112330-25] INFO  storeVehicle                   - Insert in mongodb collection successful!
   2016-06-11 15:29:27,365 [tp1620112330-23] INFO  deleteVehicle                  - Mongodb delete query: {"vehicleId":"020"}
   2016-06-11 15:29:27,366 [tp1620112330-23] INFO  deleteVehicle                  - Mongodb response: Record delete count=0
   ```
   * Use the OpenShift Web Console to login into the MongoDB Pod and verify that the records have been persisted in the data store.  If you dropped the sample XML file (from *data* directory) into the ActiveMQ Queue then there should be 10 documents in the MongoDB collection *ose* within the database. See screenshot below.
   
   ![alt tag](https://raw.githubusercontent.com/ganrad/ose-fis-jms-tx/master/images/db-verify.png)
   *  Lastly, use the ActiveMQ admin console to verify the message has been consumed (enqueued/dequeued) from the *'vehiQ'* Queue.  See screenshot below.
   ![alt tag](https://raw.githubusercontent.com/ganrad/ose-fis-jms-tx/master/images/amq-verify.png)

5.  Access the Http REST service end-points using your browser.  Substitute the correct values for route name, project name (fis-apps) and 
openshift domain name as they apply to your OpenShift environment.  You will also have to substitute values for URL parameters (excluding { } in URL's below) when issuing the corresponding GET/POST/DELETE Http calls.  All Http REST API calls return data in JSON format.
  * Retrieve vehicle info. by ID or by price range (Http GET) : 
  
  ```
  http://route name-project name.openshift domain name/AutoDMS/vehicle/{vehicleid}
  http://route name-project name.openshift domain name/AutoDMS/vehicle/pricerange/{minprice}/{maxprice}
  ```
  * Store (Create or Update) vehicle info. (Http POST) :
  
  ```
  http://route name-project name.openshift domain name/AutoDMS/vehicle
  http://route name-project name.openshift domain name/AutoDMS/vehicle/{vehicleid}
  ```
  * Delete vehicle info. (Http DELETE) :
  
  ```
  http://route name-project name.openshift domain name/AutoDMS/vehicle/{vehicleid}
  ```
  
6.  You can view the REST API responses in the Pod output / command window as shown below.

  ```
  2016-06-11 15:46:08,129 [tp1620112330-22] INFO  getVehicle                     - Mongodb response: {
  "_id" : {
    "time" : 1465673367000,
    "timestamp" : 1465673367,
    "date" : 1465673367000,
    "inc" : 29617468,
    "timeSecond" : 1465673367,
    "machine" : -458177733,
    "new" : false
  },
  "vehicleId" : "005",
  "make" : "Toyota",
  "model" : "Camry",
  "type" : "LE",
  "year" : "2016",
  "price" : 22000,
  "inventoryCount" : 4
}
  ```
  * REST API response (in browser window) is shown below.
  
  ![alt tag](https://raw.githubusercontent.com/ganrad/ose-fis-jms-tx/master/images/api-get.png)
