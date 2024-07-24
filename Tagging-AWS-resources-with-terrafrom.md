# Tagging AWS resources the right way using Terraform

Keeping your AWS resources organized and tracking costs can be a challenge, especially as your infrastructure grows.

Tagging resources is a simple yet great solution, but doing it effectively requires following best practices.

In this blog post, we’ll show you how to tag AWS resources using the Infrastructure as Code (IaC) tool ***Terraform***.

# **Introduction to Tagging AWS Resources:**

Tagging AWS resources is important for maintaining an organized and cost-effective cloud infrastructure.

Tags are ***key-value pairs*** that allow you to categorize and manage resources based on criteria like environment, application, team, etc.

Consistent tagging provides benefits like better ***resource organization, cost allocation, automation, security,*** and ***lifecycle management***.

In Terraform, you can tag resources during provisioning. For example, to tag an S3 bucket with **environment** and **team** tags:

```
resource "aws_s3_bucket" "example" {
  bucket = "my-bucket"
  tags = {
    Environment = "Production"
    Team = "DevOps"
  }
}
```

You can also define default tags at the provider level, which apply to all resources:

```
provider "aws" {
  default_tags {
    tags = {
      Environment = "Production"
      ManagedBy = "Terraform"
    }
  }
}
```

# **Overriding Default Tags at Resource Level:**

Defining default tags at the provider level using the `default_tags` block promotes consistency and reduces manual work by automatically applying common tags to all resources provisioned by that provider.

***Benefits of Default Tags***

- *Consistency:* Ensures all resources have a base set of tags applied
- *Less Manual Work:* Avoids repetitive tag definitions across resources

For example, setting default tags in the AWS provider:

```
provider "aws" {
  default_tags {
    tags = {
      Environment = "Production"
      ManagedBy   = "Terraform"
    }
  }
}
```

How to Override Default Tags or Add Extra Tags you can override the default tags or add new tags at the resource level by specifying the `tags` argument.

This argument takes precedence over the default tags defined at the provider level.

```
resource "aws_s3_bucket" "example" {
  bucket = "my-bucket"

  # Override default Environment tag and add Purpose tag
  tags = {
    Environment = "Staging"
    Purpose     = "Data Processing"
  }
}
```

In this example, the **`Environment`** tag is overridden with the value **`Staging`**, while the **`Purpose`** tag is added specifically for this S3 bucket resource.

The **`*ManagedBy*`** default tag is still applied.

***Use Cases for Resource-Level Tag Customization***

- Environment-specific tags (dev, staging, prod, etc.)
- Application/project-specific tags
- Resource-specific metadata tags (e.g., purpose, owner, expiration)
- Compliance or regulatory tags based on data sensitivity
- Cost allocation tags for specific resources

By allowing tag overrides at the resource level, you maintain the benefits of default tags while gaining the flexibility to customize tags based on the specific needs of individual resources.

# **Using Variables and Functions for Flexible Tagging:**

Terraform allows you to define tags as variables and use functions like `merge()` to combine them with other tags, promoting reusability and flexibility.

Defining tags as variables.

```
variable "default_tags" {
  default = {
    Environment = "Production"
    ManagedBy   = "Terraform"
  }
}
```

Using the `merge()` function to combine tags:

```
resource "aws_instance" "example" {
  ami           = "ami-0c94855ba95c71c99"
  instance_type = "t2.micro"

  tags = merge(
    var.default_tags,
    {
      Name    = "ExampleInstance"
      Project = "MyApp"
    }
  )
}
```

The `merge()` function combines the `default_tags` variable with additional resource-specific tags, resulting in all four tags being applied to the EC2 instance.

# **Handling Special Cases:**

Some AWS resources require specific tagging configurations or have limitations on how tags can be applied.

***Tagging Auto Scaling Groups:***

