# JEP - Joined Editors Protocol


## Protocol Overview

Communication in a typical JEP setup occurs between a frontend (editor or editor plugin) and a backend (language support logic).
Backends are typically programs which are started to provide a backend service.
There may be several instances of the same backend program running at the same time.

Frontends are typically plugins into existing editors or editors which support JEP natively.
Although one frontend plugin would be sufficient for each kind of editor, there could be several competing plugins for an editor as well.

A frontend will typically connect to several backend instances as multiple different languages/workspaces are used. 
One backend instance however, will be connected to only one frontend.

The protocol is based on messages which are exchanged between frontend and backend via a TCP connection.
Messages from one side don't require a response message from the other side unless explicitly noted.

The message content is represented in JSON, following a certain schema for each kind of message as described below.

When a file is opened in an editor (frontend), the editor will figure out the backend to be used, start it and connect to it.

While the user changes files in the editor, the frontend sends the changes to the backend.
The backend in turn responds by sending various kinds of information as it becomes available (error annotations, syntax highlighting, etc).

When the file or the editor is closed, the backend will be shut down.


## JEP Messages

The JEP protocol is based on message passing, with message content represented in JSON.
There are several kinds of messages which all have a specific structure as described in the sections below.

### Message Format

A JEP message consists of a JSON part and an optional binary part.
The latter is used to transfer binary data efficiently without transcoding or escaping.

A message further has a minimal header which tells the length of the overall message and the length of the JSON part.
The length of the binary part is the difference between the overall length and the JSON length.
Message lengths are given in ASCII characters, separated by a comma and terminated by the opening curly brace of the JSON object:

    <overall length>,<json length>{<json object>}<binary data>

Message length values exclude the header length.

Example:

    25,19{"kind": "request"}xndjs318,18{"next":"message"}

         |<--       25        -->|     |<--     18   -->|
         |<--    19     -->|           |<--     18   -->|

The example shows two messages in a stream.
The first message has an overall length of 25 bytes and a JSON length of 19 bytes.
Thus the binary length is 25 - 19 = 6 bytes.

The second message follows immediately after the first one, having an overall length of 18 bytes, a JSON length of 18 and no binary part.

### Encoding

As the JEP protocol uses JSON, the encoding of messages is UTF-8 by definition.
However, the protocol further restricts UTF-8 to only 7-BIT-ASCII characters, which effectively makes the protocol encoding 7-BIT-ASCII (or US-ASCII), which is valid UTF-8.

Frontends and backends should not apply any transcoding to the data found in input files.
The reason is, that most often information about an input file's encoding is not reliable.
If the assumption of the source encoding is wrong, transcoding just makes things worse: either data is misinterpreted or information is lost due to character replacement.

Instead, data should be passed as is, or in other words: it should be interpreted as "binary" data. 
Since the JEP protocol is restricted to 7-BIT-ASCII, all non 7-BIT-ASCII characters in the JSON part of the messages are escaped using the following pattern: 
Each byte with a value of 0x80 or higher results in three 7-BIT-ASCII characters: a leading "%" and two hexadecimal figures in lower case.
In addition, the character "%" is escaped in the same way, i.e. "%" will always be escaped as "%25" ("%" has byte value 0x25 in 7-BIT-ASCII).

Example:

The word "Übung" (german: exercise), encoded in ISO-8859-1 would result in the string:
"%dcbung" (the "Ü" Umlaut has a byte value of 0xdc is ISO-8859-1).

Note that the binary part of a JEP message doesn't require escaping and thus significantly improves performance.

### Message Schema 

The protocol description below uses a simple notation to specify the structure of the different kinds of messages.
A message schema is defined by the keyword "message" followed by a textual message identifier.

    message <message identifier>

In JSON format, the message identifier is assigned to the property "message" as a string:

    {"message": "<message identifier"}

Any message attributes are specified in curly braces, each on a separate line.
Each property declaration consists of the property name, followed by a colon, an optional multiplicity declaration and the type:

    message <message identifier> {
      <attr1>: [0,*] <type>
      ...
    }

If the multiplicity is not declared it defaults to [1,1], i.e. the property is non optional.
An optional property is expressed by [0,1].
List type property are properties with an upper limit greater than 1, e.g. [0,\*], [1,2], etc.
List type attributes are represented as JSON lists in JSON format.

Message properties directly correspond to JSON properties in the JSON format.

There are the following predefined primitive property types:
* String
* Integer

Enum types are declared with the keyword "enum" with the possible values following in curly braces.

    enum <type identifier> {
      <literal1>
      ...
    }

There is a predefined enum type "Boolean" which takes the values "true" and "false":

    enum Boolean {
      true
      false
    }

Complex types are declared separately by means of the "type" keyword and can be used to declared nested structures.
Complex types contain property declarations just like messages.

    type <type identifier> {
      <attr1>: [0,*] <type>
      ...
    }

