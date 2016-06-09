# OpenShift FIS Microservice *ose-fis-jms-tx*
**Important Note:** This project assumes the readers have a basic working knowledge of *Red Hat OpenShift Enterprise v3.1/v3.2* (or upstream project -> OpenShift Origin) & are familiar with the underlying framework components such as Docker & Kubernetes.  Readers are also advised to familiarize themselves with the *Kubernetes* API object model (high level) before beginning to work on this *microservice* implementation.  For quick reference, links to a couple of useful on-line resources are listed below.

1.  [OpenShift Enterprise Documentation](https://docs.openshift.com/)
2.  [Kubernetes Documentation](http://kubernetes.io/docs/user-guide/pods/)

This project uses OpenShift FIS (Fuse Integration Services) tools and explains how to develop, build and deploy Apache Camel based microservices in OpenShift Enterprise v3.1/v3.2.

For building Apache Camel applications within Docker containers and then deploying the resulting container images onto OpenShift, developers can take two different approaches or paths.  The steps outlined here use approach # 1 (see below) in order to build and deploy this microservice application.

1.  **S2I (Source to Image) Workflow** : Using this path, a user generates a template object definition (TOD) using the fabric8 Maven plug-in which is included in the OpenShift FIS tools package.  The TOD contains a list of kubernetes objects and also includes info. on the S2I image (builder image) which will be used to build the container image containing the camel application binaries along with the respective run-time (Fuse or Camel).  To learn more about FIS for OpenShift or types of runtimes an Apache Camel application can be deployed to, refer to this [blog] (http://blog.christianposta.com/cloud-native-camel-riding-with-jboss-fuse-and-openshift/) 
2.  **Apache Maven Workflow** : Using this path, the developer uses fabric8 Maven plug-in(s) to build the Apache Camel application, generate the docker image containing both the compiled application binary & the run-time, push the docker image to the registry & lastly generate the TOD containing the list of kubernetes objects necessary to deploy the application to OpenShift.  For more detailed info. on this workflow & steps for deploying a sample application using this workflow, please refer to this GitHub project <https://github.com/RedHatWorkshops/rider-auto-openshift>

## Description
This project buids upon the OpenShift concepts discussed in the GitHub project titled [ose-fis-auto-dealer](https://github.com/ganrad/ose-fis-auto-dealer).  Additionally, this project presumes the readers have gone thru the *ose-fis-auto-dealer* project and successfully deployed the respective artifacts (*microservices*) to OpenShift. 

This project examines and demonstrates the following OpenShift Enterprise / FIS features that are essential for building a highly performant, reliable and scalable integration application.

1.  **S2I Workflow**: Accelerate the development, testing and deployment of integration applications with OpenShift xPaaS.
2.  **Reliability**: Use of a transaction manager (Spring PTM) for guaranteed delivery of messages between source and target systems.
3.  **Stability**: Use of proven, tried and true enterprise integration patterns (EIPs) for building sophisticated integration applications using Apache Camel (included in FIS).
4.  **High Performance**: Use Spring-Boot to instantiate & run the Camel application natively in the JVM.  No application server or run-time is required to run the application.  This results in an extremely light weight application thereby increasing the application throughput and run-time performance.  
5.  **Scalability**: Build stateless microservices and deploy them on OpenShift.  Leverage OpenShift's built-in scaling feature to dynamically scale the application in order to match the data processing workload (scale up or down).
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

![alt tag](https://raw.githubusercontent.com/ganrad/ose-fis-jms-tx/master/images/description.jpg)