Auto Scaling Groups [ASG](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html?ref=dept-engineering-blog.ghost.io) and Launch Templates [LT](https://docs.aws.amazon.com/autoscaling/ec2/userguide/LaunchTemplates.html?ref=dept-engineering-blog.ghost.io) are tricky to tag correctly.

Without the right configuration, the EC2 instance and attached storage volumes launched by the ASG and LT will not have the default tags attached.

ASGs require the [propagate_at_launch](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/autoscaling_group?ref=dept-engineering-blog.ghost.io#propagate_at_launch) tag configuration.

```
resource "aws_autoscaling_group" "example" {
  # ...
  tag {
    key                 = "Environment"
    value               = "Production"
    propagate_at_launch = true
  }
}
```

***Tagging Launch Templates:***

Launch templates require the [tag_specifications](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/launch_template?ref=dept-engineering-blog.ghost.io#tag-specifications) configuration:

```
resource "aws_launch_template" "example" {
  # ...

 tag_specifications {
    resource_type = "instance"
    tags = {
      Environment = "Production"
      ManagedBy   = "Terraform"
    }
  }
  tag_specifications {
    resource_type = "volume"
    tags = {
      Persistence = "Permanent"
    }
  }
}
```

**Tagging EBS Volumes:**

When you create Elastic Compute ([EC2](https://aws.amazon.com/ec2/?ref=dept-engineering-blog.ghost.io)) instances via Terraform, the Elastic Block Store ([EBS](https://aws.amazon.com/ebs/?ref=dept-engineering-blog.ghost.io)) volumes attached to the EC2 are not automatically tagged. Untagged EBS volumes are cumbersome to administer.

You assign the EC2 default tags to the attached EBS storage volume with the aws_instance [volume_tags](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance?ref=dept-engineering-blog.ghost.io#volume_tags) .

```
resource "aws_instance" "example" {
  # ...

 volume_tags = {
    Name        = "DataVolume"
    Persistence = "Permanent"
  }
}
```

***Other Special Cases:***

Certain resources like AMIs, NAT Gateways, or VPC Endpoints may have specific tagging requirements or limitations.

Always refer to the Terraform provider documentation for the latest guidance on tagging configurations for different resource types.

# **Avoiding Common Pitfalls:**

***Inconsistent tag naming conventions:***

Using inconsistent tag keys like **`appid`**, **`*app_role*`**, and **`*AppPurpose*`** makes tags harder to use and manage.

```
resource "aws_s3_bucket" "example" {
  bucket = "my-bucket"
  tags = {
    appid     = "myapp"
    app_role  = "data-processing"
    AppPurpose = "logs"
  }
}
```

Instead, define [an explicit ruleset](https://docs.aws.amazon.com/whitepapers/latest/tagging-best-practices/adopt-a-standardized-approach-for-tag-names.html?ref=dept-engineering-blog.ghost.io) for tag key naming and stick with it.

***Not tagging all resources (including secondary resources):***

Failing to tag all AWS resources, including secondary or complementary resources like EBS volumes, leads to incomplete visibility and cost tracking.

```
resource "aws_instance" "example" {
  ami           = "ami-0c94855ba95c71c99"
  instance_type = "t2.micro"
  tags = {
    Environment = "Production"
  }
}
```

***Identical default and resource tags issue:***

Having identical tag keys and values in both `default_tags` and resource `tags`causes an error in Terraform, requiring deduplicating tags or using workarounds.

```
provider "aws" {
  default_tags {
    tags = {
      Name = "Example"
    }
  }
}

resource "aws_vpc" "example" {
  tags = {
    Name = "Example" # Error: tags are identical
  }
}
```

***Perpetual diff for partial tag matches:***

When `default_tags` and resource `tags` have some matching and some differing tags, Terraform shows a perpetual diff trying to update the matching tags on every plan, requiring workarounds.

```
provider "aws" {
  default_tags {
    tags = {
      Match1 = "A"
      Match2 = "B"
      NoMatch = "X"
    }
  }
}

resource "aws_vpc" "example" {
  tags = {
    Match1 = "A" # Perpetual diff trying
    Match2 = "B" # to update these
    NoMatch = "Y"
  }
}
```

***Infrastructure drift and tag loss:***

Losing tags due to infrastructure drift when resources are modified outside of Terraform. Using IaC consistently helps mitigate this issue.

# **Best Practices and Tips:**

***Establish a clear tagging strategy and naming convention:***

Define a consistent set of tag keys and naming conventions to use across your infrastructure.

```
variable "tag_names" {
  default = {
    environment = "Environment"
    application = "Application"
    team        = "Team"
    costcenter  = "CostCenter"
  }
}
```

***Tag resources as you provision them (not after):***

Apply tags to resources during the provisioning process, not after the fact, to ensure consistent tagging from the start.

```
resource "aws_s3_bucket" "example" {
  bucket = "my-bucket"
  tags = {
    (var.tag_names.environment) = "Production"
    (var.tag_names.application) = "MyApp"
  }
}
```

***Regularly review and audit tags:***

Periodically review and audit resource tags to ensure compliance with your tagging strategy and identify any missing or incorrect tags.

***Automate tagging where possible:***

Leverage Terraform’s features like `default_tags`, variables, and functions to automatically apply tags during provisioning, reducing manual effort and promoting consistency.

# **AWS Resource Groups and Tag Editor:**

Have you ever wanted to do the following: “Find all AWS resources in all regions that have the tag `team='platform engineering'` “?

AWS Resource Groups and Tag Editor are powerful tools that allow you to manage tags across multiple AWS resources and regions effectively.

***Resource Groups:***

Resource Groups provide a centralized way to organize and manage collections of AWS resources based on shared tags. With Resource Groups, you can:

1. Find resources across regions that have specific tags applied, such as `team='platform engineering'`.
2. Identify resources that are missing tags or have incorrect tag values.
3. Automate operations like starting/stopping instances or applying configurations based on resource group membership.
4. View consolidated information about resource status, costs, and configurations within a group.

***Tag Editor:***

The Tag Editor is a component of Resource Groups that enables bulk tagging operations across supported AWS services and regions. Using the Tag Editor, you can:

1. Search for resources based on resource types and existing tags, allowing queries like “Find all EC2 instances with `team='platform engineering'`.
2. Add, modify, or remove tags on multiple resources simultaneously, streamlining tagging efforts.
3. Preview the changes before applying them, ensuring accuracy and avoiding unintended modifications.
4. Use tag-based access control policies to manage resource access based on tag values.

# **Conclusion:**

Proper tagging is substantial for organized, cost-friendly AWS setups, make sure to use Terraform’s tagging tools and the best ways and avoid common mistakes.