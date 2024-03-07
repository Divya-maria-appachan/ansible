---
- hosts: localhost
  vars:
    aws_region: "us-east-1"  
  tasks:
    - name: Ensure S3 bucket exists
      amazon.aws.s3_bucket:
        name: youruniquebucketname111889
        versioning: true
        state: present  
        region: us-east-1
    - name: Ensure S3 bucket exists
      amazon.aws.s3_bucket:
        name: youruniquebucketname111889
        versioning: true
        state: present
        public_access:
          block_public_policy: false
        region: us-east-1
    - name: Create bucket with JSON policy
      amazon.aws.s3_bucket:
        name: youruniquebucketname111889
        policy: |
          {
            "Version": "2012-10-17",
            "Statement": [
              {
                "Effect": "Allow",
                "Principal": "*",
                "Action": [
                   "s3:*"
                ],  
                "Resource":  [
                   "arn:aws:s3:::youruniquebucketname111889",
                   "arn:aws:s3:::youruniquebucketname111889/*"
                 ]       
              }
            ]
          }
        
       
        region: us-east-1 
    - name: Upload file to S3 bucket
      community.aws.s3_sync:
        bucket: youruniquebucketname111889
        file_root: /home/ec2-user/
        key_prefix: index.html
        region: us-east-1
   
    - name: Create Custom VPC
      amazon.aws.ec2_vpc_net:
        name: "custom_vpc"
        cidr_block: "10.0.0.0/16"
        dns_support: yes
        dns_hostnames: yes
        state: present  
        region: "{{ aws_region }}"   
      register: custom_vpc
    - name: Create Internet Gateway
      ec2_vpc_igw:
        vpc_id: "{{ custom_vpc.vpc.id }}"
        state: present  
        region: "{{ aws_region }}"  
        tags:
          Name: custom_igw
      register: custom_igw
    - name: Create Public Subnet2
      ec2_vpc_subnet:
        state: present
        vpc_id: "{{ custom_vpc.vpc.id }}"
        cidr: "10.0.1.0/24"
        map_public: yes
        region: "{{ aws_region }}"  
        az: us-east-1a   
        tags:
          Name: public_subnet1
      register: public_subnet1
    - name: Create Public Subnet2
      ec2_vpc_subnet:
        state: present
        vpc_id: "{{ custom_vpc.vpc.id }}"
        cidr: "10.0.3.0/24"
        map_public: yes
        region: "{{ aws_region }}"
        az: us-east-1c  
        tags:
          Name: public_subnet2
      register: public_subnet2   
    - name: Create Private Subnet
      ec2_vpc_subnet:
        state: present
        vpc_id: "{{ custom_vpc.vpc.id }}"
        region: "{{ aws_region }}"  
        cidr: "10.0.2.0/24"
        tags:
          Name: private_subnet
      register: private_subnet

    - name: Create Public Security Group
      ec2_group:
        name: public_security_group
        description: "Allow HTTP and SSH traffic for public subnet"
        vpc_id: "{{ custom_vpc.vpc.id }}"
        region: "{{ aws_region }}"   
        rules:
          - proto: tcp
            ports: 
            -  80  
            cidr_ip: "0.0.0.0/0"
          - proto: tcp
            ports: 
            - 22
            cidr_ip: "0.0.0.0/0"
      register: public_sg

    - name: Create Private Security Group
      ec2_group:
        name: private_security_group
        description: "Allow outgoing traffic for private subnet"
        vpc_id: "{{ custom_vpc.vpc.id }}"
        region: "{{ aws_region }}"  
      register: private_sg

    - name: Create Public Route Table
      ec2_vpc_route_table:
        vpc_id: "{{ custom_vpc.vpc.id }}"
        region: "{{ aws_region }}"  
        routes:
          - dest: "0.0.0.0/0"
            gateway_id: "{{ custom_igw.gateway_id }}"
        tags:
          Name: public_route_table
        subnets:
          - "{{ public_subnet1.subnet.id }}"
          - "{{ public_subnet2.subnet.id }}"
      register: public_route_table
    - name: Create Load Balancer Target Group
      community.aws.elb_target_group:
        name: "web-elb-target-group"
        protocol: "HTTP"
        port: 80
        vpc_id: "{{ custom_vpc.vpc.id }}"
        health_check_protocol: "HTTP"
        health_check_port: "80"
        health_check_path: "/index.html"
        health_check_interval: 30
        health_check_timeout: 3
        healthy_threshold_count: 2
        unhealthy_threshold_count: 2
        deregistration_delay_timeout: 300
        state: present 
        region: "{{ aws_region }}"     
      register: target_group_result    
    - name: Create Elastic Load Balancer
      community.aws.elb_application_lb:
        name: web-elb
        security_groups:
          - "{{ public_sg.group_id }}"
        subnets:
          - "{{ public_subnet1.subnet.id }}"
          - "{{ public_subnet2.subnet.id }}" 
        listeners:
          - Protocol: HTTP
            Port: 80 
            DefaultActions:
              - Type: forward
                TargetGroupName: web-elb-target-group  
        region: "{{ aws_region }}"  
      register: elb_result      
    - name: Create Launch Configuration
      community.aws.ec2_lc:
        name: web-LC
        image_id: "ami-0e731c8a588258d0d"
        instance_type: t2.micro
        key_name: terraform
        security_groups: ["{{ public_sg.group_id }}"]
        region: "{{ aws_region }}"
        user_data: |
          #cloud-config
          runcmd:
            - sudo yum update -y
            - sudo yum install -y httpd
            - sudo systemctl start httpd
            - sudo systemctl enable httpd
            - sudo mkdir -p /var/www/html
            - sudo curl -o /var/www/html/index.html https://youruniquebucketname111889.s3.us-east-1.amazonaws.com/index.html
            - sudo chmod 644 /var/www/html/index.html
            - sudo systemctl restart httpd      
    - name: Create Autoscaling Group
      community.aws.ec2_asg:
        name: Web-asg
        availability_zones: [ 'us-east-1a', 'us-east-1c' ]  
        launch_config_name: "web-LC"
        min_size: 1
        max_size: 2
        desired_capacity: 1  
        vpc_zone_identifier: ["{{ public_subnet1.subnet.id }}", "{{ public_subnet2.subnet.id }}"]
        health_check_type: ELB
        target_group_arns: "{{ target_group_result.target_group_arn }}"

        region: "{{ aws_region }}"
        state: present
      register: asg_result
    - name: Create Autoscaling Up Policy
      community.aws.autoscaling_policy:
        name: "web_policy_up"
        adjustment_type: "ChangeInCapacity"
        scaling_adjustment: 1
        cooldown: 300
        region: "{{ aws_region }}"
        group_name: "{{ aws_launch_configuration.web.name }}-asg"
    - name: Create Autoscaling Down Policy
      community.aws.autoscaling_policy:
        name: "web_policy_down"
        adjustment_type: "ChangeInCapacity"
        scaling_adjustment: -1
        cooldown: 300
        region: "{{ aws_region }}"
        group_name: "{{ aws_launch_configuration.web.name }}-asg"
    - name: Create CloudWatch Alarm for CPU Utilization - Up
      ec2_metric_alarm:
        name: "web_cpu_alarm_up"
        comparison_operator: "GreaterThanOrEqualToThreshold"
        evaluation_periods: 2
        metric_name: "CPUUtilization"
        namespace: "AWS/EC2"
        period: 120
        statistic: "Average"
        threshold: 70
        dimensions:
          - name: "AutoScalingGroupName"
            value: "{{ aws_launch_configuration.web.name }}-asg"
        alarm_description: "This metric monitors EC2 instance CPU utilization"
        alarm_actions:
          - "{{ aws_autoscaling_policy.web_policy_up.arn }}"
    - name: Create CloudWatch Alarm for CPU Utilization - Down
      ec2_metric_alarm:
        name: "web_cpu_alarm_down"
        comparison_operator: "LessThanOrEqualToThreshold"
        evaluation_periods: 2
        metric_name: "CPUUtilization"
        namespace: "AWS/EC2"
        period: 120
        statistic: "Average"
        threshold: 30
        dimensions:
          - name: "AutoScalingGroupName"
            value: "{{ aws_launch_configuration.web.name }}-asg"
        alarm_description: "This metric monitors EC2 instance CPU utilization"
        alarm_actions:
          - "{{ aws_autoscaling_policy.web_policy_down.arn }}"
    - name: Launch EC2 Instance
      community.aws.ec2_instance:
        key_name: "terraform"
        instance_type: "t2.nano"
        wait: yes
        image_id: "ami-0e731c8a588258d0d"  
        region: "{{ aws_region }}"  
        vpc_subnet_id: "{{ public_subnet1.subnet.id }}"
        security_group: "{{ public_sg.group_id }}"  
        user_data: |
          #cloud-config
          runcmd:
            - sudo yum update -y
            - sudo yum install -y httpd
            - sudo systemctl start httpd
            - sudo systemctl enable httpd
            - sudo mkdir -p /var/www/html
            - sudo wget https://github.com/learning-zone/website-templates/archive/master.zip
            - sudo unzip master.zip
            - sudo cp -r website-templates-master/victory-educational-institution-free-html5-bootstrap-template/* /var/www/html/
            - sudo chown -R apache:apache /var/www/html
            - sudo systemctl restart httpd
             
        tags:
          Name: first

    - name: Output EC2 Public IPv4 Address
      debug:
        var: public_subnet.public_ip
