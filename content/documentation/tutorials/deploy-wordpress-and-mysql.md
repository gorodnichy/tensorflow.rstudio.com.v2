
---
date: 2016-09-13T09:00:00+00:00
title: Deploy Wordpress from the Docker hub
draft: true
---
Ths tutorial will demonstrate how Vamp builds deployments and works with gateways. We'll do this by deploying Wordpress together with mySQL using official images from the Docker hub. We are going to work with the Vamp UI, but you could just as easily perform all the described actions using the Vamp API.  

In this tutorial we will:

1. [Build a deployment from Vamp artifacts](documentation/tutorials/deploy-wordpress-and-mysql/#build-a-deployment-from-vamp-artifacts)  
  * Create breeds to describe the mySQL and Wordpress deployables and their requirements
  * Create a scale to specify the resources to be assigned at runtime
  * Create a blueprint that combines breeds with scales 
  * Deploy two instances of mySQL and Wordpress
2. [Work with Vamp gateways](documentation/tutorials/deploy-wordpress-and-mysql/#use-vamp-gateways) 
  * Add stable endpoints to access the running Wordpress deployments
  * Create a gateway to control traffic distribution between the two deployments and run a canary release
  * Connect Wordpress to the mySQL service using an external gateway
  
  

### Requirements

* A running version of Vamp (this tutorial has been tested on the [Vamp hello world set up](documentation/installation/hello-world))
* Access to the Docker hub

## Build a deployment from Vamp artifacts
Deployments to be initiated by Vamp are described in blueprints using the Vamp DSL (Domain Specific Language). A Vamp blueprint combines breed and scale artifacts to make scalable services, then groups these services into clusters for traffic distribution. 
A blueprint can reference individually stored artifacts, or everything can be described inline in the blueprint. We are going to create individual breed and scale artifacts, then reference these from our blueprint.

### Create breeds
Services to be deployed by Vamp must first be described in a breed. Breeds define the deployable to run and the internal gateway(s) to create for a single service. Settings can be specified in a breed and passed to the service at runtime, this is done using environment variables.  
[Read more about environment variables...](documentation/using-vamp/environment-variables)

For our deployment, we will need to create two breeds - one for the mySQL database and one for Wordpress. We can use environment variables to specify the login credentials for the database and add some Vamp magic to tell Wordpress where it can find the as-yet-undeployed mySQL database. The Vamp variable `host` will resolve to the host or IP address of a referenced service at runtime. Finally, we can declare the mySQL database as a dependency for the Wordpress service so Vamp will wait until the database is available before it deploys Wordpress and all environment variables can be resolved.

* **The mySQL breed**  
We will use mySQL as the Wordpress database, but you could try another database if you prefer. A quick check of the official documentation on the Docker hub tells us that the official mySQL container ([hub.docker.com - mySQL](https://hub.docker.com/_/mysql/)) exposes the standard mySQL port (3306). We can create a database and set up some user accounts with the variables `MYSQL_DATABASE`, `MYSQL_ROOT_PASSWORD`, `MYSQL_USER` and `MYSQL_PASSWORD`. That's all the details we need to create a breed for the mySQL service. 

  1. Go the BREEDS tab in the Vamp UI
  * Click ADD
  * Paste in the below breed YAML and click SAVE

```
name: mysql                    # name of our breed
deployable: mysql:latest       # the publicly available Docker image
ports:
  mysql_port: 3306/tcp         # internal gateway - the default mySQL port (tcp) 
environment_variables:         # required settings
  MYSQL_ROOT_PASSWORD: "root_password"
  MYSQL_DATABASE: "wordpress"
  MYSQL_USER: "wordpress_user"
  MYSQL_PASSWORD: "wordpress_password"
```

* **The Wordpress breed**  
Now let's do the same for our Wordpress service. The official Wordpress container ([hub.docker.com - wordpress](https://hub.docker.com/_/wordpress/)) runs apache on port 80. We can specify an external database using the variables `WORDPRESS_DB_HOST`, `WORDPRESS_DB_USER` and `WORDPRESS_DB_PASSWORD`. 

  1. Go the BREEDS tab in the Vamp UI
  * Click ADD
  * Paste in the below breed YAML and click SAVE

```
name: wp:1.0.0                 # name of our Wordpress service
deployable: wordpress:latest   # publicly available Docker image        
ports:
  webport: 80/http             # internal gateway - the default apache port
environment_variables:         
  WORDPRESS_DB_HOST: $db.host:$db.ports.mysql_port
  WORDPRESS_DB_USER: "wordpress_user"
  WORDPRESS_DB_PASSWORD: "wordpress_password"
dependencies:                  # required for the Wordpress service to run
  db: mysql                    # the mysql service from the db cluster
```


Nothing has been deployed yet, but you will be able to see the two new breeds listed in the Vamp UI under the BREEDS tab:

![](images/screens/v091/wordpress_breeds.jpg)

### Create a scale
Scales define the resources to assign to a service, that is the number of instances (servers) and allocated CPU and memory. We are going to create one scale, which will be used by both of our breeds.

  1. Go the SCALES tab in the Vamp UI
  * Click ADD
  * Paste in the below scale YAML and click SAVE

```
name: demo
instances: 1
cpu: 0.2
memory: 380MB  
```
### Create a blueprint
We can now create a simple blueprint that references our artifacts ready for deployment. A Vamp blueprint creates scalable services by joining togehter breed descriptions and a scale. Services are grouped into clusters to allow easy distribution of incoming traffic - more on this later. 

Our blueprint will use two clusters - one for the database service and one for the Wordpress service.

1. Go to the BLUEPRINTS tab in the Vamp UI 
* Click ADD (top right)
* Paste in the below blueprint YAML and click SAVE

```
name: wp_demo            # name of our blueprint
clusters:
  db:                    # name of our database cluster
    services:
      breed: mysql       # the mySQL breed we created
      scale: demo        # the scale we created
  wp:                    # name of our Wordpress cluster
    services:
      breed: wp:1.0.0    # the Wordpress breed we created
      scale: demo        # the scale we created
```

Vamp will store the blueprint and make it available for deployment. You will see it listed under the BLUEPRINTS tab.

![](images/screens/v091/wordpress_blueprints.jpg)

### Deploy the blueprint

1. From the BLUEPRINTS tab, click DEPLOY AS on the `wp_demo` blueprint
* You will be prompted to give your deployment a name, let’s call it `wp_demo_1`
* Click DEPLOY to initiate the deployment

Vamp will now run the deployment. This might take some time, especially if the images need to be pulled from the Docker hub. You can track the status of the deployed clusters and their services under the DEPLOYMENTS tab by opening the `wp_demo_1` deployment. Successfully completed deployments show the service name highlighted in grey (on the left), the IP of the deployed instance(s) and any environment variables passed to the service. Note how Vamp won't start to deploy Wordpress until mySQL has fully deployed.

![](images/screens/v091/wordpress_deployment_1.jpg)

### Deploy the blueprint again
We can now quickly deploy a second instance of Wordpress and mySQL using our existing artifacts. Vamp breeds, scales and blueprints can be re-used to initiate as many deployments as you like. Each deployment will be treated as an individual entity and Vamp will manage all the internal gateways and routing, so you don't need to worry about creating conflicts with other running services or deployments - more on this later. 

1. Go to the BLUEPRINTS tab 
* Click DEPLOY AS on the `wp_demo` blueprint
* Name the deployment `wp_demo_2`
* Click DEPLOY to initiate the deployment

We now have two separate Wordpress deployments running, they should both be listed under the DEPLOYMENTS tab:

![](images/screens/v091/wordpress_deployment_2.jpg)

## Use Vamp gateways
Vamp exposes internal and external gateways to allow access to clusters of services. Internal gateways are automatically created dynamic endpoints, external gateways are declared stable endpoints. Weights and conditions can be applied to gateways to control the traffic distribution across multiple potential routes - for example, internal gateways can control traffic distribution across the services in a cluster, whereas external gateways might control traffic distribution across routes not managed by Vamp.  
[Read more about gateway usage](documentation/using-vamp/gateways/#gateway-usage)

To get back to our demonstration, we currently have two separate deployments running. In each of our deployments, Wordpress is connected to its own instance of mySQL via a `mysql_port` internal gateway. At deployment time Vamp will automatically create new internal gateways for all the `ports` defined in a breed. Each new internal  gateway is mapped to the next available port to avoid potential conflicts between services. You can see all exposed gateways listed under the GATEWAYS tab, they are labelled in the format `deployment/cluster/port`. 

![](images/screens/v091/wordpress_internal_gateways.jpg)

### Add stable endpoints
We could use the assigned internal gateway ports to access our Wordpress deployments if we wanted to (go ahead and connect to the `<deployment>/wp/webport` ports listed under GATEWAYS). The ports assigned to internal gateways are, however, unpredictable, so it makes much more sense to declare external gateways that can act as a stable endpoint for each service. 

Vamp makes it easy to update our running deployment in this way. Go ahead and create a new external gateway that maps to an existing internal gateway.   

1. Go to the GATEWAYS tab in the Vamp UI
* Click ADD (top right)
* Paste in the below gateway YAML and click SAVE

```
---
name: wp_demo_1_gateway   # name of our gateway
port: 9050/http           # external gateway to expose
routes:
  wp_demo_1/wp/webport:   # the internal gateway exposed by the Wordpress breed
    weight: 100%          # all traffic will be sent here
```


The gateway will be exposed and we can directly access our Wordpress service at the port specified (9050). 

![](images/screens/v091/wordpress.png)

Hello Wordpress! Go ahead and finish the installation - I hear it's very quick - then you can check out your new site.

### Let's do that again
As we are running two deployments, we will also need two external gateways - one for each instance of Wordpress. Let's update Vamp again to include a second external gateway.

1. Go to the GATEWAYS tab again and click ADD (top right)
* Paste in the below gateway YAML and click SAVE

```
---
name: wp_demo_2_gateway   # name of our gateway
port: 9060/http           # external gateway to expose
routes:
  wp_demo_2/wp/webport:   # the internal gateway exposed by the Wordpress breed
    weight: 100%          # all traffic will be sent here
```
Notice that we specified a different external gateway port this time - port 9050 is already in use by the external gateway we created to point to the `wp_demo_1/wp/webport`, so we're using 9060 this time.



Once the new gateway has been created you can access the second Wordpress site on the newly exposed 9060 port (the old Wordpress site will still be available at port 9050). Complete this second Wordpress installation and make some changes to the site - select a different theme or add some content to help you easily tell the two deployments apart.

Well that was fun, but Vamp can do so much more! We can use our deployments to demonstrate some of Vamp's traffic distribution features and show how these can be used to run a canary release.  

### Use a gateway to distribute incoming traffic
Vamp distributes traffic between services via shared gateways. If our Wordpress services were running in the same deployment and cluster, they would have shared an internal gateway and we could have directly used the WEIGHT slider in the deployments section of the Vamp UI to distribute traffic between them. As we are working with two separate deployments we will need to do something a bit more cunning first.

### Create a new external gateway with two routes
We can solve this by creating a new, shared external gateway for our two deployments. We do this in much the same way as we created the previous external gateways, only this time we will be adding two routes and will use the weight setting to distribute traffic between them. You can add as many routes as you like to a gateway, just remember that _the total weight must add up to 100%_. 

1. Go to the GATEWAYS tab in the Vamp UI
* Click ADD (top right)
* Paste in the below gateway YAML and click SAVE

```
---
name: wordpress_distribution_gateway
port: 9070/http
routes:
  wp_demo_1_gateway:
    weight: 50%     # send 50% of all traffic to this route
  wp_demo_2_gateway:
    weight: 50%     # send 50% of all traffic to this route
```

The gateway wil be created. Now going to port 9070 should send you to either your old Wordpress install or your new Wordpress install. 

{{< note title="Note!" >}}
You might notice the URL in your browser switch to either :9060 or :9050 and that subsequent visits to the 9070 endpoint consistently send you back to the same port/Wordpress. So what's going on with our gateway? Visits to 9070 should be split 50% / 50% between our two Wordpress instances! Here's where things get sticky, but that's not down to Vamp.  

* On your first visit via 9070, Wordpress picked up that you had been redirected and sent out the HTTP response 301, which means "permanently moved"
* Your browser cached this 301 redirect and is now automatically sending every subsequent attempt to visit 9070 to the same destination 
  
Simply disabling the cache won't help, but you can get around this by adding, for example, a `?` to the end of the URL (e.g. `192.168.99.100:9070?`). If you prefer a more eloquent solution, you could install a Wordpress plugin to prevent the 301 from happening in the first place ([wordpress.org - Permalink Fix & Disable Canonical Redirects](https://wordpress.org/plugins/permalink-fix-disable-canonical-redirects-pack/) will do the trick), just remember to install it on both of your Wordpress deployments.
{{< /note >}}

Once you do make it back to the Vamp external gateway at 9070 you will see that everything is working as expected and you will be sent to 9050 or 9060 alternately. Thanks Wordpress.

### Run a canary release  
Now we have an external gateway set up with two routes we can use the WEIGHT slider in the GATEWAYS section of the Vamp UI to adjust the distribution of traffic between them. This will allow us to canary release our new Wordpress site to users, introducing it slowly and quickly switching back if it doesn't performing as planned. Conditions can also be added to each route in the gateway, so we could target specific user groups to send to each version.  
[Read more about using conditions](documentation/using-vamp/conditions)
 
![](images/screens/v091/wordpress_gateways_weight_slider.png)


## Summing up
You should now understand a bit more about how Vamp builds deployments from the various artifacts, resolves variables and handles internal routing. Obviously, this is not a production grade setup. The database is running in a containerised mySQL instance with no data persistence - if the container crashes you lose all your data and settings. If you're running in the Vamp hello world setup, you're also likely to hit resource problems pretty quickly. If things aren't working as expected you can try increasing the memory of Docker VirtualBox VM (3GB should be enough for this demo).

## Just for fun
Looking for more of a challenge? Try these:

* Rewrite the Wordpress and mySQL deployment as a single blueprint (define artifacts inline)
* Run this deployment using the Vamp API
  * Full instructions in [Deploy your first blueprint - using the API](documentation/tutorials/deploy-your-first-blueprint/#deploy-a-monolith)
  * Check the [API reference](documentation/api/api-reference) 
* Add a condition to the gateway - for example, send all firefox users to one version of your site and everyone else to the other
  * Read about [using conditions](documentation/using-vamp/conditions)
* Set up external gateways for the mySQL deployments and connect Wordpress to the mySQL server running in the other deployment

{{< note title="What next?" >}}
* What would you like to see for our next tutorial? [let us know](mailto:info@magnetic.io)
{{< /note >}}
