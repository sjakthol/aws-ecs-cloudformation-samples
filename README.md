CloudFormation templates for setting up an Amazon ECS cluster.

## Prerequisites

This template requires you to opt-in to the new ARN format for ECS Container Instances, Services and Tasks:

```bash
aws ecs put-account-setting-default --name serviceLongArnFormat --value enabled
aws ecs put-account-setting-default --name taskLongArnFormat --value enabled
aws ecs put-account-setting-default --name containerInstanceLongArnFormat --value enabled
```

## Deployment

Create an ECS cluster by running

```
make deploy-cluster
```

When complete, you should have an ECS cluster with at least one active container
instance.

## Implementation Details

### Blocking access to EC2 Instance Metadata Service
By default, containers can access the EC2 Instance Metadata Service. The service provides the instance temporary credentials for the IAM role associated with the instance. As containers are able to access this service, they can also read the EC2 Container Instance role credentials and perform API operations with the privileges of the host machine.

[ECS Documentation](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/instance_IAM_role.html) explains how `iptables` can be used to block containers from accessing the EC2 Instance Metadata Service. This cluster template executes the following command to block this access:

```bash
iptables --insert FORWARD 1 --in-interface docker+ --destination 169.254.169.254/32 --jump DROP
```

**Note**: This command will break applications that use the EC2 Instance Metadata Service to, for example, auto-discover the region or the availability zone they are running in. Keep this in mind if your applications relies on the availability of the EC2 Instance Metadata Service.

### Limiting scope of ECS Container Instance Role
The ECS Agent uses the IAM Role of the EC2 Instance it is running on to authenticate to ECS APIs. The default ECS Container Instance Role (as suggested in [ECS Documentation](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/instance_IAM_role.html)) does not limit the scope of ECS API calls in any way. This can cause problems if a container instance is compromised.

This setup includes a hardened container instance role that limits ECS API calls to a single ECS Cluster. This limits the blast radius of a compromise and isolates ECS clusters from one another. The following actions of the default role have been limited in this setup:

* `ecs:RegisterContainerInstance` - If not limited, a rogue container instance can register itself to any ECS cluster on the account. By registering an instance to an ECS Cluster, the ECS Control Plane starts to schedule tasks on the container instance. If these tasks use task roles, the control plane provides the instance information needed to fetch temporary credentials for the task role(s). The attacker can access the task role credentials [*] to elevate their privileges and move laterally in the account.
* `ecs:DeregisterContainerInstance` - If not limited, a rogue container instance could deregister container instances from any cluster on the account to cause a Denial of Service (if all instances are deregistered, task cannot be scheduled). This is difficult to abuse as the API requires unguessable container instance IDs (the role does not allow listing container instances so the attacker would have to obtain these through some other means).
* `ecs:Poll` - If not limited, a rogue container instance can receive control plane messages destined for any ECS cluster. Grants access to task role credentials for tasks scheduled to run on the instance (see `ecs:RegisterContainerInstance` for more details).
* `ecs:StartTelemetrySession` - If not limited, a rogue container can send fake telemetry to the ECS control plane in the name of any other container instance in the account (no practical impact).
* `ecs:SubmitContainerStateChange` - If not limited, a rogue container instance can mess up the state of containers in the ECS control plane. This is difficult to abuse as the API requires the attacker to specify unguessable identifiers (container UUIDs, task UUIDs).
* `ecs:SubmitTaskStateChange` -  If not limited, a rogue container instance can mess up the state of tasks in the ECS control plane. This is difficult to abuse as the API requires the attacker to specify unguessable identifiers (container UUIDs, task UUIDs).

[*] **Note on the ECS Task Role Credentials**: The ECS Control Plane exposes a [task metadata endpoint](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-metadata-endpoint.html) on a link-local address of the container instances. When a task with a task role is scheduled to an instance, the control plane exposes the IAM credentials for the task role via the task metadata endpoint. It also provides the ECS Agent an URL path where the credentials can be read from. The URL path includes a random UUID that cannot be guessed. The agent passes this URL path to containers in the environment which the AWS SDK uses to fetch the credentials. If an attacker can intercept the URL path where the role credentials can be obtained from, the attacker can use it to fetch credentials for the IAM role as well.
