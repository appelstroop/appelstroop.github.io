# Deploy AWS App Runner with a VPC connection using CDK

#### 28-11-2022 _Jasper Hoving_

<br />
Apprunner is one of the easiest ways to deploy containers on AWS. Launched in May 2021, AWS App Runner was positioned as a competitor to Google Cloud Run and as an alternative to running containers on Fargate.

At launch it lacked some features, most notably the ability to connect to a VPC. So, if you had an Aurora/RDS database running in your VPC, running a container in Apprunner was not really an option. Luckily, since february of this year you can connect Apprunner to a VPC!

![apprunner](/images/apprunner-banner.png)

## Apprunner vs ECS Fargate

But why not just use ECS fargate to run your container? Turns out that App Runner has a few advantages over ECS:

- Simpler configuration: a lot is abstracted away
- Load balancer is included
- Automatic scaling of containers based on concurrency
- Automatic deployments from ecr or github
- Https endpoint and certificate included
- Costs lower than ECS when service idle (see [this article](https://cloudonaut.io/fargate-vs-apprunner/) for a comparison)

But there are also some downsides to App Runner:

- Less finegrained configuration options
- More expensive when service is busy most of the day
- No support for AWS secrets in environment
- No redirect http to https (you can use another service like cloudfront connected to App Runner to achieve this)

Luckily, AWS is expanding the feature list of App Runner. For example, they created VPC support earlier this year and AWS recenly added the possiblity to create a [private endpoint](https://aws.amazon.com/blogs/containers/announcing-aws-app-runner-private-services/) in your vpc.

So, in summary, App Runner can be a cost efficient way to launch a container with a simple configuration. Especially when you have dev and staging environments, the costs can be a lot lower, because there are idle services involved.

## How to deploy with CDK?

In this post I will layout how to deploy a docker container to AWS apprunner with CDK. Because the higher level construct in the package [@aws-cdk/aws-apprunner-alpha](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-apprunner-alpha-readme.html) is still in alpha, and doesn't include all the options we want, we'll use the lower level [CfnService Apprunner](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_apprunner.CfnService.html) construct.

The complete github example repository can be found [here](https://github.com/appelstroop/apprunner-cdk-example).

First, we create a VPC:

```typescript
const vpc = new Vpc(this, "ApprunnerCdkExampleVpc", {
  maxAzs: 2,
  natGateways: 1,
});
```

We'll use an ECR repository to pull the image from (automatic builds from github is also an option). In this case we use an existing repo. Make sure there is an image uploaded to the repository, otherwise the App Runner cdk is not going to be able to deploy.

```typescript
const repository = Repository.fromRepositoryName(
  this,
  "ApprunnerCdkExampleRepo",
  "example-repo"
);
```

To illustrate communication with a service running in the VPC, let's create an Aurora DB.

```typescript
const dbCluster = new DatabaseCluster(this, "ApprunnerCdkExampleDBCluster", {
  engine: DatabaseClusterEngine.auroraPostgres({
    version: AuroraPostgresEngineVersion.VER_13_7,
  }),
  instances: 1,
  defaultDatabaseName: "postgres_api",
  instanceProps: {
    vpc: vpc,
    instanceType: InstanceType.of(
      InstanceClass.BURSTABLE3,
      InstanceSize.MEDIUM
    ),
    autoMinorVersionUpgrade: false,
    publiclyAccessible: false,
  },
  backup: {
    retention: Duration.days(7),
    preferredWindow: "01:00-02:00",
  },
  port: 5432,
  cloudwatchLogsRetention: RetentionDays.SIX_MONTHS,
  storageEncrypted: true,
  iamAuthentication: true,
});

// set security group on DB cluster
dbCluster.connections.allowFrom(dbCluster, Port.tcp(5432));
```

If you want to use the created DB secrets later on in your cdk code - to retrieve host/username/password, you can use:

```typescript
// use DB secret credentials or create new one. You can use this secret in the rest of your cdk code if you want
const dbSecrets =
  dbCluster.secret ?? new Secret(this, "ApprunnerCdkExampleDBSecrets");
```

To be able to let our App Runner service connect to the VPC, we have to create a VPC Connector. Luckily, we can also do this in CDK!

```typescript
// VCP connector to connect AppRunner to our VPC (and therefore DB)
const vpcConnector = new CfnVpcConnector(
  this,
  "ApprunnerCdkExampleVpcConnector",
  {
    subnets: vpc.selectSubnets({
      subnetType: SubnetType.PRIVATE_WITH_EGRESS,
    }).subnetIds,
    securityGroups: [dbCluster.connections.securityGroups[0].securityGroupId],
  }
);
```

Then we need to setup our IAM roles for App Runner. There are two kinds of roles relevant for app runner. The _access role_ manages the permission to the ECR repository and in the _instance role_ you can define access to other AWS services that the service needs at runtime.

```typescript
const accessRole = new Role(this, "ApprunnerCdkExampleAccessRole", {
  assumedBy: new ServicePrincipal("build.apprunner.amazonaws.com"),
});

// make sure App Runner can pull from ECR
accessRole.addToPolicy(
  new PolicyStatement({
    effect: Effect.ALLOW,
    actions: [
      "ecr:BatchCheckLayerAvailability",
      "ecr:BatchGetImage",
      "ecr:DescribeImages",
      "ecr:GetAuthorizationToken",
      "ecr:GetDownloadUrlForLayer",
    ],
    resources: ["*"],
  })
);

// allow the service to read from S3. This way you can use the AWS sdk to read from s3 inside your container
const instanceRole = new Role(this, "ApprunnerCdkExampleInstanceRole", {
  assumedBy: new ServicePrincipal("tasks.apprunner.amazonaws.com"),
  managedPolicies: [
    ManagedPolicy.fromAwsManagedPolicyName("AmazonS3ReadOnlyAccess"),
  ],
});
```

We then go on to define the environment variables we would like to have on our service. Because the CnfService of apprunner takes in a `[{ name: xyz, value: xyz }, ...]` structure, I wrote a little function to format the list of variables:

```typescript
const envVars: Record<string, string> = {
  SOME_ENVIRONMENT_VAR: "xyz",
  ANOTHER_ENV: "miauw",
  FINAL_ONE: "bark",
};
const mappedEnvVars = Object.keys(envVars).map((key) => ({
  name: key,
  value: envVars[key],
}));
```

Then we can finaly define our App Runner service 🥳 Check the [CfnService Apprunner](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_apprunner.CfnService.html) docs to see the full list of props you can provide.

```typescript
const app = new CfnService(this, "ApprunnerCdkExampleService", {
  sourceConfiguration: {
    autoDeploymentsEnabled: true,
    authenticationConfiguration: {
      accessRoleArn: accessRole.roleArn,
    },
    imageRepository: {
      imageIdentifier: `${repository.repositoryUri}:latest`,
      imageRepositoryType: "ECR",
      imageConfiguration: {
        port: "80",
        runtimeEnvironmentVariables: mappedEnvVars,
      },
    },
  },
  healthCheckConfiguration: {
    unhealthyThreshold: 5,
    interval: 5,
  },
  // optional autoscalingconfiguration
  //  autoScalingConfigurationArn: appRunnerAutoScaling.autoScalingConfigurationArn,
  instanceConfiguration: {
    instanceRoleArn: instanceRole.roleArn,
  },
  networkConfiguration: {
    egressConfiguration: {
      egressType: "VPC",
      vpcConnectorArn: vpcConnector.attrVpcConnectorArn,
    },
  },
});

// App Runner URL output
new CfnOutput(this, "AppRunnerServiceUrl", {
  value: `https://${app.attrServiceUrl}`,
});
```

And there you go! A deployed container in App Runner deployed via CDK 🚀

**Oh, one more thing.** If you want to configure autoscaling for apprunner, the only thing you can do in the _CnfService_ is to provide an ARN to the autoscaling configuration. If you want to create this configuration in cdk, you have to use a custom resource for this (as far as I know, this is the only way to do it). I've created a construct for this, see usage in the [example github repo](https://github.com/appelstroop/apprunner-cdk-example).

```typescript
export class AppRunnerAutoScaling extends Construct {
  readonly autoScalingConfigurationArn: string;
  constructor(
    scope: Construct,
    id: string,
    autoScalingConfiguration: CreateAutoScalingConfigurationCommandInput
  ) {
    super(scope, id);

    const createAutoScalingConfiguration = new AwsCustomResource(
      this,
      "CreateAutoScalingConfiguration",
      {
        onCreate: {
          service: "AppRunner",
          action: "createAutoScalingConfiguration",
          parameters: autoScalingConfiguration,
          physicalResourceId: PhysicalResourceId.fromResponse(
            "AutoScalingConfiguration.AutoScalingConfigurationArn"
          ),
        },
        policy: AwsCustomResourcePolicy.fromSdkCalls({
          resources: AwsCustomResourcePolicy.ANY_RESOURCE,
        }),
      }
    );

    const autoScalingConfigurationArn =
      createAutoScalingConfiguration.getResponseField(
        "AutoScalingConfiguration.AutoScalingConfigurationArn"
      );

    new AwsCustomResource(this, "DeleteAutoScalingConfiguration", {
      onDelete: {
        service: "AppRunner",
        action: "deleteAutoScalingConfiguration",
        parameters: {
          AutoScalingConfigurationArn: autoScalingConfigurationArn,
        },
      },
      policy: AwsCustomResourcePolicy.fromSdkCalls({
        resources: AwsCustomResourcePolicy.ANY_RESOURCE,
      }),
    });
    this.autoScalingConfigurationArn = autoScalingConfigurationArn;
  }
}
```

Then, add it to the stack and enable the _autoScalingConfigurationArn_ in the CfnService of the App Runner:

```typescript
// Create autoscaling from custom construct
const appRunnerAutoScaling = new AppRunnerAutoScaling(
  this,
  "ApprunnerAutoscaling",
  {
    AutoScalingConfigurationName: "apprunner-autoscaling",
    MinSize: 1,
    MaxSize: 3,
    MaxConcurrency: 100, // defines after how many concurrent requests app runner should scale up
  }
);

// ... appRunner CfnService
 autoScalingConfigurationArn: appRunnerAutoScaling.autoScalingConfigurationArn,
// ... rest of CfnService
```
