# ECS Metadata in logs FAQ

This examples covers all of the ways common to use ECS metadata in logs. 

### Use Cases
* **Add metadata as keys in log objects**: Inject the metadata into the log events themselves, so that events can later be searched by metadata.
* **Customize destination resource with metadata**: Use metadata in the output plugin. For CloudWatch Logs, this means customizing the log group and/or stream name using metadata. For S3, this means customizing the S3 key with metadata. 

### Examples

* 

### Techniques