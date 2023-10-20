# AWS Lambda Power Tuning

AWS Lambda Power Tuning is a state machine powered by AWS Step Functions that helps you optimize your Lambda functions for cost and/or performance in a data-driven way.

The state machine is designed to be easy to deploy and fast to execute. Also, it's language agnostic so you can optimize any Lambda functions in your account.

Basically, you can provide a Lambda function ARN as input and the state machine will invoke that function with multiple power configurations (from 128MB to 10GB, you decide which values). Then it will analyze all the execution logs and suggest you the best power configuration to minimize cost and/or maximize performance.

Please note that the input function will be executed in your AWS account - performing real HTTP requests, SDK calls, cold starts, etc. The state machine also supports cross-region invocations and you can enable parallel execution to generate results in just a few seconds.

The state machine will also generate a visualization of average cost and speed for each power configuration.


## How to execute the state machine

Once the state machine and all the Lambda functions have been deployed, you can execute the state machine and provide an input object.

You will find the new state machine in the [Step Functions Console](https://console.aws.amazon.com/states/) or in your app's `Resources` section.

The state machine name will be prefixed with `powerTuningStateMachine`. Find it and click "**Start execution**". Here you can provide the execution input and an execution id (see section below for the full documentation):

```json
{
    "lambdaARN": "your-lambda-function-arn",
    "powerValues": [128, 256, 512, 1024, 2048, 3008],
    "num": 10,
    "payload": "{}",
    "parallelInvocation": true,
    "strategy": "cost"
}
```

As soon as you click "**Start Execution**" again, you'll be able to visualize the execution.

Once the execution has completed, you will find the execution results in the "**Output**" tab of the "**Execution Details**" section. The output will contain the optimal power configuration and its corresponding average cost per execution.


## State machine input (at execution time)

The state machine accepts the following input parameters:

* **lambdaARN** (required, string): unique identifier of the Lambda function you want to optimize
* **powerValues** (optional, string or list of integers): the list of power values to be tested; if not provided, the default values configured at deploy-time are used (by default: 128MB, 256MB, 512MB, 1024MB, 1536MB, and 3008MB); you can provide any power values between 128MB and 10,240MB (⚠️ [New AWS accounts have reduced concurrency and memory quotas (3008MB max)](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html))
* **num** (required, integer): the # of invocations for each power configuration (minimum 5, recommended: between 10 and 100)
* **payload** (string, object, or list): the static payload that will be used for every invocation (object or string); when using a list, a weighted payload is expected in the shape of `[{"payload": {...}, "weight": X }, {"payload": {...}, "weight": Y }, {"payload": {...}, "weight": Z }]`, where the weights `X`, `Y`, and `Z` are treated as relative weights (not percentages); more details below in the Weighted Payloads section
* **payloadS3** (string): a reference to Amazon S3 for large payloads (>256KB), formatted as `s3://bucket/key`; it requires read-only IAM permissions, see `payloadS3Bucket` and `payloadS3Key` below and find more details in the S3 payloads section
* **parallelInvocation** (false by default): if true, all the invocations will be executed in parallel (note: depending on the value of `num`, you may experience throttling when setting `parallelInvocation` to true)
* **strategy** (string): it can be `"cost"` or `"speed"` or `"balanced"` (the default value is `"cost"`); if you use `"cost"` the state machine will suggest the cheapest option (disregarding its performance), while if you use `"speed"` the state machine will suggest the fastest option (disregarding its cost). When using `"balanced"` the state machine will choose a compromise between `"cost"` and `"speed"` according to the parameter `"balancedWeight"`
* **balancedWeight** (number between 0.0 and 1.0, by default is 0.5): parameter that express the trade-off between cost and time, 0.0 is equivalent to `"speed"` strategy, 1.0 is equivalent to `"cost"` strategy
* **autoOptimize** (false by default): if `true`, the state machine will apply the optimal configuration at the end of its execution
* **autoOptimizeAlias** (string): if provided - and only if `autoOptimize` if `true`, the state machine will create or update this alias with the new optimal power value
* **dryRun** (false by default): if true, the state machine will execute the input function only once and it will disable every functionality related to logs analysis, auto-tuning, and visualization; the dry-run mode is intended for testing purposes, for example to verify that IAM permissions are set up correctly
* **preProcessorARN** (string): it must be the ARN of a Lambda function; if provided, the function will be invoked before every invocation of `lambdaARN`; more details below in the Pre/Post-processing functions section
* **postProcessorARN** (string): it must be the ARN of a Lambda function; if provided, the function will be invoked after every invocation of `lambdaARN`; more details below in the Pre/Post-processing functions section
* **discardTopBottom** (number between 0.0 and 0.4, by default is 0.2): By default, the state machine will discard the top/bottom 20% of "outliers" (the fastest and slowest), to filter out the effects of cold starts that would bias the overall averages. You can customize this parameter by providing a value between 0 and 0.4, with 0 meaning no results are discarded and 0.4 meaning that 40% of the top/bottom results are discarded (i.e. only 20% of the results are considered).
* **sleepBetweenRunsMs** (integer) If provided, the time in milliseconds that the tuner function will sleep/wait after invoking your function, but before carrying out the Post-Processing step, should that be provided. This could be used if you have aggressive downstream rate limits you need to respect. By default this will be set to 0 and the function won't sleep between invocations. Setting this value will have no effect if running the invocations in parallel.
* **disablePayloadLogs** (boolean) If provided and set to a truthy value, suppresses `payload` from error messages and logs. If `preProcessorARN` is provided, this also suppresses the output payload of the pre-processor.


## State machine configuration (at deployment time)

The CloudFormation template accepts the following parameters:

* **PowerValues** (list of numbers): these power values will be used as the default in case no `powerValues` input parameter is provided at execution time
* **visualizationURL** (string): the base URL for the visualization tool, by default it's `lambda-power-tuning.show` but you could use your own custom tool
* **totalExecutionTimeout** (number in seconds, default=`300`): the timeout in seconds applied to all functions of the state machine
* **lambdaResource** (string, default=`*`): the `Resource` used in IAM policies; it's `*` by default but you could restrict it to a prefix or a specific function ARN
* **permissionsBoundary** (string): the ARN of a permissions boundary (policy), applied to all functions of the state machine
* **payloadS3Bucket** (string): the S3 bucket name used for large payloads (>256KB); if provided, it's added to a custom managed IAM policy that grants read-only permission to the S3 bucket; more details below in the **S3 payloads section**
* **payloadS3Key** (string, default=`*`): the S3 object key used for large payloads (>256KB); the default value grants access to all S3 objects in the bucket specified with `payloadS3Bucket`; more details below in the **S3 payloads section**
* **layerSdkName** (string): the name of the SDK layer, in case you need to customize it (optional)
* **logGroupRetentionInDays** (number, default=7): the number of days to retain log events in the Lambda log groups. Before this parameter existed, log events were retained indefinitely
* **securityGroupIds** (list of SecurityGroup IDs): List of Security Groups to use in every Lambda function's VPC Configuration (optional); please note that your VPC should be configured to allow public internet access (via NAT Gateway) or include VPC Endpoints to the Lambda service
* **subnetIds** (list of Subnet IDs): List of Subnets to use in every Lambda function's VPC Configuration (optional); please note that your VPC should be configured to allow public internet access (via NAT Gateway) or include VPC Endpoints to the Lambda service

Please note that the total execution time should stay below 300 seconds (5 min), which is the default timeout. You can easily estimate the total execution timeout based on the average duration of your functions. For example, if your function's average execution time is 5 seconds and you haven't enabled `parallelInvocation`, you should set `totalExecutionTimeout` to at least `num * 5`: 50 seconds if `num=10`, 500 seconds if `num=100`, and so on. If you have enabled `parallelInvocation`, usually you don't need to tune the value of `totalExecutionTimeout` unless your average execution time is above 5 min.

### Usage in CI/CD pipelines

If you want to run the state machine as part of your continuous integration pipeline and automatically fine-tune your functions at every deployment, you can execute it with the script `scripts/execute.sh` (or similar) by providing the following input parameters:

```json
{
    "lambdaARN": "...",
    "num": 10,
    "payload": {},
    "powerValues": [128, 256, 512, ...],
    "autoOptimize": true,
    "autoOptimizeAlias": "prod"
}
```

You can use different alias names such as `dev`, `test`, `production`, etc. If you don't configure any alias name, the state machine will only update the `$LATEST` alias.

### Weighted Payloads

Weighted payloads can be used in scenarios where the payload structure and the corresponding performance/speed could vary a lot in production and you'd like to include multiple payloads in the tuning process.

You may want to use weighted payloads also in case of functions with side effects that would be hard or impossible to test with the very same payload (for example, a function that deletes records from a database).

You configure weighted payloads as follows:

```json
{
    ...
    "num": 50,
    "payload": [
        { "payload": {...}, "weight": 5 },
        { "payload": {...}, "weight": 15 },
        { "payload": {...}, "weight": 30 }
    ]
}
```

In the example above, the weights `5`, `15`, and `30` are used as relative weights. They will correspond to `10%` (5 out of 50), `30%` (15 out of 50), and `60%` (30 out of 50) respectively - meaning that the corresponding payload will be used 10%, 30% and 60% of the time.

For example, if `num=100` the first payload will be used 10 times, the second 30 times, and the third 60 times.

To simplify these calculations, you could use weights that sum up to 100.

Note: the number of weighted payloads must always be smaller or equal than `num` (or `num >= count(payloads)`). For example, if you have 50 weighted payloads, you'll need to set at least `num: 50` so that each payload will be used at least once.


### Pre/Post-processing functions

Sometimes you need to power-tune Lambda functions that have side effects such as creating or deleting records in a database. In these cases, you may need to execute some pre-processing or post-processing logic before and/or after each function invocation.

For example, imagine that you are power-tuning a function that deletes one record from a downstream database. Since you want to execute this function `num` times you'd need to insert some records in advance and then find a way to delete all of them with a dynamic payload. Or you could simply configure a pre-processing function (using the `preProcessorARN` input parameter) that will create a brand new record before the actual function is executed.

Here's the flow in pseudo-code:

```
function Executor:
  iterate from 0 to num:
    [payload = execute Pre-processor (payload)]
    results = execute Main Function (payload)
    [execute Post-processor (results)]
```

Please also keep in mind the following:

* You can configure a pre-processor and/or a post-processor independently
* The pre-processor will receive the original payload
* If the pre-processor returns a non-empty output, it will overwrite the original payload
* The post-processor will receive the main function's output as payload
* If a pre-processor or post-processor fails, the whole power-tuning state machine will fail
* Pre/post-processors don't have to be in the same region of the main function
* Pre/post-processors don't alter the statistics related to cost and performance

### S3 payloads

In case of very large payloads above 256KB, you can provide an S3 object reference (`s3://bucket/key`) instead of an inline payload.

Your state machine input will look like this:

```json
{
    "lambdaARN": "your-lambda-function-arn",
    "powerValues": [128, 256, 512, 1024],
    "num": 50,
    "payloadS3": "s3://your-bucket/your-object.json"
}
```

Please note that the state machine will require IAM access to your S3 bucket, so you might need to redeploy the Lambda Power Tuning application and configure the `payloadS3Bucket` parameter at deployment time. This will automatically generate a custom IAM managed policy to grant read-only access to that bucket. If you want to narrow down the read-only policy to a specific object or pattern, use the `payloadS3Key` parameter (which is `*` by default).

S3 payloads work fine with weighted payloads too.


## State Machine Output

The state machine will return the following output:

```json
{
  "results": {
    "power": "128",
    "cost": 0.0000002083,
    "duration": 2.9066666666666667,
    "stateMachine": {
      "executionCost": 0.00045,
      "lambdaCost": 0.0005252,
      "visualization": "https://lambda-power-tuning.show/#<encoded_data>"
    }
  }
}
```

More details on each value:

* **results.power**: the optimal power configuration (RAM)
* **results.cost**: the corresponding average cost (per invocation)
* **results.duration**: the corresponding average duration (per invocation)
* **results.stateMachine.executionCost**: the AWS Step Functions cost corresponding to this state machine execution (fixed value for "worst" case)
* **results.stateMachine.lambdaCost**: the AWS Lambda cost corresponding to this state machine execution (depending on `num` and average execution time)
* **results.stateMachine.visualization**: if you visit this autogenerated URL, you will be able to visualize and inspect average statistics about cost and performance; important note: average statistics are NOT shared with the server since all the data is encoded in the URL hash ([example](https://lambda-power-tuning.show/#gAAAAQACAAQABsAL;ZooQR4yvkUa/pQRGRC5zRaADHUVjOftE;QdWhOEMkoziDT5Q4xhiIOMYYiDi6RNc4)), which is available only client-side

## Statistics visualization

The data visualization tool has been built by the community. It is a static website deployed via AWS Amplify Console and it's free to use.

If you don't want to use the visualization tool, you can simply ignore the `stateMachine.visualization` output. No data is ever shared with this tool.

Website repository: [matteo-ronchetti/aws-lambda-power-tuning-ui](https://github.com/matteo-ronchetti/aws-lambda-power-tuning-ui)

Optionally, you could deploy your own custom visualization tool and configure the CloudFormation Parameter named `visualizationURL` with your own URL.

## Security

All the IAM roles used by the state machine adopt the least privilege best practice, meaning that only a minimal set of `Actions` are granted to each Lambda function.

For example, the Executor function can only call `lambda:InvokeFunction`. The Analyzer function doesn't require any permission at all. On the other hand, the Initializer, Cleaner, and Optimizer functions require a broader set of actions.

Although the default resource is `"*"`, you can optionally configure the `lambdaResource` CloudFormation parameter at deploy-time to constrain the IAM permission even more.

For example, you could use a mix of the following:

* Same-region prefix: `arn:aws:lambda:us-east-1:*:function:*`
* Function name prefix: `arn:aws:lambda:*:*:function:my-prefix-*`
* Function name suffix: `arn:aws:lambda:*:*:function:*-dev`
* By account ID: `arn:aws:lambda:*:ACCOUNT_ID:function:*`

## State machine cost

There are three main costs associated with AWS Lambda Power Tuning:

* **AWS Step Functions cost**: it corresponds to the number of state transitions during the state machine execution; this cost depends on the number of tested power values, and it's approximately `0.000025 * (6 + N)` where `N` is the number of power values; for example, if you test the 6 default power values, the state machine cost will be $0.0003
* **AWS Lambda cost** related to your function's executions: it depends on three factors: 1) number of invocations that you configure as input (`num`), the number of tested power configurations (`powerValues`), and the average invocation time of your function; for example, if you test all the default power configurations with `num: 100` and all invocations take less than 100ms, the Lambda cost will be approximately $0.001
* **AWS Lambda cost** related to `Initializer`, `Executor`, `Cleaner`, and `Analyzer`: for most cases it's negligible, especially if you enable `parallelInvocation: true`; this cost is not included in the `results.stateMachine` output to keep the state machine simple and easy to read and debug

## Error handling

If something goes wrong during the initialization or execution states, the `CleanUpOnError` step will be executed. All temporary versions and aliases will be deleted as expected (the same happens in the `Cleaner` step).

You can customize the `totalExecutionTimeout` parameter at deploy time (up to 15min). This parameter will be used both for Lambda function timeouts and Step Function tasks timeouts. In case the `Executor` raises a timeout error, you will see a `States.Timeout` error. Keep in mind that the timeout you configure will vary whether you're setting `parallelInvocation` to `true` or `false`. When you enable parallel invocation, all the function executions will run concurrently (rather than in series) so that you can keep that timeout lower and your overall state machine execution faster.

In all other error cases, you will see a `Lambda.Unknown` error, which corresponds to unhandled errors in Lambda such as out-of-memory errors and hitting the concurrent Lambda invoke limit. If you encounter it as input of `CleanUpOnError`, it's very likely that something went wrong with the function you're power-tuning.

### Retry policy

The executor will retry twice in case any invocation fails. This is helpful in case of execution timeouts or memory errors. You will find the failed execution's stack trace in the `CleanUpOnError` state input.

### How do I know which executor failed and why?

You can inspect the "Execution event history" and look for the corresponding `TaskStateAborted` event type.

Additionally, you can inspect the `CleanUpOnError` state input. Here you will find the stack trace of the error.

## State Machine Internals

The AWS Step Functions state machine is composed of five Lambda functions:

* **initializer**: create N versions and aliases corresponding to the power values provided as input (e.g. 128MB, 256MB, etc.)
* **executor**: execute the given Lambda function `num` times, extract execution time from logs, and compute average cost per invocation
* **cleaner**: delete all the previously generated aliases and versions
* **analyzer**: compute the optimal power value (current logic: lowest average cost per invocation)
* **optimizer**: automatically set the power to its optimal value (only if `autoOptimize` is `true`)

Initializer, cleaner, analyzer, and optimizer are executed only once, while the executor is used by N parallel branches of the state machine (one for each configured power value). By default, the executor will execute the given Lambda function `num` consecutive times, but you can enable parallel invocation by setting `parallelInvocation` to `true`.

## CHANGELOG (SAR versioning)

From most recent to oldest, with major releases in bold:

* *4.3.2* (2023-08-16): new disablePayloadLogs flag, updated documentation
* *4.3.1* (2023-05-09): update dependencies, add VPC Configuration support, use Billed Duration instead Duration from logs, update state machine with ItemSelector
* ***4.3.0*** (2023-03-06): SnapStart support (alias waiter)
* *4.2.3* (2023-03-01): fix layer runtime (nodejs16.x)
* *4.2.2* (2023-02-15): configurable sleep parameter, bump runtime to nodejs16.x, docs updates, GH Actions, and minor bug fixes
* *4.2.1* (2022-08-02): customizable SDK layer name and logs retention value
* ***4.2.0*** (2022-01-03): support S3 payloads
* *4.1.4* (2022-01-03): sorting bugfix and updated dependencies
* *4.1.3* (2021-12-16): support simple strings as event payload
* *4.1.2* (2021-10-12): add x86_64 fallback when Graviton is not supported yet
* *4.1.1* (2021-10-12): fixed connection timeout for long-running functions
* ***4.1.0*** (2021-10-11): support Lambda functions powered by Graviton2
* ***4.0.0*** (2021-08-16): support AWS Lambda states expansion to all functions
* *3.4.2* (2020-12-03): permissions boundary bugfix (Step Functions role)
* *3.4.1* (2020-12-02): permissions boundary support
* ***3.4.0*** (2020-12-01): 1ms billing
* *3.3.3* (2020-07-17): payload logging bugfix for pre-processors
* *3.3.2* (2020-06-17): weighted payloads bugfix (for real)
* *3.3.1* (2020-06-16): weighted payloads bugfix
* ***3.3.0*** (2020-06-10): Pre/Post-processing functions, correct regional pricing, customizable execution timeouts, and other internal improvements
* *3.2.5* (2020-05-19): improved logging for weighted payloads and in case of invocation errors
* *3.2.4* (2020-03-11): dryRun bugfix
* *3.2.3* (2020-02-25): new dryRun input parameter
* *3.2.2* (2020-01-30): upgraded runtime to Node.js 12.x
* *3.2.1* (2020-01-27): improved scripts and SAR template reference
* ***3.2.0*** (2020-01-17): support for weighted payloads
* *3.1.2* (2020-01-17): improved optimal selection when same speed/cost
* *3.1.1* (2019-10-24): customizable least-privilege (lambdaResource CFN param)
* ***3.1.0*** (2019-10-24): $LATEST power reset and optional auto-tuning (new Optimizer step)
* ***3.0.0*** (2019-10-22): dynamic parallelism (powerValues as execution parameter)
* *2.1.3* (2019-10-22): upgraded runtime to Node.js 10.x
* *2.1.2* (2019-10-17): new balanced optimization strategy
* *2.1.1* (2019-10-10): custom domain for visualization URL
* ***2.1.0*** (2019-10-10): average statistics visualization (URL in state machine output)
* ***2.0.0*** (2019-07-28): multiple optimization strategies (cost and speed), new output format with AWS Step Functions and AWS Lambda cost
* *1.3.1* (2019-07-23): retry policies and failed invocations management
* ***1.3.0*** (2019-07-22): implemented error handling
* *1.2.1* (2019-07-22): Node.js refactor and updated IAM permissions (added lambda:UpdateAlias)
* ***1.2.0*** (2019-05-24): updated IAM permissions (least privilege for actions)
* *1.1.1* (2019-05-15): updated docs
* ***1.1.0*** (2019-05-15): cross-region invocation support
* *1.0.1* (2019-05-13): new README for SAR
* ***1.0.0*** (2019-05-13): AWS SAM refactor (published on SAR)
* *0.0.1* (2017-03-27): previous project (serverless framework)
