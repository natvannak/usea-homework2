# USEA Homework 2

## Project
CI/CD with Jenkins, GitHub, Docker and Docker Swarm on AWS.

## Architecture

GitHub
→ Jenkins
→ Docker
→ Amazon ECR
→ Docker Swarm
→ EC2

## Technologies

- GitHub
- Jenkins
- Docker
- Docker Swarm
- Amazon ECR
- Amazon EC2

## Local Run

docker build -t usea-homework2:local .

docker run -d \
  -p 8080:80 \
  usea-homework2:local

## ECR

Repository:
usea-homework2

Region:
us-east-1

## Swarm

Manager:
EC2 Manager

Worker:
EC2 Worker

## Deployment

docker stack deploy \
  -c docker-stack.yml \
  usea-app

## Verification

docker node ls

docker stack services usea-app

docker service ps usea-app_web

## CI/CD Flow

GitHub Push
→ Webhook
→ Jenkins
→ Docker Build
→ ECR Push
→ Swarm Deploy

## Live Application

http://YOUR_PUBLIC_IP