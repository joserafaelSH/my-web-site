---
title: "Clickops to IaC - Infrasctructure as code"
pubDate: 2024-07
description: "Creating AWS EC2 using Terraform + Docker"
author: "José Rafael S. Hermoso"
---

# Clickops to IaC - Infrastructure as code

Welcome back, I suppose that you have been passed by the first tutorial and see the IAC at the end, so here is the IAC.

Creating environments and infrastructure using the AWS console is a boring activity and very easy to make a wrong click and create a resource with an incorrect configuration. To avoid this I introduce here the TERRAFORM an infrastructure as a code tool.

For this tutorial, We need to know something about container and docker to create an image of his application, push it to the docker hub, and grab this image with the docker inside of our EC2 to run our app.

1. First, we need to generate a docker image for our applications.

   I use this Dockerfile.

   ```docker
   FROM node:lts

   WORKDIR /usr/src/app

   COPY package.json /usr/src/app
   RUN npm install --quiet

   COPY . /usr/src/app

   EXPOSE 3000

   CMD [ "npm" ,"run", "start" ]
   ```

   Use this command to generate a docker image.

   ```bash
   docker build <docker file path> -t <image name>:<tag>
   ```

2. Create a folder for the Terraform configurations and the main file. In the terraform directory, we need to create the “main.tf” and set the provider.

   ```bash
   provider "aws" {
     region = "us-east-1"
   }
   ```

   After setting the AWS as a cloud provider we can start the configurations running start commando of Terraform.

   ```bash
   terraform init
   ```

3. Now, we need to set the AWS CLI and one AWS account in our local environment. To do this we can follow the tutorial on official AWS CLI documentation. You must to set the keys of our AWS user and provide authorizations to create some resources, I use de Administrator access but the minimal privilege policy cannot be ingored in production scenario.

4. In the “main.tf” lets start to create our remote infrastructure.

   ```bash
   provider "aws" {
     region = "us-east-1"
   }

   resource "aws_security_group" "securitygroup" {
     name        = "ec2-securitygroup"
     description = "Permitir acesso HTTP e acesso a Internet"

     ingress {
       from_port   = 80
       to_port     = 80
       protocol    = "tcp"
       cidr_blocks = ["0.0.0.0/0"]
     }

     ingress {
       from_port   = 22
       to_port     = 22
       protocol    = "tcp"
       cidr_blocks = ["0.0.0.0/0"]
     }

     egress {
       from_port   = 0
       to_port     = 65535
       protocol    = "tcp"
       cidr_blocks = ["0.0.0.0/0"]

     }
   }

   resource "aws_key_pair" "keypair" {
     key_name   = "terraform-keypair"
     public_key = file("~/.ssh/id_rsa.pub")

   }

   resource "aws_instance" "servidor" {
     ami                    = "ami-00beae93a2d981137"
     instance_type          = "t2.micro"
     user_data              = file("user_data.sh")
     vpc_security_group_ids = ["${aws_security_group.securitygroup.id}"]
     key_name               = aws_key_pair.keypair.key_name

   }
   ```

   In this file, we have an AWS EC2 t2.micro with “AWS Linux image” that uses one security group with rules to accept traffic on ports 22(SSH), 80(HTTP), and ephemerous ports, and onde SSH key to provide the SSH connection. One important thing is the “user_data.sh” file, this file runs with the boot process of the EC2 instance and we use this to install Docker and get our application image from Docker Hub.

   ```bash
   #!/bin/bash

   sudo su
   yum update -y
   yum install -y docker
   service docker start
   usermod -a -G docker ec2-user
   docker run -p <expose instance port>:<docker image port> image-name
   ```

After this configuration, we can run “terraform apply” to create our infrastructure in the AWS.

Like a transaction, we can revert using “terraform destroy” to clean your created infrastructure.

Now, we can create infrastructure with a more efficient and reusable method. Of course, the developer is not must be an infrastructure master, but in currently days the developers who know things outside the lines of their code are most valued.
