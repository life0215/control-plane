# Provisioning API

The goal of this document is to describe the requirements for the Runtime Provisioner and propose an API. 

# Requirements

## Non-Functional

The basic requirements are defined as follows:

- Provisioner API is based on GraphQL. 
- Provisioner API is asynchronous.

## Functional

### Runtime management

The following operations must be supported:

- Provisioning
- Upgrade
  - Kubernetes version
  - Kyma version
- Deprovisioning

### Runtime Agent connection management

The API must support resetting the connection between Runtime and Management Plane Services. The goal of this functionality is to force the Runtime Agent installed on the Runtime to establish new connection.

### Status information retrieval

These operations must be supported:

- Getting the status of an asynchronous operation (e.g. Runtime provisioning)
- Getting the current status of an existing Runtime
- Getting the configuration of an existing Runtime

# API proposal

## General 

These are the basic assumptions for the API design:

- Cluster provisioning and Kyma installation is considered an atomic operation.
- Provisioning, deprovisioning, upgrade, and Runtime Agent reconnecting is uniquely identified by the operation ID.
- Runtime is uniquely identified by RuntimeID.
- Before you provision the Runtime, register it in Director API. Use RuntimeID returned from Director in Provisioner API.
- Only one asynchronous operation can be in progress on a given Runtime.  
- It must be possible to install minimal Kyma  (Kyma Lite) and specify additional modules.
- Two types of status information are available:
  - The status of an asynchronous operation
  - Runtime status comprised of:
    - The status of the last operation
    - Runtime Agent Connection status

## Runtime management

### Provision Runtime mutation

***provisionRuntime*** mutation provisions Runtimes. The object passed to the mutation contains these fields:

- Kyma installation settings
  - Release version
  - List of modules to install
- Kubernetes cluster settings
  - Name
  - Size
  - Memory
  - Region and zone
  - Credentials
  - Version
  - Infrastructure provider (e.g. GKE, AKS)

Some Kubernetes cluster settings (such as size, memory, and version) are optional and default values are used.

The mutation returns the operation ID which allows you to retrieve the operation status.

### Upgrade Runtime mutation

***upgradeRuntime*** mutation upgrades Runtimes. The object passed to the mutation contains these fields:

- Kyma installation settings
  - Release version
  - List of modules to be installed
- Kubernetes cluster settings
  - Version

The mutation returns the operation ID which allows you to retrieve the operation status.

### Upgrade Shoot cluster mutation

***upgradeShoot*** mutation upgrades the Gardener Shoot cluster configuration, triggering Shoot reconciliation.

>**NOTICE:** This operation is dangerous and could potentially damage your Runtime. Use it wisely and with reasonable configuration values. **There is no rollback functionality implemented**, so you would have to get your hands dirty to fix potential failures.
The object passed to the mutation contains these configurable `GardenerConfig` values:

| Field                                 |                                       Configurable                                       | Note         |
| ------------------------------------- | :--------------------------------------------------------------------------------------: | ------------ |
| **kubernetesVersion**                   |                                    :heavy_check_mark:                                    | upgrade only |
| **purpose**                             |                                    :heavy_check_mark:                                    |              |
| **machineType**                         |                                    :heavy_check_mark:                                    |              |
| **volumeSizeGB**                        |                                    :heavy_check_mark:                                    |              |
| **diskType**                            |                                    :heavy_check_mark:                                    |              |
| **autoScalerMin**                       |                                    :heavy_check_mark:                                    | min. 1       |
| **autoScalerMax**                       |                                    :heavy_check_mark:                                    |              |
| **maxSurge**                            |                                    :heavy_check_mark:                                    |              |
| **maxUnavailable**                      |                                    :heavy_check_mark:                                    |              |
| **enableKubernetesVersionAutoUpdate**   |                                    :heavy_check_mark:                                    |              |
| **enableMachineImageVersionAutoUpdate** |                                    :heavy_check_mark:                                    |              |
| **providerSpecificConfig**              | Azure :heavy_check_mark: <br/> AWS :heavy_multiplication_x: <br/> GCP :heavy_check_mark: | `zones` only |

The mutation returns the operation ID which allows you to retrieve the operation status.

### Deprovision Runtime mutation

***deprovisionRuntime*** mutation deprovisions Runtimes. Pass the RuntimeID as argument. 

The mutation returns the operation ID which allows you to retrieve the operation status.

### Reconnect Runtime Agent mutation

***reconnectRuntimeAgent*** mutation reconnects the Runtime Agent. Pass the RuntimeID as argument. 

The mutation returns the operation ID which allows you to retrieve the operation status.

## Retrieving operation status

### Operation status query

***runtimeOperationStatus*** query gets the Runtime operation status. The query takes the operation ID as a parameter and returns an object containing this information:

- Operation type (e.g. Provisioning)
- Operation status (e.g. InProgress)
- Message
- Error messages list

### Runtime status query

***runtimeOperationStatus*** query gets the current status of a Runtime. The query takes Runtime ID as a parameter and returns an object containing this information:

- Last operation status
- Runtime connection configuration (kubeconfig)
- Runtime Agent Connection status
  - Status (connected, disconnected)
  - Errors list