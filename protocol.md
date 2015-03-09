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

The message content is serialized using msgpack, following a certain schema for each kind of message as described below.

When a file is opened in an editor (frontend), the editor will figure out the backend to be used, start it and connect to it.

While the user changes files in the editor, the frontend sends the changes to the backend.
The backend in turn responds by sending various kinds of information as it becomes available (error annotations, syntax highlighting, etc).

When the file or the editor is closed, the backend will be shut down.


## JEP Messages

The JEP protocol is based on message passing, with message content represented in msgpack 2.0.
There are several kinds of messages which all have a specific structure as described in the sections below.

### Message Format

A JEP message consists of a single message object serialized with msgpack.
There is no protocol frame header and no message length information.
When receiving, frontends and backends must read input data bytes until the msgpack object is complete.

### Encoding

Frontends and backends should not apply any transcoding to the data found in input files.
The reason is, that most often information about an input file's encoding is not reliable.
If the assumption of the source encoding is wrong, transcoding just makes things worse: either data is misinterpreted or information is lost due to character replacement.
Instead, data should be passed as is, or in other words: it should be interpreted as "binary" data. 

Conceptually most data passed via JEP are strings: text fragments that are part of some user defined language.
Depending on the language as well as the users input habits these strings can have any encoding.
Since msgpack requires strings to be encoded in UTF-8 and this would lead to the problems mentioned above.

This is why JEP uses the msgpack "Binary" datatype instead whenever a string could be influenced by user input.
Note that fixed strings defined by the JEP protocol itself do not fall into this category and therefore use the regular string type.

### Message Schema 

The protocol description below uses a simple notation to specify the structure of the different kinds of messages.
A message schema is defined by the keyword "message" followed by a textual message identifier.

    message <message identifier>

The corresponding msgpack object is a Map object and the message identifier is assigned to the key "\_message" as a string:

    {"_message": "<message identifier"}

Any message properties are specified in curly braces, each on a separate line.
Each property declaration consists of the property name, followed by a colon, an optional multiplicity declaration and the type:

    message <message identifier> {
      <attr1>: [0,*] <type>
      ...
    }

If the multiplicity is not declared it defaults to [1,1], i.e. the property is non optional.
An optional property is expressed by [0,1].
List type property are properties with an upper limit greater than 1, e.g. [0,\*], [1,2], etc.
List type attributes are represented as msgpack Array objects.

Message properties directly correspond to key/value pairs in the msgpack Map object.
Property names (the keys) are represented by msgpack String objects.

For the values, there are the following predefined primitive property types:
* String  -> msgpack "Binary"
* Integer -> msgpack "Integer"

Note that string values are not represented by msgpack "String" type, but by the "Binary" type in order to avoid encoding problems.

Enum types are declared with the keyword "enum" with the possible values following in curly braces.

    enum <type identifier> {
      <literal1>
      ...
    }

Values of such enum types are represented by msgpack "String" objects.

There is a predefined enum type "Boolean" which takes the values "true" and "false":

    enum Boolean {
      true
      false
    }

This special type is represented by the msgpack "Boolean" type.

Complex types are declared separately by means of the "type" keyword and can be used to declared nested structures.
Complex types contain property declarations just like messages.

    type <type identifier> {
      <attr1>: [0,*] <type>
      ...
    }

In contrast to message identifiers, type identifiers don't show up in the actual msgpack objects.
Instead the type is implied by the corresponding property declaration.


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

File content synchronization ensures that the backend "sees" the same file content as the frontent and it gives backends a chance to change the content edited in the frontent. This means that content synchronization in general is bidirectional.

However, in order to avoid inconsistencies when the frontend and the backend change things at the same time, synchronization messages from the backend to the frontend will only be accepted in certain situations and within certain time windows. More precisely, content sync messages by the backend are only allowed if this specification explicitly allows it.

Whenever a file is changed by the frontend, the frontend needs to send the changes to the backend. 

The backend may use the contents of any number of files in the file system to determine its behavior (e.g. provide "jump to definition" links). As long as the backend has not received a content sync message for a particular file, it assumes that the file content is the same as in the file system.

