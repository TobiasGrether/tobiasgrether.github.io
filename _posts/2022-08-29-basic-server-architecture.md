## Basic Server Architecture

### Introduction
To be able to any of the deeper topics, a very basic knowledge of what a Minecraft Server is, how it works
and all of that is required.

Here I will explain the rough basics on how all of that works.

### Server & Client
Minecraft follows the "Server-Client" principle to implement multiplayer.
The **Server** is a program, for example your local Minecraft Client when inside a world. It dictates how a world and everything inside of it behaves. And then there are **Clients**. 
A Client is a single player who joins that server. Those clients are usually what you see when you are playing Minecraft itself: Its a graphical representation of what it receives from the server. A client can (usually) only be connected to one server at a time, but a server can accept an arbitrary of players at the same time, as long as the server can handle that. 
When a player moves, the client sends a movement update to the server, which the server can then broadcast to all players who need to see that movement.

### Communication
Now, that sounds fairly simple. The real question is how the server and the client exchange information. 
Minecraft: Bedrock Edition utilizes the "RakNet" transport protocol to transmit information, which is built upon UDP. Now, UDP is essentially the most basic way to get data from one computer to another in the internet: You send some information to a certain address and thats about it. 
While this is really simple and really fast, for a game like Minecraft we definitely need information to be transmitted reliably; we don't want our chat message we just sent to get lost on the way to the server.
To guarantee this, RakNet implements methods to guarantee a data packets reliability (which, to be fair, kind-of defeats the purpose of using UDP).

On top of this, Minecraft builds its own protocol. The protocol is like a language. To understand it, both ends of the conversation have to speak it.
There are different actions that might occur throughout the game: Sending chat messages, moving, spawning a player, placing blocks. All of those different actions have different parameters: Sending messages will require the message content, while placing a block will require the position where the block is supposed to be.
To realize this, Minecraft implements those actions as **packets**. Those packets have unique numeric identifiers (*packet IDs*) which make it possible for the receiving end to tell them apart.

#### Types
Every packet defines for itself how to read or write the data it contains. Different types of data can be
expressed, for example numeric values, with a fixed length: *bytes* (1 byte), *shorts*, (2 bytes), *integers* (4 bytes), *longs* (8 bytes). All of those types can either be signed or unsigned, indicating whether they can be positive or negative. Unsigned values are not able to express negative numbers, which also means that they have one bit more to their disposal, which is the bit that signed numbers use to indicate whether they are positive or negative. 
On top of that, Minecraft implements so called *VarInts*, which is short for *Var*iable *Int*eger. These are integers which, unlike the previously mentioned numeric types, do not have a fixed length.

Minecraft is also able to write characters. Characters have a numeric representation, which is being used here. Writing a certain number of characters, for example a text message, is also possible. This is also called a *String*. Strings are expressed by first writing the number of characters the string contains, followed by the characters that the string contains.

#### Example
Taking a look at an example packet: The packet to send Chat messages (PlayerChatPacket).
The ID of this packet is 4 (or 0x04 in hex).
It contains the following data fields:
- *Message*: **String** - The message content
- *Timestamp*: **Instant** - The time at which the message was sent
- *Salt*: **Long** - A number used for verification
- *Signature Length*: **VarInt** - The length of the Signature array.
- *Signature*: **ByteArray** - Signature used to verify the message

Now the fields themselves might look confusing to you. But they themselves do not matter as much as the principle they follow: They are ***clearly defined***. To be able to correctly read the chat message, the sender and the receiver of the packet have to know the same definition of this packet.  


### What the server does
The server itself is handling everything that happens in the game. As such, it handles:
- Vanilla Behaviour
- Terrain Generation & Population
- Player input/movement handling
- World Events
- Anything plugins might do

This is quite a heavy bunch of workloads for servers to handle. Generally, servers are not able to handle an unlimited amount of players, due to hardware constraints: Any computer only has so much compute power.

### Summary
So what is a server? A server accepts traffic for multiple players and handles the actions they perform

