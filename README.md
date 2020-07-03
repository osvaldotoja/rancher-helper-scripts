# Introduction

Let's say you're using Rancher to manage your clusters. You either bootstap from, or import your clusters into Rancher, which becomes the single entrypoint to your clusters. Rancher provides an API and a UI, both of which are really great. Having an unified point of access to your infrastructure is also great.

But what if you want to use kubectl?

Rancher provides `kubeconfig` files to interact with each individual cluster. The `kubeconfig` file uses Rancher as a proxy to relay all the traffic to the cluster you want to access. You don't need access to the cluster's API, only to Rancher's API endpoint.

Going through the UI to retrieve `kubeconfig` files can be cumbersome, specially when one of the reasons for using `kubectl` is to work from the CLI, avoiding UI.

Anything you can do via the UI in Rancher, can be done via the API. Getting a file is just one `curl` command away. But you'll need the cluster's id, that's two `curl` commands away.

What if we have a way for navigating among clusters, even namespaces, the same way we navigate our filesystem? 
Where we can automatically setup `kubectl` to certain namespace on a given cluster just by changing directories.

This project provides a set of helper scripts which helps you to have such a working setup.

# How it works

The tool of choice is [direnv](https://direnv.net), it allows you to set environment variables automatically whenever you change into a specific directory.

Think of direnv as your per-directory .bashrc

It combines `direnv` with `curl` calls to Rancher API, to create a directory based representation of existing clusters and their namespaces.

Changing directories will setup the proper environments variables for `kubectl` to be able to start interacting with the desired cluster/namespace.

# Directories

* Level 0: Current directory applies to our rancher server.
* Level 1: One per cluster.
* Level 2: One per namespace.

Example:

```sh
$ tree
.
├── staging-eks
│   ├── kube-system
│   ├── monitoring
│   └── myapp
├── dev-eks
└── production-eks
```

# Requirements

## rancher authentication

First create a [Bearer Token](https://rancher.com/docs/rancher/v2.x/en/api/api-tokens/). Store its content in a file, by default: `$HOME/.rancher/bearer-token`.

## direnv

```
brew install direnv
```

Using `direnv` will require running `direnv allow` command the first time we change to the directory. It's a one time only action.

## kubernetes tools

[kubectx/kubens](https://github.com/ahmetb/kubectx)

Useful for switching kubectl context and namespaces.

```sh
brew install kubectx kubens
```

Not a hard requirement, a workaround is available:

```sh
if has kubens; then
  kubens $NAMESPACE
else
  kubectl config set-context --current --namespace=$NAMESPACE
fi
```

Optional:

[stern](https://github.com/wercker/stern)

For visualizing logs.

```sh
brew install stern
```

# Usage


`cd` to the desired cluster and/or namespace. Use `kubectl` or any other cli tool as usual.

# Files

This project consists of 4 files, two helper scripts and two `.envrc` templates.

## helper scripts

The scripts will perform autodiscovery of available resources, updating current directory as required.

| Script name | Description |
| --- | --- |
| update-clusters | Creates a directory for each kubernetes cluster available on rancher. If the directory already exists the script does nothing. |
| update-ns | Creates a directory for each namespace available in the cluster. It should be executed inside a cluster's directory. |


# FAQ

## How to add a new cluster?

Run the `update-clusters` command. 

## How to add a new namespace?

Run the `update-ns` command inside a cluster's directory.

## Where are the configuration files stored?

By default inside a directory in the home directory: `$HOME/.rancher/`.

```
$ tree -a ~/.rancher
├── bearer-token
├── staging-eks
│   ├── .kubeconfig
│   └── cluster-id
├── dev-eks
│   ├── .kubeconfig
│   └── cluster-id
└── production-eks
    ├── .kubeconfig
    └── cluster-id
```

## How to update configuration files?

If you need to generate a new kubeconfig file, delete the cluster entry from the configuration directory. Running `update-clusters` will regenerate the information.

Same applies to namespaces.

Removing directories and running the scripts is the way for updating resources.

## How to get pods for myapp in production?

```sh
# only required the first time
update-clusters
# assuming that is your production cluster name
cd production-eks/
# only required the first time
update-ns
# assuming myapp is your app namespace name
cd myapp/
kubectl get pods
```

## How come helper scripts are available for execution?

Helper scripts projects eats its own food. It uses `.envrc` to add the helper scripts directory to the `PATH` variable everytime you `cd` to the project's directory.


# Vault

The following is valid only if you're using Hashicorp Vault to store your secrets. Feel free to adjust it to other secrets management solutions.

Access to the rancher API is authenticated via a bearer token. That token is retrieved from the vault server.

Login into [vault](https://vault.example.org/), copy the vault token and store in a file: `$HOME/.vault-token`.

If you're using another authentication method, like IAM roles, adjust the scripts accordingly.

Example main `.envrc` file. This code assumes vault server access to be restricted by vpn access. That might not be your case but the code is left for reference purposes.

```sh
export RANCHER_ENDPOINT=https://rancher.example.org/v3
export RANCHER_CONFDIR="$HOME/.rancher"
export VAULT_ADDR="https://vault.example.org"
export VAULT_TOKEN=$(cat "${HOME}/.vault-token")


if [ ! -f "${RANCHER_CONFDIR}/bearer-token" ]
then
  curl --silent --connect-timeout 1 "${VAULT_ADDR}" 2>&1 >/dev/null
  retVal=$?
  if [ $retVal -ne 0 ]; then
    echo "ERROR: ${VAULT_ADDR} not reachable. Are you using the VPN?"
    exit $retVal
  fi
  if [ -z "$(vault print token)" ]
  then
    if [ ! -f "${HOME}/.vault-token" ]
    then
      echo "Missing vault token"
      exit 1
    else
      vault login "${VAULT_TOKEN}"
    fi
  fi
  [ -d "${RANCHER_CONFDIR}" ] || mkdir "${RANCHER_CONFDIR}"
  # adjust the path accordingly
  vault kv get -field RANCHER_BEARER_TOKEN kv/secrets/rancher/rancher-prod > "${RANCHER_CONFDIR}/bearer-token"
fi

export RANCHER_BEARER_TOKEN=$(cat "${RANCHER_CONFDIR}/bearer-token")
```
