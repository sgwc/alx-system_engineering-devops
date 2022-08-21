# Infrastructure Design and Planning

Whether you are beginning to plan for the launch of a large website or managing the growing pains of an existing site, you will quickly come to the point where one server is not enough to handle the traffic to your site, or you need more reliability than a single server can provide. However, the leap from a single server to a multiserver infrastructure is a large one. There are a number of ways to architect your environment to ensure fast response times to your visitors and allow you the potential to further scale in the future.

Your infrastructure design will vary greatly based on your website’s requirements. What kind of high availability do you require? What response times will your users or your management consider acceptable? Does your site require any “extra” services or applications, outside of a standard LAMP stack? All of these considerations and (of course, expected traffic to the site) will need to be taken into account when designing, building, and maintaining a multiserver hosting environment. There is no silver bullet solution for everyone, but we can provide some guidelines that should make it easy to find the right solution for your particular needs.

# Horizontal and Vertical Scaling

There are two approaches to scaling out an infrastructure to add additional resources:  horizontal scaling (“scaling out”) and  vertical scaling (“scaling up”). Horizontal scaling involves adding additional servers while vertical scaling involves adding more resources to existing servers. A good infrastructure plan will take both of these scaling approaches into account, as there are times when one may be more appropriate than the other. Initially, this consideration may be used to consider when to migrate from one server to multiple servers. For larger infrastructures, the same logic can be applied at a larger scale. As an example, let’s consider a single-server infrastructure like that shown in  [Figure 7-1](https://www.oreilly.com/library/view/high-performance-drupal/9781449358013/ch07.html#Figure8-1 "Figure 7-1. Single server running web and database services").

![Single server running web and database services](https://www.oreilly.com/library/view/high-performance-drupal/9781449358013/httpatomoreillycomsourceoreillyimages1821080.png)

Figure 7-1. Single server running web and database services

Not all site admins have the resources to launch a site with multiple servers. This is not necessarily a bad thing—many sites can get away with using only one server, which gives a cost savings over hosting multiple servers as well as keeping system administration overhead to a minimum. With today’s multicore CPUs and inexpensive RAM, it’s easy and relatively cheap to host your site on a machine with enough power to run all the required services. But how do you know when one server is no longer enough?

There are a number of factors to consider when trying to answer the question of whether it’s time to add another server:

Resources

As your resource usage increases due to additional site features or an increase in traffic to the site, you may start to see resource contention on the server. For example, two processes both needing a lot of disk I/O can dramatically slow each other down. Can your current server be allocated more resources (RAM, CPU, etc.)? This is generally easier to do for virtual machines than for physical servers. For example, you may be able to solve an issue by adding more RAM to a server. However, it is also important to consider if adding more RAM to a server might mean that you are now limited by CPU, network, or I/O resources on the server—and you may not have the ability to increase those resources quite as easily.

Costs

This doesn’t just mean additional hosting costs, but may also mean more systems management overhead. As your infrastructure grows to include multiple servers, adding one more server should not mean a huge increase in systems management overhead (read up on configuration management in  [Chapter 9](https://www.oreilly.com/library/view/high-performance-drupal/9781449358013/ch09.html "Chapter 9. “DevOps”: Breaking Down Barriers Between Development and Operations")); however, making the initial jump from one server to multiple servers likely means you will need to step back and reanalyze your management plan, potentially starting to use a configuration management system for the first time.

High availability (HA)

If your entire site is hosted on a single server, it just takes that one server going offline for your site to go down. If this situation is unacceptable to clients or business owners, you will likely need multiple servers to offer better reliability. In essence, services will be offered by several servers, and any data will be synchronized across them.

Generally, outside of HA considerations, it’s fairly common to scale vertically to the extent that is cost effective to do so—once you reach the point that scaling vertically is going to be more expensive than scaling horizontally, it is time to start adding more servers. However, this doesn’t always hold true in practice, for a couple of reasons:

1.  Adding resources to a server can be more difficult and time-consuming than simply adding an additional server to your infrastructure. It also can involve bringing the server offline, which is not ideal.
2.  As sites grow to the point where they need additional resources, usually by that point there are business requirements in place that necessitate some form of high availability.
3.  As a single server becomes larger and more important, the ability to take it down for security and feature updates starts to disappear. Eventually you end up with a single outdated and complex server, which is very problematic from a systems perspective.

Therefore, most infrastructure managers figure out a general server specification that will work for them and provide some room for growth—usually this server specification will differ between different services (e.g., one spec for web servers, one spec for DB servers). When a new server is needed, it is deployed with those predetermined specifications. This may be a more powerful server than is necessary for the short term, but it will give the service room for growth and ensures you have some standardization in servers (making them easier to manage).

Many times, knowing when to scale horizontally is more difficult than knowing how to scale horizontally. The world is filled with infrastructures where there are many web servers, all bottlenecking on a shared Memcache instance. To this end, we will now discuss methodologies for categorizing services and knowing when (and where) to add more servers.

# Service Categorization

A basic Drupal site can be separated into two distinct services: web frontend, and database backend. The web service is responsible for handling incoming requests, processing Drupal’s PHP code, and returning the results back to the requesting client. The database back end is queried during Drupal’s PHP execution in order to pull out settings and data for the site. Every Drupal website will need infrastructure to support at least these two services. However, there are many more services that are often required to add features or enhance performance and scalability. These services can be grouped roughly into the following categories (starting with the web and database services we already mentioned):

Web

Apache, Nginx, or another web server of your choice, plus PHP

Database

Most commonly MySQL, though Drupal core supports many other databases to a lesser degree, including PostgreSQL, Microsoft SQL Server, Oracle, and SQLite

Frontend proxies

Varnish, CDN, Nginx, or other proxy caches

Other caches and data stores

Memcache, Redis, MongoDB, etc.

Search backends

Apache Solr, Elasticsearch, Sphinx, etc.

Load balancers

HAProxy, Nginx, IPVS, etc.

This generally categorizes most services that would be run as part of a Drupal website. The next step, once you’ve categorized your services, is to figure out which services may require their own server (or servers), and which services can safely coexist on the same server. Separating services into the different categories will give you the opportunity to scale out each service as it becomes a bottleneck for your site. As a visualization tool, we can represent each service category as a layer of the overall infrastructure, as illustrated in  [Figure 7-2](https://www.oreilly.com/library/view/high-performance-drupal/9781449358013/ch07.html#Figure8-2 "Figure 7-2. Infrastructure layers")—each layer can be expanded out horizontally by adding additional servers.

![Infrastructure layers](https://www.oreilly.com/library/view/high-performance-drupal/9781449358013/httpatomoreillycomsourceoreillyimages1821081.png)

Figure 7-2. Infrastructure layers

# Working Well Together

It’s important to analyze your services to figure out which may or may not work well together on a server. Some services require substantial memory, CPU, or disk I/O resources. Those may run fine on a server with another service that requires minimal resources, but you most likely won’t want to combine two I/O-intensive services on the same server, since they would be competing for resources. As services are split onto separate servers, network traffic also becomes a significant factor. Let’s take a look at a few common services and what their typical resource consumption is like. The following examples are intended to give a general idea of resource requirements, but your requirements may be very different depending on your site’s specifics:

Web services

Web services will consume a moderate amount of CPU for PHP processing. You will also need enough RAM to support large numbers of web server processes as your traffic spikes. While this is a general rule for any website, Drupal in particular and many PHP CMSs in general have fairly high I/O latency requirements due to how many PHP files are loaded to serve each request. While opcode caches such as APC can help with this, you don’t necessarily want to put an I/O-bound service on your web nodes since performance will deteriorate as it competes with the Drupal PHP processes.

Database

Databases typically will consume large amounts of RAM and can require moderate CPU resources. A well-tuned database will not touch the system disk often, except if it is write-heavy. However, you want the disk I/O to be as fast as possible when it is required. Because databases can require so many resources and for optimal performance may need to “hog” disk or CPU resources in spikes, the database is generally the first thing to be separated onto its own server, where it can be finely tuned and guaranteed its own resources.

Reverse proxy

Varnish is a common example of a reverse proxy used as a frontend cache sitting in front of your web server. While Varnish should be allocated a moderate amount of RAM, its CPU and disk usage are fairly minimal.

Alternate Drupal cache

Memcache, often used as an alternative cache for Drupal, can be used to offload cache queries from your database server. Much like Varnish, Memcached (the Memcache daemon) will require RAM—a small to moderate amount depending on your site’s needs—but will not consume a lot of CPU or disk resources.

Additional services

Of course, additional services will vary greatly in their resource utilization, so you will need to analyze each service independently. Solr, for example, can be memory- and I/O-heavy, depending on its usage.

Once you understand the resource requirements for the services you will be using, it will be easier to see which services can coexist on the same server without competing for resources.

# Example Two-Layer Configuration

As mentioned previously, one of the most common ways to split services for a Drupal site is to use two servers, with the first server dedicated to web services and the second dedicated to the database. If high availability is a requirement for your site, then this would become four servers: two web nodes and two database servers, all with some form of load balancing and/or failover mechanisms. For the sake of a simple example, we’ll assume no HA requirements in this setup and start with only two servers.

Let’s assume that we want to run Varnish as a reverse proxy and Memcache as an alternate Drupal cache (these configurations are covered in depth in Chapters  [19](https://www.oreilly.com/library/view/high-performance-drupal/9781449358013/ch19.html "Chapter 19. Reverse Proxies and Content Delivery Networks")  and  [16](https://www.oreilly.com/library/view/high-performance-drupal/9781449358013/ch16.html "Chapter 16. Alternative Storage and Cache Backends"), respectively). Considering that we are working with two servers, it makes the most sense to run those services on the web server, along with Apache, httpd, Nginx, or whichever web server you have selected.

In most environments, we will isolate the database service on its own server if at all possible. As we went over earlier, this is so that it can be guaranteed resources and won’t impact other services with its I/O and memory demands. Both Varnish and Memcache are able to run on the web node without stealing too much CPU time away from httpd—they will both have a memory limit, so that httpd is sure to have enough to deal with traffic spikes when it may need to spin up a lot of processes.  [Figure 7-3](https://www.oreilly.com/library/view/high-performance-drupal/9781449358013/ch07.html#Figure8-3 "Figure 7-3. Two server configuration")  shows which services are running on which server in this setup.

![Two server configuration](https://www.oreilly.com/library/view/high-performance-drupal/9781449358013/httpatomoreillycomsourceoreillyimages1821082.png)

Figure 7-3. Two server configuration

As your site grows in complexity and traffic, you are likely to see performance bottlenecks in either the web layer or the database layer, or sometimes both. An example bottleneck in the web layer would be if you are consistently serving a high number of clients and the web node is not able to handle them all. An example bottleneck in the database layer would be if you have a large number of queries or a few large queries that are slowing down the database response time.

In order to deal with the increased demands of the site, you can scale each of the layers horizontally by adding more servers. Most services are pretty trivial to scale in this manner, and growing the infrastructure in this way allows you to specifically scale up each service layer as it becomes a priority. (The database layer requires a bit more thought and strategy to scale beyond one server; we’ll go into more detail in  [Chapter 13](https://www.oreilly.com/library/view/high-performance-drupal/9781449358013/ch13.html "Chapter 13. MySQL")  on scaling strategies specific to the database layer.)

This “two-layer” approach can work for some time, though eventually you may want to reassess your layer definitions and potentially separate things even further. For example, if you want to add Solr search to your site, that may not fit well on your current web nodes, and it’s not typically something that you would want to run on the database servers due to I/O and RAM contention. Thus, we often add an additional layer for the Solr service. Another example would be if Varnish needs more RAM than is available for it on the web servers: in that case, you could separate it into its own layer of servers that sit in front of the web nodes. The benefit here is that you can then scale out either the frontend caching layer or the web layer separately, as needed. Our next example will show a larger infrastructure with such a setup.

# Example Larger-Scale Infrastructure

The majority of websites will get along quite well following the preceding example of a two-layer setup separating web services (and some additional caching services) into one layer of servers and databases into a second layer. But for larger sites, or those that have specific HA, security, or performance requirements, it may be necessary to implement an infrastructure with more than two layers. As in the previous example, in this case, we will strive to separate services out into their own layers so that they can be managed and scaled separately.

### NOTE

As soon as your infrastructure includes more than one web server, you will need to configure some sort of shared or synced file system for the Drupal  _files/_  directory. We cover a number of options for sharing files between servers in  [Chapter 10](https://www.oreilly.com/library/view/high-performance-drupal/9781449358013/ch10.html "Chapter 10. File Storage for Multiple Web Servers").

For this example, we’ll assume that the requirements for a frontend cache (we’ll use Varnish in this example) are large enough that it warrants having its own dedicated layer of servers. We’ll also assume that the site will use Solr heavily enough that the Solr service should be separated off into its own layer. Add to that the web and database layers, and we end up with the service layers shown in  [Figure 7-4](https://www.oreilly.com/library/view/high-performance-drupal/9781449358013/ch07.html#Figure8-4 "Figure 7-4. Example multilayer configuration").

![service layers, Varnish on top, followed by web, db, and solr](https://www.oreilly.com/library/view/high-performance-drupal/9781449358013/httpatomoreillycomsourceoreillyimages1821083.png)

Figure 7-4. Example multilayer configuration

One benefit of separating Varnish onto its own servers is that it can then provide load balancing and fault tolerance for your web nodes. Given a pool of web servers, Varnish can distribute the load between them during normal service, and if one of the web servers goes offline, Varnish will notice the failing server and stop using it for backend web requests. While some larger infrastructures will have their own dedicated load balancing devices, for many people, using Varnish to provide basic load balancing works well and is cost-effective.

Going into such an infrastructure design, it’s likely that you will want some form of high availability for the site, so we’ll assume you’re starting with at least two servers per layer in order to avoid a single point of failure.

Each service layer here starts with two servers, and the servers within a layer have replication or shared data between them such that if one server in a layer goes away, the other server is able to serve requests in its place. During times that all servers are online, load is shared between all servers in a layer in order to improve performance. Refer to  [Chapter 19](https://www.oreilly.com/library/view/high-performance-drupal/9781449358013/ch19.html "Chapter 19. Reverse Proxies and Content Delivery Networks")  (Varnish),  [Chapter 18](https://www.oreilly.com/library/view/high-performance-drupal/9781449358013/ch18.html "Chapter 18. PHP and httpd Configuration")  (httpd),  [Chapter 13](https://www.oreilly.com/library/view/high-performance-drupal/9781449358013/ch13.html "Chapter 13. MySQL")  (MySQL), and  [Chapter 17](https://www.oreilly.com/library/view/high-performance-drupal/9781449358013/ch17.html "Chapter 17. Solr Search")  (Solr) for specific setup and configuration settings for a similar infrastructure design.

# Development and Staging Environments

Once you have designed your production infrastructure, an important next step is to allocate resources for development and staging environments. As the complexity and availability requirements of your site grow, it becomes even more critical to have a separate staging environment to test out code changes, module updates, and configuration changes.

For a staging environment to provide the most benefit, it should mimic the production environment as closely as possible, complete with load balancers, database replication, and any other configuration found in the production environment. Obviously this can become quite expensive if you are literally doubling your infrastructure to provide a staging environment. There are a number of ways to cut costs for the staging environment, but it’s important to keep in mind the trade-offs you are making when you don’t have an exact replica of the production environment. Some frequently used methods for cutting costs when setting up a staging environment include:

Reducing the number of redundant servers

For example, the production environment may have five or six different web servers. For staging, you could likely get by with only two servers set up in a similar fashion. While this doesn’t mimic production 100%, the configuration should be close enough to catch most bugs triggered by having multiple web nodes.

Virtualizing resources

Even if your production environment is mostly or entirely run on physical servers, it’s possible that you could run a staging environment using some or all virtual servers. It’s important to remember, though that this—like any difference between the two environments—means that you may run into bugs in production that you don’t hit in the staging (or vice versa).

Using lower-end servers

It is possible to use less expensive servers to host your staging environment. However, this can lead to a couple of issues. First of all, as with the previous examples, this can lead to bugs or performance issues in the production environment that are not repeatable in the staging environment. Secondly, if the staging servers are so underpowered that they don’t perform well at all, it could lead to developers totally ignoring the staging environment because “it’s too much of a pain,” which in turn could lead to untested or poorly tested code making its way to production.

The staging environment should be used not only to test code and database updates, but also to test software and OS updates. Because these servers closely mimic your production servers, testing software updates on staging servers can help prevent downtime in the production environment when an update has unexpected consequences.

Providing a stable environment for your developers is another important consideration when designing an infrastructure. As mentioned previously, one way to implement a development environment is to share the staging server resources—for example, setting up separate development virtual hosts and databases on the same servers used for staging. Another option would be to use virtualized resources (or a lower-cost virtualized server if your other servers are already virtualized). In order to cut costs, sites with lower development activity could get away with only spinning up the development virtualized environment occasionally, when needed, instead of leaving it running all the time. A third option is to distribute the development environment so that it can be run locally by your developers on their workstations or laptops. Most developers can’t or won’t want to deal with setting up the system-level applications such as httpd or  _mysqld_; however, by providing a virtual machine image (or using a tool such as Vagrant to build one on the fly using Puppet or Chef configuration management scripts), developers can pretty easily run a virtual machine that pretty closely mimics the production and staging configuration. We cover Vagrant in more depth in  [Chapter 9](https://www.oreilly.com/library/view/high-performance-drupal/9781449358013/ch09.html "Chapter 9. “DevOps”: Breaking Down Barriers Between Development and Operations").

# Internal Network Layout

If you are in a hosted environment, you may not have much choice about how the network is configured. However, if possible, it’s preferable to set up a separate backend network for your devices to communicate on instead of using their public network interfaces. It’s best that most servers not even be accessible on a public IP address—only those with user-facing services need to have a public IP.

Setting up a backend network not only can improve performance, but also can increase security. In almost every case, there is no reason for your database servers or other not-public-facing services to be open to the Internet. In order to access the servers, you can connect through one of your public-facing services or, even better, create a dedicated “jump host” that is accessible to the outside world and is used specifically to access your internal hosts (see  [Figure 7-5](https://www.oreilly.com/library/view/high-performance-drupal/9781449358013/ch07.html#Figure8-7 "Figure 7-5. Jump host")). Don’t overlook the importance of securing your jump host; it does no good to segregate your hosts onto a separate network if it’s easy for an attacker to gain access through your jump host.

When your frontend web nodes are experiencing high traffic, if they also had to communicate with the database over the same network interface you would start to see slower database response times as well. Using a separate network for backend requests such as database traffic, as shown in  [Figure 7-6](https://www.oreilly.com/library/view/high-performance-drupal/9781449358013/ch07.html#Figure8-8 "Figure 7-6. Backend network traffic"), means that you won’t be competing with network traffic from external sources when trying to communicate with your internal hosts. By offloading this traffic onto its own internal network, you can also avoid potential traffic charges with your hosting company and avoid possibly congesting the public-facing firewall or router with additional traffic.

![jump host used for ssh connections to the internal network](https://www.oreilly.com/library/view/high-performance-drupal/9781449358013/httpatomoreillycomsourceoreillyimages1821084.png)

Figure 7-5. Jump host

![network traffic to/from web nodes from Internet, and separate traffic to/from database servers from web nodes](https://www.oreilly.com/library/view/high-performance-drupal/9781449358013/httpatomoreillycomsourceoreillyimages1821085.png)

Figure 7-6. Backend network traffic

### NOTE

Be sure to ask your provider about the availability of its “backend network.” While providers spend a lot of money on redundant networking equipment for the public network, the backend network sometimes doesn’t get as much attention. If a backend switch going down can take down your website, it doesn’t matter how many redundant switches are on the public network.

# Utility Servers

As your site and infrastructure grow, you may end up with a number of services that don’t directly power your website, but are used to manage the servers, monitor services, or host code repositories. In many situations, infrastructures are deployed without a utility server and all of these “side” services end up littered across various servers without much thought being given to where they should reside. This is not a very good solution and can lead to problems as you scale. As you are planning your infrastructure, you may want to consider a utility server that is built specifically for these sorts of supporting services, or a VM host to create dedicated VMs for “one-off” services.

Here are some common uses for a utility server:

Continuous integration

Depending on your workflow, you may be using a continuous integration server such as Jenkins in order to test and deploy code between environments.

Code revision control

The utility server provides a place to host your revision control system, such as Git or Subversion.

Monitoring

As discussed earlier in this book, it’s important to monitor servers and services for performance and stability. A utility server is potentially a good place to run such monitoring services.

Periodic jobs

A utility server can provide a place to run all the “odds and ends” scripts that do something against either the website or the database—for example, Drupal cron and queue workers that need to run perodically and may require more resources than a typical web request.

Configuration management

As the number of servers you are managing increases, you’ll benefit immensely by using a configuration management system (popular options include Puppet, Chef, and CFEngine). The utility server would be an ideal choice to run as the central configuration management server.

There are, of course, other things that you could run on a utility server; the main idea is to have a secure, internal server that can be used to run jobs and services that may require trusted access to other servers. In general, this should  _not_  include any services that are publicly accessible from the Internet, because this host typically ends up being trusted to some degree on all of your other hosts—and that makes it an ideal target for attackers.

# High Availability and Failover

For many sites, being offline for any significant amount of time is unacceptable. For those situations, it is necessary to set up redundant servers and a method for detecting failures and automatically switching over to the secondary server.

At the very least, for a Drupal site to remain online during downtime for a server, you will need to plan failover for web and database services. If you have additional services that are used by your Drupal site, you may consider it acceptable for those to be offline for some period of time as long as the core website is functioning.

There are different ways to approach high availability, and your approach may depend on a number of factors:

-   Business requirements set for acceptable downtime
-   Whether or not failover needs to be completely automated
-   Whether certain failures could put the site in a “read-only” mode (for database access and/or file uploads)
-   Whether you have a budget for hardware and staff time to implement high availability

In subsequent chapters, we cover specific configurations to accomplish high availability with various services. At this point, it’s important to figure out if HA is something you require (or may require in the future) for your website, and to look at the various services, servers, and network equipment you will be hosting to get an idea of all the single points of failure you may have in your infrastructure.

# Hosting Considerations

Some websites end up being hosted internally if there are sufficient hosting resources; other situations call for paying for an external server (or service) host. If you are looking for an external hosting provider, there are many important aspects to consider, including:

Cost

Analyze the cost for hosting your planned infrastructure. Include additional bandwidth costs that you may incur. Also, look at how cost will increase should you need to grow your infrastructure by adding additional servers or dealing with increased bandwidth requirements.

Uptime

What kind of uptime does the host guarantee, and what is its average uptime for services, etc.? Consider reading through public customer forums or hosting “status” sites, if it has such a thing. What kind of redundancy does it have in place for power, network, and cooling?

Support

What type of support is offered, and how does this correspond to your support needs? For example, do you want a host who will support only its own equipment, leaving you to handle all OS and application support? Or would you rather have a host who manages the servers for you, so you only have to deal with Drupal specifically? Does support cost extra, or is it included with your hosting plan (and to what extent)?

Backups

Are servers backed up, and if so, on what schedule? Is backup size limited? How long are backups kept? Does the backup service cost extra? Where are the backups stored, and are they mirrored to another location?

Security

What kind of physical and network/server-based security is provided?

Failover and geographic distribution

Does the host offer servers at different geographic locations that could be used for failover in the event of a disaster at one data center?

Virtualized Versus physical servers

Are you looking to host physical servers (owned by you, or rented from the host), or virtualized resources? How will this decision affect your ability to scale in the future and to deal with estimated traffic loads now? Does the host offer the ability to mix physical hosts and virtualized hosts as part of the same hosting plan?
