# Deploy with Fuse Fabric
JBoss Fuse has a fantastic management capability leveraging Fuse Fabric. Please take a look at the documentation on RedHat.com for more detail about Fuse Fabric, but for a quick bullet point summary, here's what fabric is/does:

Fabric is a microservices deployment/management platform for JBoss Fuse. It gives the following functionality:

* Centralized configuration management
* Service/module versioning
* Rolling deployments/rollbacks
* Audit capabilities of what changed in the environment
* Service discover
* Load balancing
* Failover
* Master/Slave elections
* more......


## Getting started with Fabric
Before we let Fuse Fabric manage our deployments, let's first create a Fabric container. If you don't have fuse running yet, navigate to the root directory and run:

    ./bin/fuse

You will notice that shell prompts you to create a user with the message:

```No user found in etc/users.properties. Please use the 'esb:create-admin-user' command to create one.```

Run the ```esb:create-admin-user``` command and create the user ```admin``` with password ```admin```.

    
After this we will create the fabric, to this by running this command:

    JBossFuse:karaf@root> fabric:create --clean --wait-for-provisioning --profile fabric
    
Wait a few moments for the JVM to transform into a fabric container. Once that's happened, navigate over to `http://localhost:8181` and login with `admin/admin`. You should see a screen similar to this (click "containers" tab if you're greeted with the welcome page first):

![Fabric welcome](images/fabric-welcome.png)

> Note, the intention isn't to go super deep with what Fabric is or how it works. Feel free to tweet me `@christianposta` or open an issue for this repo with the detail you feel would be useful to make the flow of this POC work better


Click on the `Wiki` tab which will show you a list of `profiles` or "applications" that we can deploy:

![Fabric welcome](images/fabric-profiles.png)

Feel free to poke around and see what default applications/profiles exist out of the box.

## Deploying ActiveMQ
You should see a screen similar to this:

![Fabric welcome](images/fabric-mq-services.png)

What we want to do is create a master/slave deployemnt of ActiveMQ so that if a broker fails, the passive broker will take over from where the failed one left off. This gives us message-level guarantees, ie, that if a message gets stored to disk, it will not get lost (assuming no SPOF of disk) and will _eventually_ get delivered (note, there are some important implications of that statement). 

So click on the `Broker` button to configure a new set of ActiveMQ profiles:

![Fabric welcome](images/fabric-mq-config.png)

Click the Create button and you should see the new broker profiles in the UI. 

### Configure A-MQ Port
By default, A-MQ will use a randomly chosen port, however the healthcare-poc applications are configured to look for A-MQ on port 61616. In order for the application to work correctly, we need to configure A-MQ to use a statically assigned port. To do this, click on the healthbrokers link outlined in red as shown below:

![Fabric mq configure port 1](images/fabric-mq-port-1.png)

This will bring you to the configuration page shown below. Click the Edit button in the upper right and then add the openwire-port property as shown outlined in red below.

![Fabric mq configure port 2](images/fabric-mq-port-2.png)

### Create A-MQ Containers
One other thing to note is that the new broker profiles we created have a few red icons indicating that the profiles are not in a valid state. Hover over the red icons and you'll see why:

![Fabric welcome](images/fabric-invalid-mq.png)

Go ahead and click those red icons which should take you to a "Create New Container" page. This will enable you to create new Fuse JVMs that have the new `healthbrokers` ActiveMQ profiles assigned. Give the containers a name and click "create and start container":
 
![Fabric mq brokers containers](images/fabric-create-mq-containers.png)

Now we should have 3 Fuse JVMs running. We should have the `root` container as well as the new `healthbroker` containers running. The `healthbroker` containers have ActiveMQ running in them specifically set up in a master-slave mode. 

![Fabric mq brokers containers](images/fabric-running-brokers.png)

