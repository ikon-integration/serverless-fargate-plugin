# serverless-fargate-plugin ![Node.js Package](https://github.com/ikon-integration/serverless-fargate-plugin/workflows/Node.js%20Package/badge.svg)

Based on templates found here: https://github.com/nathanpeck/aws-cloudformation-fargate

### Overall

- ![npm](https://img.shields.io/npm/dy/@ikonintegration/serverless-fargate-plugin) ![npm](https://img.shields.io/npm/v/@ikonintegration/serverless-fargate-plugin) ![npm (tag)](https://img.shields.io/npm/v/@ikonintegration/serverless-fargate-plugin/latest) ![Libraries.io dependency status for latest release, scoped npm package](https://img.shields.io/librariesio/release/npm/@ikonintegration/serverless-fargate-plugin)
- ![GitHub commit activity](http://sprice.redirectme.net:8555/github/commit-activity/m/ikon-integration/serverless-fargate-plugin)
- ![GitHub last commit](http://sprice.redirectme.net:8555/github/last-commit/ikon-integration/serverless-fargate-plugin)

#### About
This plugin will create a cluster, load balancer, vpc, subnets, and one or more services to associate with it. This plugin implements the following approaches:

- Public VPC / Public ELB / Public Subnet 
- Private VPC / Private ELB / Private ELB

If you would like to reference the VPC elsewhere (such as in the [serverless-aurora-plugin](https://github.com/honerlaw/serverless-aurora-plugin)). The VPC will be called `VPC{stage}` where `{stage}` is the stage in the serverless.yml. The subnets will be called `SubnetName{stage}{index}` where `{stage}`is the stage in the serverless.yml, and `{index}` references the index of the subnet that was specified in the subnets array. *THESE ARE NOT ADDED TO OUTPUT*. So you can only reference them in the same serverless.yml / same cf stack.

#### Notes
- This plugin only supports AWS
- Docker image must be built / uploaded / and properly tagged
- It is assumed that the process running in the docker container is listening for HTTP requests.

#### Options
```javascript
Array<{
    tags: {
      owner: Me
      Customer: You
    };
    executionRoleArn?: string; // execution role for services, generated if not specified
    disableELB?: boolean; //disable ELB creation and bindings, default to false. Usefull for long running processes
    elbListenerArn?: string; //optionally pass a ELB listener to use instead of create an ELB + listener -- carefull with ports, they must be the same pf the listener
    timeout?: number; //ELB timeout, defaults to 30
    clusterArns?: { //Indicates if the cluster will not be created and an shared ECS cluster should be used instead
        ecsClusterArn: string; //ECS cluster ARN
        ecsIngressSecGroupId: string; //Ingress ECS VPC Group 
    };
    vpc: {
        //if this options are specified it will create a VPC
        cidr: string;
        subnets: string[]; // subnet cidrs
        //If this options are specified it will attach to existing VPC.
        //all of then are required, if one missing it will turn to self-created 
        //VPC as described above -- All vpc parameters below are intrinsic safe 
        //ivars meaning that all of then accept intrinsic functions 💪
        vpcId: string;
        securityGroupIds: string[]
        subnetIds: string[]
        //If not specified, subnetIds will be used
        elbSubnetIds?: string[]
    };
    services: Array<{
        name: string; // name of the service
        cpu: number;
        memory: number;
        public: boolean; //Will it be facing internet? This affects directly what security groups will be auto created
        port: number; // docker port (the port exposed on the docker image) - if not specified random port will be used (usefull for EC2 task types or busy private subnets where new services (and **re-deployments**) are allowed to have random listener ports)
        hostname?: string | string[]; //optional hostname for filter on ELB 
        limitSourceIPs?: string | string[]; //optional limit source IPs on ELB (only request made by the specified source IPs are allowed)
        limitHeaders?: {Name: string, Value: string | string[]}[]; //optional limit headers on ELB (only requests made with the specified headers are allowed)
        disableELB?: boolean; //useful for disabling ELB listeners on a cluster that has ELB and more tasks with ELB enabled
        entryPoint: string[]; // same as docker's entry point
        environment: { [key: string]: string }; // environment variables passed to docker container
        protocols: Array<{
            protocol: "HTTP" | "HTTPS";
            certificateArns?: string[]; // needed for https
            authorizer?: {
              poolArn: string;
              clientId: string;
              poolDomain: string;
            }; //available on HTTPS only
        }>;
        autoScale?: {
              min?: number; //default to 1
              max?: number; //default to 1
              metric: AutoScalingMetricType;
              cooldown?: number; //defaults to 30
              cooldownIn?: number; //defaults to cooldown but has priority over it
              cooldownOut?: number; //defaults to cooldown but has priority over it
              targetValue: number;
        }
        image?: string; // full image name, REPOSITORY[:TAG]
        imageRepository?: string; // image repository (used if image option is not provided)
        imageTag?: string; // image tag (used if image option is not provided)
        priority?: number; // priority for routing, defaults to 1
        path?: string | { path: string, method?: string }[]; // path the LB should send traffic to, defaults '*' (everything) - keyword 'ANY' is allowed on method
        desiredCount?: number; // number of tasks wanted by default - if not specified defaults to 1
        shouldUseEC2?: boolean; //defaults to false, if true will launch task into EC2
        taskRoleArn?: string;
        healthCheckUri?: string; // defaults to "/"
        healthCheckProtocol?: string; // defaults to "HTTP"
        healthCheckInterval?: number // in seconds, defaults to 6 seconds
        healthCheckTimeout?: number; // in seconds, defaults to 5 seconds
        healthCheckHealthyCount?: number; // defaults to 2
        healthCheckUnhealthyCount?: number; // defaults to 2
        logsMultilinePattern?: string; //regex pattern to match multiline logs (useful for js objects for example)
    }>
}>
```

#### Examples
```yaml
service: example-service

provider:
  name: aws
  region: us-east-1
  stage: example

plugins:
- serverless-fargate-plugin

custom:
  fargate:
    clusterName: Test
    vpc:
      cidr: 10.0.0.0/16
      subnets:
      - 10.0.0.0/24
      - 10.0.1.0/24
    tags:
      customer: You
      owner: Me
    disableELB: false
    services:
    - name: example-name
      cpu: 512
      memory: 1024
      port: 80
      healthCheckUri: /health
      healthCheckInterval: 6
      imageTag: 1.0.0
      imageRepository: xxx.amazonaws.com/xxx
      autoScale:
        min: 1
        max: 10
        cooldownIn: 30
        cooldownOut: 60
        metric: ECSServiceAverageCPUUtilization
        targetValue: 75
      entryPoint:
      - npm
      - run
      - start
      environment:
        PRODUCTION: true
        ECS_ENABLE_CONTAINER_METADATA: true #https://stackoverflow.com/questions/48819809/how-to-get-task-id-from-within-ecs-container
      protocols:
      - protocol: HTTP
      - protocol: HTTPS
        certificateArns:
        - xxxx

```

####Outputs
  For the configuration above CF will have the reference `ECSTestClusterExampleNameServiceHTTP` to be used on your serverless template as `${cf:stackName.ECSTestClusterExampleNameServiceHTTP}`

  For more information about your stack name, please, check [here][1] 
  
  [1]: https://serverless.com/framework/docs/providers/aws/guide/variables#reference-cloudformation-outputs
  
#### TODO
- Tests
- Better TS Definitions
- Option to not use ELB
- Auto certification through plugin https://github.com/schwamster/serverless-certificate-creator
- More options
