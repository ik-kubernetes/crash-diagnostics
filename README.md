![](https://github.com/vmware-tanzu/crash-diagnostics/workflows/Crash%20Diagnostics%20Build/badge.svg)

# Crash Recovery and Diagnostics for Kubernetes

Crash Recovery and Diagnostics for Kubernetes (*Crash Diagnostics* for short) is designed to help human operators who are investigating and troubleshooting unhealthy or unresponsive Kubernetes clusters.  It is a project designed to automate the diagnosis of problem clusters that may be in an unstable state including completely inoperable.  In its introductory release, Crash Diagnostics provides cluster operators the ability to automatically collect machine states and other information from each node in a cluster.  The collected information is then bundled in a tar file for further analysis. 

## Collecting information for troubleshooting
To specify the resources to collect from cluster machines, a series of commands are declared in a file called a diagnostics file.  Like a Dockerfile, the diagnostics file is a collection of line-by-line directives with commands that are executed on each specified cluster machine.  The output of the commands is then added to a tar file and saved for further analysis.    

For instance, when the following diagnostics file (saved as Diagnostics.file) is executed, it will collect information from the two cluster machines (specified with the `FROM` directive): 

```
FROM  192.168.176.100:22 192.168.176.102:22 
AUTHCONFIG username:${remoteuser}  private-key:${HOME}/.ssh/id_rsa 
WORKDIR /tmp/crashout 

# copy log files 
COPY /var/log/kube-apiserver.log 
COPY /var/log/kube-scheduler.log 
COPY /var/log/kube-controller-manager.log 
COPY /var/log/kubelet.log 
COPY /var/log/kube-proxy.log 

# Capture service status output 
CAPTURE journalctl -l -u kubelet 
CAPTURE journalctl -l -u kube-apiserver 

# Collect docker-related logs 
CAPTURE journalctl -l -u docker 
CAPTURE /bin/sh -c "docker ps | grep apiserver"

# Collect objects and logs from API server if available
KUBECONFIG $HOME/.kube/kind-config-kind
KUGEGET objects namespaces:"kube-system default" kind:"deployments" 
KUBEGET logs namespaces:"default" containers:"hello-app"

OUTPUT ./crash-out.tar.gz 
```
Note that the tool can also collect resource data from the API server, if available, using `KUBECONFIG` and the `KUBEGET` command.

## Features
* Simple declarative script with flexible format
* Support for multiple directives to execute user-provided commands
* Ability to declare or use existing environment variables in commands
* Easily transfer files from cluster machines
* Execute commands on remote machines and captures the results
* Automatically collect information from multiple machines
* Collect resource data and pod logs from an available API server

See the complete list of supported [directives here](./docs/README.md).


## Running Diagnostics
The tool is compiled into a single binary named `crash-diagnostics`.  For instance, when the following command runs, by default it will search for and execute diagnostics script file named `./Diagnostics.file`:

```
crash-diagnostics run
```

Flag `--file` can be used to specify a different diagnostics file: 

```
crash-diagnostics --file test-diagnostics.file 
```

The output file generated by the tool can be specified using flag `--output` (which overrides value in script):

```
crash-diagnostics --file test-diagnostics.file --output test-cluster.tar.gz
```


When you use the `--debug` flag, you should see log messages on the screen similar to the following:
```
$> crash-diagnostics run --debug

DEBU[0000] Parsing script file
DEBU[0000] Parsing [1: FROM local]
DEBU[0000] FROM parsed OK
DEBU[0000] Parsing [2: WORKDIR /tmp/crasdir]
...
DEBU[0000] Archiving [/tmp/crashdir] in out.tar.gz
DEBU[0000] Archived /tmp/crashdir/local/df_-i.txt
DEBU[0000] Archived /tmp/crashdir/local/lsof_-i.txt
DEBU[0000] Archived /tmp/crashdir/local/netstat_-an.txt
DEBU[0000] Archived /tmp/crashdir/local/ps_-ef.txt
DEBU[0000] Archived /tmp/crashdir/local/var/log/syslog
INFO[0000] Created archive out.tar.gz
INFO[0002] Created archive out.tar.gz
INFO[0002] Output done
```

## Compile and Run
`crash-diagnostics` is written in Go and requires version 1.11 or later.  Clone the source from its repo or download it to your local directory.  From the project's root directory, compile the code with the
following:

```
GO111MODULE=on go install .
```

This should place the compiled `crash-diagnostics` binary in `$(go env GOPATH)/bin`.  You can test this with:
```
crash-diagnostics --help
```
If this does not work properly, ensure that your Go environment is setup properly.

## Roadmap
This project has numerous possibilities ahead of it.  Read about our evolving [roadmap here](ROADMAP.md).


## Contributing

New contributors will need to sign a CLA (contributor license agreement). Details are described in our [contributing](CONTRIBUTING.md) documentation.


## License
This project is available under the [Apache License, Version 2.0](LICENSE.txt)