### Turning on Virtual Topics
ActiveMQ has a wonderful feature called [Virtual Topics](http://activemq.apache.org/virtual-destinations.html). This feature allows you to do pub-sub broadcast semantics but back the subscriptions with queues. This has many benefits, but chief among them, allows to loadbalance against a single stream of messages (with durable subscriptions, you cannot do that). This allows publishers to publish to a topic and consumers consume from the stream as though they were consuming from a queue. 

One usecase I've run into with the healthcare community is the need to manually correct messages at the head of the queue. Oftentimes this behavior would be an anti-pattern as querying the queue and identifying specific messages sounds much more like a database. However, in some use cases it makes sense, and ActiveMQ allows you to do this even if using a pub-sub broadcast model with topics.

Let's configure ActiveMQ for this Virtual Topic behavior. First thing we want to do is find the `mq-base` profile and edit the `broker.xml` file that comes out of the box. This is essentially the same as the `activemq.xml` file you would expect to find in the ActiveMQ broker you know and love. Click on the `Wiki` tab, then `mq`, then `base`. You should be greeted with a screen like this:

![Fabric mq brokers containers](images/fabric-mq-base-profile.png)

Click on the `broker.xml` file which should take you to an screen that displays the config. Click `Edit`. In older versions of ActiveMQ the location of the next set of elements was location dependent, but nowadays you can just put it wheverever you like:


```xml
    <broker xmlns="http://activemq.apache.org/schema/core" brokerName="${broker-name}" dataDirectory="${data}" start="false" restartAllowed="false">
    ...
    <destinationInterceptors>
      <virtualDestinationInterceptor>
        <virtualDestinations>
          <virtualTopic name=">" selectorAware="false"/>
        </virtualDestinations>
      </virtualDestinationInterceptor>
    </destinationInterceptors>    
    ...
```

Now when a consumer publishes to a topic, e.g., `org.jboss.fuse.event` they can publish to `VirtualTopic.org.jboss.fuse.event` and consumers can listen to a queue named `Consumer.<name>.VirtualTopic.org.jboss.fuse.event` and receive the stream of events. For example, our consumer will listen on `Consumer.1.VirtualTopic.org.jboss.fuse.event`

As soon as you save this configuration, the changes will be propogated to the running brokers and applied in place.

## Deploying our Healthcare POC to Fuse Fabric
We can now deploy our application to fabric using profiles. If you take a look at (for example) the [hl7-ingress/pom.xml](../hl7-ingress/pom.xml), you'll see these two snippets of code;


```xml
      <plugin>
        <groupId>io.fabric8</groupId>
        <artifactId>fabric8-maven-plugin</artifactId>
      </plugin>
```

This is part of the `<build/>` section and declares the [Fabric8 maven plugin](http://fabric8.io/gitbook/mavenPlugin.html). 

> NOTE: you will need to set up the fabric8 maven plugin locally to know exactly where your fabric is running. [Please follow these directions to set it up correctly, including your maven settings.xml](http://fabric8.io/gitbook/mavenPlugin.html)

Also note the `<properties/>` section of the pom.xml:

```xml
    <fabric8.profile>healthcare-poc.hl7ingress</fabric8.profile>
    <fabric8.parentProfiles>feature-camel</fabric8.parentProfiles>
    <fabric8.features>camel-spring camel-hl7 camel-netty4</fabric8.features>
    <fabric8.bundles>mvn:${project.parent.groupId}/${project.artifactId}/${project.parent.version}</fabric8.bundles>
```

These properties define the features and parent profiles needed to implement these profiles.

From the root of the cloned project, run this command:

    mvn clean install -DskipTests -Pfabric fabric8:deploy
    
> NOTE: For JBoss Fuse 6.21, the Maven version must be 3.2.4 or less in order for the fabric8:deploy goal to work as it is not compatible with later versions of Maven.

> NOTE: We skip running the unit tests here since the unit tests start an A-MQ broker on port 61616. Since we already have the Fabric A-MQ running on that port the unit tests will fail. If you want to run the unit tests, shut down the fabric containers first.

This will generate the profiles and upload them to your locally running fabric. Note, if your maven plugin is not configured correctly this will fail. If you have issues find me `@christianposta` on twitter or log an issue in this repo. 

You should see this if built successfully:

    b-services/1.0.0-SNAPSHOT/stub-services-1.0.0-SNAPSHOT.pom
    [INFO] 
    [INFO] <<< fabric8-maven-plugin:1.2.0.redhat-133:deploy (default-cli) @ stub-services <<<
    [INFO] 
    [INFO] --- fabric8-maven-plugin:1.2.0.redhat-133:deploy (default-cli) @ stub-services ---
    [INFO] ------------------------------------------------------------------------
    [INFO] Reactor Summary:
    [INFO] 
    [INFO] Health Care POC: Parent POM ....................... SUCCESS [6.367s]
    [INFO] Health Care POC: Features XML File ................ SUCCESS [5.117s]
    [INFO] Health Care POC: Ingress of HL7 Messages .......... SUCCESS [23.417s]
    [INFO] Health Care POC: Consumer 1 of HL7 Messages ....... SUCCESS [22.810s]
    [INFO] Health Care POC: Transformer 1 .................... SUCCESS [26.200s]
    [INFO] Health Care POC: Stub services .................... SUCCESS [0.705s]
    [INFO] ------------------------------------------------------------------------
    [INFO] BUILD SUCCESS
    [INFO] ------------------------------------------------------------------------
    [INFO] Total time: 1:25.603s
    [INFO] Finished at: Mon Oct 05 19:58:43 MST 2015
    [INFO] Final Memory: 70M/1151M
    [INFO] ------------------------------------------------------------------------
    
Now if you go to the `Wiki` tab again, you should see our new healthcare profiles:

![Fabric mq brokers containers](images/fabric-healthcare-profiels.png)

Now that we have the profiles inside of Fabric, let's create the Ingress routes which listen for HL7 connections do the heavy lifting of receiving the message, decoding it, and putting it into a queue. Click "Create" from the "Containers" tab and fill in the container name and choose the associated profile from the profile list.

![Fabric mq brokers containers](images/fabric-ingress-container.png)

![Fabric mq brokers containers](images/fabric-ingress-success.png)


After we have the ingress container, let's create the Fuse JVM that will host the "transform" modules. click "Create" from the "containers" tab to create a new Fuse JVM and we'll assign the "hl7-transform-1" profile to it:

![Fabric mq brokers containers](images/fabric-transform-create.png)

Do the same thing (repeat the steps of creating the JVM and assigning profiles) for the "hl7-consumer-1" profile. We should see that all of the containers eventually come up:

![Fabric mq brokers containers](images/fabric-consumer-create.png)

![Fabric mq brokers containers](images/fabric-all-containers.png)


Now we have all of the pieces in place and can begin sending messages. Fire up our [HAPI Test Panel](http://hl7api.sourceforge.net/hapi-testpanel/) again and let's fire a test off to port 8888:

![Fabric mq brokers containers](images/fabric-hapi-started-8888.png)

![Fabric mq brokers containers](images/fabric-hapi-send.png)


Now let's take a look at the routes and the queues. If we click the "connect to" icon next to the "transformhl7" container, we can see the Camel routes and preview what happened:

![Fabric mq brokers containers](images/fabric-transform-worked.png)


Do the same thing; open the ActiveMQ broker. You maybe be wondering which broker to open? We know we deployed an ActiveMQ master/slave topology, but which container is the master? Click on "Services" --> "MQ" and you should see the brokers deployed. The "green" broker is the active _master_ broker in the pair:

![Fabric mq brokers containers](images/fabric-master-broker.png)

Once you've connected to the master broker, expand the Queues and Topics part of the tree to reveal some of the destinations that were involved with that last transaction:

![Fabric mq brokers containers](images/fabric-brokers-worked.png)

Woah... we see the queues/topics involved, but why is there something in the DLQ? Let's take a look at the `consumerhl7` container to investigate why. Click the same "connect" icon that we found for the previous containers and connect into the `consumerhl7` container. If we scroll to the bottom of the logs page we see this:

![Fabric mq brokers containers](images/fabric-consumer-logs.png)

Things were failing!! We also see some retry logic in there... which is something we can do with camel and if you take a closer look at the routes you'll see that in there. Let's expand the ERROR log statement and see what happened:


![Fabric mq brokers containers](images/fabric-expand-log.png)

OH! we didn't have a stub listener for this message. The "Consumer" will actually receive a message from ActiveMQ and then try to call out to an external HL7 endpoint. Let's start one of those up with the HAPI test panel. 

> Also note, the [stub-services](../stub-services) project contains a stub service that can be used here too

Go to the HAPI Test Panel and start a stub service that receives messages and sends back an ACK. Make sure to open the port with `9999`:

![Fabric mq brokers containers](images/fabric-hapi-receive.png)

Now try to send the message again, and you should see it go through:

![Fabric mq brokers containers](images/fabric-hapi-full-send.png)


