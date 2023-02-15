# skjutsgruppen-infra

This repo describes setup and maintenance of the hosting platform behind Skjutsgruppen.

## Contributing

### Dependencies

You will need a bunch of CLI tools to be able to work with the infrastructure.

```shell
brew install jq kubens kubectl kubectx skaffold
```

### Access

Once you have kubectl installed, add the skjutsgruppen cluster context to your `~/.kube/config`. Ask someone in the project to give it to you.

If you need SSH access to the servers, you need to ask someone in the project to add your SSH key to each server.

## Cluster setup

The cluster is currently running on two instances. 2 vCPU, 4 GB RAM and 40 GB disk each.
