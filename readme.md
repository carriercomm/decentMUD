**DecentMUD 1.0**
==============

Jessica YANG

Fall 2014

CPSC 426 Final Project

Note for GitHub users
=====================
DecentMUD currently contains some of the ugliest code I have written in my entire life, unfit for public display, and so the repository currently contains just the executable. -- Jessica Yang 12/23/2014 5:18:11 PM 

Overview  
========
For this project I will be creating a slight variant on the traditional
multi-user-dungeon (MUD) game. Such games were popular before being usurped by
graphical multiplayer online games, but are still played by (diminishing)
numbers of people, including, sometimes, myself. 

Multi-user Dungeons
-------------------
A MUD is usually comprised of a
set of interconnected "rooms", which contain items and non-player characters.
The game world (i.e. rooms, objects) and state are hosted on a central server,
and then players can connect to the server and interact with the game world. The
player interacts with the game world through a text-based interface that may
sometimes make use of ASCII art, but mostly through descriptions of what is
happening in the world.

DecentMUD
--------------
Instead of having all players connect to a central server, I implemented a fully decentralized game world that takes advantage of the existing
Peerster network structure. It will still use the same basic concept of
interconnected rooms and borrows terminology and gameplay from traditional MUDs.

Furthermore, in addition to the core game functionality, the game will also
include an implementation of a fault detection system, in which each node
maintains a record of the actions taken by the player at that node, and this
record can then be requested by other peers, who will examine the record against
a reference implementation and report any inconsistencies.

**Feature Summary**

- Fully decentralized MUD
- Room creation
- Connecting rooms with doors
- Setting descriptions for rooms and players
- Looking at rooms and players
- Point system (allow players to give and receive "gems")
- Accountability system for points


APIs (kind of)
===========
Messaging
-------------
Many types of messages are used in this implementation to carry out
the various actions. The functions used are:

	QVariantMap buildDescMsg(QString type);
	QVariantMap buildGemMsg(QString type);
	QVariantMap buildIPMsg(QHostAddress ip);
	QVariantMap buildIPMsg(QString type);
	QVariantMap buildDoorMsg(QString type);
	QVariantMap buildMoveRequest(QString type);
	QVariantMap buildMoveConfirmed(QString type);
	QVariantMap buildUpdate();
	QVariantMap buildUpdate(QString msg);

As you can see, messages are all wrapped in a `QVariantMap`. Due to 
the constraints of `QVariantMap` (e.g. it requires a considerable amount
of work to get it to accept custom types) certain types of data are
not stored in the most efficient way possible - e.g. instead of having
just one map that maps `nodeID -> (playerName, nodeAddress)` we have to have
two separate maps: `nodeID -> playerName` and `nodeID -> nodeAddress`.

Information
---------------
Each node maintains two main sets of pertinent information: properties
about the room that it is hosting (referred to throughout the code as
`homeRm`) as well as the room that its player is currently standing in 
(`currentRm`).

These properties are set directly in the code without intermediary
getter or setter functions.

Room Creation
--------------
The room is created automatically upon running the executable and
specifying a username, which is validated by a QValidator, and
after submitted the name is further transformed to begin with a capital
letter and end with a lowercase letter by `namify(QString)`.

Connecting rooms with doors
----------------------------

	void addDoor(QString hostname, quint16 port);
	void addDoor(QHostAddress ip, quint16 port);

These functions, called when pressing the Add Door button in the GUI, will
either add that ip:port pair directly, or resolve it to a hostname and add
the peer after it has been resolved. `addDoor()` will not create two doors 
to the same destination in the same room. In more detail, `addDoor()` relies
on the function `buildDoorMsg()` to send requests to establish a new door,
and to receive confirmations that doors have been established on the "other side" so that the node can update its room accordingly.

Command Parser
---------------
The command parser is based on simple regex, for example:

	rxSetRmDesc.setPattern("setrmdesc \"(.*)\"");
	rxLookAt.setPattern("look (.*)");

A match for one of these regex calls a corresponding function that executes
the action(s) related to that command. The full set of commands in v1.0 is:

	void setPcDesc(QString str);
	void setRmDesc(QString str);
	void lookAt(QString str);
	void look();
	void exits();
	void move(QString str);
	void giveGem(QString str);
	void printGems();


Walking between rooms
----------------------
When a command to move player A to room B is read, if there is a door 
available in that direction, A calls `buildMoveRequest()` to inform B that it
is going to enter. B updates its table of players who are in room B, and responds to A. A calls `buildMoveRequest()` again and informs the room that player A is currently in that player A is going to leave. The room receiving
this then removes player A from its table of "home" players. Finally, A updates `currentRmAddr` to the address of room B.

Synchronization
-------------------
One of the two update functions in particular is used to keep all players synchronized about the "true" state of the room, which is known by the host of the room.

	QVariantMap buildUpdate();
	
This function builds a message that contains information about the host's room that is pertinent to the players standing in the room, such as the names of players and the exits, and sends it to everyone standing in the room. Upon receipt of such a message, a player updates their `currentRm` properties.

