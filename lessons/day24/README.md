 # Day 24: Monitoring and Observability

## Learning Objectives

By the end of this lesson, you will be able to:

- Understand infrastructure monitoring concepts and their importance
- Implement comprehensive logging strategies for Terraform-managed infrastructure
- Set up alerting and notification systems
- Apply observability best practices for infrastructure
- Configure monitoring tools and dashboards
- Implement distributed tracing and metrics collection

## Core Concepts

### 1. Infrastructure Monitoring Overview

Infrastructure monitoring involves:
- **Metrics Collection**: Performance and resource utilization data
- **Log Aggregation**: Centralized logging from all infrastructure components
- **Alerting**: Proactive notification of issues and anomalies
- **Visualization**: Dashboards and reports for operational insights
- **Distributed Tracing**: End-to-end request tracking across services

### 2. Observability Pillars

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│    Metrics      │    │     Logs        │    │     Traces      │
│                 │    │                 │    │                 │
│ • Performance   │    │ • Application   │    │ • Request Flow  │
│ • Resources     │    │ • System        │    │ • Dependencies  │
│ • Business      │    │ • Security      │    │ • Latency       │
│ • Custom        │    │ • Audit         │    │ • Errors        │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

## Practical Examples

### 1. CloudWatch Monitoring Setup

```hcl
# monitoring.tf
# CloudWatch Dashboard
resource "aws_cloudwatch_dashboard" "main" {
  dashboard_name = "${var.environment}-infrastructure-dashboard"

  dashboard_body = jsonencode({
    widgets = [
      {
        type   = "metric"
        x      = 0
        y      = 0
        width  = 12
        height = 6
        properties = {
          metrics = [
            ["AWS/EC2", "CPUUtilization", "AutoScalingGroupName", aws_autoscaling_group.main.name],
            [".", "NetworkIn", ".", "."],
            [".", "NetworkOut", ".", "."]
          ]
          period = 300
          stat   = "Average"
          region = var.aws_region
          title  = "EC2 Metrics"
        }
      },
      {
        type   = "metric"
        x      = 12
        y      = 0
        width  = 12
        height = 6
        properties = {
          metrics = [
            ["AWS/ApplicationELB", "TargetResponseTime", "LoadBalancer", aws_lb.main.arn_suffix],
            [".", "RequestCount", ".", "."],
            [".", "HTTPCode_Target_2XX_Count", ".", "."],
            [".", "HTTPCode_Target_4XX_Count", ".", "."],
            [".", "HTTPCode_Target_5XX_Count", ".", "."]
          ]
          period = 300
          stat   = "Sum"
          region = var.aws_region
          title  = "ALB Metrics"
        }
      },
      {
        type   = "metric"
        x      = 0
        y      = 6
        width  = 12
        height = 6
        properties = {
          metrics = [
            ["AWS/RDS", "CPUUtilization", "DBInstanceIdentifier", aws_db_instance.main.id],
            [".", "DatabaseConnections", ".", "."],
            [".", "FreeStorageSpace", ".", "."]
          ]
          period = 300
          stat   = "Average"
          region = var.aws_region
          title  = "RDS Metrics"
        }
      },
      {
        type   = "metric"
        x      = 12
        y      = 6
        width  = 12
        height = 6
        properties = {
          metrics = [
            ["AWS/ElastiCache", "CPUUtilization", "CacheClusterId", aws_elasticache_replication_group.main.id],
            [".", "DatabaseMemoryUsagePercentage", ".", "."],
            [".", "CurrConnections", ".", "."]
          ]
          period = 300
          stat   = "Average"
          region = var.aws_region
          title  = "ElastiCache Metrics"
        }
      }
    ]
  })
}

# CloudWatch Alarms
resource "aws_cloudwatch_metric_alarm" "cpu_high" {
  alarm_name          = "${var.environment}-cpu-utilization-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = "300"
  statistic           = "Average"
  threshold           = "80"
  alarm_description   = "This metric monitors EC2 CPU utilization"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.main.name
  }

  tags = {
    Environment = var.environment
    Component   = "EC2"
  }
}

resource "aws_cloudwatch_metric_alarm" "memory_high" {
  alarm_name          = "${var.environment}-memory-utilization-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "MemoryUtilization"
  namespace           = "System/Linux"
  period              = "300"
  statistic           = "Average"
  threshold           = "85"
  alarm_description   = "This metric monitors memory utilization"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.main.name
  }

  tags = {
    Environment = var.environment
    Component   = "EC2"
  }
}

resource "aws_cloudwatch_metric_alarm" "disk_high" {
  alarm_name          = "${var.environment}-disk-utilization-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "DiskSpaceUtilization"
  namespace           = "System/Linux"
  period              = "300"
  statistic           = "Average"
  threshold           = "90"
  alarm_description   = "This metric monitors disk space utilization"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.main.name
  }

  tags = {
    Environment = var.environment
    Component   = "EC2"
  }
}

# SNS Topic for Alerts
resource "aws_sns_topic" "alerts" {
  name = "${var.environment}-infrastructure-alerts"
  
  tags = {
    Environment = var.environment
    Purpose     = "Monitoring"
  }
}

resource "aws_sns_topic_subscription" "email" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "email"
  endpoint  = var.alert_email
}

resource "aws_sns_topic_subscription" "slack" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "https"
  endpoint  = var.slack_webhook_url
}
```

