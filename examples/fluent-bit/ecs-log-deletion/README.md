# ECS FireLens Log Deletion Example

This example will walk you through options to set up log file deletion/clean up in your ECS FireLens task.

This example is based upon the tail example in the [ecs-log-collection](https://github.com/aws-samples/amazon-ecs-firelens-examples/tree/mainline/examples/fluent-bit/ecs-log-collection) FireLens example. It adds log deletion.

## Scenario

For this example, consider that you app produces log files with paths like `/var/log/service-{timestamp}.log`. Each hour, your app write to a log file with a new timestamp. Thus, if your app runs for a very long time, you will quickly have lots of log files sitting around on disk. This is not ideal for two reasons:
1. Your disk may eventually fill up.
2. The large number of log files may slow down Fluent Bit. Fluent Bit continuously scans for files matched by your configured `Path` and runs a syscall on them to check if they were modified. If you have tons of log files that it must scan through, this will cause it to slow down, and in extreme cases, you may even lose logs. 

Fluent Bit will use this base configuration (based on the ECS Log Collection example) to tail the log files:

```
[INPUT]
    Name              tail
    Tag               service-log
    Path              /var/log/service*.log
    DB                /var/log/flb_service.db
    DB.locking        true
    Skip_Long_Lines   On
    Refresh_Interval  10
    Rotate_Wait       30

# Output for stdout logs
# Change the match if your container is not named 'app'
[OUTPUT]
    Name cloudwatch
    Match   app-firelens*
    region us-east-1
    log_group_name firelens-tutorial-$(ecs_cluster)
    log_stream_name /logs/app/$(ec2_instance_id)-$(ecs_task_id)
    auto_create_group true
    retry_limit 2

# Output for the file logs
[OUTPUT]
    Name cloudwatch
    Match   service-log
    region us-east-1
    log_group_name firelens-tutorial-$(ecs_cluster)
    log_stream_name /logs/service/$(ec2_instance_id)-$(ecs_task_id)
    auto_create_group true
    retry_limit 2
```

## Log Deletion Options

### 1. Use Fluent Bit Exec Input to run deletion command

Prerequisites:
* Flue
* The volume mount for Fluent Bit must not be read-only

### 2. Run deletion command with cron

### 3. Use log4j delete action