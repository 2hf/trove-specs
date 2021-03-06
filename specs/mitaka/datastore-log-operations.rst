..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

 Sections of this template were taken directly from the Nova spec
 template at:
 https://github.com/openstack/nova-specs/blob/master/specs/juno-template.rst

..
  This template should be in ReSTructured text. The filename in the git
  repository should match the launchpad URL, for example a URL of
  https://blueprints.launchpad.net/trove/+spec/awesome-thing should be named
  awesome-thing.rst.

  Please do not delete any of the sections in this template.  If you
  have nothing to say for a whole section, just write: None


=============
Trove Logging
=============

Provide end-user access to various type of logs on guest instances.

Launchpad Blueprint:
https://blueprints.launchpad.net/trove/+spec/datastore-log-operations


Problem Description
===================

In the current implementation, it is not possible for the user to
retrieve any logs from the guest agent without ssh access to the
instance.  The user should be able to retrieve logs as defined by the
datastore.


Proposed Change
===============

This document outlines a proposal to provide access to guest logs by
storing them in a Swift container.

Each datastore will define a number of log files which it can make
available to the user.  Logs can be either system logs such as the
Trove guest-agent log which are always enabled, or the log can be user
logs such as the MySQL Slow Query Log which can be enabled or disabled
by the user.

The contents of the appropriate log files will be copied from the
guest instance to a Swift container via execution of a log-publish
command, and may then be streamed to the user.  Subsequent invocations
of the log-publish command will only copy log entries made since the
last log-publish command to efficiently utilize network resources and
minimize impact on database performance.

Swift Log Container
-------------------

Each log will be composed of multiple objects stored in a Swift container (with
a predefined prefix), with each object being a subset of the logging
information.  The entire log will be reconstructed by concatenating the objects
in the container that pertain to the specified log; order can be determined by
examining the object metadata for each object, or simply by alphabetically
sorting the objects by filename (assuming a suitable naming convention).  The
prefix will be generated by using a know pattern.  All the corresponding files
can then be retrieved from Swift using this prefix.  The suggested prefix will
be: '%(instance_id)s/%(datastore)s-%(log)s/'

|

Each log will store a metadata file in the container, which will have the
following information associated with it:

============   ===========
Key            Value
============   ===========
Log Name       Name of log
Log Type       SYS or USER
Log File       File where data is published from
Log Hash       Hash of log file
Log Size       Log file size at last publish
Log Lines      Log file line count at last publish
============   ===========

|

A list of all the instances with logs in the container could be viewed with the
following command::

  $ swift list database_logs --delimiter='/'

Response::

  fa8c452e-6568-438a-9f04-b2878290fb90/

|

A list of all the logs for an instance could be viewed with the following
command::

  $ swift list database_logs --delimiter='/' --prefix fa8c452e-6568-438a-9f04-b2878290fb90/

Response::

  fa8c452e-6568-438a-9f04-b2878290fb90/mysql-guest/
  fa8c452e-6568-438a-9f04-b2878290fb90/mysql-guest_metafile

|

A list of all the actual log files associated with a specific datastore log
could be viewed with the following command::

  $ swift list database_logs --delimiter='/' --prefix fa8c452e-6568-438a-9f04-b2878290fb90/mysql-guest/

Response::

  fa8c452e-6568-438a-9f04-b2878290fb90/mysql-guest/log-2015-11-18T21:28:09.975754

|

Configuration
-------------

Datastore CONF settings:

- guest_log_exposed_logs - Used to configure which logs will be exposed by
  the logging component
- guest_log_container_name - pattern used to generate the name of the
  contain which will hold the logs
- guest_log_long_query_time - the time to set to identify if a query is
  taking a long time
- guest_log_limit - max size (in bytes) for any chunk pushed to Swift
- guest_log_expiry - time in seconds after which the log component is
  removed from the Swift container (0 for no expiry)

guest_log_exposed_logs
......................

A guest_log_exposed_logs config value will enumerate the logs available for a
datastore.  By default, the guestagent log would be supported by all
datastores plus any other logs defined and supported by the datastore's
guestagent manager.

Excerpt from trove-guestagent.conf::

   guest_log_exposed_logs = error,guest,slow_query,general

Any USER log not mentioned in the guest_log_exposed_logs list will be disabled
by default.  A special value of "ALL" will enable all logs supported by
the guest agent manager.

An admin user will always be able to see all the logs, and to publish and
view the contents, regardless of whether they are 'exposed' or not.

