# Child Process

    Stability: 3 - Stable

Node 通过 `child_process` 模块提供了处理三向数据流的 `popen(3)` 功能。


它能以完全非阻塞的方式与子进程的 `stdin`、`stdout` 和 `stderr` 用流的方式传递数据。
注意，某些程序在内部使用行缓冲 I/O，
虽然这不会影响到 node.js，
但传递给子进程的数据不会被立即处理。

使用 `require('child_process').spawn()` 或 `require('child_process').fork()` 创建子进程。
这两种方法的语义有些区别，
[下面](#child_process_asynchronous_process_creation)会有解释。

For scripting purposes you may find the
[synchronous counterparts](#child_process_synchronous_process_creation) more
convenient.

## Class: ChildProcess

`ChildProcess` 是一个 [EventEmitter][]。

子进程有三个与之相关的流：
`child.stdin`、`child.stdout` 和 `child.stderr`。
它们可以共享父进程的标准输入输出流，
也可以作为可导的独立流对象。

ChildProcess 类不能被直接使用，
使用 `spawn()`、`exec()`、`execFile()` 或 `fork()` 方法创建子进程实例。


### Event:  'error'

* `err` {Error Object} the error.

触发于：

1. 进程不能被创建
2. 进程不能被终止
3. 由任何原因引起的数据发送到子进程失败

注意，`exit` 事件在错误发生后可能会也可能不会触发。
如果同时监听了两个事件去触发某个函数，
需要防备会触发两次。

参阅 [`ChildProcess#kill()`](#child_process_child_kill_signal) 和
[`ChildProcess#send()`](#child_process_child_send_message_sendhandle)。

### Event:  'exit'

* `code` {Number} the exit code, if it exited normally.
* `signal` {String} the signal passed to kill the child process, if it
  was killed by the parent.

This event is emitted after the child process ends. If the process terminated
normally, `code` is the final exit code of the process, otherwise `null`. If
the process terminated due to receipt of a signal, `signal` is the string name
of the signal, otherwise `null`.

Note that the child process stdio streams might still be open.

Also, note that node establishes signal handlers for `'SIGINT'` and `'SIGTERM`',
so it will not terminate due to receipt of those signals, it will exit.

See `waitpid(2)`.

### Event: 'close'

* `code` {Number} the exit code, if it exited normally.
* `signal` {String} the signal passed to kill the child process, if it
  was killed by the parent.

This event is emitted when the stdio streams of a child process have all
terminated.  This is distinct from 'exit', since multiple processes
might share the same stdio streams.

### Event: 'disconnect'

This event is emitted after calling the `.disconnect()` method in the parent
or in the child. After disconnecting it is no longer possible to send messages,
and the `.connected` property is false.

### Event: 'message'

* `message` {Object} a parsed JSON object or primitive value
* `sendHandle` {Handle object} a Socket or Server object

Messages send by `.send(message, [sendHandle])` are obtained using the
`message` event.

### child.stdin

* {Stream object}

A `Writable Stream` that represents the child process's `stdin`.
Closing this stream via `end()` often causes the child process to terminate.

If the child stdio streams are shared with the parent, then this will
not be set.

### child.stdout

* {Stream object}

A `Readable Stream` that represents the child process's `stdout`.

If the child stdio streams are shared with the parent, then this will
not be set.

### child.stderr

* {Stream object}

A `Readable Stream` that represents the child process's `stderr`.

If the child stdio streams are shared with the parent, then this will
not be set.

### child.pid

* {Integer}

子进程的 PID。

例子：

    var spawn = require('child_process').spawn,
        grep  = spawn('grep', ['ssh']);

    console.log('Spawned child pid: ' + grep.pid);
    grep.stdin.end();

### child.connected

* {Boolean} Set to false after `.disconnect' is called

If `.connected` is false, it is no longer possible to send messages.

### child.kill([signal])

* `signal` {String}

Send a signal to the child process. If no argument is given, the process will
be sent `'SIGTERM'`. See `signal(7)` for a list of available signals.

    var spawn = require('child_process').spawn,
        grep  = spawn('grep', ['ssh']);

    grep.on('close', function (code, signal) {
      console.log('child process terminated due to receipt of signal '+signal);
    });

    // send SIGHUP to process
    grep.kill('SIGHUP');

May emit an `'error'` event when the signal cannot be delivered. Sending a
signal to a child process that has already exited is not an error but may
have unforeseen consequences: if the PID (the process ID) has been reassigned
to another process, the signal will be delivered to that process instead.
What happens next is anyone's guess.

Note that while the function is called `kill`, the signal delivered to the
child process may not actually kill it.  `kill` really just sends a signal
to a process.

See `kill(2)`

### child.send(message, [sendHandle])

* `message` {Object}
* `sendHandle` {Handle object}

When using `child_process.fork()` you can write to the child using
`child.send(message, [sendHandle])` and messages are received by
a `'message'` event on the child.

For example:

    var cp = require('child_process');

    var n = cp.fork(__dirname + '/sub.js');

    n.on('message', function(m) {
      console.log('PARENT got message:', m);
    });

    n.send({ hello: 'world' });

And then the child script, `'sub.js'` might look like this:

    process.on('message', function(m) {
      console.log('CHILD got message:', m);
    });

    process.send({ foo: 'bar' });

In the child the `process` object will have a `send()` method, and `process`
will emit objects each time it receives a message on its channel.

Please note that the `send()` method on both the parent and child are
synchronous - sending large chunks of data is not advised (pipes can be used
instead, see
[`child_process.spawn`](#child_process_child_process_spawn_command_args_options)).

There is a special case when sending a `{cmd: 'NODE_foo'}` message. All messages
containing a `NODE_` prefix in its `cmd` property will not be emitted in
the `message` event, since they are internal messages used by node core.
Messages containing the prefix are emitted in the `internalMessage` event, you
should by all means avoid using this feature, it is subject to change without notice.

The `sendHandle` option to `child.send()` is for sending a TCP server or
socket object to another process. The child will receive the object as its
second argument to the `message` event.

Emits an `'error'` event if the message cannot be sent, for example because
the child process has already exited.

#### Example: sending server object

Here is an example of sending a server:

    var child = require('child_process').fork('child.js');

    // Open up the server object and send the handle.
    var server = require('net').createServer();
    server.on('connection', function (socket) {
      socket.end('handled by parent');
    });
    server.listen(1337, function() {
      child.send('server', server);
    });

And the child would the receive the server object as:

    process.on('message', function(m, server) {
      if (m === 'server') {
        server.on('connection', function (socket) {
          socket.end('handled by child');
        });
      }
    });

Note that the server is now shared between the parent and child, this means
that some connections will be handled by the parent and some by the child.

For `dgram` servers the workflow is exactly the same.  Here you listen on
a `message` event instead of `connection` and use `server.bind` instead of
`server.listen`.  (Currently only supported on UNIX platforms.)

#### Example: sending socket object

Here is an example of sending a socket. It will spawn two children and handle
connections with the remote address `74.125.127.100` as VIP by sending the
socket to a "special" child process. Other sockets will go to a "normal" process.

    var normal = require('child_process').fork('child.js', ['normal']);
    var special = require('child_process').fork('child.js', ['special']);

    // Open up the server and send sockets to child
    var server = require('net').createServer();
    server.on('connection', function (socket) {

      // if this is a VIP
      if (socket.remoteAddress === '74.125.127.100') {
        special.send('socket', socket);
        return;
      }
      // just the usual dudes
      normal.send('socket', socket);
    });
    server.listen(1337);

The `child.js` could look like this:

    process.on('message', function(m, socket) {
      if (m === 'socket') {
        socket.end('You were handled as a ' + process.argv[2] + ' person');
      }
    });

Note that once a single socket has been sent to a child the parent can no
longer keep track of when the socket is destroyed. To indicate this condition
the `.connections` property becomes `null`.
It is also recommended not to use `.maxConnections` in this condition.

### child.disconnect()

Close the IPC channel between parent and child, allowing the child to exit
gracefully once there are no other connections keeping it alive. After calling
this method the `.connected` flag will be set to `false` in both the parent and
child, and it is no longer possible to send messages.

The 'disconnect' event will be emitted when there are no messages in the process
of being received, most likely immediately.

Note that you can also call `process.disconnect()` in the child process when the
child process has any open IPC channels with the parent (i.e `fork()`).

## Asynchronous Process Creation

These methods follow the common async programming patterns (accepting a
callback or returning an EventEmitter).

### child_process.spawn(command, [args], [options])

* `command` {String} The command to run
* `args` {Array} List of string arguments
* `options` {Object}
  * `cwd` {String} Current working directory of the child process
  * `stdio` {Array|String} Child's stdio configuration. (See below)
  * `env` {Object} Environment key-value pairs
  * `detached` {Boolean} The child will be a process group leader.  (See below)
  * `uid` {Number} Sets the user identity of the process. (See setuid(2).)
  * `gid` {Number} Sets the group identity of the process. (See setgid(2).)
* return: {ChildProcess object}

Launches a new process with the given `command`, with  command line arguments in `args`.
If omitted, `args` defaults to an empty Array.

The third argument is used to specify additional options, with these defaults:

    { cwd: undefined,
      env: process.env
    }

`cwd` allows you to specify the working directory from which the process is spawned.
Use `env` to specify environment variables that will be visible to the new process.

Example of running `ls -lh /usr`, capturing `stdout`, `stderr`, and the exit code:

    var spawn = require('child_process').spawn,
        ls    = spawn('ls', ['-lh', '/usr']);

    ls.stdout.on('data', function (data) {
      console.log('stdout: ' + data);
    });

    ls.stderr.on('data', function (data) {
      console.log('stderr: ' + data);
    });

    ls.on('close', function (code) {
      console.log('child process exited with code ' + code);
    });


Example: A very elaborate way to run 'ps ax | grep ssh'

    var spawn = require('child_process').spawn,
        ps    = spawn('ps', ['ax']),
        grep  = spawn('grep', ['ssh']);

    ps.stdout.on('data', function (data) {
      grep.stdin.write(data);
    });

    ps.stderr.on('data', function (data) {
      console.log('ps stderr: ' + data);
    });

    ps.on('close', function (code) {
      if (code !== 0) {
        console.log('ps process exited with code ' + code);
      }
      grep.stdin.end();
    });

    grep.stdout.on('data', function (data) {
      console.log('' + data);
    });

    grep.stderr.on('data', function (data) {
      console.log('grep stderr: ' + data);
    });

    grep.on('close', function (code) {
      if (code !== 0) {
        console.log('grep process exited with code ' + code);
      }
    });


Example of checking for failed exec:

    var spawn = require('child_process').spawn,
        child = spawn('bad_command');

    child.on('error', function (err) {
      console.log('Failed to start child process.');
    });

The 'stdio' option to `child_process.spawn()` is an array where each
index corresponds to a fd in the child.  The value is one of the following:

1. `'pipe'` - Create a pipe between the child process and the parent process.
   The parent end of the pipe is exposed to the parent as a property on the
   `child_process` object as `ChildProcess.stdio[fd]`. Pipes created for
   fds 0 - 2 are also available as ChildProcess.stdin, ChildProcess.stdout
   and ChildProcess.stderr, respectively.
2. `'ipc'` - Create an IPC channel for passing messages/file descriptors
   between parent and child. A ChildProcess may have at most *one* IPC stdio
   file descriptor. Setting this option enables the ChildProcess.send() method.
   If the child writes JSON messages to this file descriptor, then this will
   trigger ChildProcess.on('message').  If the child is a Node.js program, then
   the presence of an IPC channel will enable process.send() and
   process.on('message').
3. `'ignore'` - Do not set this file descriptor in the child. Note that Node
   will always open fd 0 - 2 for the processes it spawns. When any of these is
   ignored node will open `/dev/null` and attach it to the child's fd.
4. `Stream` object - Share a readable or writable stream that refers to a tty,
   file, socket, or a pipe with the child process. The stream's underlying
   file descriptor is duplicated in the child process to the fd that 
   corresponds to the index in the `stdio` array. Note that the stream must
   have an underlying descriptor (file streams do not until the `'open'`
   event has occurred).
5. Positive integer - The integer value is interpreted as a file descriptor 
   that is is currently open in the parent process. It is shared with the child
   process, similar to how `Stream` objects can be shared.
6. `null`, `undefined` - Use default value. For stdio fds 0, 1 and 2 (in other
   words, stdin, stdout, and stderr) a pipe is created. For fd 3 and up, the
   default is `'ignore'`.

As a shorthand, the `stdio` argument may also be one of the following
strings, rather than an array:

* `ignore` - `['ignore', 'ignore', 'ignore']`
* `pipe` - `['pipe', 'pipe', 'pipe']`
* `inherit` - `[process.stdin, process.stdout, process.stderr]` or `[0,1,2]`

Example:

    var spawn = require('child_process').spawn;

    // Child will use parent's stdios
    spawn('prg', [], { stdio: 'inherit' });

    // Spawn child sharing only stderr
    spawn('prg', [], { stdio: ['pipe', 'pipe', process.stderr] });

    // Open an extra fd=4, to interact with programs present a
    // startd-style interface.
    spawn('prg', [], { stdio: ['pipe', null, null, null, 'pipe'] });

If the `detached` option is set, the child process will be made the leader of a
new process group.  This makes it possible for the child to continue running 
after the parent exits.

By default, the parent will wait for the detached child to exit.  To prevent
the parent from waiting for a given `child`, use the `child.unref()` method,
and the parent's event loop will not include the child in its reference count.

Example of detaching a long-running process and redirecting its output to a
file:

     var fs = require('fs'),
         spawn = require('child_process').spawn,
         out = fs.openSync('./out.log', 'a'),
         err = fs.openSync('./out.log', 'a');

     var child = spawn('prg', [], {
       detached: true,
       stdio: [ 'ignore', out, err ]
     });

     child.unref();

When using the `detached` option to start a long-running process, the process
will not stay running in the background unless it is provided with a `stdio`
configuration that is not connected to the parent.  If the parent's `stdio` is
inherited, the child will remain attached to the controlling terminal.

See also: `child_process.exec()` and `child_process.fork()`

### child_process.exec(command, [options], callback)

* `command` {String} The command to run, with space-separated arguments
* `options` {Object}
  * `cwd` {String} Current working directory of the child process
  * `env` {Object} Environment key-value pairs
  * `encoding` {String} (Default: 'utf8')
  * `shell` {String} Shell to execute the command with
    (Default: '/bin/sh' on UNIX, 'cmd.exe' on Windows,  The shell should
     understand the `-c` switch on UNIX or `/s /c` on Windows. On Windows,
     command line parsing should be compatible with `cmd.exe`.)
  * `timeout` {Number} (Default: 0)
  * `maxBuffer` {Number} (Default: `200*1024`)
  * `killSignal` {String} (Default: 'SIGTERM')
  * `uid` {Number} Sets the user identity of the process. (See setuid(2).)
  * `gid` {Number} Sets the group identity of the process. (See setgid(2).)
* `callback` {Function} called with the output when process terminates
  * `error` {Error}
  * `stdout` {Buffer}
  * `stderr` {Buffer}
* Return: ChildProcess object

Runs a command in a shell and buffers the output.

    var exec = require('child_process').exec,
        child;

    child = exec('cat *.js bad_file | wc -l',
      function (error, stdout, stderr) {
        console.log('stdout: ' + stdout);
        console.log('stderr: ' + stderr);
        if (error !== null) {
          console.log('exec error: ' + error);
        }
    });

The callback gets the arguments `(error, stdout, stderr)`. On success, `error`
will be `null`.  On error, `error` will be an instance of `Error` and `error.code`
will be the exit code of the child process, and `error.signal` will be set to the
signal that terminated the process.

There is a second optional argument to specify several options. The
default options are

    { encoding: 'utf8',
      timeout: 0,
      maxBuffer: 200*1024,
      killSignal: 'SIGTERM',
      cwd: null,
      env: null }

If `timeout` is greater than 0, then it will kill the child process
if it runs longer than `timeout` milliseconds. The child process is killed with
`killSignal` (default: `'SIGTERM'`). `maxBuffer` specifies the largest
amount of data allowed on stdout or stderr - if this value is exceeded then
the child process is killed.


### child_process.execFile(file, [args], [options], [callback])

* `file` {String} The filename of the program to run
* `args` {Array} List of string arguments
* `options` {Object}
  * `cwd` {String} Current working directory of the child process
  * `env` {Object} Environment key-value pairs
  * `encoding` {String} (Default: 'utf8')
  * `timeout` {Number} (Default: 0)
  * `maxBuffer` {Number} (Default: 200\*1024)
  * `killSignal` {String} (Default: 'SIGTERM')
  * `uid` {Number} Sets the user identity of the process. (See setuid(2).)
  * `gid` {Number} Sets the group identity of the process. (See setgid(2).)
* `callback` {Function} called with the output when process terminates
  * `error` {Error}
  * `stdout` {Buffer}
  * `stderr` {Buffer}
* Return: ChildProcess object

This is similar to `child_process.exec()` except it does not execute a
subshell but rather the specified file directly. This makes it slightly
leaner than `child_process.exec`. It has the same options.


### child_process.fork(modulePath, [args], [options])

* `modulePath` {String} The module to run in the child
* `args` {Array} List of string arguments
* `options` {Object}
  * `cwd` {String} Current working directory of the child process
  * `env` {Object} Environment key-value pairs
  * `encoding` {String} (Default: 'utf8')
  * `execPath` {String} Executable used to create the child process
  * `execArgv` {Array} List of string arguments passed to the executable
    (Default: `process.execArgv`)
  * `silent` {Boolean} If true, stdin, stdout, and stderr of the child will be
    piped to the parent, otherwise they will be inherited from the parent, see
    the "pipe" and "inherit" options for `spawn()`'s `stdio` for more details
    (default is false)
  * `uid` {Number} Sets the user identity of the process. (See setuid(2).)
  * `gid` {Number} Sets the group identity of the process. (See setgid(2).)
* Return: ChildProcess object

This is a special case of the `spawn()` functionality for spawning Node
processes. In addition to having all the methods in a normal ChildProcess
instance, the returned object has a communication channel built-in. See
`child.send(message, [sendHandle])` for details.

These child Nodes are still whole new instances of V8. Assume at least 30ms
startup and 10mb memory for each new Node. That is, you cannot create many
thousands of them.

The `execPath` property in the `options` object allows for a process to be
created for the child rather than the current `node` executable. This should be
done with care and by default will talk over the fd represented an
environmental variable `NODE_CHANNEL_FD` on the child process. The input and
output on this fd is expected to be line delimited JSON objects.

## Synchronous Process Creation

These methods are **synchronous**, meaning they **WILL** block the event loop,
pausing execution of your code until the spawned process exits.

Blocking calls like these are mostly useful for simplifying general purpose
scripting tasks and for simplifying the loading/processing of application
configuration at startup.

### child_process.spawnSync(command, [args], [options])

* `command` {String} The command to run
* `args` {Array} List of string arguments
* `options` {Object}
  * `cwd` {String} Current working directory of the child process
  * `input` {String|Buffer} The value which will be passed as stdin to the spawned process
    - supplying this value will override `stdio[0]`
  * `stdio` {Array} Child's stdio configuration.
  * `env` {Object} Environment key-value pairs
  * `uid` {Number} Sets the user identity of the process. (See setuid(2).)
  * `gid` {Number} Sets the group identity of the process. (See setgid(2).)
  * `timeout` {Number} In milliseconds the maximum amount of time the process is allowed to run. (Default: undefined)
  * `killSignal` {String} The signal value to be used when the spawned process will be killed. (Default: 'SIGTERM')
  * `maxBuffer` {Number}
  * `encoding` {String} The encoding used for all stdio inputs and outputs. (Default: 'buffer')
* return: {Object}
  * `pid` {Number} Pid of the child process
  * `output` {Array} Array of results from stdio output
  * `stdout` {Buffer|String} The contents of `output[1]`
  * `stderr` {Buffer|String} The contents of `output[2]`
  * `status` {Number} The exit code of the child process
  * `signal` {String} The signal used to kill the child process
  * `error` {Error} The error object if the child process failed or timed out

`spawnSync` will not return until the child process has fully closed. When a
timeout has been encountered and `killSignal` is sent, the method won't return
until the process has completely exited. That is to say, if the process handles
the `SIGTERM` signal and doesn't exit, your process will wait until the child
process has exited.

### child_process.execFileSync(command, [args], [options])

* `command` {String} The command to run
* `args` {Array} List of string arguments
* `options` {Object}
  * `cwd` {String} Current working directory of the child process
  * `input` {String|Buffer} The value which will be passed as stdin to the spawned process
    - supplying this value will override `stdio[0]`
  * `stdio` {Array} Child's stdio configuration. (Default: 'pipe')
    - `stderr` by default will be output to the parent process' stderr unless
      `stdio` is specified
  * `env` {Object} Environment key-value pairs
  * `uid` {Number} Sets the user identity of the process. (See setuid(2).)
  * `gid` {Number} Sets the group identity of the process. (See setgid(2).)
  * `timeout` {Number} In milliseconds the maximum amount of time the process is allowed to run. (Default: undefined)
  * `killSignal` {String} The signal value to be used when the spawned process will be killed. (Default: 'SIGTERM')
  * `maxBuffer` {Number}
  * `encoding` {String} The encoding used for all stdio inputs and outputs. (Default: 'buffer')
* return: {Buffer|String} The stdout from the command

`execFileSync` will not return until the child process has fully closed. When a
timeout has been encountered and `killSignal` is sent, the method won't return
until the process has completely exited. That is to say, if the process handles
the `SIGTERM` signal and doesn't exit, your process will wait until the child
process has exited.

If the process times out, or has a non-zero exit code, this method ***will***
throw.  The `Error` object will contain the entire result from
[`child_process.spawnSync`](#child_process_child_process_spawnsync_command_args_options)


### child_process.execSync(command, [options])

* `command` {String} The command to run
* `options` {Object}
  * `cwd` {String} Current working directory of the child process
  * `input` {String|Buffer} The value which will be passed as stdin to the spawned process
    - supplying this value will override `stdio[0]`
  * `stdio` {Array} Child's stdio configuration. (Default: 'pipe')
    - `stderr` by default will be output to the parent process' stderr unless
      `stdio` is specified
  * `env` {Object} Environment key-value pairs
  * `uid` {Number} Sets the user identity of the process. (See setuid(2).)
  * `gid` {Number} Sets the group identity of the process. (See setgid(2).)
  * `timeout` {Number} In milliseconds the maximum amount of time the process is allowed to run. (Default: undefined)
  * `killSignal` {String} The signal value to be used when the spawned process will be killed. (Default: 'SIGTERM')
  * `maxBuffer` {Number}
  * `encoding` {String} The encoding used for all stdio inputs and outputs. (Default: 'buffer')
* return: {Buffer|String} The stdout from the command

`execSync` will not return until the child process has fully closed. When a
timeout has been encountered and `killSignal` is sent, the method won't return
until the process has completely exited. That is to say, if the process handles
the `SIGTERM` signal and doesn't exit, your process will wait until the child
process has exited.

If the process times out, or has a non-zero exit code, this method ***will***
throw.  The `Error` object will contain the entire result from
[`child_process.spawnSync`](#child_process_child_process_spawnsync_command_args_options)

[EventEmitter]: events.html#events_class_events_eventemitter
