# ServerEngine

ServerEngine is a framework to implement robust multiprocess servers like Unicorn.

**Main features:**

```
                  Heartbeat via pipe
                      & auto-restart
                 /                \               ---+
+------------+  /   +----------+   \  +--------+     |
| Supervisor |------|  Server  |------| Worker |     |
+------------+      +----------+\     +--------+     | Multi-process
                        /         \                  | or multi-thread
                       /            \ +--------+     |
      Dynamic reconfiguration         | Worker |     |
     and live restart support         +--------+     |
                                                  ---+
```

**Other features:**

- logging and log rotation
- signal handlers
- stacktrace and heap dump on signal
- chuser, chgroup and chumask
- changing process names


## API

### Simplest server

What you need to implement at least is a worker module which has `run` and `stop` methods.

```ruby
require 'serverengine'

module MyWorker
  def run
    until @stop
      logger.info "Awesome work!"
      sleep 1
    end
  end

  def stop
    @stop = true
  end
end

se = ServerEngine.create(nil, MyWorker, {
  :daemonize => true,
  :log => 'myserver.log',
  :pid_path => 'myserver.pid',
})
se.run
```

Send `TERM` signal to kill the daemon. See also **Signals** section bellow for details.


### Multiprocess server

Simply set **worker_type=process** or **worker_type=thread** parameter, and set number of workers to `workers` parameter.

```ruby
se = ServerEngine.create(nil, MyWorker, {
  :daemonize => true,
  :log => 'myserver.log',
  :pid_path => 'myserver.pid',
  :worker_type => 'process',
  :workers => 4,
})
se.run
```

See also **Worker types** section bellow.


### Multiprocess TCP server

One of the typical implementation styles of TCP servers is that a parent process listens socket and child processes accept connections from clients.

ServerEngine allows you to optionally implement a server module to control the parent process:

```ruby
# Server module controls the parent process
module MyServer
  def before_run
    @sock = TCPServer.new(config[:bind], config[:port])
  end

  attr_reader :sock
end

# Worker module controls child processes
module MyWorker
  def run
    until @stop
      # you should use Cool.io or EventMachine actually
      c = server.sock.accept
      c.write "Awesome work!"
      c.close
    end
  end

  def stop
    @stop = true
  end
end

se = ServerEngine.create(MyServer, MyWorker, {
  :daemonize => true,
  :log => 'myserver.log',
  :pid_path => 'myserver.pid',
  :worker_type => 'process',
  :workers => 4,
  :bind => '0.0.0.0',
  :port => 9071,
})
se.run
```


### Logging

ServerEngine logger rotates logs by 1MB and keeps 5 generations by default.

```ruby
se = ServerEngine.create(MyServer, MyWorker, {
  :log => 'myserver.log',
  :log_level => 'debug',
  :log_rotate_age => 5,
  :log_rotate_size => 1*1024*1024,
})
se.run
```

ServerEngine's default logger extends from Ruby's standard Logger library to:

* support multiprocess aware log rotation
* support reopening of log file
* support 'trace' level, which is lower level than 'debug'

See also **Configuration** section bellow.


### Enabling supervisor process

Server programs running 24x7 hours need to survive even if a process stalled because of unexpected memory swapping or network errors.

Supervisor process runs as the parent process of the server process and monitor it to restart automatically.

```ruby
se = ServerEngine.create(nil, MyWorker, {
  :daemonize => true,
  :pid_path => 'myserver.pid',
  :supervisor => true,  # enable supervisor process
})
se.run
```


### Live restart

You can restart a server process without waiting for completion of all workers using `INT` signal (`supervisor=true` and `enable_detach=true` parameters must be enabled).
This feature allows you to minimize downtime where workers take long time to complete a task.

