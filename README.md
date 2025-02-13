# fabric-builder-k8s

Proof of concept Fabric builder for Kubernetes

Advantages:
- prepublished chaincode images avoids compile issues at deploy time
- standard CI/CD pipelines can be used to publish chaincode images
- traceability of installed chaincode's implementation (demo uses Git commit hash as image tag)

Status: it _should_ just about work now but there are a few issues to iron out (and tests to write) before it's properly usable!

## Usage

The k8s builder can be run in cluster using the `KUBERNETES_SERVICE_HOST` and `KUBERNETES_SERVICE_PORT` environment variables, or it can connect using a `KUBECONFIG_PATH` environment variable.

An optional `FABRIC_CHAINCODE_NAMESPACE` can be used to specify the namespace to deploy chaincode to.

A `CORE_PEER_ID` environment variable is also currently required.

External builders are configured in the `core.yaml` file, for example:

```
  externalBuilders:
    - name: k8s_builder
      path: /opt/hyperledger/k8s_builder
      propagateEnvironment:
        - CORE_PEER_ID
        - FABRIC_CHAINCODE_NAMESPACE
        - KUBERNETES_SERVICE_HOST
        - KUBERNETES_SERVICE_PORT
```

See [External Builders and Launchers](https://hyperledger-fabric.readthedocs.io/en/latest/cc_launcher.html) for details of Hyperledger Fabric builders.

There are addition docs with more detailed usage instructions for specific Fabric network deployments:

- [Kubernetes Test Network](docs/TEST_NETWORK_K8S.md)
- [Nano Test Network](docs/TEST_NETWORK_NANO.md)

## Chaincode Docker image

Unlike the traditional chaincode language support for Go, Java, and Node.js, the k8s builder *does not* build a chaincode Docker image using Docker-in-Docker.
Instead, a chaincode Docker image must be built and published before it can be used with the k8s builder.

The chaincode will have access to the following environment variables:

- CORE_CHAINCODE_ID_NAME
- CORE_PEER_ADDRESS
- CORE_PEER_TLS_ENABLED
- CORE_PEER_TLS_ROOTCERT_FILE
- CORE_TLS_CLIENT_KEY_PATH
- CORE_TLS_CLIENT_CERT_PATH
- CORE_TLS_CLIENT_KEY_FILE
- CORE_TLS_CLIENT_CERT_FILE
- CORE_PEER_LOCALMSPID

See [conga-nft-contract](https://github.com/hyperledgendary/conga-nft-contract) for an example project which publishes a chaincode image using GitHub Actions.

## Chaincode package

The k8s chaincode package file, which is installed by the `peer lifecycle chaincode install` command, must contain the Docker image name and tag of the chaincode being deployed.

[Fabric chaincode packages](https://hyperledger-fabric.readthedocs.io/en/latest/cc_launcher.html#chaincode-packages) are `.tgz` files which contain two files:

- metadata.json - the chaincode label and type
- code.tar.gz - source artifacts for the chaincode

To create a k8s chaincode package file, start by creating an `image.json` file.
For example,

```shell
cat << IMAGEJSON-EOF > image.json
{
  "name": "ghcr.io/hyperledgendary/conga-nft-contract",
  "tag": "b96d4701d6a04e6109bc51ef1c148a149bfc6200"
}
IMAGEJSON-EOF
```

Create a `code.tar.gz` archive containing the `image.json` file.

```shell
tar -czf code.tar.gz image.json
```

Create a `metadata.json` file for the chaincode package.
For example,

```shell
cat << METADATAJSON-EOF > metadata.json
{
    "type": "k8s",
    "label": "conga-nft-contract"
}
METADATAJSON-EOF
```

Create the final chaincode package archive.

```shell
tar -czf conga-nft-contract.tgz metadata.json code.tar.gz
```

Ideally the chaincode package should be created in the same CI/CD pipeline which builds the docker image.
There is an example [package-k8s-chaincode-action](https://github.com/hyperledgendary/package-k8s-chaincode-action) GitHub Action which can create the required k8s chaincode package.

The GitHub Action repository includes a basic shell script which can also be used for automating the process above outside GitHub workflows.
For example, to create a basic k8s chaincode package using the `pkgk8scc.sh` helper script.

```shell
curl -fsSL https://raw.githubusercontent.com/hyperledgendary/package-k8s-chaincode-action/main/pkgk8scc.sh -o pkgk8scc.sh && chmod u+x pkgk8scc.sh
./pkgk8scc.sh -l conga-nft-contract -n ghcr.io/hyperledgendary/conga-nft-contract -t b96d4701d6a04e6109bc51ef1c148a149bfc6200
```

## Chaincode deploy

Deploy the chaincode package as usual, starting by installing the k8s chaincode package.

```shell
peer lifecycle chaincode install conga-nft-contract.tgz
```
