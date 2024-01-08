---
title: Kubectl Plugins - Building a `kubectl`-like CLI with Go
slug: kubectl-mapr-ticket
date: 2024-01-07
lastmod: 2024-01-08

description: |-
  A write-up of my experience building a Kubectl plugin with Go using the `cobra`, `cli-runtime`, and `client-go` libraries for the first time.
summary: |-
  A write-up of my experience building a Kubectl plugin with Go using the `cobra`, `cli-runtime`, and `client-go` libraries for the first time.

tags:
  - mapr
  - kubernetes
  - kubectl
  - go

draft: false
showReadingTime: true

coverCaption: Created with [Bing Create](https://bing.com/create).
---

The motivation behind this article was my interest in developing a Kubectl plugin for handling MapR tickets, leveraging the [reverse-engineered MapR ticket format]({{< ref "/posts/reverse-engineering-mapr-ticket-format" >}}) I discussed in an earlier post.
As this was my first venture into creating a Kubectl plugin, it required a fair amount of research, leading to the creation of this to share my learnings at least in an abbreviated form.

This post will not go too much into the details of the plugin itself but rather focus on the process of creating a Kubectl plugin and making it available to the world. The plugin itself is available on [GitHub](https://github.com/nobbs/kubectl-mapr-ticket).

## What is a Kubectl Plugin?

[Kubectl](https://kubernetes.io/docs/reference/kubectl/) is the official command-line interface (CLI) tool to interact with the control plane of a Kubernetes cluster. It is a very powerful tool and probably the most important tool in the Kubernetes ecosystem for developers and operators alike.

Kubectl provides an [official plugin mechanism](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/) that allows developers to extend the functionality of it. Plugins can be written in any language as they are just executables starting with `kubectl-` that have to be placed somewhere in the `PATH` of the user. Kubectl will automatically find and execute these plugins when the user runs `kubectl <plugin-name>` passing all the arguments and flags to the plugin.

Kubectl comes with a built-in command to list all the installed and discoverable plugins:

```console
$ kubectl plugin list
The following compatible plugins are available:

/Users/nobbs/.krew/bin/kubectl-mapr_ticket
/Users/nobbs/.krew/bin/kubectl-neat
/Users/nobbs/.krew/bin/kubectl-slice
/opt/homebrew/bin/kubectl-krew
```

That's it. No magic involved. The only thing that is required is that the plugin executable is named `kubectl-<plugin-name>` and is placed somewhere in the `PATH` of the user, to make it available as `kubectl <plugin-name>`.

_Oh, alright, one more thing: if your plugin name contains dashes, you have to replace them with underscores in the executable name. So, if your plugin is called `kubectl my-plugin`, the executable has to be named `kubectl-my_plugin`._

### Managing Kubectl Plugins

Right, the Kubectl plugin mechanism is pretty simple. But how do you manage your plugins? How do you find new plugins? How do you keep them up-to-date? How do you share them with others?

That's where [Krew](https://krew.sigs.k8s.io/) comes into play. Krew is the de facto package manager for Kubectl plugins. It is a plugin itself and provides all of the above functionality. [Installation](https://krew.sigs.k8s.io/docs/user-guide/setup/install/) is pretty simple and straightforward.

Once installed, you can use Krew to search for plugins and install them, e.g.:

```console
$ kubectl krew search mapr-ticket
NAME         DESCRIPTION                                  INSTALLED
mapr-ticket  Get information about deployed MapR tickets  yes

$ kubectl krew install mapr-ticket
Updated the local copy of plugin index.
Installing plugin: mapr-ticket
Skipping plugin "mapr-ticket", it is already installed
```

Or, you can simply browse the [Krew index](https://krew.sigs.k8s.io/plugins/) to find new plugins. Getting listed there is quite simple, too. Follow the [instructions in the developer guide](https://krew.sigs.k8s.io/docs/developer-guide/) and submit a pull request to the [Krew index repository](https://sigs.k8s.io/krew-index) to get your plugin listed.

_Note, that not all plugins will be accepted. The Krew maintainers decide on a case-by-case basis whether a plugin is a good fit for the index. The `mapr-ticket` plugin was accepted, so the bar is not too high 😉._

## Creating a Kubectl Plugin

Now that we know what a Kubectl plugin is and how to manage them, let's take a look at how to create one using [the Go programming language](https://golang.org/) based on commonly used libraries to make our lives easier.

Other languages can be used as well, as the only requirement is that the plugin is an executable binary. You will also find libraries to interact with the Kubernetes API in other languages, e.g. [Python](https://pypi.org/project/kubernetes/). But as Go is the language of choice for most of the Kubernetes ecosystem, we will use it here as well.

Most of the Go-based Kubectl plugins out there make use of the following libraries:

- [cobra](https://github.com/spf13/cobra) for creating the CLI - it is the same library that Kubectl itself uses
- [client-go](https://github.com/kubernetes/client-go) for interacting with the Kubernetes API - that's the official Kubernetes client library
- [cli-runtime](https://github.com/kubernetes/cli-runtime) providing various helpers and utilities for writing Kubectl plugins to keep them consistent with the usual `kubectl` behavior

### The Cobra CLI Framework

Cobra is a very powerful CLI framework for Go and probably the most popular one. It is used by Kubectl itself and many other CLI tools in the Kubernetes ecosystem. It provides a very simple and intuitive way to create a CLI with subcommands, flags, and arguments. Even shell autocompletion can be implemented with just a few lines of code.

Creating a basic `root` command with Cobra is as simple as this:

```go {linenos=table}
package main

import (
	"os"

	"github.com/spf13/cobra"
)

func main() {
	rootCmd := &cobra.Command{
		Use:   "kubectl-mapr-ticket",
		Short: "A kubectl plugin to list and inspect MapR tickets",
		Long: `A kubectl plugin that allows you to list and inspect MapR tickets from a
Kubernetes cluster, including details stored in the ticket itself without
requiring access to the MapR cluster.`,
	}

	if err := rootCmd.Execute(); err != nil {
		os.Exit(1)
	}
}
```

A `root` command is the main entry point for a CLI tool. It can have subcommands, flags, and arguments. In the above example, we only set some metadata for the command but don't add any functionality yet. We will do that in the next section.

Running the above code will result in the following output:

```console
$ go run main.go
A kubectl plugin that allows you to list and inspect MapR tickets from a
Kubernetes cluster, including details stored in the ticket itself without
requiring access to the MapR cluster.
```

#### Adding Subcommands

Not very exciting, but it's a start. Now, let's add a `list` subcommand to the `root` command that we will use to list all deployed MapR tickets - doing so is very similar to creating the `root` command itself. We just have to create a new `cobra.Command` and add it to the `root` command. Let's take a look at the code that we add to the above code snippet in place of line 17:

```go {linenos=table,linenostart=17}
listCmd := &cobra.Command{
    Aliases: []string{"ls"}, // add an `ls` alias for the list command
    Use:     "list",
    Short:   "List all secrets containing MapR tickets in the current namespace",
    Long: `List all secrets containing MapR tickets in the current namespace and print
some information about them.`,
    Args: cobra.NoArgs, // no arguments allowed for this command
    RunE: func(cmd *cobra.Command, args []string) error {
        // todo: do stuff when the command is called
        return nil
    },
}

rootCmd.AddCommand(listCmd)
```

There is still no functionality to any of the commands, but we already have a nice CLI with a `root` command and a `list` subcommand. Running the code will result in the following output:

```console
$ go run main.go
A kubectl plugin that allows you to list and inspect MapR tickets from a
Kubernetes cluster, including details stored in the ticket itself without
requiring access to the MapR cluster.

Usage:
  kubectl-mapr-ticket [command]

Available Commands:
  completion  Generate the autocompletion script for the specified shell
  help        Help about any command
  list        List all secrets containing MapR tickets in the current namespace

Flags:
  -h, --help   help for kubectl-mapr-ticket

Use "kubectl-mapr-ticket [command] --help" for more information about a command.

$ go run main.go list -h
List all secrets containing MapR tickets in the current namespace and print
some information about them.

Usage:
  kubectl-mapr-ticket list [flags]

Aliases:
  list, ls

Flags:
  -h, --help   help for list
```

#### Adding Flags

Right, we haven't done much yet, but we already have a nice CLI. Let's add some flags to the `list` command to make it a bit more useful. Let's add a `--all-namespaces` flag that will allow us to list MapR tickets in all namespaces instead of just the one of the current context. Again, very simple, let's add it in line 29 of the above code snippet:

```go {linenos=table,linenostart=29}
listOptions := &struct {
    // AllNamespaces indicates whether to list tickets in all namespaces
    AllNamespaces bool
}{}

listCmd.Flags().BoolVarP(
    &listOptions.AllNamespaces,
    "all-namespaces",
    "A",
    false,
    "If present, list tickets in all namespaces",
)
```

We create a new struct `listOptions` that will hold the values of the flags (and potentially arguments). Using a `*Options` struct is a common pattern in building CLIs with Cobra, as we can simply pass it to whatever function we want to call when the command is executed.

We then add a new flag to the `list` command using the `listCmd.Flags().BoolVarP()` function. The `BoolVarP()` function takes a pointer to a `bool` variable, the name of the flag, a shorthand name, a default value, and a description. The `BoolVarP()` function will automatically set the value of the `bool` variable to the value of the flag when the command is executed.
There are similar functions for all kinds of flags, e.g. `IntVarP()` for `int` flags, `StringVarP()` for `string` flags, etc.

Now, let's run the code again and see what we get:

```console
$ go run main.go list -h
List all secrets containing MapR tickets in the current namespace and print
some information about them.

Usage:
  kubectl-mapr-ticket list [flags]

Aliases:
  list, ls

Flags:
  -A, --all-namespaces   If present, list tickets in all namespaces
  -h, --help             help for list
```

All very nice - you quickly get a feeling for how easy it is to create a CLI with Cobra. And then there is all kinds of more advanced stuff that you can do with it, like autocompletion, e.g. something to the liking of running `kubectl get secret <TAB><TAB>` and getting a list of all secrets in the current namespace. That's again simple to do with Cobra, but we won't go into that here. Check out the [Cobra repository](https://github.com/spf13/cobra) for more information, or take a look into the implementation of my [`kubectl-mapr-ticket` plugin](https://github.com/nobbs/kubectl-mapr-ticket/tree/main).

### cli-runtime

Great, we now know how to create a basic CLI with Cobra. You could start implementing all the options available to all `kubectl` commands, like `--kubeconfig`, `--context`, `--namespace`, etc. But that would be a lot of work and we would have to reimplement a lot of functionality that is already available in the `kubectl` CLI. That's where the `cli-runtime` library comes into play.

The library provides a bunch of helpers for building `kubectl`-like CLIs. It provides functionality for setting up a default set of flags, as well as methods for printing output in a `kubectl`-like way, i.e. tables, JSON, YAML, etc. We will use the `cli-runtime` library to add the default set of `kubectl` to all our commands. To do so, we simply have to add the following code to our `root` command, e.g. right after the `rootCmd.AddCommand(listCmd)` call:

```go {linenos=table,linenostart=49}
// Create a set of flags to pass to the CLI
flags := pflag.NewFlagSet("kubectl-mapr-ticket", pflag.ExitOnError)
pflag.CommandLine = flags

// Create a set of default Kubernetes flags
kubernetesConfigFlags := genericclioptions.NewConfigFlags(true)

// add default kubernetes flags as global flags
kubernetesConfigFlags.AddFlags(rootCmd.PersistentFlags())
```

Running the same command as before will now result in the following output:

```console
$ go run main.go list -h
List all secrets containing MapR tickets in the current namespace and print
some information about them.

Usage:
  kubectl-mapr_ticket list [flags]

Aliases:
  list, ls

Flags:
  -A, --all-namespaces   If present, list tickets in all namespaces
  -h, --help             help for list

Global Flags:
      --as string                      Username to impersonate for the operation. User could be a regular user or a service account in a namespace.
      --as-group stringArray           Group to impersonate for the operation, this flag can be repeated to specify multiple groups.
      --as-uid string                  UID to impersonate for the operation.
      --cache-dir string               Default cache directory (default "/Users/nobbs/.kube/cache")
      --certificate-authority string   Path to a cert file for the certificate authority
      --client-certificate string      Path to a client certificate file for TLS
      --client-key string              Path to a client key file for TLS
      --cluster string                 The name of the kubeconfig cluster to use
      --context string                 The name of the kubeconfig context to use
      --disable-compression            If true, opt-out of response compression for all requests to the server
      --insecure-skip-tls-verify       If true, the server's certificate will not be checked for validity. This will make your HTTPS connections insecure
      --kubeconfig string              Path to the kubeconfig file to use for CLI requests.
  -n, --namespace string               If present, the namespace scope for this CLI request
      --request-timeout string         The length of time to wait before giving up on a single server request. Non-zero values should contain a corresponding time unit (e.g. 1s, 2m, 3h). A value of zero means don't timeout requests. (default "0")
  -s, --server string                  The address and port of the Kubernetes API server
      --tls-server-name string         Server name to use for server certificate validation. If it is not provided, the hostname used to contact the server is used
      --token string                   Bearer token for authentication to the API server
      --user string                    The name of the kubeconfig user to use
```

Oh, wow, that's a lot of flags. But that's exactly what we wanted, right? We now have all the default `kubectl` flags available to our plugin. Luckily, `cli-runtime` also parses all these default flags for us and provides us with simple functions to generate a Kubernetes client configuration from them which then can be used with the `client-go` library to interact with the Kubernetes API.

### client-go

The last library will only get a brief mention here, as [`client-go`](https://github.com/kubernetes/client-go) is a very powerful library that we cannot do justice in this post. It is the official Kubernetes client library, so it provides all the functionality to interact with the Kubernetes API you will ever need.

## Conclusion

That's it for this post. We have learned what Kubectl plugins are, how to manage them, and how to at least get started with creating one. We have also learned about some of the libraries that are commonly used to create Kubectl plugins in Go.

If you want to learn more about Kubectl plugins, I can highly recommend the [official documentation](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/). There you will find a lot more information about the plugin mechanism, as well as some links to more resources and sample plugins.

---

## Addendum - Full Sample Code

Here is the full sample code for CLI we've built in this post - it's obviously very basic and simplistic, but it should give you a good idea of how to get started.

```go {linenos=table}
package main

import (
	"os"

	"github.com/spf13/cobra"
	"github.com/spf13/pflag"
	"k8s.io/cli-runtime/pkg/genericclioptions"
)

func main() {
	rootCmd := &cobra.Command{
		Use:   "kubectl-mapr_ticket",
		Short: "A kubectl plugin to list and inspect MapR tickets",
		Long: `A kubectl plugin that allows you to list and inspect MapR tickets from a
Kubernetes cluster, including details stored in the ticket itself without
requiring access to the MapR cluster.`,
	}

	listCmd := &cobra.Command{
		Aliases: []string{"ls"},
		Use:     "list",
		Short:   "List all secrets containing MapR tickets in the current namespace",
		Long: `List all secrets containing MapR tickets in the current namespace and print
some information about them.`,
		Args: cobra.NoArgs,
		RunE: func(cmd *cobra.Command, args []string) error {
			// do stuff when the command is called, passing the listOptions struct to
			// whatever function needs it
			return nil
		},
	}

	listOptions := &struct {
		// AllNamespaces indicates whether to list tickets in all namespaces
		AllNamespaces bool
	}{}

	listCmd.Flags().BoolVarP(
		&listOptions.AllNamespaces,
		"all-namespaces",
		"A",
		false,
		"If present, list tickets in all namespaces",
	)

	rootCmd.AddCommand(listCmd)

	// Create a set of flags to pass to the CLI
	flags := pflag.NewFlagSet("kubectl-mapr-ticket", pflag.ExitOnError)
	pflag.CommandLine = flags

	// Create a set of default Kubernetes flags
	kubernetesConfigFlags := genericclioptions.NewConfigFlags(true)

	// add default kubernetes flags as global flags
	kubernetesConfigFlags.AddFlags(rootCmd.PersistentFlags())

	if err := rootCmd.Execute(); err != nil {
		os.Exit(1)
	}
}
```
