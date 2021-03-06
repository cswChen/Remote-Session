[![Maven Central](https://img.shields.io/maven-central/v/com.nordstrom.tools/remote-session.svg)](https://mvnrepository.com/artifact/com.nordstrom.tools/remote-session)

# REMOTE SESSION UTILITIES

**Remote Session** is a small collection of utility classes for interacting with remote systems via Secure Shell (SSH) protocol. Built around the [JSch](http://www.jcraft.com/jsch/) library from _JCraft_, available functions include:

* Secure file transfer
* Remote interactive shell
* Remote command execution

**Remote Session** wraps the capabilities of **JSch** in a simplified API that handles many of the details related to setting up and managing remote sessions. Originated by [Mykhaylo Adamovych](http://stackoverflow.com/users/448078/mykhaylo-adamovych), the implementation was copied verbatim from a post on [Stack Overflow](http://stackoverflow.com/questions/2405885/run-a-command-over-ssh-with-jsch). JavaDoc has been added for completeness and comprehensibility.

## Secure File Transfer

**Remote Session** enables clients to upload and download files via secure file transfer protocol (SFTP).

```java
    SshUtils.sftp("file:/C:/home/file.txt", "ssh://user:pass@host/home");
    SshUtils.sftp("ssh://user:pass@host/home/file.txt", "file:/C:/home");
```

**NOTE**: The transferred file retains its original name. If specified, the name component of `toUrl` will be ignored.  
**NOTE**: As indicated by the examples, source and target URIs must refer to opposing locations: `file` for local file system and `ssh` for remote file system.

* For upload: **fromUri** = `file`; **toUri** = `ssh`
* For download: **fromUri** = `ssh`; **toUri** = `file`

## Remote Interactive Shell

**Remote Session** supports interacting with remote systems via a secure `shell` channel, in which input and output are streamed between local and remote systems. Each of the three methods provided for secure shell interaction exhibit different operational characteristics.

###### Unbounded Stream I/O
```java
    InputStream is = System.in;
    OutputStream os = System.out;
    SshUtils.shell("ssh://user:pass@host", is, os);
    PrintStream ps = new PrintStream(is, true);
    ps.println("ls -la");
    ps.println("exit");
    System.out.println(IOUtils.toString(os, Charset.defaultCharset()));
```

With unbounded stream I/O, the channel remains open until the input stream is closed or an `exit` command is submitted.

###### Submit Specified Command
```java
    String remoteOutput = SshUtils.shell("ssh://user:pass@host/work/dir/path", "ls");
```

From the client perspective, this is effectively equivalent to `exec(String, String)`. The primary difference is the channel used for communication (`shell` instead of `exec`).

###### Submit Command with Streamed Output
```java
    OutputStream os = System.out;
    SshUtils.shell("ssh://user:pass@host", "ls -la", os);
    System.out.println(IOUtils.toString(os, Charset.defaultCharset()));
```

This is essentially a hybrid of the previous two secure-shell methods, well-suited for long-running commands that you'd like to leave running while you handle other tasks.

## Remote Command Execution

**Remote Session** enables you to submit commands to remote systems atomically. Instead of submitting commands through an input stream, you specify each command as a property of the channel and execute it.

###### Execute Specified Command
```java
    System.out.println(SshUtils.exec("ssh://user:pass@host/work/dir/path", "ls -t | head -n1"));
```

###### Execute Command 
```java
    String connectUri = "ssh://user:pass@host/work/dir/path";
    String command = "ls -t | head -n1";
    try (SessionHolder<ChannelExec> session = new SessionHolder<>(ChannelType.EXEC, URI.create(connectUri))) {
        String workDir = session.getWorkDir();
        if (workDir != null) command = "cd " + workDir + " && " + command;
        System.out.println(session, command);
    }
```

## JSch Reference Implementation (SshUtils)

**SshUtils** is a reference implementation of a **JSch** client. It enables you to execute the specified command, optionally executing an initial command to switch to an alternate user first.

```java
    String userName = "user";
    String password = "password";
    String hostName = "host";
    String sudoCmd = "sudo su - admin";
    String batchDir = "/work/dir/path";
    String batchCmd = "./script.ksh parm1 parm2";
    String output = SshUtils.executeBatch(userName, password, hostName, sudoCmd, batchDir, batchCmd);
    System.out.println(output);
```

The implementation of `SshUtils` demonstrates how to use a couple of important **Remote Session** classes:
* `SessionHolder` - This is a wrapper class for objects that extend the `Channel` class. The wrapper implements the `Closeable` interface, and `SshUtils` uses a "try-with-resources" block to ensure that the channel is always closed regardless of the outcome of command execution. `SessionHolder` includes these methods (among others):
  * `getChannel` - Get the channel to the remote session created for this `SessionHolder`.
  * `getChannelStream` - Get a new channel stream object for this session.
  * `disconnect` - Disconnect channel and session.
  * `assertExitStatus` - Verifies that the remote task completed normally.
* `ChannelStream` - This class encapsulates input/output operation for the channel attached to this session. It includes these methods:
  * `waitForInput` - Wait for input to be available.
  * `writeln` - Write the specified string to the remote host, followed by a carriage return.
  * `waitForPrompt` - Wait for the specified prompt to be received from the remote host.
  * `readChannel` - Read the input from the channel.
  
