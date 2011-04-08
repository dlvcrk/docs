
RunDeck Plugins
===========

***RunDeck plugin system is currently under development, so this document
is subject to change.***

***[Insert BETA sticker here]***

Plugins for RunDeck contain new Providers for some of the Services used by
the RunDeck core.

RunDeck comes with some built-in providers for these services, but Plugins
let you write your own, or use third-party implementations.

RunDeck currently comes installed with a few useful plugins: script-plugin and 
stub-plugin.


## About Services and Providers

The RunDeck core makes use of several different "Services" that provide
functionality for the different steps necessary to execute workflows, jobs, 
and commands across multiple nodes.

Each Service makes use of "Providers". Each Provider has a unique "Provider Name"
that is used to identify it, and most Services have default Providers unless
you specify different ones to use.

![RunDeck Services and Providers](figures/fig1101.png)

There are currently two types of Providers that can be used and developed for
RunDeck:

1. Node Executor Providers - these define ways of executing a command on a Node (local or remote)
2. File Copier Providers - these define ways of copying files to a Node

Specifics of how providers of these plugins work is listed below.

RunDeck Plugins can contain more than one Provider.

## Using Providers

The Providers are "enabled" for particular nodes on a node-specific basis,
or set as a default provider for a project or for the system.

If multiple providers are defined the most specific definition takes precedence
in this order:

1. Node specific
2. Project scope
3. Framework scope

### Node Specific

To enable a Node Executor provider for a node, add an attribute to the node definition:

`node-executor`

:    specify the provider name for a non-local node.

`local-node-executor`

:    specify the provider name for the local (server) node.


To enable a FileCopier provider for a node, add an attribute named either:

`file-copier`

:    specify the provider by name for a non-local node.

`local-file-copier`

:    specify the provider by name for the local (server) node.

### Project or Framework Scope

You can define the default provider to use for nodes at either the Project or
Framework scope (or both).  To do so, configure one of the following properties
in the `project.properties` or the `framework.properties` files.  

`service.NodeExecutor.default.provider`

:   Specifies the default NodeExecutor provider for remote nodes

`service.NodeExecutor.default.local.provider`

:   Specifies the default Node Executor provider for the  local node.

`service.FileCopier.default.provider`

:   Specifies the default File Copier provider for remote nodes.

`service.FileCopier.default.local.provider`

:   Specifies the default File Copier provider for the local node.

## When providers are invoked

RunDeck executes Command items on Nodes.  The command may be part of a Workflow as defined
in a Job, and it may be executed multiple times on different nodes.

Currently three "kinds" of Command items can be specified in Workflows:

1. "exec" commands - simple system command strings
2. "script" commands - either embedded script content, or server-local script 
files can be sent to the specified node and then executed with a set of input arguments.
3. "jobref" commands - references to other Jobs by name that will be executed with
a set of input arguments.

RunDeck uses the NodeExecutor and FileCopier services as part of the process of 
executing these command types.

The procedure for executing an "exec" command is:

1. load the NodeExecutor provider for the node and context
2. call the NodeExecutor#executeCommand method

The procedure for executing a "script" command is:

1. load the FileCopier provider for the node and context
2. call the FileCopier#copy* method 
3. load the NodeExecutor provider for the node and context
4. Possibly execute an intermediate command (such as "chmod +x" on the copied file)
5. execute the NodeExecutor#executeCommand method, passing the filepath of the 
  copied file, and any arguments to the script command.

## built-in providers

RunDeck uses a few built-in providers to provide the default service:

For NodeExecutor, these providers:

`local`

:   local execution of a command 

`jsch-ssh`

:   remote execution of a command via SSH, requiring the "hostname", and "username" attributes on a node.

For FileCopier, these providers:

`local`

:   creates a local temp file for a script

`jsch-scp`

:   remote copy of a command via SCP, requiring the "hostname" and  "username" attributes on a node.

Plugin Development
=============

This is a work in progress, and the plugin system is likely to change.

There are currently two ways to develop plugins:

1. Develop Java code that is distributed within a Jar 
file.  See [Java plugin development](#java-plugin-development).
2. Write shell/system scripts that implement your desired behavior and put them
in a zip file with some metadata.   See [Script Plugin Development](#script-plugin-development).

Either way, the resultant plugin archive file, either a .jar java archive, 
or a .zip file archive, will be placed in the `$RDECK_BASE/libext` dir.

Java Plugin Development
--------

Java plugins are distributed as .jar files containing the necessary classes for 
one or more service provider.

The `.jar` file you distribute must have this metdata within the main Manifest
forthe jar file to be correctly loaded by the system:

* `Rundeck-Plugin-Archive: true`
* `Rundeck-Plugin-Classnames: classname,..`

Each classname listed must be a valid "Provider Class" as defined below.

### Provider Classes

A "Provider Class" is a java class that implements a particular interface and declares
itself as a provider for a particular RunDeck "Service".  

Each plugin also defines a "Name" that identifies it for use in RunDeck.  The Name
of a plugin is also referred to as a "Provider Name", as the plugin class is a
provider of a particular service.

You should choose a unique but simple name for your provider.

Each plugin class must have the "com.dtolabs.rundeck.core.plugins.Plugin"
annotation applied to it.

    @Plugin(name="myprovider", service="NodeExecutor")
    public class MyProvider implements NodeExecutor{
    ...

Your provider class must have at least a zero-argument constructor, and optionally 
can have a single-argument constructor with a 
`com.dtolabs.rundeck.core.common.Framework` parameter, in which case your
class will be constructed with this constructor and passed the Framework
instance.

You may log messages to the ExecutionListener available via 
`ExecutionContext#getExecutionListener()` method.

You can also send output to `System.err` and `System.out` and it will be 
captured as output of the execution.

### Available Services:

* `NodeExecutor` - executes a command on a node
* `FileCopier` - copies a file to a node

## Provider Lifecycle

Provider classes are instantiated when needed by the Framework object, and the
instance is retained within the Service for future reuse. The Framework
object may exist across multiple executions, and the provider instance may be
reused.

Provider instances may also be used by multiple threads.

Your provider class should not use any instance fields and should be
careful not to use un-threadsafe operations.

### Node Executor Providers

A Node Executor provider executes a certain command on a remote or 
local node.

Your provider class must implement the `com.dtolabs.rundeck.core.execution.service.NodeExecutor` interface:

    public interface NodeExecutor {
        public NodeExecutorResult executeCommand(ExecutionContext context, 
            String[] command, INodeEntry node) throws ExecutionException;
    }

More information is available in the Javadoc.

### File Copier Providers

A File Copier provider copies a file or script to a remote
or local node.

Your provider class must implement the `com.dtolabs.rundeck.core.execution.service.FileCopier` interface:

    public interface FileCopier {
        public String copyFileStream(final ExecutionContext context, InputStream input, INodeEntry node) throws
            FileCopierException;

        public String copyFile(final ExecutionContext context, File file, INodeEntry node) throws FileCopierException;

        public String copyScriptContent(final ExecutionContext context, String script, INodeEntry node) throws
            FileCopierException;
    }

More information is available in the Javadoc.

Script Plugin Development
-----------

Script plugins can provide the same services as Java plugins, but they do so
with a script that is invoked in an external system processes by the JVM. 

You must create a zip file with the following structure:


    [something]-plugin.zip -- container (root)
    |- plugin.yaml -- plugin metadata file
    \- contents/  -- plugin contents directory
       \- script/resource files...

The filename of the plugin zip must end with "-plugin.zip" to be recognized as a
plugin archive. (NB: to support a common confusion with zip files, if the zip 
contains a top-level directory with the same base name as the zip file (sans ".zip") 
then that directory is considered the root.)

The file `plugin.yaml` must have this structure:

    # yaml plugin metadata
    
    name: plugin name
    author: author name
    version: version info
    date: release data
    providers:
        - name: [provider name]
          service: [service name]
          plugin-type: script
          script-file: [script file name]
          script-args: [script file args]

This provides the necessary metadata about the plugin, including one or more 
entries in the `providers` list to declare those providers defined in the plugin.

Each provider must have a name unique for the service it provides.

Each provider must declare one of these valid services:

* `NodeExecutor`
* `FileCopier`

The `script-file` must be the name of a file relative to the `contents` directory.

`script-args` is the arguments to use when invoking the script file. This can
contain data-context properties such as `${node.name}`.  Additionally, the
specific Service will provide some additional context properties that can be used:

* `NodeExecutor` will define "${exec.command}" containing the command to be executed
* `FileCopier` will define "${file-copy.file}" containing the local path to the file
that needs to be copied.

### Service requirements

The specific service has expectations about
the way your provider script behaves:

* Exit code of 0 indicates success
* Any other exit code indiates failure

For `NodeExecutor`

:   All output to `STDOUT`/`STDERR` will be captured for the job's output

For `FileCopier`

:   The first line of output of `STDOUT` MUST be the filepath of the file copied 
    to the target node.  Other output is ignored. All output to `STDERR` will be
    captured for the job's output.

Architecture
------

Installing Java Plugins

Copy the `plugin.jar` to the RunDeck server's libext dir:

    cp plugin.jar \
        $RDECK_BASE/libext