As soon as a file is opened in the frontend, the backend may no longer use the contents from the file system as the contents shown in the frontend might differ (either because the user started editing or because the file changed in the file system and wasn't reloaded yet). For that reason, the frontend needs to tell the backend whenever it opens a file. 

This is done by sending a synchronization message with the full file contents. Note that sending a notification only and letting the backend load the file from the file system is not sufficient as the file may change before the backend actually loads it. 

Any subsequent changes may be transmitted by sending only partial updates.

In general, a ContentSync message transports file contents via the "data" property. This data is inserted into the backend's view of the file overwriting the bytes from the start index position to the end index position including the byte at the end index position. Default start and end indexes ensure that the whole content is overwritten if no indices are specified. An initial full content sync message must either omit the indices are set them to 0.

    message ContentSync {
      file: String             // absolute file name
      start: [0,1] Integer     // update region start byte index, default: 0
      end: [0,1] Integer       // update region end byte index (exclusive), default: after last byte
      data: String             // replacement data
    }

In case a partial content synchronization message (i.e. a message with a start byte index greater than 0) is sent before an initial full synchronization message, the backend will respond with a synchronization loss indication. In this case, the frontend needs to send a full synchronization message to recover from this state.

    message OutOfSync {
      file: String             // absolute file name
    }

[note: there may be a "check sync" message in the future which allows the backend to check its view of a file, e.g. with a checksum; if this check fails, the backend would also respond with an OutOfSync message; there could also be an additional "checksum" property which is sent with the ContentSync message ]

[note: there may be a "sync from file" message in the future which instructs the backend to update its internal view from the file system; this is meant as an optimization to avoid sending large file contents; in this case there should either be a checksum included in the message or frontend should send a "check sync" message and the backend would possibly send an OutOfSync message ]


## Problem Markers

Whenever the backend detects a change in the known problems, it sends a ProblemUpdate message. Typically, this will happen some time after the frontend sent a ContentSync message.

Problem updates can also be incremental. This happens on two levels: On the first level, update information may only be sent for a subset of all model files if the "partial" property is set to true. In this case, only the problems of the named files will be updated, the problem lists of any other files sent earlier remain unchanged. On the second level, problem updates per file may be partial as well. If start and/or end index are set, the list of problems transmitted will be used to replace the existing problems in the given range. If one of start/end is not given, it defaults to the first and last element respectively. Indices are zero based. The start index marks the position at which thetransmitted problem list will be inserted. The end index marks the first element which will not be replaced. The end index must be greater than or equal to the start index.

Index Examples:

    Assuming the following problem array:

    [ a, b, c, d ]

    {-, -}    replace full list (defaults are {0, 4})
    {-, 1}    replace first element (defaults are {0, 1})
    {1, -}    replace last 3 elements (defaults are {1, 4})
    {0, 0}    insert at position 0
    {0, 1}    replace element at position 0
    {4, 4}    insert after the end (append)

    

    message ProblemUpdate {
      fileProblems: [0,*] FileProblems
      partial: [0,1] Boolean    // if true, the fileProblems list contains updates for
                                // a subset of the files, default: false
    }

    type FileProblems {
      file: String              // absolute file name
      problems: [0,*] Problem
      total: [0,1] Integer      // total number of problems if the number of returned 
                                // problems is smaller than the real number of problems
      start: [0,1] Integer      // update region start problem index, default: 0
      end: [0,1] Integer        // update region end problem index (exclusive), default: after last problem
    }

    type Problem {
      message: String
      severity: SeverityEnum
      line: Integer
    }

    enum SeverityEnum {
      debug
      info
      warn
      error
      fatal
    }


## Content Completion

The frontend may ask for content completion at a certain cursor position in a given file:

    message CompletionRequest {
      file: String             // absolute file name
      pos: Integer             // completion position, byte index after cursor
      limit: [0,1] Integer     // maximum number of options to be returned, default: no limit
      token: String            // request token returned in response message
    }

The request also carries an arbitrary string value token which must be repeated in the response message. This way the frontend can associate responses to previous requests.

The backend may limit the number of options returned by means of the limit property. If the limit is exceeded, this will be indicated in the response message by the limitExceeded property.

The backend replies by providing the completion options along with information about which part of the existing content is to be replaced. The frontend should remove the part ranging from the start position to the end position and replace it by the completion options. 

    message CompletionResponse {
      token: String            // token from request message
      start: Integer           // insertion start position
      end: Integer             // insertion end position
      options: [0,*] CompletionOption
      limitExceeded: Boolean   // set if a limit was given and the limit is exceeded
    }

    type CompletionOption {
      insert: String           // the string to be inserted when the option is chosen
      desc: [0,1] String       // description of the option
      longDesc: [0,1] String   // longer description of the option
      semantics: [0,1] SemanticType // semantic information about the option
      extensionId: [0,1] String // option identifier
    }

When the user has chosen an option the frontend should insert the insert text. If the chosen option contained an "extensionId" property, then the frontend must indicate the invocation of the option by a separate message and wait for an response by a ContentSync message. This gives the backend a chance to do more advanced insertions. The frontend must not allow users to type until the ContentSync response was received or a timeout occurred. 

    message CompletionInvocation {
      extensionId: String      // extension ID as defined by the completion option "extensionId"
    }


## Semantics

Semantic information about things like text snippets or completion options is necessary in order to present things properly to the user. For example a string should be highlighted in a different way than a number. An completion option completing a type (like a class in OO) might be highlighted different than a method or maybe a constant.

At the same time this semantic information must be independant of the actual presentation attributes like the color or the font. JEP doesn't allow backends to explicitly prescribe these attributes. Instead frontends can allow users to choose coloring schemes independent of JEP or the actual language edited with JEP at a particular point of time.

In order to achieve this, JEP pre-defines a set of semantic types. Frontends may associate visual styles with these types and backends may choose which types to use for the particular parts of a language. It seems practical to use terms commonly found in programming languages as the names of the semantic types because this way frontend and backend creators can have a common idea about how to highlight them. This doesn't mean of course that some entity associated with a specfic type in a specific language actually has to be the same thing. Instead it means something like "highlight this thing as you would highlight a class in C++".

The following is the list of semantic things currently supported by JEP:
  * Comment
  * Type (e.g. datatype, class, etc.)
  * String
  * Number
  * Identifier (e.g. a variable name)
  * Keyword (e.g. programming language keywords like "if" or "for")
  * Label (e.g. named arguments, keys in maps, etc.)
  * Link (something cross-referencing somewhere else, e.g. a HTML link)
  * Special1
  * Special2
  * Special3
  * Special4
  * Special5

JEP commands can use these definitions by the following enum:

    enum SemanticType {
      comment
      type
      string
      number
      identifier
      keyword
      label
      link
      special1
      special2
      special3
      special4
      special5
    }


## Syntax Highlighting

## Folding

## Outline information



