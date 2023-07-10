# Customizing S3 Key with Metadata

In S3, the "file name" in a bucket is called the S3 key.

*This example discusses options to inject metadata values into the S3 key, using ECS metadata as a use case. However, the techniques here can be used for any type of metadata.*

In Fluent Bit, [every log record is given a tag](https://docs.fluentbit.io/manual/concepts/key-concepts), which defines how it is routed through the pipeline and which plugin configurations apply to it.

The [S3 output](https://docs.fluentbit.io/manual/pipeline/outputs/s3/) has an option `s3_key_format` which can use the log tag and environment variables in the S3 key.

This tutorial has two options depending on how you obtain ECS Metadata in logs. 
* [Use Metadata Environment Variables from ECS Init Tag](#use-metadata-environment-variables-from-ecs-init-tag)
* [ Inject metadata from keys in logs with FireLens enable-ecs-log-metadata](#inject-metadata-from-keys-in-logs-with-firelens-enable-ecs-log-metadata)

Please also see our [FAQ on ECS Metadata](https://github.com/aws-samples/amazon-ecs-firelens-examples/tree/mainline/examples/fluent-bit/metadata-s3-key/).

## Use Metadata Environment Variables from ECS Init Tag 

Follow the [init ECS Metadata example](../init-metadata/) to understand that the Fluent Bit init tag adds ECS metadata environment variables.

Then simply reference those variables in your S3 output `s3_key_format`:

```
[OUTPUT]
    Name                         s3
    Match                        *
    bucket                       my-bucket
    region                       us-west-2
    total_file_size              250M
    s3_key_format                /${ECS_CLUSTER}/${ECS_FAMILY}/${ECS_TASK_ID}/%Y/%m/%d/%H/%M/%S/$UUID.gz
```

The init image can also [pull configuration files from S3](https://github.com/aws-samples/amazon-ecs-firelens-examples/tree/mainline/examples/fluent-bit/multi-config-support), which is used in the [task-definition-init.json](task-definition-init.json).

## Inject metadata from keys in logs with FireLens enable-ecs-log-metadata

FireLens can add metadata to your logs with the `enable-ecs-log-metadata` setting that defaults to true:

```
"firelensConfiguration": {
				"type": "fluentbit",
				"enable-ecs-log-metadata": "true",
}
```

This [adds keys](https://github.com/aws/aws-for-fluent-bit/blob/mainline/troubleshooting/debugging.md#what-will-the-logs-collected-by-fluent-bit-look-like) to your logs like:
```
"ecs_cluster": "cluster-name",
"ecs_task_arn": "arn:aws:ecs:region:111122223333:task/cluster-name/f2ad7dba413f45ddb4EXAMPLE",
"ecs_task_definition": "task-def-name:revision",
```

On EC2, the instance ID will also be added:
```
"ec2_instance_id": "i-06bc83dbc2ac2fdf8"
``` 

The Fluentd log driver also always adds the following keys:
```
"source": "stdout",
"container_id": "e54cccfac2b87417f71877907f67879068420042828067ae0867e60a63529d35",
"container_name": "/ecs-demo-6-container2-a4eafbb3d4c7f1e16e00"
```

Please notice that the Task ID is not a separate piece of metadata, this is one disadvantage to this option.

This is achieved by a [record_modifier filter](https://github.com/aws-samples/amazon-ecs-firelens-under-the-hood/blob/mainline/generated-configs/fluent-bit/generated_by_firelens.conf#L21) in the [FireLens generated config](https://aws.amazon.com/blogs/containers/under-the-hood-firelens-for-amazon-ecs-tasks/) that matches `*`. This means the metadata will be added to stdout & stderr as well as any logs ingested by [custom inputs](https://github.com/aws-samples/amazon-ecs-firelens-examples/tree/mainline/examples/fluent-bit/ecs-log-collection) you add. 

Remember, the `s3_key_format` can include the log tag. To customize the S3 key with ECS metadata, we first must customize the tag with ECS metadata. With the [rewrite tag filter](https://docs.fluentbit.io/manual/pipeline/filters/rewrite-tag), we can customize the tag using keys in the logs. 

```
[FILTER]
    Name                  rewrite_tag
    Match                 *
    Rule                  $log  ^[\S]+$  app.$ecs_cluster.$container_name false
    Emitter_Name          re_emitted_metadata
    Emitter_Mem_Buf_Limit 100M
```

The rule:
```
$log  ^[\S]+$  app.$ecs_cluster.$container_name
```

Applies if the `log` key exists and has any non-whitespace characters. FireLens will send your container stdout and stderr log lines wrapped in a key called `log`. The rule then changes the tag to be `app.{ECS Cluster}.{container name}`. 

Obviously, both of these pieces of information static at the time when you deploy your application task, and thus this information could simply be statically defined in your configuration for the S3 key. However, this example shows the general technique of customizing an S3 key using metadata keys in logs, which is useful for other use cases. 

Next, we define the S3 output:

```

[OUTPUT]
    Name                         s3
    Match                        app*
    bucket                       ${BUCKET_NAME_TASK}
    region                       ${BUCKET_REGION_TASK}
    upload_timeout               2m
    total_file_size              10M
    s3_key_format                /aws/ecs/$TAG[1]/myservice/$TAG[2]/%Y_%m_%d/%H_%M_%S
    s3_key_format_tag_delimiters .
    store_dir                    /var/fluent-bit/s3-store/task
```

[Fluent Bit S3](https://docs.fluentbit.io/manual/pipeline/outputs/s3/) supports indexing "parts" of the tag with a `$TAG[n]` syntax. The tag is split on the `s3_key_format_tag_delimiters` chracters. 

Assuming the tag is, `app.my-cluster.my-container`, the S3 key will be `/aws/ecs/my-cluster/myservice/my-container`. 


