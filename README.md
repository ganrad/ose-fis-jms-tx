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
2.  **Reliability**: Use of a transaction manager (Spring PTM) for guaranteed delivery of messages between source and target systems.  Data integrity and consistency are key requirements for almost all enterprise applications.  To ensure data integrity and consistency, the application has to read/write messages from/to source/destination systems under the control of a transaction manager (TM). The TM will ensure the reads and writes are performed in an atomic fashion and will ensure data read/persisted across all applications involved in the transaction is consistent.   The TM will also notify the message broker (ActiveMQ or other JMS provider) when the messages cannot be delivered to the recipient system to allow re-delivery of the messages at a later time.  This ensures data doesn't get lost in-flight between the source and target systems and the data is always consistent across all systems participating in the transaction. 
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

Refer to the Apache ActiveMQ website [documentation] (http://activemq.apache.org/getting-started.html) for installing and configuring an ActiveMQ messaging server / broker.  Note down the server hostname (or IP Address) and listener port number (default: 61616) as we will need to provide these values while configuring the microservice (Step B).

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
  * Use the OpenShift Web UI to create the application.  Click 'Add to Project' button at the top.  Next, enter 'fis' in the 'Select Image or Template' text field.  Then select the 'fis-jms-tx-template.  See the screenshot below.
  
  ![alt tag]()
  * On the next screen, specify values for GIT_REPO, JMS_HOST and JMS_PORT application parameters. See screenshot below.
  
  ![alt tag]()

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
  
  ![alt tag]()
8.  Open a command line window and tail the output from the application Pod.
   
   ```
   $ oc get pods
   $ oc log pod -f <pod name>
   ```
   Substitute the name of your Pod in the command above.  

### B] Test *ose-fis-jms-tx* microservice
**NOTE:** The microservice [ose-fis-auto-dealer](https://github.com/ganrad/ose-fis-auto-dealer) should have been deployed to OpenShift and the corresponding Pods for the microservice and backend datastore (MongoDB) should be running.  The *ose-fis-auto-dealer* microservice exposes REST API service end-points which will be consumed by this microservice.

1.  Create and save a few XML Vehicle data files (*'batch update'*) into a working directory preferably on the server/machine where your Apache ActiveMQ server is deployed/running.  A sample XML data file is provided in the *'data'* directory of this project. 
2.  Use the Apache ActiveMQ Admin web application and login using admin credentials.  See Admin URL below.  Replace *'<host>'* with the hostname/IP address of the ActiveMQ server.

  ```
  ActiveMQ Admin URL : https://<host>:8161/admin
  ```
3.  On the admin web page, click on the link *'Send'*.   In the next page, specify **'vehiQ'** as the *'Queue'* name in the *Destination* field.  Copy the XML message you saved in Step 1 into the *'Message body'* text box below.  Then click on *'Send'*.  See screenshot below.

  ![alt tag](https://raw.githubusercontent.com/ganrad/ose-fis-jms-tx/master/images/amq-send.png)
  
4.  The Vehicle (*Batch* update) XML message should be immediately read from the queue by this microservice.  This message would then be split into individual XML messages and each message would be sent to the respective REST service end-point.  You should be able to view Http response code returned by the REST service end-points in the command window as shown below.

   ```
   2016-05-17 22:52:08,531 [e://target/data] INFO  readVehicleFiles               - Read Vehicle Data File : /deployments/target/data/vn01.xml
   <?xml version="1.0"?>
   <vehicle>
	      <vehicleId>001</vehicleId>
	      <make>Honda</make>
	      <model>Civic</model>
	      <type>LX</type>
	      <year>2016</year>
	      <price>18999</price>
	      <inventoryCount>2</inventoryCount>
   </vehicle>
   ```
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
  2016-05-17 22:53:24,788 [tp1244815033-20] INFO  getVehicle                     - {
  "vehicleId" : "001",
  "make" : "Honda",
  "model" : "Civic",
  "type" : "LX",
  "year" : "2016",
  "price" : 18999,
  "inventoryCount" : 2
}
  ```
  * REST API response shown in browser window below.
  
  ![alt tag](https://raw.githubusercontent.com/ganrad/ose-fis-auto-dealer/master/images/results01.png)