### 2. Centralized Logging with CloudWatch Logs

```hcl
# logging.tf
# CloudWatch Log Groups
resource "aws_cloudwatch_log_group" "application" {
  name              = "/aws/ec2/${var.environment}/application"
  retention_in_days = 30

  tags = {
    Environment = var.environment
    Component   = "Application"
  }
}

resource "aws_cloudwatch_log_group" "system" {
  name              = "/aws/ec2/${var.environment}/system"
  retention_in_days = 30

  tags = {
    Environment = var.environment
    Component   = "System"
  }
}

resource "aws_cloudwatch_log_group" "security" {
  name              = "/aws/ec2/${var.environment}/security"
  retention_in_days = 90

  tags = {
    Environment = var.environment
    Component   = "Security"
  }
}

# CloudWatch Log Metric Filters
resource "aws_cloudwatch_log_metric_filter" "error_logs" {
  name           = "${var.environment}-error-logs"
  pattern        = "[timestamp, level=ERROR, message]"
  log_group_name = aws_cloudwatch_log_group.application.name

  metric_transformation {
    name      = "ErrorCount"
    namespace = "Application"
    value     = "1"
  }
}

resource "aws_cloudwatch_log_metric_filter" "security_events" {
  name           = "${var.environment}-security-events"
  pattern        = "[timestamp, event=SECURITY, details]"
  log_group_name = aws_cloudwatch_log_group.security.name

  metric_transformation {
    name      = "SecurityEventCount"
    namespace = "Security"
    value     = "1"
  }
}

# CloudWatch Alarms for Log Metrics
resource "aws_cloudwatch_metric_alarm" "error_rate_high" {
  alarm_name          = "${var.environment}-error-rate-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "ErrorCount"
  namespace           = "Application"
  period              = "300"
  statistic           = "Sum"
  threshold           = "10"
  alarm_description   = "This metric monitors application error rate"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  tags = {
    Environment = var.environment
    Component   = "Application"
  }
}
```

### 3. Prometheus and Grafana Monitoring