```
# 1. starts server
+------------+    +----------+    +-----------+
| Supervisor |----|  Server  |----| Worker(s) |
+------------+    +----------+    +-----------+

# 2. receives SIGINT and waits for shutdown of the server for server_detach_wait
+------------+    +----------+    +-----------+
| Supervisor |    |  Server  |----| Worker(s) |
+------------+    +----------+    +-----------+

# 3. starts new server if the server doesn't exit in server_detach_wait time
+------------+    +----------+    +-----------+
| Supervisor |\   |  Server  |----| Worker(s) |
+------------+ |  +----------+    +-----------+
               |
               |  +----------+    +-----------+
               \--|  Server  |----| Worker(s) |
                  +----------+    +-----------+

# 4. old server exits eventually
+------------+
| Supervisor |\
+------------+ |
               |
               |  +----------+    +-----------+
               \--|  Server  |----| Worker(s) |
                  +----------+    +-----------+
```

Note that network servers (which listen sockets) shouldn't use live restart because it causes "Address already in use" error at the server process. Instead, simply use `worker_type=process` configuration and send `USR1` to restart workers instead of the server. It restarts a worker without waiting for shutdown of the other workers. This way doesn't cause downtime because server process doesn't close listening sockets and keeps accepting new clients (See also `restart_server_process` parameter if necessary).


### Dynamic configuration reloading

Robust servers should not restart only to update configuration parameters.

```ruby
module MyWorker
  def initialize
    reload
  end

  def reload
    @message = config[:message] || "Awesome work!"
    @sleep = config[:sleep] || 1
  end

  def run
    until @stop
      logger.info @message
      sleep @sleep
    end
  end

  def stop
    @stop = true
  end
end

se = ServerEngine.create(nil, MyWorker) do
  YAML.load_file("config.yml").merge({
    :daemonize => true,
    :worker_type => 'process',
  })
end
se.run
```

Send `USR2` signal to reload configuration file.


## Utilities

### BlockingFlag

`ServerEngine::BlockingFlag` is recommended to stop workers because `stop` method is called by a different thread from the `run` thread.

```ruby
module MyWorker
  def initialize
    @stop_flag = ServerEngine::BlockingFlag.new
  end

  def run
    until @stop_flag.wait_for_set(1.0)  # or @stop_flag.set?
      logger.info @message
    end
  end

  def stop
    @stop_flag.set!
  end
end

se = ServerEngine.create(nil, MyWorker) do
  YAML.load_file(config).merge({
    :daemonize => true,
    :worker_type => 'process'
  })
end
se.run
```


## Module methods

### Worker module

- interface
  - `initialize` is called in the parent process (or thread) in contrast to the other methods
  - `before_fork` is called before fork for each worker process (available only if `worker_type` is "process" or "thread")
  - `run` is the required method
  - `stop` is called when TERM signal is received
  - `reload` is called when USR2 signal is received
  - `after_start` is called after starting the worker process in the parent process (or thread) (available only if `worker_type` is "process" or "thread")
- api
  - `server` server instance
  - `config` configuration
  - `logger` logger
  - `worker_id` serial id of workers beginning from 0


### Server module

- interface
  - `initialize` is called in the parent process in contrast to the other methods
  - `before_run` is called before starting workers
  - `after_run` is called before shutting down
  - `after_start` is called after starting the server process in the parent process (available only if `supervisor` parameter is true)
- hook points (call `super` in these methods)
  - `reload_config`
  - `stop(stop_graceful)`
  - `restart(stop_graceful)`
- api
  - `config` configuration
  - `logger` logger


## Worker types

ServerEngine supports 3 worker types:

- **embedded**: uses a thread to run worker module (default). This type doesn't support immediate shutdown or immediate restart.
- **thread**: uses threads to run worker modules. This type doesn't support immediate shutdown or immediate restart.
- **process**: uses processes to run worker modules. This type doesn't work on Win32 platform.


## Signals