guest_log_container_name
........................

The name of the container used to store log components can be
specified in the configuration file.

Excerpt from trove-guestagent.conf:

    guest_log_container_name = database_logs

The value shown above is the default value.

guest_log_long_query_time
..........................

The amount of time to set for 'slow_query' logging.  This value
is datastore specific, and may mean different things for different
datastores.  For example, MySQL has a slow query log that these queries
are written into, whereas PostgreSQL would use the field to decide what
queries to write into its general log.


Log Rotation
------------

Many systems will use log rotation to ensure that logs do not exceed
the amount of available disk space on a system.  At any point in time,
the current log file could be renamed to "<logfile>.1" (or some other
name) and a new log file started for ongoing log messages.

To account for this, the logging feature will keep track of a hash of
the first line of the current log file that exists during a
log-publish operation.  The current hash value will be stored in the
x-container-meta-log-header-digest value associated with the log file
container.  Subsequent log-publish operations will use the hash value
to determine whether the log has indeed been rotated.  If so, the
current container will be purged and the new log file published to it.


Database
--------

n/a


Public API
----------

For log-list:

Request::

    GET v1/instance/{id}/log

Response::

    {
        'logs' : [
            {
                'name': 'guest',
                'type': 'SYS',
                'status': 'Ready',
                'published': '0',
                'pending': '4234',
                'container': 'None'
                'prefix': 'None',
                'metafile': '<id>/mysql-guest_metafile',
            },
            {
                'name': 'general',
                'type': 'USER',
                'status': 'Disabled',
                'published': '0',
                'pending': '0',
                'container': 'None'
                'prefix': 'None',
                'metafile': '<id>/mysql-general_metafile',
            },
            {
                'name': 'slow_query',
                'type': 'USER',
                'status': 'Partial',
                'published': '1009',
                'pending': '304',
                'container': 'database_logs'
                'prefix': '<id>/mysql-slow_query/',
                'metafile': '<id>/mysql-slow_query_metafile',
            },
        ]
    }


For log-show:

Request::

    POST v1/instance/{id}/log
    { 'name': 'general' }

Response::

    {
        'log': {
            'name': 'guest',
            'type': 'SYS',
            'status': 'Partial',
            'published': 218913,
            'pending': 2636234
            'container': 'database_logs',
            'prefix': '<id>/mysql-guest/',
            'metafile': '<id>/mysql-guest_metafile',
        }
    }


For log-enable:

Request::

    POST v1/instance/{id}/log
    { 'name': 'general', 'enable': 'True' }

Response::

    {
        'log': {
            'name': 'general',
            'type': 'USER',
            'status': 'Enabled',
            'published': '0',
            'pending': '0',
            'container': 'None'
            'prefix': 'None',
            'metafile': '<id>/mysql-general_metafile',
            }
        ]
    }


For log-disable:

Request::

    POST v1/instance/{id}/log
    { 'name': 'general', 'disable': 'True' }

Response::

    {
        'log': {
            'name': 'general',
            'type': 'USER',
            'status': 'Disabled',
            'published': '30103',
            'pending': '0',
            'container': 'log-mysql-general-<id>'
            'prefix': '<id>/mysql-general/',
            'metafile': '<id>/mysql-general_metafile',
            }
        ]
    }


For log-publish:
(Note that 'publish' will automatically 'enable' a log)

Request::

    POST v1/instance/{id}/log
    { 'name': 'general', 'publish': 'True' }

Response::

    {
        'log': {
            'name': 'guest',
            'type': 'SYS',
            'status': 'Published',
            'published': '443',
            'pending': '0',
            'container': 'log-mysql-guest-<id>'
            'prefix': '<id>/mysql-guest/',
            'metafile': '<id>/mysql-guest_metafile',
            }
        ]
    }


For log-discard

Request::

    POST v1/instance/{id}/log
    { 'name': 'general', 'discard': 'True' }

Response::

    {
        'log': {
            'name': 'general',
            'type': 'USER',
            'status': 'Ready',
            'published': '0',
            'pending': '30103',
            'container': 'None'
            'prefix': 'None',
            'metafile': '<id>/mysql-general_metafile',
            }
        ]
    }


Python API
----------

