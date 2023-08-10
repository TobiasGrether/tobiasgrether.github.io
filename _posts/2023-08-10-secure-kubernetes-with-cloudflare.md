# Cloudflare Gateway as a zero-trust corporate VPN

## Introduction
Security is a challenge of its own, especially with developers that work from remote locations all over the planet.
At NetherGames, we faced the issue of having a kubernetes cluster that contained development and production components, but no safe way to let developers access it.

Our primary requirements were:
- Ability to use Single Sign On via our company-internal IdP
- Ability to restrict groups to only connect to certain IP address ranges or kubernetes services
- Ability to easily manage the entire setup from a Web UI
- Minimal Cost

## Options
Our first VPN setup for developers was a simple Wireguard service setup with Firezone for user management. This meant developers simply proxied into the kubernetes cluster, but there was no further control or restriction involved. Every developer would be able to move freely inside the cluster and connect to whatever they want. This of course poses a huge security attack surface, going right against the least privilige principle. 

When this didn't work (and we accidentally deleted it too while migrating clusters), we sought out for a different option. Immediately services like Tailscale came to mind, but Tailscale is not really cost-effective, costing 10$/user/m for the setup that we would have required.
This lead me to search for different options. At first, I tested NetBird, which is a pretty nice open source project for a wireguard VPN server with user management and access control which you can selfhost easily, which would mean zero operational cost. However during testing, we found it to be partially unreliable, thus not qualifying for further deployment.

After that I had a discussion about a similar topic with a colleague of mine, which reminded me of Cloudflare. NetherGames relies on Cloudflare Products in many ways; We run staging previews for our web pages, proxy and protect web traffic as well as use their DNS load balancer to get all the players to their nearest proxy server. 
Cloudflare also has a product called Zero Trust, which we rely heavily upon to protect operational services like Web Interfaces for databases, control panels etc. from unauthorised access.

## Cloudflare Access
Cloudflare Access works by using Cloudflare as a proxy between your client and the application. Cloudflare then uses a previously defined authentication strategy like OIDC or SAML to authenticate the request against a set of predefined rules.
For example, one could have a rule: User must have group Administrator and the request must come from this IP range.
Cloudflare would then check the user for that access, essentially preventing unauthorised access as a whole.
It has been an irreplacable asset for us as it helps us keep the entire network infrastructure secure and at the same time unified and simple. It can be deployed in front of any web application with a few clicks and is very effective at protecting it.

## Cloudflare Zero Trust
Cloudflare Access is part of Cloudflare Zero Trust. Zero Trust means what it says: Don't trust anyone, validate and confirm instead. Zero Trust provides a suite of tools that allow enterprises to keep their infrastructure safe and is used by a wide variety of companies. Zero Trust also provides a service called Tunnels, which can be primarily used to expose applications to the internet without doing any port forwarding. 
This essentially works by letting the device that the web service is running on connect to cloudflare servers and cloudflare will proxy all connections through that tunnel. This allows people without an IPv4 address or people who value their privacy to easily expose applications to the web; A very useful tool.

However, tunnels can also be used for a different purpose. They can also be used as exit-gates for a virtual network. That means that we connect from our kubernetes cluster to cloudflare and allow cloudflare to proxy data into our cluster that way.


## Cloudflare Gateway & WARP
Thats pretty nice, but how is that different to Wireguard?
Well, thats a fair question. It still proxies traffic into the cluster. 

For a little bit more context, Cloudflare has a product called WARP. WARP is a VPN that leverages their 1.1.1.1 DNS Nameservers as well as traffic routing to keep traffic private from evesdroppers like ISPs. WARP can be installed on any device of your choice and connects to Cloudflares servers to proxy traffic to the internet.

Luckily, Cloudflare did a great job at integrating Warp with Zero Trust. With little configuration, you are able to log in to the Zero Trust Network with the WARP client, using our IdP (requirement: check!). This essentially puts your device in a virtual network with all other devices that are also connected to the same Zero Trust instance.

