# What is this?
A fun little job interview challenge I had the other day. Despite not a match I was happy to pick up some idiomatic ansible along the way after being isolated from outside world tooling for 8 years. Putting it up here with identifiable details anonymized.


# Requirements
* Detailed notes of your work - challenges, design choices, etc. Please be descriptive, so that we get, not just the end result, but an idea of your thought process, reasons for the approaches you take, how you solve problems you encounter along the way, etc.
* Two web servers - one serves 'a', the other 'b' at index.html
* Add a load balancer on the 3rd server to load balance between the webservers
* Configure the load balancer
  * Any balancing scheme (round robin, random, load based, etc.)
  * Configure the load balancer to be "sticky" - the same host should hit the same webserver for repeat requests, only switching when a webserver goes down, and not switching back when the webserver goes back up
  * Pass the original requesting ip to the webservers
  * Make port range 60000-65000 on the load balancer all get fed to the web servers on port 80.
* Add nagios to the 4th server and configure to monitor the servers and the load balancer.
* Add a user ‘client’ to the boxes
  * Grant sudo access and install the attached public keys as its authentication credential
* Lock down the network:
  * Allow only one server public ssh access
  * From that server, be able to access the others via ssh
  * Block all other unused/unnecessary ports

# Recommendations
These are the recommendations or questions I would raise if it were a real life project.
* Have a separately harden instance acting as bastion or jump box, with verbose authN/Z logging and rate limiting.
* I can see that these are EC2 instances, normally I would set it up so only the bastion and load balancer have public IP, the rest only VPC private IPs. 
* Consider not having a reverse proxy listening on 5000 ports.
  * Why listen on that many ports in the first place? What problem is it trying to solve?
  * In case is a workaround for client side (of the LB) port exhaustion, who is the client? If the clients are diverse port exhaustion shouldn't be a problem.
  * The tiny 2 vCPU 1 GB ram instance would struggle. I haven't figured out what's the exact effect and it's not a pattern I'm familiar with, I wouldn't be able to find out the effect on these instances through for example load test either because they're small. It's most likely going to be CPU or memory bound. Whether the reverse proxy of choice allocate a new process / thread for each port should be quite irrelevant, this setup is likely going to be pure overhead
  * Could there be alternate architecture of choice that does't involve a single big instance? Scalability is a concern, VRRP based primary / secondary failover is generally less desirable than solutions that spread loads. Perhaps something that involves weighted DNS, or some form of control plane.
  * In an AWS environment consider ALB/ELB/NLB or API Gateway, usual tradeoff analysis between money versus setup / running/ maintenance cost applies for the same level of availability.
