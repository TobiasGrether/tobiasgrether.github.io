## Introduction to Proxies

Sometimes, one server just is not enough. You might want to split your server up on multiple physical machines, or you might want to run separate servers with separate plugins. It's easy to just set up a separate server and let them both run along each other on different network ports.

Minecraft implements the TransferPacket, which gives you the ability to forcefully transfer a player from one server to another. This allows for transfers when you click on an item or punch an NPC.

With that said, this transfer will re-do the entire login sequence, re-establishing encryption, compression and all the game properties. This is a lengthy process, especially, for people with a high network latency. It can easily take 3-5 seconds until you are on the actual destination server.

## So what is a proxy?
Proxies are a piece of software which allows server admins to transfer players between servers without the need for the client to recreate the entire connection.

It acts as a middleware between the player and the server. The player connects to the proxy, which then performs the initial login. The proxy then decides what server to connect the player to, and establishes a connection to that server. It then performs the login sequence with the data that the player has previously sent it, and relays data that the server sends to the player, and vice versa.

Imagine a proxy like a game: Person 1 tells Person 2 something, and Person 2 tells Person 3 what Person 1 said. Person 2 can tell Person 3 something different than what Person 1 actually said. Person 2 can also talk to another Person, Person 4 entirely, without Person 1 knowing.

In this scenario, Person 1 is the Game Client, Person 2 is the Proxy, and Person 3 or Person 4 are downstream servers.

## Directions
In the proxy landscape, two terms are mainly used to indicate the direction of the traffic: **Downstream** and **Upstream**.
The "Downstream Server" is referring to the current server that the proxy is connected to, while the "Upstream Client" is referring to the client being connected to the proxy.
As such, if traffic is Downstream-Bound, its traffic from Client -> Proxy or Proxy -> Server, generally with the server as the target.
From that, if traffic is Upstream-Bound, its coming from the Server or proxy in the direction of the client.

I personally think that these naming conventions are bad because they are not self-explanatory. I instead prefer *Clientbound* and *Serverbound*, which are much easier to understand. Clientbound packets go towards the client, while serverbound packets go towards the server.

**Important**: I've had a discussion with YouTuber LifeOverflow over the terminology here. It seems that in the Minecraft community, these terms are switched up compared to how they are used outside of the Minecraft scene. This strengthens my point of changing these terms.

## Switching servers
So you might think "how does this help with transfers"?
Since the player is no longer directly connected to the server, but has the proxy as a middleware, the proxy can implement its own transfer mechanism. 
When the proxy transfers a player, it cuts the connection with the previous server the player was connected to, then establishes a new connection to the new destination server.
The player, however, will not be disconnected from the proxy. He stays connected to it while the proxy does the transfer. 
To make the transfer possible, the proxy has to keep the information of the player stored so it essentially "replays" the initial login of the player.

## Transfer Masking
When switching the server, players usually expect to see some sort of transfer screen, like it is the case in vanilla Minecraft. To recreate this feeling, we can simulate a dimension switch. Dimension switches happen when you go from the Overworld to the Nether or from the Overworld to the end, and they are mainly recognized through a screen which looks a bit like the screen you see when you initially join a server.

This screen can be used to give the user some visual feedback that something is happening.
It also serves another purpose however. The Minecraft Bedrock client caches some data. If servers do not tell the client to remove that data properly, then that cache will fill up over time. While that might sound small in the beginning, if a player plays on a mobile device for an extended period of time and changes servers frequently, that cache might keep data it no longer needs from a while back. Mobile devices generally do not have a large amount of resources available, so the cache might fill up to the point where the games performance is degraded because of this.

## ID rewrites
### The basics
In Minecraft, every entity spawned has a so called `entityRuntimeId` (also `actorRuntimeId`). This ID is used by both client and server to communicate which entity is affected by a certain action (for example, hitting an entity). With a non-proxied setup, the client and server entity IDs are always in sync, so they can just send them over the network as usual. However, with a proxy in between, it gets more complicated.

Players are entities, just like animals and mobs. They also have a runtimeId, which is used to identify them when performing actions. However there is one major issue:
Important to know is: The client needs the entityRuntimeId of the player. To know that, the server sends the client its entityRuntimeId in one of the first packet, the **StartGamePacket**. The difficult part: The StartGamePacket can only be sent **once**. Which means that once the client knows its entityId, it cannot be changed.

### The problem
Now why is this a problem? Well, when we switch servers, the downstream servers will each assign their own runtimeId to the client. But since the clients self-known runtime id cannot be changed, there would be a difference there.