This is where many of the Cloudflare products come together:
- Cloudflare Tunnel gives the Zero Trust Network access to our on-premise kubernetes cluster
- Cloudflare WARP gives our devices access to the Zero Trust Network

Putting two and two together, we can bridge the gap and connect everything.
We can now tell Cloudflare using their Private Network Configuration for Tunnels that there are certain ip ranges that we want to bridge from the kubernetes cluster tunnel to all our clients, in our case the kubernetes pod and service CIDRs.

After that, without any rules, we can already access the IPs from the kubernetes cluster on our local devices. A simple

```bash
curl http://10.103.99.169:8080/health
```

Will show us that the connection succeeded. But wait, we didn't provide any security setting: Why can we even access this?
Well, Gateway has a Allow All default policy. To mitigate this, we will create a new Gateway Firewall Network policy, which simply denies all kinds of traffic to that cluster. But thats not what we want, right? We want to allow contact to certain ip addresses in the cluster to make them usable.

And this is where a second policy comes in: We will now define an ALLOW policy that will allow certain user groups (derived from our IdP) to connect to certain IP addresses, the IP addresses that we want to make available to that group.
Now, its important to understand that these two policies, the BLOCK ALL policy and the ALLOW policy work hand in hand.
The BLOCK ALL policy dictates that all traffic must be dropped, and the ALLOW policy overrides this decision in certain cases, specifically when a user has the correct permissions and tries to connect to one service of a predefined, certain group.

This will allow traffic only to certain services only with a certain group assigned.

## KubeDNS
*This is great, but nobody wants to remember IPv4 addresses for services by heart**.

And thats easy to solve. In Kubernetes, each service is externally reachable by a FQDN. A service named "observer" in the "infra-prod" namespace (with our cluster configuration) will be reachable under `http://observer.infra-prod.svc.ngmc.cluster`. That is much nicer, isn't it?

This is provided through KubeDNS, a DNS server shipped in every kubernetes cluster that all kubernetes pods are configured to use. Now, but how do we tell our client devices, like the MacBook that I'm typing this blog post on, to use this DNS server to resolve these DNS requests. Obviously, Cloudflare has thought about it too.

It allows you to provide Local Domain Fallbacks: this means that you can configure DNS requests for certain domains or subdomains to go to a specific nameserver, in our case the KubeDNS server. By configuring the KubeDNS server as the DNS server for the domain *.ngmc.cluster, we should now be able to use the DNS server, right? Wrong!

Remember that Block All rule that we set up earlier? This will also block access to the KubeDNS address, so we need to create a new ALLOW rule for the KubeDNS IP first. Once that is done, we can confirm the setup to be working by running

```bash
dig @10.96.0.10 observer.infra-prod.svc.ngmc.cluster
```

Which should return with a status of NOERROR. Now we can run
```bash
curl http://observer.infra-prod.svc.ngmc.cluster/health
```
to get to the same page as with the direct ip, just much nicer.

## Conclusion
Cloudflare Access solved all of our problems:
- It ties in perfectly with our IdP by supporting all kinds of SSO strategies
- It allows developers from around the world to access cluster resources easily
- It is **completely free**. I have no idea how they actually manage to provide all that for free.
- It has a nice (but sometimes a bit slow) web UI to manage everything.

This is the most mature solution we could find for a setup like this without spending any money, and we strongly believe in the products cloudflare makes. This setup increases security and ease of development alike. It doesn't cost anything to operate and supports everything we need and more. It also provides us with a logging interface to log suspicious activities and react to security threats inside of our network quickly.

## Note
This is obviously a difficult setup and hard to explain in a blog where I want to go into technical depth. This blog had a bit of a different style, expressing my thoughts while solving a business problem. I do hope though that this has been interesting. I plan to attach some screenshots of the configurations used here to make it easier to understand and replicate.

Cheers