Looking
----------
The `void lookAt(QString str)` function is a bit more interesting than its sibling `look()`. It messages the target of the look directly to ask for a description, which is then shown to the player.

Gems (Points)
-----------------
A very simple point system in the game is used where players give or receive gems to one another. Everyone starts with 10 and the corresponding messaging occurs using `buildGemMsg()`, which includes subtypes such as "give" and "received". It also uses a mechanism similar to that of door creation, where confirmation of receipt must occur before the local gem count is updated.

Logging
--------
When the executable is first launched, `void initLog()` is called, which creates a log file with the filename `[nodeID].log`. The other function `void log(QString msg)` is called to add a line to the log file. Each entry consists of the format:

	dd.MM.yy hh:mm:ss.zzz    command/message
	                      ^ Tab character

The file is opened and closed every time a log entry is appended.

Cheat Detection (Accountability)
--------------------------------
Each node has a set of witnesses who periodically audit its log. Auditing is done on-line and not after the end of the game.

The witnesses are, in this implementation, all neighbors of the node (i.e. everyone who is joined to the node by a "door"). In other words, each node monitors the nodes who request a direct in-game connection to it. These node IDs are stored in a `QStringList monitoredNodes`. This way we can guarantee that everyone on the network will be monitored by at least one witness. Currently the related variables,  like`monitoredNodes`, are updated directly, without intermediate functions.

### Behavior of monitored nodes
Each node regularly re-uploads its own log file using the `uploadLog()` function. Since we are chunking uploads and keeping track of them by block hashes *a la* Lab 3, there is minimal worry about memory usage getting too large if outdated log files are not regularly deleted. Since logs should be append-only, earlier (complete) chunks will always have the same hash result, and will not be re-uploaded by the application.

### Behavior of witnesses
Once per user-specified interval, a witness node sends a search request for the log file for a monitored node. It also asks the node using `buildGemMsg()` how many gems it currently has.

Auditing the log is done by 'replaying' the events documented in the log. Each node has a reference implementation of sorts, which the reference implementation runs through (deterministcally) to obtain a "correct" state which is then compared with the reported state of the node. 

In v1.0 we detect the following issues:

- Outdated log sent
- Log is not in chronological order
- Player's state* (sent together with log) is inconsistent with actions in log

The progress of the accountability system can be monitored in the terminal window that is running the executable. If a fault is detected, a message is also broadcast to every node on the network, that the player is cheating.

In v1.0 the only player state of interest is the number of gems.

### Cheat codes
A special command, `xyzzy`, can be used to test the anti-cheating system. Running the command `xyzzy` causes the user to no longer lose gems even when `give gems` is used. However, other nodes still receive the gems.


Miscellaneous Functions
------------------------------
	void autoScroll();
Scrolls the text view automatically as new text is entered.

There are some other utility functions in the code that aren't really important.

User Guide
===========
Type "tutorial" in-game to bring up a tutorial that shows some commands you can
use in the MUD.

Sample sequence of commands that can be used for testing:

**Node 1**
1. (Launch executable and enter a name)
2. setdesc "A bedraggled student."
3. setrmdesc "The office has a lived-in look to it. Perhaps too lived-in."
4. (Add a door to Node 2 via the Add Door function)
5. south (or whatever direction the exits command shows)
6. look
7. look *[node 2 player name]*
8. give *[node 2 player name]* gem

**Node 2**

1. (Launch executable and enter a name)
2. setdesc "A powerful professor."
3. setrmdesc "The walls of this room are plastered with news clippings and comic strips."
4. (Wait for Node 1 to create a door)
5. (Wait for Node 1 to enter)
6. give *[node 2 player name]* gem
7. (After this action, the accountability system should report a passed audit.)
8. xyzzy
8. give *[node 2 player name]* gem
9. (After this, the accountability system should report a failed audit and announce that this node is cheating.)

### Full command list
(When x is used, it refers to the name of a player in the room.)

- `setrmdesc "..."`
- `setdesc "..."` 
- `look`
- `look x`
- `north`, `south`, `northeast` etc. as well as `enter portal` (when more than 8 doors are present; all doors from No. 9 onwards are in the portal. Portals were not fully tested, but the v1.0 code definitely supports up to 8 doors per room.)
- `give x gem`
- `gems`

Planned Features
==============
These were planned features dropped due to a lack of time. I have a clear idea of how they would work (can explain upon request) but implementing them would have been quite time-consuming. These include, in no order of priority:

- Automatically kick players who are reported as cheaters after a certain threshold is passed; or, if the peer review system (see 3rd point) is implemented, automatically kick *confirmed* cheaters
- Graceful handling of node disconnecting from network (boot the other players standing in that room onto other rooms that are still alive)
- If witness(es) detect misbehavior they generate evidence and make
it available to other nodes, who verify that the witness is correct, before announcing that the player is cheating.
- Further refinements to the game commands allowed and their presentation
- Add objects to the game: allow creation, pickup and drop (possibly in another room) of arbitrary objects, giving objects to other plays.
- Allow social functions such as `say` or `broadcast` or `emote` (e.g. `emote "is having a bad day"` would print `Jessica is having a bad day`.)

