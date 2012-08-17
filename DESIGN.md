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
 * Logger
  * Writes log information to a specified file
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
Note that log messages are not 

Supervisor
-----
 * Startup
  * Launches a Storage process
  * Lanuches a Configuration process
  * Calls Configuration ! GetConfiguration to retrieve the configuration for all Component instances that shall be started (in general, one transport shall be started for each foreign domain)
 * Configure
  * Launches a Configuration instance with the given data
 * LanuchComponents
  * Launches Component instances
 * Storage dies
  * If the reason is a shutdown, ignore it
  * Otherwise, launch a new Storage instance and propagate its Pid to the Configuration instance and all Components
 * Configuration dies
  * If the reason is a shutdown, ignore it
  * Otherwise, aunch a new Configuration instance and propagate its Pid to all Components
 * DistributeConfigurationChange
  * Sends the new value of a configuration variable to all running Components, Configuration and Storage processes
 * Shutdown
  * Send shutdown signals to all Components and the Configuration and Storage processes

Storage
-----
 * Startup
  * Opens the xmppmx Mnesia database
  * If the schema is not yet propagated with initial values, read the values from the configuration file, create tables and fill in appropriate values
 * GetConfigurationOption
  * Retrieves the specified configuration option and sends a DistributeConfigurationChange message to Supervisor
 * GetConfiguration
  * Retrieves the entire global configuration and calls Supervisor ! Configure
 * GetUsers
  * Retrieves the users of the given Component and pushes a Component ! AddUser message for each of them
 * AddUser
  * Stores a user in the database. If the user did not exist already, sends a Component ! AddUser message.
 * SetConfiguration
  * Stores a configuration option and upon success calls Supervisor ! DistributeConfigurationChange
 * Shutdown
  * Closes the database
  * Dies with a reason of "shutdown".

Configuration
-----


COMMAND LINE PARAMETERS
======
The following parameters may be passed as command line parameters:
 * directory **REQUIRED**
  * The directory for storing the Mnesia database in

CONFIGURATION
======
The global configuration consists of the following values:
 * log-level
  * One of "ERROR", "WARNING", "INFO" or "DEBUG"
 * log-file
  * Self-explanatory. Defaults to ${dir}/xmppmx.log.
 * allowed-servers
  * A list of domains from which users are allowed to be registered. Accepts wildcards. '*' allows access from all domains.
 * admins
  * A list of JIDs that may set configuration values, query internal data and kill components
 * require-tls
  * Whether to require TLS when connecting to the local server

The following configuration options are set per component:
 * name
  * The human-interpretable name of this component, will also be used in Discovery replies
 * jid
  * The component\'s JID
 * ip
  * The IP of this component
 * port
  * The component port
 * secret
  * The component secret, used to authenticate this component with the server
 * instructions
  * A string used to instruct the user how to perform registration (this is sent along with the registration form)
 * remote-server
  * The remote domain to connect this instance to
 * remote-connect-server
  * If non-blank (or non-void), the actual server to connect to (instead of using DNS SRV)
 * remote-port
  * The port on the remote server to connect to
 * remote-require-tls
  * Require TLS when connecting to the remote server