::

    def log_list(self, instance):
        """Get a list of all guest logs.

        :param instance: The :class:`Instance` (or its ID) of the database
        instance to get the log from.
        :rtype: list of :class:`DataStoreLog`.
        """

    def log_show(self, instance, log):
        """Show details of a log.

        :param instance: The :class:`Instance` (or its ID) of the database
        instance to get the log from.
        :param log: The type of <log> to enable
        :rtype: List of :class:`DataStoreLog`.
        """

    def log_enable(self, instance, log):
        """Enable the writing of a log.

        :param instance: The :class:`Instance` (or its ID) of the database
        instance to get the log from.
        :param log: The type of <log> to enable
        :rtype: List of :class:`DataStoreLog`.
        """

    def log_disable(self, instance, log):
        """Disable the writing of a log.

        :param instance: The :class:`Instance` (or its ID) of the database
        instance to get the log from.
        :param log: The type of <log> to disable
        :rtype: List of :class:`DataStoreLog`.
        """

    def log_publish(self, instance, log, disable=None, discard=None):
        """Publish guest log to Swift container.

        :param instance: The :class:`Instance` (or its ID) of the database
        instance to get the log from.
        :param log: The type of <log> to publish
        :param disable: Turn off <log>
        :param discard: Delete the associated container
        :rtype: List of :class:`DataStoreLog`.
        """

    def log_discard(self, instance, log):
        """Discard (delete) the published log container in Swift.

        :param instance: The :class:`Instance` (or its ID) of the database
        instance to get the log from.
        :param log: The type of <log> to discard
        :rtype: List of :class:`DataStoreLog`.
        """

    def log_generator(self, instance, log, publish=None, lines=50):
        """Return generator to yield the last <lines> lines of guest log.

        :param instance: The :class:`Instance` (or its ID) of the database
        instance to to get the log from.
        :param log: The type of <log> to publish
        :param publish: Publish updates before displaying log
        :param lines: Display last <lines> lines of log (0 for all lines)
        :rtype: generator function to yield log as chunks.
        """

    def log_save(self, instance, log, publish=None, filename=None):
        """Saves a guest log to a file.

        :param instance: The :class:`Instance` (or its ID) of the database
        instance to get the log from.
        :param log: The type of <log> to publish
        :param publish: Publish updates before displaying log
        :rtype: Filename to which log was saved
        """

CLI (python-troveclient)
------------------------

Log List
........

The log-list command provides information about each log available on
a given Trove instance.

::

    $ trove log-list <instance>
    +------------+------+-------------+-----------+---------+---------------+---------------------+
    | Name       | Type | Status      | Published | Pending | Container     | Prefix              |
    +------------+------+-------------+-----------+---------+---------------+---------------------+
    | error      | SYS  | Unavailable |         0 |       0 | None          |                     |
    | general    | USER | Published   |      1009 |       0 | database_logs | <id>/mysql-general/ |
    | guest      | SYS  | Ready       |         0 |  499850 | None          |                     |
    | slow_query | USER | Disabled    |         0 |       0 | None          |                     |
    +------------+------+-------------+-----------+---------+---------------+---------------------+


+-------------+---------------------------------------------------------------+
+ Column      + Description                                                   +
+=============+===============================================================+
+ Name        + Name of the log component                                     +
+-------------+---------------------------------------------------------------+
+ Type        + SYS: System log, always on                                    +
+             +---------------------------------------------------------------+
+             + USER: Managed by user                                         +
+-------------+---------------------------------------------------------------+
+ Status      + Disabled: Inital state of USER log                            +
+             +---------------------------------------------------------------+
+             + Enabled: Initial state of a SYS log or a USER log with no     +
+             + data in it                                                    +
+             +---------------------------------------------------------------+
+             + Unavailable: SYS log that has no data in it                   +
+             +---------------------------------------------------------------+
+             + Ready: Log has data available for publishing                  +
+             +---------------------------------------------------------------+
+             + Published: Log file has been fully published                  +
+             +---------------------------------------------------------------+
+             + Partial: Log file has been partially published                +
+             +---------------------------------------------------------------+
+             + Rotated: Log file has rotated, so next publish will delete    +
+             + the container first                                           +
+             +---------------------------------------------------------------+
+             + Restart Required: Datastore requires a restart in order to    +
+             + begin writing to the log file                                 +
+             +---------------------------------------------------------------+
+             + Restart Completed: Internal state so the guest log knows to   +
+             + begin reporting the actual state again                        +
+-------------+---------------------------------------------------------------+
+ Published   + Amount of data published to container                         +
+-------------+---------------------------------------------------------------+
+ Pending     + Amount of data available to be published by log-publish       +
+-------------+---------------------------------------------------------------+
+ Container   + Swift container that holds the components of the log          +
+-------------+---------------------------------------------------------------+
+ Prefix      + Prefix to send to Swift to get just the relevant log parts    +
+-------------+---------------------------------------------------------------+