```hcl
# prometheus.tf
# ECS Cluster for Monitoring Stack
resource "aws_ecs_cluster" "monitoring" {
  name = "${var.environment}-monitoring"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }

  tags = {
    Environment = var.environment
    Purpose     = "Monitoring"
  }
}

# Prometheus Task Definition
resource "aws_ecs_task_definition" "prometheus" {
  family                   = "${var.environment}-prometheus"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = 512
  memory                   = 1024

  container_definitions = jsonencode([
    {
      name  = "prometheus"
      image = "prom/prometheus:latest"

      portMappings = [
        {
          containerPort = 9090
          protocol      = "tcp"
        }
      ]

      mountPoints = [
        {
          sourceVolume  = "prometheus-config"
          containerPath = "/etc/prometheus"
          readOnly      = true
        }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          awslogs-group         = aws_cloudwatch_log_group.prometheus.name
          awslogs-region        = var.aws_region
          awslogs-stream-prefix = "ecs"
        }
      }
    }
  ])

  volume {
    name = "prometheus-config"
    efs_volume_configuration {
      file_system_id = aws_efs_file_system.prometheus_config.id
      root_directory = "/"
    }
  }

  tags = {
    Environment = var.environment
    Component   = "Prometheus"
  }
}

# Grafana Task Definition
resource "aws_ecs_task_definition" "grafana" {
  family                   = "${var.environment}-grafana"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = 256
  memory                   = 512

  container_definitions = jsonencode([
    {
      name  = "grafana"
      image = "grafana/grafana:latest"

      portMappings = [
        {
          containerPort = 3000
          protocol      = "tcp"
        }
      ]

      environment = [
        {
          name  = "GF_SECURITY_ADMIN_PASSWORD"
          value = var.grafana_admin_password
        },
        {
          name  = "GF_INSTALL_PLUGINS"
          value = "grafana-piechart-panel,grafana-worldmap-panel"
        }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          awslogs-group         = aws_cloudwatch_log_group.grafana.name
          awslogs-region        = var.aws_region
          awslogs-stream-prefix = "ecs"
        }
      }
    }
  ])

  tags = {
    Environment = var.environment
    Component   = "Grafana"
  }
}

# ECS Services
resource "aws_ecs_service" "prometheus" {
  name            = "${var.environment}-prometheus"
  cluster         = aws_ecs_cluster.monitoring.id
  task_definition = aws_ecs_task_definition.prometheus.arn
  desired_count   = 1

  network_configuration {
    subnets          = aws_subnet.private[*].id
    security_groups  = [aws_security_group.monitoring.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.prometheus.arn
    container_name   = "prometheus"
    container_port   = 9090
  }

  tags = {
    Environment = var.environment
    Component   = "Prometheus"
  }
}

resource "aws_ecs_service" "grafana" {
  name            = "${var.environment}-grafana"
  cluster         = aws_ecs_cluster.monitoring.id
  task_definition = aws_ecs_task_definition.grafana.arn
  desired_count   = 1

  network_configuration {
    subnets          = aws_subnet.private[*].id
    security_groups  = [aws_security_group.monitoring.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.grafana.arn
    container_name   = "grafana"
    container_port   = 3000
  }

  tags = {
    Environment = var.environment
    Component   = "Grafana"
  }
}

# CloudWatch Log Groups for Monitoring
resource "aws_cloudwatch_log_group" "prometheus" {
  name              = "/ecs/${var.environment}-prometheus"
  retention_in_days = 7

  tags = {
    Environment = var.environment
    Component   = "Prometheus"
  }
}

resource "aws_cloudwatch_log_group" "grafana" {
  name              = "/ecs/${var.environment}-grafana"
  retention_in_days = 7

  tags = {
    Environment = var.environment
    Component   = "Grafana"
  }
}
```

### 4. Distributed Tracing with AWS X-Ray

```hcl
# xray.tf
# X-Ray Daemon Task Definition
resource "aws_ecs_task_definition" "xray_daemon" {
  family                   = "${var.environment}-xray-daemon"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = 128
  memory                   = 256

  container_definitions = jsonencode([
    {
      name  = "xray-daemon"
      image = "amazon/aws-xray-daemon:latest"

      portMappings = [
        {
          containerPort = 2000
          protocol      = "udp"
        }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          awslogs-group         = aws_cloudwatch_log_group.xray.name
          awslogs-region        = var.aws_region
          awslogs-stream-prefix = "ecs"
        }
      }
    }
  ])

  tags = {
    Environment = var.environment
    Component   = "X-Ray"
  }
}

# X-Ray Daemon Service
resource "aws_ecs_service" "xray_daemon" {
  name            = "${var.environment}-xray-daemon"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.xray_daemon.arn
  desired_count   = 1

  network_configuration {
    subnets          = aws_subnet.private[*].id
    security_groups  = [aws_security_group.xray.id]
    assign_public_ip = false
  }

  tags = {
    Environment = var.environment
    Component   = "X-Ray"
  }
}

# CloudWatch Log Group for X-Ray
resource "aws_cloudwatch_log_group" "xray" {
  name              = "/ecs/${var.environment}-xray-daemon"
  retention_in_days = 7

  tags = {
    Environment = var.environment
    Component   = "X-Ray"
  }
}

# X-Ray Group
resource "aws_xray_group" "main" {
  group_name        = "${var.environment}-application"
  filter_expression = "service(\"${var.environment}-app\")"

  tags = {
    Environment = var.environment
    Component   = "X-Ray"
  }
}
```

