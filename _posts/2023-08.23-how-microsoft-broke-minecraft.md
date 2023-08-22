# How Microsoft broke all pre-1.20 Minecraft clients

Just two hours ago, many players playing Minecraft: Bedrock Edition started facing authentication errors when trying to log into any Minecraft server while using a Minecraft version pre-1.20. This sometimes happens when Minecraft servers are down, however this time, it is very likely permanent. This is a technical / non-technical explanation on what happened.

## Non-Technical
Whenever you want to play on online servers in Minecraft, you have to authenticate with Minecrafts Multiplayer Service. The multiplayer service, after you authenticated yourself, gives your client a bundle of information that tells you some things about the user, for example, what his username and XBOX ID are. 
This data is also passed on directly to the servers that the player connects to later. 

To make sure that the data the clients send to the server and the client receives itself is authentic, Microsoft uses something called a JWT. Its a programmatic standard to present data in a way where it can be made sure that data you receive is originating from a certain (trusted) source. This works with clever math which I'm not going to explain here.
THe short version of it is: There are two values, a private key and a public key. The private key is, as the name implies, private. Only Microsoft has it. The public key however can be shared in whatever way you want without any issue. 
The Private Key is able to mathematically sign data, and the public key is later able to verify that data was previously signed using the private key. This also means that anyone who has the private key will be able to create trusted data.

Minecraft uses this by signing the data from the authentication service using their private key, and the client as well as third-party servers use the public key to validate that the data is indeed from Microsoft and not forged or otherwise manipulated.

Recently, Microsofts private key has apparently been compromised, where and how is not really known. If someone had the private key, they could authenticate as any player they want since they could just sign any authentication data they want. This is of course a huge security issue and Microsoft promptly reacted by planning to replace the old private key, which was compromised, with a new one. A clear date and time for this was not known. 

When the private key changes, so does the public key.

### The big issue
All client versions have the private key contained inside the game code. This is used, as explained above, to validate the integrity of the player data. That is an issue because that means the client will try to validate data signed with the new private key with the old public key, which will fail and thus cause the client to treat the (correct) data it receives from Microsofts Authentication service as invalid or untrusted, thus not logging you in. 
Because the client doesn't think its logged in, it will also not send your player data to the servers, which means they think you are not actually logged into Xbox Live. 

That all leads to all clients pre-1.20 not being able to join any Minecraft Server that has XBOX authentication available.

### Why even care
Some might ask why you would even support older game versions; In the end, newer versions have more features and bugfixes. 
However, a lot of players, especially from the Bedrock PvP community have been using older client versions for a long time, mainly for performance and stability reasons. This is why it was also a priority of many of the big bedrock servers to support the old versions as well as the current version, to give players the ability to play with old versions to improve their gameplay experience.

### TL;DR
Microsoft permanently broke all pre-1.20 bedrock clients unless they are reverting to the old private key, which would be a massive security risk and is not likely to happen. There is nothing any server can do about this.