# Docker Swarm

_Archived page from 2020_

CASIT Web services has been researching a developing a system to host our WordPress, Drupal and other legacy sites. The system as designed revolves around a few key technologies. Primarily Docker Swarm as the management and orchestration system. Nginx as the reverse proxy, loadbalancer, SSL processor and caching system. NFS and Convoy as the persistent data storage system hosted on Netapp filers. The system is designed around a single user community not as a shared services model.

## Container Cluster Orchestration
We selected Docker swarm after investigating many other orchestration stacks. All of the other stacks required picking up several different technologies to provide a working solution. Swarm’s only missing layer is the intelligent layer seven reverse proxy. We therefore landed on swarm to limit our technology foot print and as a way to stay developer focused. Many of the other systems had a more operations direction. Since we are primarily a development shop we wanted the stack to lean that way.

While it would be more effective to be on hardware directly we decided that it would be simplest to use VM’s as the platform of choice initially but not tie ourselves to it long term.

Minimally a production swarm cluster requires 3 nodes. It may operate better at 5 nodes. Because there is no cluster hardware the cluster addresses partitioning by a voting system. More then have of the nodes have to be in the surviving cluster. We think 3 nodes with 2 vCPU’s and 16GB of RAM will be a reasonable replacement for our production VM’s. The size of the VM’s memory and the number of nodes could easily be scaled further. Our web sites are not typically CPU bound and because we are moving SSL and caching to the Reverse Proxies we expect this CPU to RAM sizing ratio to be reasonable.

The design while well balanced does have outstanding issues. Because containers cannot be migrated without stopping the application there is the tendency on rebooting the cluster for the last node to be empty. All of it’s containers were stopped and restarted elsewhere before it was able to return to the cluster. This and in several areas Swarm is still under very active development. We expect many of these issues to be resolved in the next two calendar years.

## Intelligent Layer 7 Reverse Proxy
Docker Swarm introduced an interesting problem as it comes with layer 4 load balancing built in. While this is a very useful configuration it hides source information from the application servers. Swarm does distribute traffic and handles hosts going down. What we needed was a way to inspect the traffic and put information in the http headers so that the applications can be aware of the load balancing in front of them. As additional bonus it would be great to manage ssl from a single point and not inject it into the application server. It also provides a central point that traffic logging and status can be reported from. We selected NGiNX over HA proxy and using the campus F5 for various reasons. Primarily the cost advantage and the ease of configuration of NGiNX put it at the head of the pack.

Our design requires that we do not have any single points of failure. Therefore we have specified a fail over cluster pair of NGiNX nodes. They use RHEL’s built in keepalived failover service to float an IP address between them. If the primary node fails or the the NGiNX instance fails or the keepalived is stopped the floating ip is started on the secondary node.

The configuration will be stored in our puppet repo and will have a separate keepalived configuration file for each node. The configuration files for the services will be stored in the same repo and will be identical between the two servers. The general configuration will handle defaults, SSL, and caching. Individual sites will have include file for each site mapping http to https and the default port to the ports they are presented in in the Docker Swarm.

This service will require high CPU performance and ideally have decent disk performance to it’s cache directories. We have sized them with 4vCPU’s and 4GB of RAM with a 100GB cache.

While part of RHEL keepalived is very poorly documented and has a lot of complex functionality. We have a simple working configuration but it is as simple as possible. It is possible that this lack of documentation clarity for keepalived could be a risk long term. NGiNX plus includes a keepalived configuration that I have follow generally. Their paid version would provide more directly supported solution for ip failover.

## Persistent Storage
Many container systems are designed around impermanence at the application and often even the database layer. Since we were designing around historical wordpress applications most of those models don’t work. The application needs to be able to write to update itself. Pluggins download and load the latest code. Databases are small but there are many of them. So rather then build another layer for each of these issues we decided to work with Docker Volumes directly for the DB data storage and the web server code and unstructured file storage. This meant we need to select a plugin to map this persistent storage to all of the nodes. Docker Swarm does not yet do this. Several plugins work with NFS shares as backends. We chose the most generic method with Convoy from Rancher labs. It isn’t clear where the development of Convoy is headed. Chris believes that Docker Swarm will implement this functionality internally. If Convoy was to stop working the NetApp plugin would be an option.

Using a NetApp based NFS share allows us to leverage our existing backup technology and the built in snapshots to provide roll back and versioning of containers. Convoy then maps this directory into a single namespace across the different Docker Swarm nodes.

We have provision a 1TB volume from our CAS SVM on the IS cluster in the CC. This volume has been mounted with NFSv3 with root access to our three nodes. Access should be limited by firewall rules to the filer. Currently it is protected by NFSv3 permisions on the filer.

## Visualizer
View our running containers on the swarm at Docker Visualizer (only available on campus).

## Acknowledgments

### Information Services (IS)
IS provided the infrastructure and support to help design and implement our solution.

* Lucas Crownover
* Micah Sardell
* Matthew Shepard
* Derek Smith
* Justin Spencer

### College of Arts and Sciences Information Technology Support Service (CASIT)
CASIT researched the solution and are the primary users of it.

* Ben Brinkley
* Loring Hummel
* Bill Madden
* Daniel Mundra
* Chris Schafer
* Cameron Seright
* John Zhao