**Note: Where the values for 'Container' and 'Prefix' for the logs in the
example above are 'None,' this signifies that that log has not had a
log-publish operation executed against it**


Log Show
........

The log-show command provides full information about a specific log available
on a given Trove instance.

::

   $ trove log-show <instance> general
    +--------------+-----------------------------+
    | Property     | Value                       |
    +--------------+-----------------------------+
    | name         | slow_query                  |
    | type         | USER                        |
    | status       | Enabled                     |
    | published    | 135                         |
    | pending      | 2156                        |
    | container    | database_logs               |
    | prefix       | <id>/mysql-slow_query/      |
    | metafile     | <id>/mysql-general_metafile |
    +--------------+-----------------------------+


Log Enable
..........

The log-enable command will instruct the guest agent to begin writing
information to the specified log file.  Only 'USER' logs can be enabled
as 'SYS' logs are enabled by default (and cannot be disabled).  Depending
on the datastore, this may cause the log to go into a 'Restart Required'
state where it will remain until the datastore is restarted.  This can
be configured on a per-datastore basis, and should only be done if there
is no way to dynamically start the logging (i.e. PostgreSQL must be
restarted in order to change logging, so it would require this configuration).

::

   $ trove log-enable <instance> general
    +--------------+-----------------------------+
    | Property     | Value                       |
    +--------------+-----------------------------+
    | name         | general                     |
    | type         | USER                        |
    | status       | Enabled                     |
    | published    | 0                           |
    | pending      | 0                           |
    | container    | None                        |
    | prefix       | None                        |
    | metafile     | <id>/mysql-general_metafile |
    +--------------+-----------------------------+


Log Disable
...........

The log-disable command will instruct the guest agent to stop writing
information to the specified log file.  Only 'USER' logs can be disabled.
As with log-enable, this may cause the log to go into a 'Restart Required'
state.  See log-enable for more details.

::

   $ trove log-disable <instance> general
    +--------------+-----------------------------+
    | Property     | Value                       |
    +--------------+-----------------------------+
    | name         | general                     |
    | type         | USER                        |
    | status       | Disabled                    |
    | published    | 34658                       |
    | pending      | 2532                        |
    | container    | database_logs               |
    | prefix       | <id>/mysql-general/         |
    | metafile     | <id>/mysql-general_metafile |
    +--------------+-----------------------------+


Log Publish
...........

The log-publish command will instruct the guest agent to push any
updates to the specified log to the Swift container, which will be
created if required.  One log-publish command could result in multiple
objects being pushed to the Swift container in order to keep each
object below the maximum object size as configured by the
guest_log_limit CONF value.

The log-publish command will execute asynchronously.  When the
log-publish command is executed, the Trove instance will be put in the
LOGGING state, returning to ACTIVE when objects have been pushed to
the logging container so as to successfully finish execution of the
command.

When an object is pushed to the Swift container, an X-Delete-After
header is used to specify a time-to-live for the container
object. This will result in objects automatically being removed from
the container after a period of time as specified by the
log_expiry CONF value.

An optional --disable parameter will be supported to disable logging
for a particular USER log.  An optional --discard parameter will be
supported to first discard (delete) the associated container.

::

   $ trove log-publish <instance> slow_query
    +--------------+--------------------------------+
    | Property     | Value                          |
    +--------------+--------------------------------+
    | name         | slow_query                     |
    | type         | USER                           |
    | status       | Published                      |
    | published    | 43242                          |
    | pending      | 0                              |
    | container    | database_logs                  |
    | prefix       | <id>/mysql-slow_query/         |
    | metafile     | <id>/mysql-slow_query_metafile |
    +--------------+--------------------------------+


Log Discard
...........

The log-discard command will discard (delete) the container where the current
log information resides.

::

   $ trove log-discard <instance> general
    +--------------+-----------------------------+
    | Property     | Value                       |
    +--------------+-----------------------------+
    | name         | general                     |
    | type         | USER                        |
    | status       | Enabled                     |
    | published    | 0                           |
    | pending      | 37190                       |
    | container    | None                        |
    | prefix       | None                        |
    | metafile     | <id>/mysql-general_metafile |
    +--------------+-----------------------------+


Log Tail
........