### The solution
To resolve this issue, the proxy does whats known as **Entity ID Rewrites**. This essentially means that when the client initially connects to the proxy, the proxy sends an arbitrary, random ID to the client which the client now knows as its own runtimeId (known as *client-known* ID or just *runtimeId*). It also stores this ID for later.
When the downstream server the proxy connects the player to now sends its own StartGamePacket with its own runtimeId, the proxy intercepts the packet. It then takes the runtimeId that the downstream server has given the client and saves it as well (known as *server-known* ID or *originalEntityId*).
The proxy now has the runtimeId that the client thinks it has, the client-known ID, as well as the runtimeId that the server thinks the client has, the server-known ID. Now, the actual rewrite is taking place.
When a player is being transferred, then the proxy needs to switch out the originalEntityId with the new one.

### The actual rewrite
With both of these values, the rewrite is fairly simple. We take every packet that is being transmitted. If it contains an entityRuntimeId, then we need to rewrite it. For this, we need to take into consideration the direction of the packet. If the packet is sent from the client to the server, then we need to replace the client-known ID with the server-known ID. And for server -> client, we need to do the opposite, replacing server-known ID with client-known ID.
Using this, we can ensure that there is no switch-up between these entities.

### Overhead 
While this is necessary, it also a noticable overhead. To determine which packets have to be rewritten, we need to take every single batch of packets, take a look into them and see if any of them contains an entityId. To do that, we need to take every single packet batch, decompress it, then check all the contained packets for any packets that have to be rewritten. *If* any packets need to be rewritten, we have to recompress the entire batch. (If not, we can just relay the original batch without compressing it again)
While this sounds trivial, Minecraft Bedrock uses ZLib for its compression. ZLib is inherently slower than some of its alternatives, and as such, these re-compression actions can consume a significant amount of CPU time. 

## Capabilities through Proxies
Before proxies were a thing, most servers were composed of a single game server. That game server was overloaded by players fairly quickly and had to use multi-world capabilities to enable game isolation. Proxies brought a completely different kind of Minecraft Server to life: The "Minecraft Networks".
In contrast to regular Minecraft Servers, Networks are composed of multiple, sometimes dozens or even hundreds of Game Servers. Players can switch between servers using commands, entities they can punch or signs they can click. This enabled networks to handle thousands of players by designing their servers to be scalable and simply creating more servers. 
Too many players, need another SkyWars server? Start one up and start shifting players towards that new server when they search for a match.

This has brought us to the Minecraft Multiplayer Experience that we now have. Every major server is using proxies to scale. The Hubs / Lobbies are where the player selects where they want to go. If they click on a game, the server finds a suitable server of that game, and tells the proxy to redirect the player there. 

This major shift in design enabled it for Minecraft servers to scale to thousands and hundreds of thousands of players. (More on scaling in a future post)

## Proxy responsibilities
The proxy has much more responsibility in a certain way than a game server has. A proxy usually handles hundreds of players at once, while game servers only handle a couple dozen. It is for that reason that proxies should try to isolate exceptions. If a players connection encounters an exception, you do not want to kick everyone from your proxy, right?
The proxy can act as a central authority to transfer players, kick them, send messages to them, or do a lot of things.
I personally believe that the proxy is only supposed to do the part of the business logic that cannot be taken care of by the downstream servers.

## Security
### The XBOX Authentication problem
When you set up a proxy, you will always be asked to enable xbox authentication on your downstream servers. That is necessary because the XBOX login credentials can only be verified by the original server, not any of the servers coming after. This, however, causes a highly dangerous security issue: If someone finds out the IP/Port combination of your game servers, they can log onto there with any username they want, since they do not need to be authorized by XBOX anymore.

### Proxies as entry-gates
In a good network setup, proxies act as entry gates.
Your network can only be entered through the proxies. Traffic inside the network can only be sent by the game servers and the proxies. The game servers are only reachable by the the proxies, they cannot be reached by anything else.
This can be done fairly easy using for example IPTables on linux:

```bash
iptables -A INPUT -p udp -s my-proxy-ip --dport 19133 -j ACCEPT
iptables -A INPUT -p tcp --dport 19133 -j DROP
```

Now what does this do?
- The first line specifies to allow traffic from the proxy to the program running on UDP port 19133.
- The second line specifies that all traffic which is not explicitly allowed by another IPtable rule for this port shall be dropped.

In general: Only the proxy can connect to the game server


## Conclusion
I've tried to talk about some of the interesting things about proxies here, however there are much more. I will likely alter this post to include even more things afterwards.

The TL;DR is: Proxies are very useful pieces of software that allowed Minecraft to grow way past what it was initially capable of.

A good example of an open source proxy is (WaterdogPE)[https://waterdog.dev].
