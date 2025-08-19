# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

DevFlow Engine is an AI-powered development automation system that enables automatic code editing and Pull Request creation through natural language prompts. The system integrates with GitHub repositories and provides a web-based interface for developers to interact with AI-generated code changes.

## Language and Communication

- **All responses, comments, and communications should be in Japanese (日本語)**
- This is explicitly stated in the GEMINI.md file as a basic rule for the project

## Architecture

This is a documentation-only repository containing system design specifications for a three-tier AWS-based architecture:

### Frontend (React + TypeScript)
- **Technology Stack**: React, TypeScript, Material-UI, Vite
- **Components**: 
  - SessionPanel: Repository URL and initial prompt input
  - HistoryPanel: Displays execution history and PR links
  - PromptPanel: Additional prompt input for ongoing sessions
  - ActionBar: Web Editor access and session management
- **State Management**: React useState and useContext with SessionContext

### Backend (Python + AWS)
- **Technology Stack**: Python, FastAPI, Boto3
- **Infrastructure**: AWS Lambda, API Gateway, ECS on Fargate, DynamoDB, Secrets Manager
- **API Endpoints**:
  - `POST /sessions` - Start new session and create PR
  - `POST /sessions/{id}/prompts` - Execute additional prompts
  - `GET /sessions/{id}` - Get session status and history
  - `GET /sessions/{id}/editor` - Get Web Editor URL
  - `DELETE /sessions/{id}` - Stop session and destroy container

### Container Environment
- **Base Image**: `codercom/code-server`
- **Additional Tools**: aws-cli, gh (GitHub CLI), claudecode
- **Workflow**: Repository cloning → AI-based code editing → Commit/Push → PR creation → Web Editor serving

## Key Features

1. **Automated PR Creation**: Natural language prompts generate code changes and create GitHub PRs
2. **Session Management**: Track multiple user sessions with DynamoDB
3. **Web-based IDE**: code-server integration for manual code editing
4. **GitHub Integration**: Secure GitHub App authentication via AWS Secrets Manager
5. **Containerized Execution**: On-demand ECS Fargate containers for isolated development environments

## Infrastructure & Deployment

- **Cloud Provider**: AWS with serverless architecture
- **Provisioning**: AWS CDK (TypeScript)
- **CI/CD**: GitHub Actions for frontend (S3/CloudFront), backend (Lambda), and container (ECR) deployments
- **Security**: IAM roles with least privilege, GitHub App authentication, VPC with public/private subnets

## Development Notes

- This repository contains only documentation and design specifications
- No actual source code implementation exists yet
- The project is designed for AWS deployment with Japanese language support
- Focus on understanding the system architecture through the comprehensive documentation in the `docs/` directory

## Documentation Structure

- `docs/backend_design.md` - Backend API and Lambda function specifications
- `docs/frontend_design.md` - React component design and API integration
- `docs/infrastructure_design.md` - AWS resource architecture and deployment
- `docs/design/requirements.md` - System requirements (minimal content)
- `docs/design/usecase.md` - Detailed use cases and user flows  
- `docs/design/機能一覧.md` - Feature list and component mapping