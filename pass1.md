provider "aws" {
  region = var.aws_region
}

variable "aws_region" {
  description = "AWS region"
  default     = "us-east-1"
}

variable "aws_az" {
  description = "AWS Region Az"
  default     = "us-east-1a"
}

variable "vpc_cidr_block" {
  description = "CIDR block for VPC"
  default     = "10.0.0.0/16"
}

variable "private_subnet_cidr_block" {
  description = "CIDR block for private subnet"
  default     = "10.0.1.0/24"
}

variable "public_subnet_cidr_block" {
  description = "CIDR block for public subnet"
  default     = "10.0.2.0/24"
}

variable "ami_id" {
  description = "AMI ID for instances"
  default     = "ami-0005e0cfe09cc9050"
}

variable "key_name" {
  description = "Name of the AWS key pair"
  default     = "new-trail"
}

variable "instance_type" {
  description = "Name of the AWS instance type"
  default     = "t2.micro"
}

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr_block

  tags = {
    Name = "Main VPC"
  }
}

resource "aws_subnet" "private" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.private_subnet_cidr_block
  availability_zone       = var.aws_az
  map_public_ip_on_launch = false

  tags = {
    Name = "Private Subnet"
  }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidr_block
  availability_zone       = var.aws_az
  map_public_ip_on_launch = true

  tags = {
    Name = "Public Subnet"
  }
}

resource "aws_instance" "private_instance" {
  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = aws_subnet.private.id
  key_name      = var.key_name
  

  tags = {
    Name = "Private Instance"
  }
}

resource "aws_security_group" "private_sg" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "Private Security Group"
  }
}

resource "aws_instance" "nginx_instance" {
  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = aws_subnet.public.id
  key_name      = var.key_name
  security_groups = [aws_security_group.my_security_group.id]
  depends_on = [aws_instance.private_instance]

  user_data = <<-EOF
              #!/bin/bash
              # Update and install Nginx
              sudo yum update -y
              sudo yum install -y nginx
              sudo systemctl enable nginx
              sudo systemctl start nginx
              # Get the private IP of another EC2 instance
              #private_ip=$(aws ec2 describe-instances --instance-ids <instance_id> --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)


              # Create Nginx directories if they don't exist, Create sites-available and sites-enabled directories
              sudo mkdir -p /etc/nginx/sites-available/
              sudo mkdir -p /etc/nginx/sites-enabled/
                    
              # Get the private IP of another EC2 instance from Terraform output
              private_ip=${aws_instance.private_instance.private_ip}

              # Create Nginx reverse proxy configuration file
              cat <<EOL | sudo tee /etc/nginx/sites-available/reverse-proxy > /dev/null
              server {
                  listen 80;
                  server_name _;

                  location / {
                      proxy_pass http://$private_ip;
                      proxy_set_header Host \$host;
                      proxy_set_header X-Real-IP \$remote_addr;
                      proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
                      proxy_set_header X-Forwarded-Proto \$scheme;
                  }

                  location /ping {
                      proxy_pass http://www.google.com;
                  }
              }
              EOL

              echo "Nginx reverse proxy configuration created successfully"

             
              # Create symlink only if it doesn't exist
              symlink_path="/etc/nginx/sites-enabled/reverse-proxy"

              echo "reverse proxy symlink path: $symlink_path"
              
              if [ ! -e "$symlink_path" ]; then
                  sudo ln -s /etc/nginx/sites-available/reverse-proxy "$symlink_path"
              fi

              # Restart Nginx
              sudo systemctl restart nginx
              echo "Nginx reverse proxy configuration created successfully"
        
              EOF
  tags = {
    Name = "Nginx Instance"
  }
}


# Create Security Group
resource "aws_security_group" "my_security_group" {
  name        = "my_security_group"
  description = "Allow inbound SSH and HTTP traffic"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group_rule" "private_sg_ingress" {
  security_group_id        = aws_security_group.private_sg.id
  type                     = "ingress"
  from_port                = 0
  to_port                  = 0
  protocol                 = "-1"
  source_security_group_id = aws_security_group.private_sg.id
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "Internet Gateway"
  }
}


resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
  tags = {
    Name = "Public Route Table"
  }
}

# Associate Route Table with Subnet
resource "aws_route_table_association" "my_route_table_association" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public_rt.id
}

output "vpc_id" {
  value = aws_vpc.main.id
}

output "private_subnet_id" {
  value = aws_subnet.private.id
}

output "public_subnet_id" {
  value = aws_subnet.public.id
}

output "private_instance_private_ip" {
  value = aws_instance.private_instance.private_ip
}

output "nginx_instance_public_ip" {
  value = aws_instance.nginx_instance.public_ip
}