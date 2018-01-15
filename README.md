# terraform-awselasticsearch

Terraform module to setup all resources needed for setting up an AWS Elasticsearch Service domain.

## awselasticseacrh

### Available variables

* [`project`]: String(required): Project name
* [`environment`]: String(required): Environment name
* [`name`]: String(optional, \"es\"): Name to use for the Elasticsearch domain
* [`version`]: String(optional, \"5.5\": Version of the Elasticsearch domain
* [`advanced_options`]: Map(optional, {}): An Elasticsearch advanced_options block, which consists of Elasticsearch configuration options as key-value pairs
* [`logging_enabled`]: Bool(optional, false): Whether to enable Elasticsearch slow logs in Cloudwatch
* [`logging_retention`]: Int(optional, 30): How many days to retain Elasticsearch logs in Cloudwatch
* [`instance_count`]: Int(optional, 1): Size of the Elasticsearch domain
* [`instance_type`]: String(optional, t2.small.elasticsearch): Instance type to use for the Elasticsearch domain
* [`dedicated_master_enabled`]: Bool(optional, false): Whether dedicated master nodes are enabled for the domain
* [`dedicated_master_type`]: String(optional, t2.small.elasticsearch): Instance type of the dedicated master nodes in the domain
* [`dedicated_master_count`]: Int(optional, 1): Number of dedicated master nodes in the domain
* [`volume_type`]: String(optional, \"gp2\"): EBS volume type to use for the Elasticsearch domain
* [`volume_size`]: Int(required): EBS volume size (in GB) to use for the Elasticsearch domain
* [`volume_iops`]: Int(required if volume_type=\"io1\"): Amount of provisioned IOPS for the EBS volume
* [`vpc_id`]: String(required*): VPC ID where to deploy the Elasticsearch domain. If set, you also need to specify `subnet_ids`. If not set, the module creates a public domain
* [`subnet_ids`]: List(required*): Subnet IDs for the VPC enabled Elasticsearch domain endpoints to be created in"
* [`security_group_ids`]: List(optional): Extra security group IDs to attach to the Elasticsearch domain. Note: a default SG is already created and exposed via outputs
* [`snapshot_start_hour`]: Int(optional, 3): Hour during which an automated daily snapshot is taken of the Elasticsearch indices
* [`snapshot_bucket_enabled`]: Bool(optional, false): Whether to create a bucket for custom Elasticsearch backups (other than the default daily one)
* [`tags`]: Map(optional, {}): Optional tags

**(*)** If the `vpc_id` and `subnet_ids` are not specified, this module will create a public Elasticsearch domain. 

### Outputs

* [`arn`]: ARN of the Elasticsearch domain
* [`domain_id`]: ID of the Elasticsearch domain
* [`domain_name`]: Name of the Elasticsearch domain
* [`endpoint`]: DNS endpoint of the Elasticsearch domain
* [`sg_id`]: ID of the Elasticsearch security group
* [`role_arn`]: ARN of the IAM role (eg to attach to an instance or user) allowing access to the Elasticsearch snapshot bucket
* [`role_id`]: ID of the IAM role (eg to attach to an instance or user) allowing access to the Elasticsearch snapshot bucket

### Example

```terraform
module "elasticsearch" {
  source                   = "github.com/skyscrapers/terraform-awselasticsearch?ref=0.1"
  name                     = "es"
  project                  = "${var.project}"
  environment              = "${terraform.workspace}"
  version                  = "5.5"
  instance_count           = 2
  instance_type            = "t2.medium.elasticsearch"
  volume_size              = 100
  vpc_id                   = "${data.terraform_remote_state.static.vpc_id}"
  subnet_ids               = ["${slice(data.terraform_remote_state.static.db_subnets,0,var.es_instance_count)}"]
  dedicated_master_enabled = false

  advanced_options {
    "indices.fielddata.cache.size"           = ""
    "rest.action.multi.allow_explicit_index" = "true"
  }
}

resource "aws_elasticsearch_domain_policy" "es_policy" {
  domain_name = "${module.elasticsearch.domain_name}"

  access_policies = <<POLICY
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": "es:*",
            "Principal": [
              "AWS": "${aws_iam_user.es_user.arn}"
            ],
            "Effect": "Allow",
            "Resource": "${module.elasticsearch.arn}/*"
        }
    ]
}
POLICY
}
```

## Backups

The AWS Elasticsearch Service handles backups automatically via daily snapshots. You can control when this happens by setting `snapshot_start_hour`.

It's possible to create a custom backup schedule by using the normal Elasticsearch API for snapshotting. This module can create an S3 bucket and IAM role allowing such scenario's (`snapshot_bucket_enabled = true`). More info on how to create custom snapshots can be found in the [AWS documentation](https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/es-managedomains-snapshots.html).

## Logging

This module by default creates Cloudwatch Log Groups & IAM permissions for ElasticSearch slow logging, but we don't enable these logs by default. You can control logging behavior via the `logging_enabled` and `logging_retention` parameters. When enabling this, make sure you also enable this on Elasticsearch side, following the [AWS documentation](https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/es-createupdatedomains.html#es-createdomain-configure-slow-logs).