In contrast to message identifiers, type identifiers don't show up in the actual JSON representation.
Instead the type is implied by the corresponding property declaration.

As an example, consider the following declaration:

    message FoodOrder {

    }

    type 


## Backend Startup

There needs to be a backend instance (i.e. an operating system process) for each file viewed or edited with a JEP frontend.
The same backend instance may be used for several files but for one file there is exactly one responsible backend instance.
If the backend instance is not yet running, it needs to be started.
If it is already running, it needs to be found for a particular file.

The association between files and backend instances is done via JEP config files.
These config files contain file patterns and associated command lines as described below.

If a file pattern is matching a file being edited/viewed, the corresponding command line is executed in order to startup the backend process. The working directory for the new backend process is the directory containing the config file.

When the backend starts up, it must announce the TCP port it will listen on on standard output using the fixed string:

    JEP service, listening on port <port>\n

Note, that it's not necessary for this to be the first output on standard output. The backend may for instance print it's version first.
The frontend will open a TCP connection to the port announced by the backend.

If a backend instance is already running, it should not be started again but the existing instance and TCP connection should be used. For that, backend instances are identified by the pair of JEP config file with absolute path and the matching file pattern. The frontend should keep track of the started backend instances and should reuse them if the pair of config file and file pattern is matching.

Note that with this approach, backend instances can only be shared for a particular frontend. If the same file is edited with two different frontends, two backend instances will be created.

### JEP Configuration Files

Each configuration file consists of a list of service specs, each holding a list of file patterns and a command line.

    jep_config        ::= <service_spec>+
    service_spec      ::= <file_pattern_list>:\n<command line string>\n
    file_pattern_list ::= <file_pattern> | <file_pattern>, <file_pattern_list>
    file_pattern      ::= <filename without path> | *.<extension>

A file pattern is either the full name of a file without any path information or an asterisk followed by a dot and a file extension.

TODO: glob pattern, relative/absolute

### Backend Discovery

Backend discovery is the process of finding a JEP config file with a matching file pattern.
The discovery process needs to be done for each file being loaded or edited by a JEP frontend.

There are two ways to find config files which are tried one after the other in the given order as presented below.

#### Config Files in Parent Directories

For each file, the frontend searches for a configuration file named ".jep".
The search starts within the directory containing the file being edited.
As long as the configuration file isn't found, the search continues in the parent directory.
When a configuration file is found, its content is searched for a matching file pattern.
If a matching pattern is found, the search is complete.
Otherwise it continues in the parent directory.

Note that these ".jep" config files are a simple way to establish something like workspaces:
All files in the directory containing the .jep file and all files in subdirectories will be handled by this .jep file and thus by the backend instances created for this .jep file.
If the backend knows the file pattern and since its current working directory will be the directory containing the config file, it can regard all matching files as belonging together in a workspace.
The set of workspace files could then be used to check for example if cross references from one file to another can be resolved or not.

#### Config Files in Central Directories 

There are two places where frontends should look for configuration files:
1. the directory .jep in the user's home 
2. the directory /etc/jep in the central system configuration 

If present, the directories are searched one by one in the given order.
All files in a directory are considered as JEP configuration files independent of their name.
The files in a directory are search in alphabetical order of the file names.
The discovery process stops on the first file which contains a matching file pattern.

Note that this method of finding config files makes creating workspaces more difficult:
There can be many files from various locations in the file system which are associated with a backend service started via a config file in a central directory. Without further information, a backend can not know which of these files belong together. One solution could be to have backend specific configuration files which define workspaces and effectively play the same role as the ".jep" config files described above. 



## Backend Alive Message

As soon as the frontend has opened the connection to the backend, the backend should start to send alive messages once every 1 second.
This way, frontends can check if a backend is still alive and restart it if necessary.

    message BackendAlive

Note that frontends may also monitor the backend process they have started and this way will know when the backend stopped or crashed. 
The alive message however is an additional means to check the liveliness of the backend proccess which might not be responding for whatever reason.
In this case, the frontend might kill and restart the backend.



## Backend Shutdown

Backend shutdown is conducted by means of a special shutdown message.

    message Shutdown

When a backend receives this message, it should shutdown immediately.
Frontends should send this message when the backend is no longer needed.
In particular, they should send this message when the frontend is shut down by itself.

Since frontends may miss to send the shutdown command, backends should implement a timeout mechanism and shutdown automatically when the timeout expires.
Note, that the protocol is designed in a way that allows backends to shut down without any loss of information.


## Content Synchronization

    message ContentSync {
      file: String             // file name
      start: [0,1] Integer     // update region start byte index, default: 0
      end: [0,1] Integer       // update region end byte index, default: last byte
    }


## Syntax Highlighting

## Folding

## Outline information

## Error Markers


