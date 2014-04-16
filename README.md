# Overcast

![Screenshot](http://i.imgur.com/qWUV684.png)

Overcast is a simple terminal-based cloud management tool that was designed to make it easy to spin up and manage clusters of servers in a consistent, scriptable way. Inspired by [Packer.io](http://packer.io).

## Concepts

1. **Instances** are any machine you can SSH into. Instances can be local or remote, virtual or physical. Each instance has a name, IP, SSH key and port.
2. **Clusters** are sets of instances.

## Features

- Define clusters and instances using the command line or by editing a simple JSON file.

  ```sh
  $ overcast cluster create db
  $ overcast cluster create app
  $ overcast instance import app.01 --cluster app --ip 127.0.0.2 \
    --ssh-port 22222 --ssh-key $HOME/.ssh/id_rsa
  $ overcast instance import app.02 --cluster app --ip 127.0.0.3 \
    --ssh-port 22222 --ssh-key $HOME/.ssh/id_rsa
  ```

- Create, snapshot and destroy instances on DigitalOcean (EC2/Linode support is on the roadmap).

  ```sh
  # Create a new Ubuntu 12.04 instance:
  $ overcast digitalocean create db.01 --cluster db
  # Configure the instance to your liking:
  $ overcast run db.01 install/core install/redis
  $ overcast expose db.01 22 6379
  # Create a snapshot:
  $ overcast digitalocean snapshot db.01 my.db.snapshot
  # Spin up a cluster using your snapshot:
  $ overcast digitalocean create db.02 --cluster db --image-name my.db.snapshot
  $ overcast digitalocean create db.03 --cluster db --image-name my.db.snapshot
  $ overcast digitalocean create db.04 --cluster db --image-name my.db.snapshot
  ```

- Run commands or script files across any number of servers. Commands can be run sequentially or in parallel.

  ```sh
  $ overcast run db install/core install/redis
  $ overcast run all uptime "free-m" "df -h" --parallel
  ```

- Push and pull files between your local machine and an instance, a cluster, or all clusters. Dynamically rewrite file paths to include the instance name.

  ```sh
  $ overcast push app nginx/myapp.conf /etc/nginx/sites-enabled/myapp.conf
  $ overcast pull all /etc/nginx/sites-enabled/myapp.conf nginx/{instance}.myapp.conf
  ```

- Overcast is a thin wrapper around your native SSH client, and doesn't install or leave anything on the servers you communicate with.

- A [script library](https://github.com/andrewchilds/overcast/tree/master/scripts) and [recipe library](https://github.com/andrewchilds/overcast/tree/master/recipes) are included to make it easy to deploy common software stacks and applications. The library was written for Ubuntu servers, but could be extended to include other distributions.

## Installation

1. Install [Node.js](https://github.com/joyent/node/wiki/Installing-Node.js-via-package-manager) if not already installed.

2. Install Overcast using npm.

    ```sh
    $ npm -g install overcast
    ```

3. You can now use Overcast from any directory.

    ```sh
    $ overcast help
    ```

## Configuration

Overcast looks for an `.overcast` directory in the current directory, or in some parent directory, otherwise falling back to `$HOME/.overcast`. This allows you to have multiple configurations, and to check your cluster definitions and scripts into a repo, like source code.

The command `overcast init` will create a new configuration in the current directory. The config directory looks like this:

```sh
/.overcast
  /files            # Files to be pushed to / pulled from instances
  /keys             # SSH keys, can be your own or auto-generated by overcast
    overcast.key
    overcast.key.pub
  /scripts          # Scripts to be run on instances
  clusters.json     # Cluster/instance definitions (see example.clusters.json)
  variables.json    # API keys, etc (see example.variables.json)
```

## API Documentation

### overcast cluster

```
  overcast cluster count [name]
    Return the number of instances in a cluster.

    Example:
    $ overcast cluster count db
    > 0
    $ overcast instance create db.01 --cluster db
    > ...
    $ overcast cluster count db
    > 1

  overcast cluster create [name]
    Creates a new cluster.

    Example:
    $ overcast cluster create db

  overcast cluster rename [name] [new-name]
    Renames a cluster.

    Example:
    $ overcast cluster rename app-cluster app-cluster-renamed

  overcast cluster remove [name]
    Removes a cluster from the index. If the cluster has any instances
    attached to it, they will be moved to the "orphaned" cluster.

    Example:
    $ overcast cluster remove db
```

### overcast completions

```
  overcast completions
    Return an array of commands, cluster names, and instance names for use
    in bash tab completion.

    To enable tab completion in bash, add this to your .bash_profile:

    _overcast_completions() {
      local cur=${COMP_WORDS[COMP_CWORD]}
      COMPREPLY=($(compgen -W "`overcast completions`" -- "$cur"))
      return 0
    }
    complete -F _overcast_completions overcast
```

### overcast digitalocean

```
  These functions require the following values set in .overcast/variables.json:
    DIGITALOCEAN_CLIENT_ID
    DIGITALOCEAN_API_KEY

  overcast digitalocean create [name] [options]
    Creates a new instance on DigitalOcean.

    The instance will start out using the auto-generated SSH key found here:
    /path/to/.overcast/keys/overcast.key.pub

    You can specify region, image, and size of the droplet using -id or -slug.
    You can also specify an image or snapshot using --image-name.

      Option               | Default
      --cluster CLUSTER    |
      --ssh-port PORT      | 22
      --region-slug NAME   | nyc2
      --region-id ID       |
      --image-slug NAME    | ubuntu-12-04-x64
      --image-id ID        |
      --image-name NAME    |
      --size-slug NAME     | 512mb
      --size-id ID         |

    Example:
    $ overcast instance create db.01 --cluster db --size-slug 1gb --region-slug sfo1

  overcast digitalocean destroy [instance]
    Destroys a DigitalOcean droplet and removes it from your account.
    Using --force overrides the confirm dialog. This is irreversible.

      Option               | Default
      --force              | false

    Example:
    $ overcast digitalocean destroy app.01

  overcast digitalocean droplets
    List all DigitalOcean droplets in your account.

  overcast digitalocean images
    List all available DigitalOcean images. Includes snapshots.

  overcast digitalocean poweron [instance]
    Power on a powered off droplet.

  overcast digitalocean reboot [instance]
    Reboots a DigitalOcean droplet. According to the API docs, "this is the
    preferred method to use if a server is not responding."

    Example:
    $ overcast digitalocean reboot app.01

  overcast digitalocean rebuild [instance] [options]
    Rebuild a DigitalOcean droplet using a specified image name, slug or ID.
    According to the API docs, "This is useful if you want to start again but
    retain the same IP address for your droplet."

      Option               | Default
      --image-slug SLUG    | ubuntu-12-04-x64
      --image-name NAME    |
      --image-id ID        |

    Example:
    $ overcast digitalocean rebuild app.01 --name my.app.snapshot

  overcast digitalocean regions
    List available DigitalOcean regions (nyc2, sfo1, etc).

  overcast digitalocean resize [name] [options]
    Shutdown, resize, and reboot a DigitalOcean droplet.
    If --skipboot flag is used, the droplet will stay in a powered-off state.

      Option               | Default
      --size-slug NAME     |
      --size-id ID         |
      --skipBoot           | false

    Example:
    $ overcast instance resize db.01 --size-slug 2gb

  overcast digitalocean sizes
    List available DigitalOcean sizes (512mb, 1gb, etc).

  overcast digitalocean shutdown [instance]
    Shut down a DigitalOcean droplet.

    Example:
    $ overcast digitalocean shutdown app.01

  overcast digitalocean snapshot [instance] [snapshot-name]
    Creates a named snapshot of a droplet. This process will reboot the instance.

    Example:
    $ overcast digitalocean snapshot app.01

  overcast digitalocean snapshots
    Lists available snapshots in your DigitalOcean account.
```

### overcast expose

```
  overcast expose [instance|cluster|all] [port...]
    Reset the exposed ports on the instance or cluster using iptables.
    This will fail if you don't include the current SSH port.
    Specifying --whitelist will restrict all ports to the specified address(es).
    These can be individual IPs or CIDR ranges, such as "192.168.0.0/24".

    Expects an Ubuntu server, untested on other distributions.

      Option
      --user=NAME
      --whitelist "IP|RANGE..."
      --whitelist-PORT "IP|RANGE..."

    Examples:
    # Allow SSH, HTTP and HTTPS connections from anywhere:
    $ overcast expose app 22 80 443
    # Allow SSH from anywhere, only allow Redis connections from 1.2.3.4:
    $ overcast expose redis 22 6379 --whitelist-6379 "1.2.3.4"
    # Only allow SSH and MySQL connections from 1.2.3.4 or from 5.6.7.xxx:
    $ overcast expose mysql 22 3306 --whitelist "1.2.3.4 5.6.7.0/24"
```

### overcast exposed

```
  overcast exposed [instance|cluster|all]
    List the exposed ports on the instance or cluster.
    Expects an Ubuntu server, untested on other distributions.

      Option        | Default
      --user NAME   |
```

### overcast health

```
  overcast health [instance|cluster|all]
    Export common health statistics in JSON format.
    Expects an Ubuntu server, untested on other distributions.

    Example JSON:
    {
      "my_instance_name": {
        "cpu_1min": 0.53,
        "cpu_5min": 0.05,
        "cpu_15min": 0.10,
        "disk_total": 19592,     // in MB
        "disk_used": 13445,      // in MB
        "disk_free": 5339,       // in MB
        "mem_total": 1000,       // in MB
        "mem_used": 904,         // in MB
        "mem_free": 96,          // in MB
        "cache_used": 589,       // in MB
        "cache_free": 410,       // in MB
        "swap_total": 255,       // in MB
        "swap_used": 124,        // in MB
        "swap_free": 131,        // in MB
        "tcp": 152,              // open TCP connections
        "rx_bytes": 196396703,   // total bytes received
        "tx_bytes": 47183785,    // total bytes transmitted
        "io_reads": 1871210,     // total bytes read
        "io_writes": 6446448,    // total bytes written
        "processes": [
          {
            "user": "root",
            "pid": 1,
            "cpu%": 0,
            "mem%": 0,
            "time": "0:01",
            "command": "/sbin/init"
          }
        ]
      }
    }
```

### overcast help

```
  Overcast v0.2.0

  Code repo, issues, pull requests:
    https://github.com/andrewchilds/overcast

  Usage:
    overcast [command] [options...]

  Help:
    overcast help
    overcast help [command]
    overcast [command] help

  Commands:
    overcast cluster list
    overcast cluster count [name]
    overcast cluster create [name]
    overcast cluster rename [name] [new-name]
    overcast cluster remove [name]
    overcast completions
    overcast digitalocean create [instance] [options]
    overcast digitalocean destroy [instance]
    overcast digitalocean droplets
    overcast digitalocean images
    overcast digitalocean poweron [instance]
    overcast digitalocean reboot [instance]
    overcast digitalocean rebuild [instance] [options]
    overcast digitalocean regions
    overcast digitalocean resize
    overcast digitalocean sizes
    overcast digitalocean shutdown [instance]
    overcast digitalocean snapshot [instance] [snapshot-name]
    overcast digitalocean snapshots
    overcast expose [instance|cluster|all] [port...] [options]
    overcast exposed [instance|cluster|all]
    overcast health [instance|cluster|all]
    overcast info
    overcast init
    overcast instance get [name] [attr...]
    overcast instance import [name] [options]
    overcast instance list [cluster...]
    overcast instance remove [name]
    overcast instance update [name] [options]
    overcast list
    overcast ping [instance|cluster|all]
    overcast port [instance|cluster|all] [port]
    overcast pull [instance|cluster|all] [source] [dest]
    overcast push [instance|cluster|all] [source] [dest]
    overcast run [instance|cluster|all] [command...]
    overcast run [instance|cluster|all] [file...]
    overcast ssh [instance]

  Config directory:
    /path/to/.overcast
```

### overcast info

```
  overcast info
    Pretty-prints the complete clusters.json file, stored here:
    /path/to/.overcast/clusters.json
```

### overcast init

```
  overcast init
    Create an .overcast config directory in the current working directory.
    No action taken if one already exists.
```

### overcast instance

```
  overcast instance get [name] [attr...]
    Returns the instance attribute(s), one per line.

    Examples:
    $ overcast instance get app.01 ssh-port ip
    > 22
    > 127.0.0.1
    $ overcast instance get app.01 user
    > appuser

  overcast instance import [name] [options]
    Imports an existing instance to a cluster.

      Option               | Default
      --cluster CLUSTER    |
      --ip IP              |
      --ssh-port PORT      | 22
      --ssh-key PATH       | .overcast/keys/overcast.key
      --user USERNAME      | root

    Example:
    $ overcast instance import app.01 --cluster app --ip 127.0.0.1 \
        --ssh-port 22222 --ssh-key $HOME/.ssh/id_rsa

  overcast instance list [cluster...]
    Returns all instance names, one per line. Optionally limit to one or more clusters.

    Examples:
    $ overcast instance list
    $ overcast instance list app-cluster db-cluster

  overcast instance remove [name]
    Removes an instance from the index.
    The server itself is not affected by this action.

    Example:
    $ overcast instance remove app.01

  overcast instance update [name] [options]
    Update any instance property. Specifying --cluster will move the instance to
    that cluster. Specifying --name will rename the instance.

      Option               | Default
      --name NAME          |
      --cluster CLUSTER    |
      --ip IP              |
      --ssh-port PORT      |
      --ssh-key PATH       |
      --user USERNAME      |

    Example:
    $ overcast instance update app.01 --user differentuser --ssh-key /path/to/another/key
```

### overcast list

```
  overcast list
    Short list of your cluster and instance definitions, stored here:
    /path/to/.overcast/clusters.json
```

### overcast ping

```
  overcast ping [instance|cluster|all]
    Ping an instance or cluster.

      Option    | Default
      --count N | 3

    Examples:
    $ overcast ping app.01
    $ overcast ping db --count 5
```

### overcast port

```
  overcast port [instance|cluster|all] [port]
    Change the SSH port for an instance or a cluster.
    This command will fail if the new port is not opened by iptables.

    Examples:
    $ overcast port app.01 22222
    $ overcast port db 22222
```

### overcast pull

```
  overcast pull [instance|cluster|all] [source] [dest]
    Pull a file or directory from an instance or cluster using scp. Source is absolute.
    Destination can be absolute or relative to the .overcast/files directory.

    Any reference to {instance} in the destination will be replaced with the instance name.

      Option
      --user NAME

    Example:
    Assuming instances "app.01" and "app.02", this will expand to:
      - .overcast/files/nginx/app.01.myapp.conf
      - .overcast/files/nginx/app.02.myapp.conf
    $ overcast pull app /etc/nginx/sites-enabled/myapp.conf nginx/{instance}.myapp.conf
```

### overcast push

```
  overcast push [instance|cluster|all] [source] [dest]
    Push a file or directory to an instance or cluster using scp. Source can be
    absolute, or relative to the .overcast/files directory. Destination is absolute.

    Any reference to {instance} in the source will be replaced with the instance name.

      Option
      --user NAME

    Example:
    Assuming instances "app.01" and "app.02", this will expand to:
      - .overcast/files/nginx/app.01.myapp.conf
      - .overcast/files/nginx/app.02.myapp.conf
    $ overcast push app nginx/{instance}.myapp.conf /etc/nginx/sites-enabled/myapp.conf
```

### overcast run

```
  overcast run [instance|cluster|all] [command...]
    Runs a command or series of commands on an instance or cluster.
    Commands will run sequentially unless you use the --parallel flag,
    in which case each command will run on all instances simultanously.

      Option                          | Default
      --env "KEY=VAL KEY='1 2 3'"     |
      --user NAME                     |
      --ssh-key PATH                  |
      --parallel -p                   | false
      --continueOnError               | false

    Examples:
    $ overcast run app --env "foo='bar bar' testing=123" env
    $ overcast run all uptime "free -m" "df -h"

  overcast run [instance|cluster|all] [file...]
    Executes a script file or files on an instance or cluster.
    Script files can be either absolute or relative path.
    Script files will run sequentially unless you use the --parallel flag,
    in which case each file will execute on all instances simultanously.

      Option                          | Default
      --env "KEY=VAL KEY='1 2 3'"     |
      --user NAME                     |
      --ssh-key PATH                  |
      --shell-command "COMMAND"       | bash -s
      --parallel -p                   | false
      --continueOnError               | false

    Relative paths are relative to the cwd, or to these directories:
    /path/to/.overcast/scripts
    /path/to/installed/overcast/scripts

    Example:
    $ overcast run db install/core install/redis
```

### overcast ssh

```
  overcast ssh [instance]
    Opens an SSH connection to an instance.

    Option
    --ssh-key PATH
    --user NAME
```

## Design Goals &amp; Motivation

There are a lot of server management frameworks out there already (Chef, Puppet, Ansible, Salt), but they all involve either a complex server-client implementation, a steep learning curve or a giant, monolithic conceptual framework that requires taking a course to understand.

I wanted something that had little to no learning curve, that just focused on multi-server provisioning and communication and leaves problems like process/state management and system monitoring to tools designed specifically for those problems (Monit, Munin, Nagios, etc).

## Running the Tests

[![Build Status](https://travis-ci.org/andrewchilds/overcast.svg?branch=master)](https://travis-ci.org/andrewchilds/overcast)

```sh
npm install
npm test
```

## Upgrading Overcast

```sh
npm -g update overcast
```

Configuration files are left alone during an upgrade.

## Contributing

Contributions are very welcome. If you've got an idea for a feature or found a bug, please [open an issue](https://github.com/andrewchilds/overcast/issues). If you're a developer and want to help make Overcast better, [open a pull request](https://github.com/andrewchilds/overcast/pulls) with your changes.

## Roadmap

- Linode support
- AWS EC2 support
- Better test coverage
- Improved script library bundle

## License

MIT. Copyright &copy; 2014 [Andrew Childs](http://twitter.com/andrewchilds).