- **TERM:** graceful shutdown
- **QUIT:** immediate shutdown (available only when `worker_type` is "process")
- **USR1:** graceful restart
- **HUP:** immediate restart (available only when `worker_type` is "process")
- **USR2:** reload config file and reopen log file
- **INT:** detach process for live restarting (available only when `supervisor` and `enable_detach` parameters are true. otherwise graceful shutdown)
- **CONT:** dump stacktrace and memory information to /tmp/sigdump-<pid>.log file

Immediate shutdown and restart send SIGQUIT signal to worker processes which kills the processes.
Graceful shutdown and restart call `Worker#stop` method and wait for completion of `Worker#run` method.


## Configuration

- Daemon
  - **daemonize** enables daemonize (default: false)
  - **pid_path** sets the path to pid file (default: don't create pid file)
  - **supervisor** enables supervisor if it's true (default: false)
  - **daemon_process_name** changes process name ($0) of server or supervisor process
  - **chuser** changes execution user
  - **chgroup** changes execution group
  - **chumask** changes umask
  - **daemonize_error_exit_code** exit code when daemonize, changing user or changing group fails (default: 1)
- Supervisor: available only when `supervisor` parameters is true
  - **server_process_name** changes process name ($0) of server process
  - **restart_server_process** restarts server process when it receives USR1 or HUP signal. (default: false)
  - **enable_detach** enables INT signal (default: true)
  - **exit_on_detach** exits supervisor after detaching server process instead of restarting it (default: false)
  - **disable_reload** disables USR2 signal (default: false)
  - **server_restart_wait** sets wait time before restarting server after last restarting (default: 1.0) [dynamic reloadable]
  - **server_detach_wait** sets wait time before starting live restart (default: 10.0) [dynamic reloadable]
- Multithread server and multiprocess server: available only when `worker_type` is thread or process
  - **workers** sets number of workers (default: 1) [dynamic reloadable]
  - **start_worker_delay** sets wait time before starting a new worker (default: 0) [dynamic reloadable]
  - **start_worker_delay_rand** randomizes start_worker_delay at this ratio (default: 0.2) [dynamic reloadable]
- Multiprocess server: available only when `worker_type` is "process" [dynamic reloadable]
  - **worker_process_name** changes process name ($0) of workers [dynamic reloadable]
  - **worker_heartbeat_interval** sets interval of heartbeats in seconds (default: 1.0) [dynamic reloadable]
  - **worker_heartbeat_timeout** sets timeout of heartbeat in seconds (default: 180) [dynamic reloadable]
  - **worker_graceful_kill_interval** sets the first interval of TERM signals in seconds (default: 15) [dynamic reloadable]
  - **worker_graceful_kill_interval_increment** sets increment of TERM signal interval in seconds (default: 10) [dynamic reloadable]
  - **worker_graceful_kill_timeout** sets promotion timeout from TERM to QUIT signal in seconds. -1 means no timeout (default: 600) [dynamic reloadable]
  - **worker_immediate_kill_interval** sets the first interval of QUIT signals in seconds (default: 10) [dynamic reloadable]
  - **worker_immediate_kill_interval_increment** sets increment of QUIT signal interval in seconds (default: 10) [dynamic reloadable]
  - **worker_immediate_kill_timeout** sets promotion timeout from QUIT to KILL signal in seconds. -1 means no timeout (default: 600) [dynamic reloadable]
- Logger
  - **log** sets path to log file. Set "-" for STDOUT (default: STDERR) [dynamic reloadable]
  - **log_level** log level: trace, debug, info, warn, error or fatal. (default: debug) [dynamic reloadable]
  - **log_rotate_age** generations to keep rotated log files (default: 5)
  - **log_rotate_size** sets the size to rotate log files (default: 1048576)
  - **log_stdout** hooks STDOUT to log file (default: true)
  - **log_stderr** hooks STDERR to log file (default: true)
  - **logger_class** class of the logger instance (default: ServerEngine::DaemonLogger)

---

```
Author:    Sadayuki Furuhashi
Copyright: Copyright (c) 2012-2013 Sadayuki Furuhashi
License:   Apache License, Version 2.0
```

