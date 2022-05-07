### EC2 IAM Role
Policy:

```
{ 

  "Version": "2012-10-17", 

  "Statement": [ 

    { 

      "Sid": "CustomCloudwatchPolicy", 

      "Effect": "Allow", 

      "Action": [ 

        "cloudwatch:PutMetricData", 

        "cloudwatch:GetMetricStatistics", 

        "cloudwatch:ListMetrics" 

      ], 

      "Resource": "*" 

    } 

  ] 

} 
```

### EC2 User Data
OS: Ubuntu
```
#!/bin/bash 
 
# Install Cloudwatch Agent 
sudo wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb -O /tmp/amazon-cloudwatch-agent.deb 
sudo dpkg -i /tmp/amazon-cloudwatch-agent.deb 
 
mkdir -p /home/ubuntu/app/logs 
sudo chmod +x /home/ubuntu/app/logs 
sudo tee -a /opt/aws/amazon-cloudwatch-agent/bin/config.json << EOF 
{ 
   "agent": { 
      "run_as_user": "root" 
   }, 
   "metrics":{ 
      "namespace":"CwAgent", 
      "append_dimensions": { 
          "ImageId": "\${aws:ImageId}", 
          "InstanceId": "\${aws:InstanceId}", 
          "InstanceType": "\${aws:InstanceType}" 
      }, 
      "aggregation_dimensions" : [["InstanceId"]], 
      "metrics_collected":{ 
         "mem":{ 
            "measurement":[ 
               "mem_used_percent" 
            ] 
         }, 
         "disk":{ 
            "measurement":[ 
               "used_percent" 
            ], 
            "resources":[ 
               "/" 
            ] 
         } 
      } 
   }, 
   "logs":{ 
      "logs_collected":{ 
         "files":{ 
            "collect_list":[ 
               { 
                 "file_path": "/home/ubuntu/app/logs/laravel*", 
                 "log_group_name": "/api", 
                 "log_stream_name": "{instance_id}" 
               } 
            ] 
         } 
      } 
   } 
} 
EOF 
 
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json -s 

sudo amazon-cloudwatch-agent-ctl -a status 
```