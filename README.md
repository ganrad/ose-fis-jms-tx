# OpenShift FIS Microservice *ose-fis-jms-tx*
**Important Note:** This project assumes the readers have a basic working knowledge of *Red Hat OpenShift Enterprise v3.1/v3.2* (or upstream project -> OpenShift Origin) & are familiar with the underlying framework components such as Docker & Kubernetes.  Readers are also advised to familiarize themselves with the *Kubernetes* API object model (high level) before beginning to work on this *microservice* implementation.  For quick reference, links to a couple of useful on-line resources are listed below.

1.  [OpenShift Enterprise Documentation](https://docs.openshift.com/)
2.  [Kubernetes Documentation](http://kubernetes.io/docs/user-guide/pods/)

This project uses OpenShift FIS (Fuse Integration Services) tools and explains how to develop, build and deploy Apache Camel based microservices in OpenShift Enterprise v3.1/v3.2.

For building Apache Camel applications within Docker containers and then deploying the resulting container images onto OpenShift, developers can take two different approaches or paths.  The steps outlined here use approach # 1 (see below) in order to build and deploy this microservice application.

1.  S2I (Source to Image) workflow : Using this path, a user generates a template object definition (TOD) using the fabric8 Maven plug-in which is included in the OpenShift FIS tools package.  The TOD contains a list of kubernetes objects and also includes info. on the S2I image (builder image) which will be used to build the container image containing the camel application binaries along with the respective run-time (Fuse or Camel).  To learn more about FIS for OpenShift or types of runtimes an Apache Camel application can be deployed to, refer to this [blog] (http://blog.christianposta.com/cloud-native-camel-riding-with-jboss-fuse-and-openshift/) 
2.  Apache Maven Workflow : Using this path, the developer uses fabric8 Maven plug-in(s) to build the Apache Camel application, generate the docker image containing both the compiled application binary & the run-time, push the docker image to the registry & lastly generate the TOD containing the list of kubernetes objects necessary to deploy the application to OpenShift.  For more detailed info. on this workflow & steps for deploying a sample application using this workflow, please refer to this GitHub project <https://github.com/RedHatWorkshops/rider-auto-openshift>

## Description
This microservice 