* Requirement of stickiness implies stateful web service. Stateless web services removes multiple classes of availability and scalability concerns. There are notable exceptions such as [high bandwidth traffic](https://www.haproxy.com/user-spotlight-series/haproxy-load-balancing-at-vimeo/) where locality of web server cache matters, otherwise it is usually desirable to put states in client, database or cache depending on is nature.

# Design choices
## Tenants
* Respect Gall's law, fulfill the requirements without over-engineering the solution. Prefer simple solutions that solves complex problems. Optimize iteratively.
* Be liberal at considerations but conservative at implementation plan. Get a good tradeoff that is less likely to introduce future tech debt within a reasonable timebox.
* Security comes first, then availability, then the rest of the factors

## High level choices
1. Fixed number of 4 hosts in total, future scaling assumed to be out of scope, unknown pre-existing deployment habit.
   1. Avoids unnecessary pros / cons analysis
   2. No service discovery or orchestration layer such as Kubernetes.
   3. No containers such as docker
   4. No CI/CD considered
   5. No scaling considered
2. Unknown future resources prepared for this simulation scenario, choose paths of least frictions with least maintenance need:
   1. Avoid paths that involve building from source, the added bonus of up to date (security) patches and tighter controls diminish when it is not maintained.
      * This is an nuanced topic that I don't want to dive into for this scenario. Ideally they should never be updated through distro package manager directly, but given the constraints this is the most sane way of doing it.
   2. Avoid volunteer / community maintained packages or libraries, some factors when considering exceptions:
      1. Well known, well funded with lots of eyeballs on it
      2. Have consistent and recent activities

## Configuration management framework
### Considerations
Certain axioms that are tried and true:
* All of these must be infrastructure as code. Over the years I've learned that the effort to do this is pretty much the same as manual configuration, even a hackthon project deserves all the goodness:
  * Inventory
  * Reproducibility 
  * State with baseline predictability
* Declarative frameworks eliminates one class of availability risk, compared to procedural ones.

Configuration management has a strong overlap with provisioning frameworks. I'm strongly supportive of immutable deployments for its stronger availability characteristics. It eliminates another class of availability risk thus overheads.That means no configuration changes, just provisioning of compute units with tested new images. However,
* The trade off function / curve only makes sense when the cost of itself becomes insignificant at certain scale
* No access to underlying AWS account in this scenario

That means Terraform is out of consideration.

Docker would've been worthy of a pros/cons analysis, however given high level choice 1.2 above and that the choice of compute is already defined as VMs for which running containers on top of it removes a lot of its pros, it is also out of consideration.

In this case I'd like to use a framework that is the most powerful for any generic environment without preparation, that means what works most naturally masterless and agentless. Since I predominantly used Cloudformation / CDK / internal AWS framework, to maintain these, I have to refresh my memories and learn about the latest status quo from outside sources. I find this [article](https://blog.gruntwork.io/why-we-use-terraform-and-not-chef-puppet-ansible-saltstack-or-cloudformation-7989dad2865c#64a6) a good source, although I personally do not know about the reputation of this community / company, the clearly outlined factors they considered matches what I had considered in the past. Given the non-prod nature and limited time for an interview take home simulation, this is good enough for validating what I roughly believed to be true based on past experiences. 

The conclusion of it is Ansible. It is not strictly declarative however:
* Other factors trumps this one
* A lot of modules are designed to be used declaratively
* Can be written in a style that is mostly declarative, or a least idempotent. e.g. restart service instead of start service

There was a concern of team / developer familiarity (me). The last time I touched any configuration management was over 8 years ago helping a company migrate from poorly maintained Puppet to SaltStack. On top of being framework agnostic knowing general good practices, the languages I recently used a lot are: Typescript, Kotlin, Java, Python, Ruby. Particularly for Python, I owned and designed an analytics project around it, often dive into lowest level of details to debug (pdb is fine, gdb into C component less so), so I'm confident I could unblock myself when running into any problems.

### Challenges
* I've never touched Ansible before.
  * The way I learn best is when I read code examples.
    * Thus ansible's own [repo](https://github.com/ansible/ansible-examples/blob/master/lamp_haproxy/site.yml) was a good starting point.
    * I've also read a few other Github repos with strong traction either on itself or on [galaxy](https://galaxy.ansible.com).
    * When I do these I do put on my code reviewer glasses as a filter. Even without knowing Ansible intimately, I can capture certain universal values: how they name things, style of abstraction, whether certain configurations are hardcoded and how painful is it to break out of it when that is the case, etc
  * Then I backtrack into the [concepts](https://docs.ansible.com/ansible/latest/getting_started/basic_concepts.html) to make sure I grasp the design philosophy right.
  * When certain things are unclear, [source](https://github.com/ansible/ansible/blob/1491ec8019b064374145dace41b1320e04fb494b/docs/docsite/rst/reference_appendices/special_variables.rst) is always authoritative as a last resort.


### Structure
It uses roles to define shared desired state for groups of servers instead of a parent child tree structure. This simplifies categorization which is always hard and encourages reuse.
* common: applies to every host as a baseline
* service: applies to all service hosts, this includes web and loadbalancer servers. Services control and data planes.
* management: management plane. Only one host in this case
* group fo each service, in this case web servers and loadbalancers

### Result
I'm happy with the end result. I think I've picked up reasonably strong expertise on Ansible after this and I can author respectable roles, playbooks or any reusable components. In terms of code, there are room for improvements, in particular certain strings can be extracted as variables; atomic units can be extract as blocks, to be more DRY (my threshold is 3 repeats). However the base structure is at level that I'm mostly comfortable to be left to perfect at a later time even for production services.

### Caveats
* Not all the configuration changes are being validated before service restart. This could bring down a service, I'm only comfortable leaving it as is for a simulation.
* Service could restart unnecessarily, when it should be conditional e.g. config changes. Certain services accept NOHUP signal instead which could further reduce down time. This is however moot in an immutable provisioning environment.
* No traffic draining logic defined for add / removal of backend services. See high level consideration #1. Also for a simulation I don't want to reinvent a deployment system or spend time to evaluate whether there are trustworthy community maintained collections / roles out there, see high level consideration #2.1


## Security, Networking and Firewall
As per recommendations above, in an AWS environment with full provisioning access, networking and firewalling can be dealt with at VPC / SecurityGroup / Network ACL level. With that limitation in mind, below are the chosen implementations.

### Security
Not possible given the constraints:
* Role based access control is much more manageable as a single authoritative source of identity, 
* Desirable to limit SSH access for strictly last resort escape hatched. Services like SSM provides a framework for structural and reproducible changes that is reviewed, vetted and battle tested. 

### Networking
Perhaps restricting the use of public IPs, and making sure instances are distributed among AZs is desirable. Given the restrictions, noops here. 

### Firewall
From the bit and pieces I read over the years nftable is considered to be the recommended firewall for Linux. However:
* Netfilter has been running on production servers for a long time
* The logics necessary to fulfill the requirements is simple
* Small environment with small capacity where it is unlikely for it to become a bottleneck
* ufw is under Ansible "standard library"

Thus despite there is a [third party role](https://github.com/ipr-cnrs/nftables) for nftables with respectable traction, I chose to use ufw. See high level considerations.


## Loadbalancer
There are only a handful of reputable production grade software loadbalancer out there. According to tenants and high level considerations, I pre-filtered the choices to just Nginx and HAProxy.

A quick web search validated my vague memory that only paid version of Nginx supports sticky sessions so the remaining choice is HAProxy.

There are only community maintained ansible roles in galaxy. I reviewed a few of them, they're either source based (Consideration 2.1), staled (Consideration 2.2) or the abstraction limits the flexibility so that it takes more work to apply the configuration I desire (send-proxy, surge queue, tcp). So I decided to write my own role.

The choice of TCP or HTTP reverse proxy depends on whether the scenario's ultimate goal requires end-to-end encryption. Reasonably likely since Company does financial transactions. Although this is a nuanced topic since connection in VPC is somehow encrypted and encapsulated in multiple layers making it hard to intercept, I cannot get a definitive answer without the capacity to do a proper threat modelling. I chose TCP reverse proxy with PROXY protocol to preserve client IP because it comes with less moving parts. This has been my default choice unless requirements call for otherwise, in particular if cert rotation is a solved problem.

Since the requirement's listening port range lies within the common ephemeral source port range, to avoid potential service (re)start blockage, I've updated the end to be right before the listening port range (59999), while extend the start to 10000 since no services would listen on those ports. On the other hand, as far as I understand there are certain risks associated with timewait port reuse, so I left it there as disabled state to be explored later.

Some speculative defaults:
* File descriptor ulimit - roughly matches 2 * listening port while being a power of 2
* Enable surge queue - inside of AWS we consider LIFO as the holy grail, it's one of the many repeating themes in correct of error reviews where massive backlog became an issue. Interestingly, HAProxy [thinks](https://github.com/haproxy/haproxy/issues/177) otherwise with sound reasoning. I wouldn't have time right now to find out who is right, or whether scales / service guarantees matter, however it would be an interesting topic to look into in the future.
* Maxconn - I don't have an even rough idea how much maxconn is feasible for t2 instances serving single character static content. So I just put in some value that roughly matches to number of listening ports for consistency.
* Sticktable - nothing particularly here. Without detailed requirements, simple IP hash should suffice. This is something that can be optimized at later stage. I just set it up according to [HAProxy's blog](https://www.haproxy.com/blog/tag/persistence/) so it doesn't become a landmine in case of a future multi-instances setup, such as a keepalived VRRP failover. The above blogs category also provides a reasonable future investigation entry point.
* Health check - depth of health check is another somewhat philosophical (bikeshedding magnet) question, how deep should it, and service canaries should be? I just choose a basic TCP ping here instead of getting into the rabbit hole.

## Web server
This is similar to loadbalancer. For simple static content, the most well known are Apache and Nginx, both would handle the requirement without issues. It's only worthwhile to do a manual benchmarking of them, and perhaps other newer contenders, when the scenarios calls for it. Even then it could be left as a later optimization task. I pick Nginx because back then when I read about it Nginx performs better with default configurations and I like there config syntax more.

Here, despite consideration 1 above, I chose a source based ansible module because it is vended by Nginx Inc themselves. I did customize the default webroot to a more conventional location. Originally I attempted to use their default templates as a baseline since it saves work for a simulation scenario, however I ran into some issues (see challenges below), at the end I switched to copying my own config after several hours of debugging. Here I did not put a lot of energy into customize the logging format either. For production, these days I prefer structural logs such as JSON for easier later parsing.

### Challenges
I ran into a hard to debug [issue](https://pastebin.com/v3EZcwZk) with a cryptic exception, which I couldn't root cause even after making it debuggable locally with pdb. I will write about it more once I have the root cause. It likely involves issue that is a combination of the template, python version and jinja1 module version. As an early summary, this is the problematic block: https://github.com/nginxinc/ansible-role-nginx-config/blob/main/templates/http/core.j2#L159 . I tried changing my local version so only "address" and "proxy_protocol" are left and it worked. Adding the ssl line (with that being defined as the same value in variables) would fail, so it's not simple typing issue.


## Nagios
Similar to loadbalancer situation I ended up writing my own tiny role for this from scratch.

I only defined some basic NRPE checks that comes with ubuntu maintained collection of plugins. One notable miss is memory: available, free, used, dirty page and perhaps slab.

A lot of the metrics should be timeseries instead, with major focus on the services themselves. I haven't re-dive too deep into RRD for a test scenario, which I for sure did when I was revamping the monitoring system for another company. For a production scenario I would have looked into how it operates with / versus statsd or other timeseries daemons.

Some service level metics I consider important for both LB and web:
* Count (divided into 2xx, 4xx and 5xx)
* Latency (divided into 2xx, 4xx and 5xx) 
* Availability (success / count)
* Throttles

Since DNS / TLS are not in scope, instead of password protecting a plain text port I bind it to localhost so it can only be accessed through SSH tunnel.
