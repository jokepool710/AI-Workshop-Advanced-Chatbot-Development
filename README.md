# AI-Workshop-Advanced-Chatbot-Development
Create the Terraform File

Python 
provider "aws" {
  region = "us-east-1"
}

resource "aws_vpc" "chatbot_vpc" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.chatbot_vpc.id
}

resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.chatbot_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
}

resource "aws_subnet" "chatbot_subnet_1" {
  vpc_id                  = aws_vpc.chatbot_vpc.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true
}

resource "aws_subnet" "chatbot_subnet_2" {
  vpc_id                  = aws_vpc.chatbot_vpc.id
  cidr_block              = "10.0.2.0/24"
  availability_zone       = "us-east-1b"
  map_public_ip_on_launch = true
}

resource "aws_subnet" "chatbot_subnet_3" {
  vpc_id                  = aws_vpc.chatbot_vpc.id
  cidr_block              = "10.0.3.0/24"
  availability_zone       = "us-east-1c"
  map_public_ip_on_launch = true
}

resource "aws_route_table_association" "a" {
  subnet_id      = aws_subnet.chatbot_subnet_1.id
  route_table_id = aws_route_table.public_rt.id
}

resource "aws_route_table_association" "b" {
  subnet_id      = aws_subnet.chatbot_subnet_2.id
  route_table_id = aws_route_table.public_rt.id
}

resource "aws_route_table_association" "c" {
  subnet_id      = aws_subnet.chatbot_subnet_3.id
  route_table_id = aws_route_table.public_rt.id
}

resource "aws_security_group" "chatbot_sg" {
  name        = "chatbot-sg"
  description = "Allow HTTP traffic"
  vpc_id      = aws_vpc.chatbot_vpc.id

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

resource "aws_ecs_cluster" "chatbot_cluster" {
  name = "chatbot-cluster"
}

resource "aws_iam_role" "ecsTaskExecutionRole" {
  name = "ecsTaskExecutionRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Action = "sts:AssumeRole",
      Effect = "Allow",
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      }
    }]
  })

  managed_policy_arns = [
    "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy",
    "arn:aws:iam::aws:policy/CloudWatchLogsFullAccess"
  ]
}

resource "aws_ecs_task_definition" "chatbot_task" {
  family                   = "chatbot-task"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "8192"
  memory                   = "20480"
  execution_role_arn       = aws_iam_role.ecsTaskExecutionRole.arn

  container_definitions = jsonencode([{
    name      = "chatbot"
    image     = "your_dockerhub_username/chatbot:latest"
    essential = true
    portMappings = [{
      containerPort = 80
      hostPort      = 80
    }]
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        awslogs-group         = "/ecs/chatbot"
        awslogs-region        = "us-east-1"
        awslogs-stream-prefix = "ecs"
      }
    }
  }])
}

resource "aws_ecs_service" "chatbot_service" {
  name            = "chatbot-service"
  cluster         = aws_ecs_cluster.chatbot_cluster.id
  task_definition = aws_ecs_task_definition.chatbot_task.arn
  desired_count   = 1

  launch_type = "FARGATE"

  network_configuration {
    subnets          = [aws_subnet.chatbot_subnet_1.id, aws_subnet.chatbot_subnet_2.id, aws_subnet.chatbot_subnet_3.id]
    security_groups  = [aws_security_group.chatbot_sg.id]
    assign_public_ip = true
  }
}

resource "aws_cloudwatch_log_group" "ecs_log_group" {
  name              = "/ecs/chatbot"
  retention_in_days = 7
}

output "service_url" {
  value = aws_ecs_service.chatbot_service.network_configuration