### 5. Custom Metrics and Dashboards

```hcl
# custom-metrics.tf
# Custom CloudWatch Metrics
resource "aws_cloudwatch_metric_alarm" "custom_business_metric" {
  alarm_name          = "${var.environment}-business-metric"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "BusinessMetric"
  namespace           = "Custom/Business"
  period              = "300"
  statistic           = "Average"
  threshold           = "100"
  alarm_description   = "This metric monitors business KPIs"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  tags = {
    Environment = var.environment
    Component   = "Business"
  }
}

# Custom Dashboard for Business Metrics
resource "aws_cloudwatch_dashboard" "business" {
  dashboard_name = "${var.environment}-business-dashboard"

  dashboard_body = jsonencode({
    widgets = [
      {
        type   = "metric"
        x      = 0
        y      = 0
        width  = 12
        height = 6
        properties = {
          metrics = [
            ["Custom/Business", "BusinessMetric", "Environment", var.environment],
            [".", "UserSessions", ".", "."],
            [".", "Revenue", ".", "."]
          ]
          period = 300
          stat   = "Sum"
          region = var.aws_region
          title  = "Business Metrics"
        }
      },
      {
        type   = "log"
        x      = 12
        y      = 0
        width  = 12
        height = 6
        properties = {
          query   = "SOURCE '/aws/ec2/${var.environment}/application' | fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 20"
          region  = var.aws_region
          title   = "Recent Errors"
          view    = "table"
        }
      }
    ]
  })
}
```

## Best Practices

### 1. Monitoring Strategy

```hcl
# monitoring-strategy.tf
locals {
  monitoring_tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
    Purpose     = "monitoring"
  }
}

# Comprehensive monitoring setup
resource "aws_cloudwatch_metric_alarm" "comprehensive" {
  for_each = {
    cpu    = { metric = "CPUUtilization", threshold = 80, operator = "GreaterThanThreshold" }
    memory = { metric = "MemoryUtilization", threshold = 85, operator = "GreaterThanThreshold" }
    disk   = { metric = "DiskSpaceUtilization", threshold = 90, operator = "GreaterThanThreshold" }
    error  = { metric = "ErrorCount", threshold = 10, operator = "GreaterThanThreshold" }
  }

  alarm_name          = "${var.environment}-${each.key}-alarm"
  comparison_operator = each.value.operator
  evaluation_periods  = "2"
  metric_name         = each.value.metric
  namespace           = "AWS/EC2"
  period              = "300"
  statistic           = "Average"
  threshold           = each.value.threshold
  alarm_description   = "This metric monitors ${each.key}"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  tags = local.monitoring_tags
}
```

### 2. Logging Strategy

```hcl
# logging-strategy.tf
# Structured logging configuration
resource "aws_cloudwatch_log_group" "structured" {
  for_each = toset([
    "application",
    "system",
    "security",
    "audit",
    "performance"
  ])

  name              = "/aws/ec2/${var.environment}/${each.value}"
  retention_in_days = each.value == "security" || each.value == "audit" ? 90 : 30

  tags = {
    Environment = var.environment
    Component   = each.value
    Purpose     = "logging"
  }
}

# Log metric filters for structured logs
resource "aws_cloudwatch_log_metric_filter" "structured" {
  for_each = {
    error   = { pattern = "[timestamp, level=ERROR, service, message]", metric = "ErrorCount" }
    warning = { pattern = "[timestamp, level=WARN, service, message]", metric = "WarningCount" }
    info    = { pattern = "[timestamp, level=INFO, service, message]", metric = "InfoCount" }
  }

  name           = "${var.environment}-${each.key}-logs"
  pattern        = each.value.pattern
  log_group_name = aws_cloudwatch_log_group.structured["application"].name

  metric_transformation {
    name      = each.value.metric
    namespace = "Application"
    value     = "1"
  }
}
```

### 3. Alerting Strategy

