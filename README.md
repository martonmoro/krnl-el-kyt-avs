# KRNL EigenLayer KYT AVS

Basic repo demoing a simple AVS middleware with full eigenlayer integration.

## Dependencies

You will need [foundry](https://book.getfoundry.sh/getting-started/installation) and [zap-pretty](https://github.com/maoueh/zap-pretty) to run.
```bash
curl -L https://foundry.paradigm.xyz | bash
foundryup
go install github.com/maoueh/zap-pretty@latest
```
Alternatively you can install zap-pretty with Homebrew running:
```bash
brew install maoueh/tap/zap-pretty
``` 

You will also need to install the submodules (eigenlayer-middleware and forge-std). 
```bash
git submodule update --init --recursive
```

Go version can also cause an issue so make sure you have 1.21 or higher.

## Running via make

This simple session illustrates the basic flow of the AVS. The makefile commands are hardcoded for a single operator, but it's however easy to create new operator config files, and start more operators manually (see the actual commands that the makefile calls).

Start anvil in a separate terminal:

```bash
make start-anvil-chain-with-el-and-avs-deployed
```

The above command starts a local anvil chain from a [saved state](./tests/anvil/avs-and-eigenlayer-deployed-anvil-state.json) with eigenlayer and kyt contracts already deployed (but no operator registered).

Start the aggregator:

```bash
make start-aggregator
```

Register the operator with eigenlayer and kyt, and then start the process:

```bash
make start-operator
```

## Avs Task Description

The architecture of the AVS contains:

- [Eigenlayer core](https://github.com/Layr-Labs/eigenlayer-contracts/tree/master) contracts
- AVS contracts
  - [ServiceManager](contracts/src/IncredibleSquaringServiceManager.sol) which will eventually contain slashing logic but for M2 is just a placeholder.
  - [TaskManager](contracts/src/KYTTaskManager.sol) which contains [task creation](contracts/src/KYTTaskManager.sol#L86) and [task response](contracts/src/KYTTaskManager.sol#L105) logic.
  - The [challenge](contracts/src/KYTTaskManager.sol#L185) logic could be separated into its own contract, but we have decided to include it in the TaskManager for this simple task.
  - Set of [registry contracts](https://github.com/Layr-Labs/eigenlayer-middleware) to manage operators opted in to this avs
- Task Generator
  - in a real world scenario, this could be a separate entity, but for this simple demo, the aggregator also acts as the task generator
- Aggregator
  - aggregates BLS signatures from operators and posts the aggregated response to the task manager
  - For this simple demo, the aggregator is not an operator, and thus does not need to register with eigenlayer or the AVS contract. It's IP address is simply hardcoded into the operators' config.
- Operators
  - Check the address to be KYTd sent to the task manager by the task generator, sign it, and send it to the aggregator

1. A task generator (in our case, same as the aggregator) publishes tasks (when received through the httpServer) to the KYTTaskManager contract's [createNewTask](contracts/src/KYTTaskManager.sol#L86) function. Each task specifies an address `addressToKYT` for which it wants the currently opted-in operators to determine the KYT result. `createNewTask` also takes `quorumNumbers` and `quorumThresholdPercentage` which requests that each listed quorum (we only use quorumNumber 0) needs to reach at least thresholdPercentage of operator signatures.
   - The task has to be sent to the `/send-task-KYT` endpoint
   - An example task looks the followings:
      ```json
      {
        "address": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266"
      }
      ```

2. A [registry](https://github.com/Layr-Labs/eigenlayer-middleware/blob/master/src/BLSRegistryCoordinatorWithIndices.sol) contract is deployed that allows any eigenlayer operator with at least 1 delegated [mockerc20](contracts/src/ERC20Mock.sol) token to opt-in to this AVS and also de-register from this AVS.

3. [Operator] The operators who are currently opted-in with the AVS need to read the task number from the Task contract, do the KYT process, sign on that computed result (over the BN254 curve) and send their taskResponse and signature to the aggregator.

4. [Aggregator] The aggregator collects the signatures from the operators and aggregates them using BLS aggregation. If any response passes the [quorumThresholdPercentage](contracts/src/IKYTTaskManager.sol#L38) set by the task generator when posting the task, the aggregator posts the aggregated response to the Task contract.

5. (Currently challenge functionalities are not implemented fully but we leave this part in to give a full picture about the AVS flow) 
   If a response was sent within the [response window](contracts/src/KYTTaskManager.sol#L125), we enter the [Dispute resolution] period.
   - [Off-chain] A challenge window is launched during which anyone can [raise a dispute](contracts/src/KYTTaskManager.sol#L185) in a DisputeResolution contract (in our case, this is the same as the TaskManager contract)
   - [On-chain] The DisputeResolution contract resolves that a particular operator’s response is not the correct response or the opted-in operator didn’t respond during the response window. If the dispute is resolved, the operator will be frozen in the Registration contract and the veto committee will decide whether to veto the freezing request or not.

## Avs node spec compliance

Every AVS node implementation is required to abide by the [Eigenlayer AVS Node Specification](https://eigen.nethermind.io/). The hard requirements are currently only to:
- implement the [AVS Node API](https://eigen.nethermind.io/docs/category/avs-node-api)
- implement the [eigen prometheus metrics](https://eigen.nethermind.io/docs/category/metrics)

## Ports

- The local anvil chain runs on `8546`
- The aggregator server ip port address is `8090`
- The endpoint to send kyt tasks to the task generator is `8081/send-task-KYT`


