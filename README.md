# quarkus-cheatsheet
This repository lists some of the useful features &amp; commands used while working with RH Quarkus.

### What is Quarkus?
Quarkus is Kubernetes Native java stack crafted from the best of breed Java libraries and standards. It is a part of Red Hat runtimes used for building fast, lightweight microservices, and serverless applications. Also focused on developer experience, making things just work with little to no configuration and allowing to do live coding.

We have listed some of the very useful features in Quarkus. This cheat-sheet is tested with Quarkus 1.8.1.Final.

### Architecture
In this guide, we will create a straightforward application serving a hello endpoint. To demonstrate dependency injection, this endpoint uses a greeting bean. 
![Component Workflow](getting-started-architecture.png "Component Workflow")

### Prerequisites
To complete this guide, you need:

* Java development kit (JDK 11+)
* Apache Maven 3.6.2+

Follow the link below to install the same.

```
https://tecadmin.net/install-apache-maven-on-centos/
```

### Creating new project namespace

```
oc create ns quarkus-meetup
oc project quarkus-meetup
```

### Bootstrapping of project 
We will create a basic restful java application, which will use the quarkus maven plugin and generate a basic project.

```
mvn io.quarkus:quarkus-maven-plugin:1.8.1.Final:create  -DprojectGroupId=org.acme  -DprojectArtifactId=getting-started  -DplatformVersion=1.8.1.Final  -DclassName="org.acme.quickstart.GreetingResource"   -Dpath="/hello" -X
```

### Development mode (live coding)
This mode enabled hot deployment with background compilation, which means that when you modify your Java files and/or your resource files and refresh your browser, these changes will automatically take effect.

First change the directory in which the project was created.

```
cd getting-started
```

Now we are ready to run our application. 

```
./mvnw compile quarkus:dev
```

Once started, you can request the provided endpoint in the browser using the links below. Please change the IP's as per your infrastructure node IP.

```
http://<INFRA_NODE_IP>:8080/hello
```

Now, let's exercise the live reload capabilities of Quarkus. Please open the below endpoint and change return "hello"; to return "quarkus-meetup"; on line 14 in the editor. Don't recompile or restart anything. Just try to reload the browser. You should see the updated quarkus-meetup message.

```
vim src/main/java/org/acme/quickstart/GreetingResource.java
http://<INFRA_NODE_IP>:8080/hello
```

### Injection (Arc extension)

As a developer, you may want to use this feature to test your customized rest end points. Quarkus is based on ArC (advanced rest client) which is a CDI-based dependency injection env for you, with powerful features like live coding etc.

Let's modify the application and add a companion bean. Create the file with the following content.

```
vim src/main/java/org/acme/quickstart/GreetingService.java

package org.acme.quickstart;

import javax.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class GreetingService {

    private String hostname = System.getenv().getOrDefault("HOSTNAME", "unknown");

    public String greeting(String name) {
        return "hello " + name + " from " + hostname;
    }

}
```

Now, let's open the GreetingResource class to inject the new bean and create a new endpoint using it.

```
vim  src/main/java/org/acme/quickstart/GreetingResource.java

package org.acme.quickstart;

import javax.inject.Inject;
import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.PathParam;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;

@Path("/hello")
public class GreetingResource {

    @Inject
    GreetingService service;

    @GET
    @Produces(MediaType.TEXT_PLAIN)
    @Path("/greeting/{name}")
    public String greeting(@PathParam("name") String name) {
        return service.greeting(name);
    }

    @GET
    @Produces(MediaType.TEXT_PLAIN)
    public String hello() {
        return "hello";
    }
}
```

Inspect the results.

```
http://<INFRA_NODE_IP>:8080/hello 
http://<INFRA_NODE_IP>:8080/hello/greeting/quarkus
http://<INFRA_NODE_IP>:8080/hello/greeting/quarkus-meetup
```

### Application Packaging
In this step, we will package our application and run it as a self-contained jar file.

```
mvn package -DskipTests
java -jar target/getting-started-1.0-SNAPSHOT-runner.jar
```

Inspect the results.

```
http://<INFRA_NODE_IP>:8080/hello
http://<INFRA_NODE_IP>:8080/hello/greeting/quarkus
http://<INFRA_NODE_IP>:8080/hello/greeting/quarkus-meetup
```

### OpenShift Extension
Now that we have our app built, let's move it into containers and into the cloud. Quarkus offers the ability to automatically generate OpenShift resources based on the user supplied configuration.

The OpenShift extension is actually a wrapper extension that brings together the kubernetes and container-image-s2i extensions with defaults so that it’s easier for the user to get started with Quarkus on OpenShift. Run the following commands to add it to our project.

```
mvn quarkus:add-extension -Dextensions="openshift"
mvn quarkus:add-extension -Dextensions="container-image-s2i"
```

**S2I extension uses the S2I binary builds in order to perform container builds inside the cluster. S2I requires a buildconfig and 2 image streams, one for builder image and one for output image. The creation of such objects is being taken care by quarkus kubernetes extensions.**

Run the following command which will build and deploy the Quarkus native application using the OpenShift extension.

```
mvn clean package \
-Dquarkus.kubernetes-client.trust-certs=true \
-Dquarkus.container-image.build=true \
-Dquarkus.kubernetes.deploy=true \
-Dquarkus.kubernetes.deployment-target=openshift \
-Dquarkus.openshift.expose=true \
-Dquarkus.openshift.labels.app.openshift.io/runtime=java
```

Once the deployment is finished with "BUILD SUCCESS" message, make sure it's actually done rolling out.

```
oc rollout status -w dc/getting-started
oc get is -n quarkus-meetup
oc get pods
oc get route
curl <ROUTE_URL>
curl <ROUTE_URL>/hello/greeting/quarkus-on-openshift
```

You can also see the app deployed in the OpenShift Developer Topology.

```
<CONSOLE_URL>/topology/ns/quarkus-meetup/graph
```

### Application scaling
With that set, let's see how fast our app can scale up to 10 instances.

```
oc scale --replicas=10 dc/getting-started
oc get dc
oc get pods
```

Back in the OpenShift Developer Topology you'll see the app scaling dynamically upto 10 pods.

### Traffic load Balancing
Now that we have 10 pods running, let's hit it with some load.

```
oc get route
for i in {1..10} ; do curl \
<ROUTE_URL>/hello/greeting/quarkus-on-openshift ; echo ""; sleep .05 ; done
```

