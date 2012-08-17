DESIGN
=====
This document describes the design of xmppmx. In particular, it describes which Erlang processes will be spawned (which, coincidentally, each handle one set of capabilities needed to facilitate an XMPP-to-XMPP proxy).

PARTS
=====
I'm not using the word 'components' here for a reason: One of the parts of this project actually is what is known as a XMPP Component (which is very much like a common client, but with a different authentication mechanism, purpose and permissions - please see XEP-0114 for details).

 * Supervisor
  * This process launches the remaining ones and also monitors their health, restarting them if necessary (e.g. if the Storage process suddenly dies, Supervisor will launch it again).
  * Also the first process to be launched, i.e. this is the program's "main" function.
 * Storage
  * Administers a Mnesia database to persist information about registered users and configuration.
 * Configuration
  * Manages xmppmx's configuration.
  * Receives requests to set or get configuration values
  * Sends out information about configuration changes
 * Component
  * This is the XMPP component as defined in XEP-0114. It provides what clients see as the transport.
  * Handles registration (XEP-0077), JID escaping (XEP-0106, actually returns escaped values in their escaped form using jabber:iq:gateway) and all other interactions as defined in XEP-0100 (Gateway Interaction).
 * Translation
  * Translates messages between the transport's domain and the domains we're connecting to
  * Receives messages from both Client and Component processes and forwards them to the respective counterparts
 * Client
  * This part acts as a XMPP client on behalf of registered users.

Note that multiple instances of some of these parts may exist. In particular, Component, Translation and Client may exist multiple times (Translation and Client may actually be spawned once for each registered account), whereas Supervisor, Storage and Configuration should not be spawned more than once (unless one of them dies).


DETAILED BEHAVIOUR
=====
This section describes the behaviour of each part in detail. Since this is the most natural way to describe Erlang apps, this description mostly consists of reactions to incoming messages.

Supervisor
-----
 * Startup
  * Launches a Storage process
  * Lanuches a Configuration process
  * Retrieves the configuration for all Component instances that shall be started (in general, one transport shall be started for each foreign domain)
  * Launches Component instances
 * Storage dies
  * If the reason is a shutdown, ignore it
  * Otherwise, launch a new Storage instance and propagate its Pid to the Configuration instance and all Components
 * Configuration dies
  * If the reason is a shutdown, ignore it
  * Otherwise, aunch a new Configuration instance and propagate its Pid to all Components
 * Shutdown
  * Send shutdown signals to all Components and the Configuration and Storage processes

Storage
-----