By default, log-tail outputs the 50 lines at the end of the
log.  With the --lines=n option, log-tail will output the last n
lines of the log.  If n is negative, output will start past line n and
continue to the end; --lines=0 will output the entire log.

With the --publish option, log-tail will first execute a log-publish
command and wait for the log to be published before beginning output.

It should be noted that the actual display of the log will take place
in the Python API only.  There will be no REST APIs to facilitate
display of the log; such APIs would put undue stress on the system due
to the requirements of buffering and streaming from Swift.

::

  $ trove log-tail <instance> slow_query --publish
  /usr/local/mysql/libexec/mysqld, Version: 3.23.54-log, started with:
  Tcp port: 3306  Unix socket: /tmp/mysql.sock
  Time                 Id Command    Argument
  # Time: 030207 15:03:33
  # User@Host: wsuser[wsuser] @ localhost.localdomain [127.0.0.1]
  # Query_time: 13  Lock_time: 0  Rows_sent: 117  Rows_examined: 234
  use wsdb;
  SELECT l FROM un WHERE ip='209.xx.xxx.xx';


Log Save
........

Like the log-tail command, the log-save command will execute in the
python-troveclient, but where log-tail will output the log to the
console, the log-save command will save the log into a file in the
filesystem.  This will allow the user to download extremely large log
files without overwhelming the client or browser.

With the --file option, the log will be saved to the named file.
Without the --file option, the log will be output to <logname>.log in
the current directory.

With the --publish option, log-tail will first execute a log-publish
command and wait for the log to be published before beginning output.

It should be noted that the actual saving of the log will take place
in the Python API only.  There will be no REST APIs to facilitate
display of the log; such APIs would put undue stress on the system due
to the requirements of buffering and streaming from Swift.

::

  $ trove log-save <instance> slow_query --file=/tmp/my.log --publish


Public API Security
-------------------

The Swift containers will be created with the same security
credentials as used for backup.


Internal API
------------

Appropriate API methods will be added to both the task manager and the guest.

No changes to existing APIs are foreseen.


Guest Agent
-----------

No compatibility issues are foreseen.


Alternatives
------------

Alternate API
.............

An alternative API could be implemented that does away with some
of the simpler commands.

Note:  This alternate section was modified after describing the process to
documentation personnel, where it was determined that having the 'simple'
commands made it significantly easier to understand the logging process.

|

Openstack functionality:

==============   ============
API              Description
==============   ============
log-list         As above
log-show         Removed
log-enable       Removed
log-disable      Removed
log-publish      As above
log-discard      Removed
log-tail         As above
log-save         As above
==============   ============

--publish by default
....................

There has also been a suggestion to make the --publish option the
default on the log-tail and log-save commands.  If this were done, it
seems unlikely that anyone would ever execute --no-publish and the
log-publish command would really only ever execute "log-publish
--disable", so it would be better to just eliminate the
--publish/--no-publish options, have log-tail and log-save always
publish first, and replace the log-publish command with log-disable.

Admin override for guest log
............................

It is possible for the operator to exclude the guest agent log from the list of
logs returned to the user.  Doing so, however, will not prevent the operator
from seeing the guest log, as they would access it through the admin account.
Currently the admin user will still stream the logs to the same tenant as the
Trove user.  This could be enhanced (in the future) to have the admin user
provide a tenant that could be used to host the log containers.


Dashboard Impact (UX)
=====================

TBD (section added after approval)


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  vgnbkr (morgan@tesora.com)
Secondary assignees:
  peterstac (peter@tesora.com)
  atomic77 (atomic@tesora.com)

Documentation:
  laurelm (lmichaels@tesora.com)


Milestones
----------

Target Milestone for completion:
  Mitaka-1

Work Items
----------

This component has been largely implemented.


Upgrade Implications
====================

As this is entirely new functionality, no upgrade implications are foreseen.


Dependencies
============

n/a


Testing
=======

An integration scenario test will be added to retrieve the guest log, execute a
command, retrieve the guest log again and confirm that additional logging
details were captured.  The log will then be deleted via the log-discard
command and the removal of the files from the container from Swift will be
verified.  The TestHelper class will be enhanced to include the 'default' log
list (which is simply 'guest'), and the MySQL one the additional defined USER
logs.


Documentation Impact
====================

New documentation describing the added functionality will need to be
written.


References
==========

https://etherpad.openstack.org/p/trove-2015-vancouver-logging

Appendix
========

None
