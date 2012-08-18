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
 * Logger
  * Writes log information to a specified file
 * Component
  * This is the XMPP component as defined in XEP-0114. It provides what clients see as the transport.
  * It also synthesizes the local pseudo JIDs corresponding to remote JIDs
  * Handles registration (XEP-0077), JID escaping (XEP-0106, actually returns escaped values in their escaped form using jabber:iq:gateway) and all other interactions as defined in XEP-0100 (Gateway Interaction).
 * Client
  * This part acts as a XMPP client on behalf of registered users.
  * Translates messages between the local and remote domain and vice versa

Note that multiple instances of some of these parts may exist. In particular, Component, Translation and Client may exist multiple times (Translation and Client may actually be spawned once for each registered account), whereas Supervisor, Storage and Configuration should not be spawned more than once (unless one of them dies).


DETAILED BEHAVIOUR
=====
This section describes the behaviour of each part in detail. Since this is the most natural way to describe Erlang apps, this description mostly consists of reactions to incoming messages.
Note that calls to write log messages are not explicitly documented here; these will be added in the actual implementation. 

Supervisor
-----
 * Startup
  * Launches the Logger process
  * Launches a Storage process
  * Launches a Configuration process
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
  * Sends the new value of a configuration variable to all running Components, Logger, Configuration and Storage processes
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
 * Startup
  * Calls Supervisor ! DistributeConfigurationChange on all appropriate (global configuration) values
  * Calls Supervisor ! LaunchComponents with all individual component configurations
 * GetConfiguration
  * Returns the current value of the configuration option to the sender as Pid ! Info message
 * SetConfiguration
  * If the configuration option exists and the current value of it differs from the new value, sends a Storage ! SetConfiguration message
  * Otherwise, ignores it
 * ConfigurationChange
  * Stores the new value of the option
 * Shutdown
  * Dies with a reason of "shutdown"

Logger
-----
 * Startup
  * Opens the specified log file
  * Stores the current log level
 * Log
  * If the current log level calls for it, writes the specified string (plus information about the log level it occurred on) to the log file
  * Otherwise, ignores the message
 * ConfigurationChange
  * If the value of log-file changed, writes a log message to the current file, close it and open the new log file instead
  * If the value of log-level changed, stores the new value
  * Otherwise, ignore it
 * Shutdown
  * Writes a final log message documenting the upcoming shutdown
  * Closes the log file
  * Terminates with reason "shutdown"

Component
-----
 * Startup
  * Sends presence probes to all registered users
 * Record
  * If presence and not unavailable, determine if this is a registered user and if we have a Client running for the user
   * If not, start a Client for the given user with the current presence information
   * If there is already a client running for this user, send Client ! Iq with the presence
  * Otherwise, send Client ! RecordOut with the presence, which will shut down the Client
  * If message, check if message to raw component or to real JID
   * If raw component...
    * TODO: Registration
    * TODO: Disco
    * TODO: Other interaction (JID translation?)
    * TODO: Administration
   * If real JID, call Client ! RecordOut
 * RecordIn
  * Generate an appropriate stanza from the record and send it to the recipient (a local registered user)
 * Client dies
  * If reason is "shutdown", ignore
  * Otherwise, sends presence probe to corresponding user
 * Shutdown
  * Sends shutdown message to all clients
  * Sends presence unavailable to all registered users
  * Exits with reason "shutdown" 

Client
-----
 * Startup
  * Connect to the remote server with the given credentials, settings and presence
  * Send a roster request
 * translateJIDs()
  * Translates JIDs between networks, operates on Records
  * Depending on the direction:
   * If inbound, translate the JID of the user this Client is logged in as to the JID of the local user and translate remote JIDs to gateway pseudo JIDs
   * If outbound, translate the JID of the local user to the JID of the user this client is logged in as and translate gateway pseudo JIDs to remote JIDs
 * Record
  * TODO: Roster management, roster push (translate remote JIDs and add to local roster; translate roster list after request to roster push [rosterx])
  * Translate JIDs and send Component ! RecordIn
 * RecordOut
  * If presence and unavailable, translates and sends the given stanza out and shuts down the client
  * Translates and sends the given stanza out
 * Shutdown
  * Send presence unavailable as the JID of the logged in user
  * Exit with reason "shutdown"

COMMAND LINE PARAMETERS
======
The following parameters may be passed as command line parameters:
 * directory
  * The directory for storing the Mnesia database in. The default value is the current directory.
 * log-level
  * See the global configuration value below. The default value is "INFO".
 * log-file
  * See the global configuration value below. Defaults to ${dir}/xmppmx.log.

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