```hcl
# alerting-strategy.tf
# Multi-channel alerting
resource "aws_sns_topic" "alerts" {
  name = "${var.environment}-alerts"
  
  tags = {
    Environment = var.environment
    Purpose     = "alerting"
  }
}

resource "aws_sns_topic_subscription" "email_critical" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "email"
  endpoint  = var.critical_email
  filter_policy = jsonencode({
    severity = ["critical", "high"]
  })
}

resource "aws_sns_topic_subscription" "slack_all" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "https"
  endpoint  = var.slack_webhook_url
}

resource "aws_sns_topic_subscription" "pagerduty" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "https"
  endpoint  = var.pagerduty_webhook_url
  filter_policy = jsonencode({
    severity = ["critical"]
  })
}
```

## Common Commands

### 1. Monitoring Commands

```bash
# Check CloudWatch metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=AutoScalingGroupName,Value=my-asg \
  --start-time 2023-01-01T00:00:00Z \
  --end-time 2023-01-01T23:59:59Z \
  --period 300 \
  --statistics Average

# List CloudWatch alarms
aws cloudwatch describe-alarms \
  --alarm-names-prefix my-environment

# Get log streams
aws logs describe-log-streams \
  --log-group-name /aws/ec2/my-environment/application \
  --order-by LastEventTime \
  --descending
```

### 2. Observability Commands

```bash
# Query CloudWatch Logs
aws logs filter-log-events \
  --log-group-name /aws/ec2/my-environment/application \
  --filter-pattern "ERROR" \
  --start-time 1640995200000

# Get X-Ray traces
aws xray get-traces \
  --trace-ids 1-5759e988-bd862e3fe1be46a994272793

# Export metrics
aws cloudwatch get-metric-data \
  --metric-data-queries file://metric-query.json \
  --start-time 2023-01-01T00:00:00Z \
  --end-time 2023-01-01T23:59:59Z
```

## Troubleshooting

### 1. Common Monitoring Issues

**Issue**: Metrics not appearing in CloudWatch
```bash
# Solution: Check IAM permissions
aws iam get-role-policy --role-name EC2Role --policy-name CloudWatchPolicy

# Verify CloudWatch agent is running
sudo systemctl status amazon-cloudwatch-agent
```

**Issue**: Logs not being sent to CloudWatch
```bash
# Solution: Check CloudWatch agent configuration
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard

# Restart the agent
sudo systemctl restart amazon-cloudwatch-agent
```

**Issue**: Alarms not triggering
```bash
# Solution: Check alarm configuration
aws cloudwatch describe-alarms --alarm-names my-alarm

# Test alarm
aws cloudwatch set-alarm-state \
  --alarm-name my-alarm \
  --state-value ALARM \
  --state-reason "Testing alarm"
```

### 2. Observability Best Practices

```hcl
# observability-best-practices.tf
# Implement comprehensive monitoring
resource "aws_cloudwatch_metric_alarm" "best_practices" {
  # Use meaningful alarm names
  alarm_name = "${var.environment}-${var.component}-${var.metric}-alarm"
  
  # Set appropriate thresholds
  threshold = var.environment == "production" ? 80 : 90
  
  # Use multiple evaluation periods
  evaluation_periods = 2
  
  # Include detailed descriptions
  alarm_description = "Monitors ${var.metric} for ${var.component} in ${var.environment}"
  
  # Add proper tags
  tags = {
    Environment = var.environment
    Component   = var.component
    Metric      = var.metric
    ManagedBy   = "terraform"
  }
}

# Implement log retention policies
resource "aws_cloudwatch_log_group" "with_retention" {
  name              = "/aws/ec2/${var.environment}/${var.component}"
  retention_in_days = var.log_retention_days
  
  tags = {
    Environment = var.environment
    Component   = var.component
    Retention   = var.log_retention_days
  }
}
```

## Summary

In this lesson, you learned about:

1. **Infrastructure Monitoring Concepts**: Metrics, logs, and distributed tracing
2. **Logging Strategies**: Centralized logging with CloudWatch Logs
3. **Alerting and Notification**: Multi-channel alerting with SNS
4. **Observability Best Practices**: Comprehensive monitoring and troubleshooting
5. **Advanced Monitoring Tools**: Prometheus, Grafana, and X-Ray integration

## Next Steps

- Implement comprehensive monitoring for your infrastructure
- Set up advanced observability tools and dashboards
- Configure automated alerting and notification systems
- Practice with different monitoring platforms and tools

## Additional Resources

- [AWS CloudWatch Documentation](https://docs.aws.amazon.com/cloudwatch/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [AWS X-Ray Documentation](https://docs.aws.amazon.com/xray/)