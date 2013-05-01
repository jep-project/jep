# JEP - Joined Editors Protocol

JEP is a protocol for decoupling editor frontends from language supporting backends.
This way, support for a particular textual language can be reused in many different editors.
As an additional benefit, language backends can be deployed independently of the editor frontends.
This allows for example, to ship a tool with support for its DSL and thus make sure that language support is up-to-date.

The primary focus of JEP is editing on a local machine, in which case communication between frontends and backends is local.
As an additional usecase though, frontends may run in a web browser with the backend running on a remote server.

This Github repo contains the JEP protocol description in the file "protocol.md".

