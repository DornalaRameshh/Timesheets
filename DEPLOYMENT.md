# Deployment Configuration

## Overview
This repository is configured for automated deployment of AWS Lambda functions using GitHub Actions and AWS SAM.

## Files Changed

### 1. `template.yaml`
- **Purpose**: SAM template defining all Lambda functions and API Gateway
- **Key Features**:
  - Creates a unified API Gateway for all endpoints
  - Defines 13 Lambda functions (one per module)
  - Includes CORS configuration
  - Outputs API Gateway URL and ID for reference

### 2. `.github/workflows/deploy.yml`
- **Purpose**: GitHub Actions workflow for automated deployment
- **Trigger**: Pushes to `main` branch affecting module folders
- **Process**:
  1. Detects which modules changed
  2. If any changes detected, deploys entire stack
  3. Uses AWS IAM role for authentication
  4. Builds and deploys using SAM
  5. Outputs the API Gateway URL

### 3. `requirements.txt`
- **Purpose**: Python dependencies for Lambda functions
- **Dependencies**: boto3, botocore

## Deployment Strategy

### Single Stack Approach
- All Lambda functions are deployed as one CloudFormation stack
- Stack name: `timesheets-api-stack`
- When any module changes, the entire stack is updated
- This ensures consistency and avoids API Gateway conflicts

### API Gateway Configuration
- SAM automatically creates an API Gateway
- Each function gets its own path (e.g., `/approvals`, `/contacts`)
- CORS is enabled for all endpoints
- API supports ANY method for all paths

## Module Structure
Each module (folder) contains:
- `lambda_function.py` with `lambda_handler` function
- Supporting files (handlers, models, services, utils)
- All modules follow the same structure pattern

## Environment Variables
The workflow uses:
- `arn:aws:iam::026090520154:role/aws_github` for AWS authentication
- `us-east-1` as the deployment region

## API Endpoints
After deployment, endpoints will be available at:
- `https://{api-id}.execute-api.us-east-1.amazonaws.com/Prod/approvals`
- `https://{api-id}.execute-api.us-east-1.amazonaws.com/Prod/client_table`
- `https://{api-id}.execute-api.us-east-1.amazonaws.com/Prod/contacts`
- ... (one for each module)

## Monitoring
- CloudWatch logs are automatically created for each Lambda function
- Function names follow pattern: `{module}-handler`
- Stack outputs provide API Gateway URL and ID for reference

## Security
- Functions use `AWSLambdaBasicExecutionRole` policy
- GitHub Actions uses OIDC with IAM role (no long-term credentials)
- CORS is configured for web application access

## Next Steps
1. Push changes to trigger first deployment
2. Note the API Gateway URL from the workflow output
3. Update any client applications with the new API endpoints
4. Monitor CloudWatch logs for any issues

## Troubleshooting
- Check GitHub Actions logs for deployment issues
- Use AWS CloudFormation console to monitor stack status
- CloudWatch logs contain Lambda execution details
- SAM CLI can be used locally for testing: `sam local start-api`

## Local Development Setup

This guide covers setting up local development and testing using AWS SAM Local with Docker.

### Prerequisites
1. **Install AWS SAM CLI**:
   - Download from: https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html
   - Verify: `sam --version`

2. **Install Docker**:
   - Download Docker Desktop for Windows
   - Ensure Docker is running
   - Verify: `docker --version`

3. **Install AWS CLI**:
   - Download from: https://aws.amazon.com/cli/
   - Verify: `aws --version`

### Step 1: Set Up AWS Credentials for Local Development

Create a dedicated AWS profile for dev environment access:

#### Option A: Access Keys
```bash
aws configure --profile timesheets-dev
```
Enter:
- AWS Access Key ID
- AWS Secret Access Key  
- Default region: us-east-1
- Default output format: json

#### Option B: AWS SSO (IAM Identity Center)
```bash
aws configure sso --profile timesheets-dev
aws sso login --profile timesheets-dev
```

### Step 2: Create Environment Configuration

Create `env.local.json` in your project root with dev table names:

```json
{
  "ApprovalsFunction": {
    "APPROVALS_TABLE": "dev.Approvals.ddb-table",
    "LOG_LEVEL": "DEBUG"
  },
  "ContactsFunction": {
    "CONTACTS_TABLE": "dev.Contacts.ddb-table",
    "LOG_LEVEL": "DEBUG"
  },
  "TasksFunction": {
    "TASKS_TABLE": "dev.Tasks.ddb-table",
    "LOG_LEVEL": "DEBUG"
  },
  "TimeentriesFunction": {
    "TIMEENTRIES_TABLE": "dev.TimeEntries.ddb-table",
    "LOG_LEVEL": "DEBUG"
  },
  "ProjectsTableFunction": {
    "PROJECTS_TABLE": "dev.Projects.ddb-table",
    "LOG_LEVEL": "DEBUG"
  },
  "ProjectAssignmentFunction": {
    "ASSIGNMENTS_TABLE": "dev.ProjectAssignments.ddb-table",
    "LOG_LEVEL": "DEBUG"
  },
  "UserLoginFunction": {
    "USERS_TABLE": "dev.Users.ddb-table",
    "LOG_LEVEL": "DEBUG"
  },
  "UserRoutesFunction": {
    "USERS_TABLE": "dev.Users.ddb-table",
    "LOG_LEVEL": "DEBUG"
  },
  "ClientTableFunction": {
    "CLIENTS_TABLE": "dev.Clients.ddb-table",
    "LOG_LEVEL": "DEBUG"
  },
  "LookupFunction": {
    "LOOKUPS_TABLE": "dev.Lookups.ddb-table",
    "LOG_LEVEL": "DEBUG"
  },
  "IamFunction": {
    "USER_GRANTS_TABLE": "dev.UserGrants.ddb-table",
    "ROLES_TABLE": "dev.roles_t.ddb-table",
    "POLICY_DEFS_TABLE": "dev.PolicyDefinitions.ddb-table",
    "LOG_LEVEL": "DEBUG"
  },
  "DashboardFunction": {
    "LOG_LEVEL": "DEBUG"
  },
  "UpdatePasswordFunction": {
    "USERS_TABLE": "dev.Users.ddb-table",
    "LOG_LEVEL": "DEBUG"
  }
}
```

**Note**: Match the environment variable keys to what your code reads via `os.getenv()`. Update table names to match your actual dev DynamoDB tables.

### Step 3: Prepare the Project

Navigate to your project directory:
```bash
cd timesheets-project
```

Validate the SAM template:
```bash
sam validate --region us-east-1
```

Build the project:
```bash
sam build
```

### Step 4: Start Local API Gateway

Start the local API Gateway with your dev credentials:
```bash
sam local start-api --profile timesheets-dev --region us-east-1 --env-vars env.local.json
```

This will:
- Start a local API Gateway on `http://127.0.0.1:3000`
- Mount all Lambda functions at their respective paths
- Use real AWS credentials to access dev DynamoDB tables
- Enable debug logging

### Step 5: Test Local Endpoints

Test your API endpoints using curl, Postman, or browser:

```bash
# Test GET endpoint
curl http://127.0.0.1:3000/approvals

# Test POST endpoint with JSON body
curl -X POST http://127.0.0.1:3000/user_login \
  -H "Content-Type: application/json" \
  -d '{"username": "test", "password": "test"}'
```

Available endpoints:
- `/approvals` - Approvals management
- `/client_table` - Client data
- `/contacts` - Contact management
- `/dashboard` - Dashboard data
- `/iam` - IAM operations
- `/lookup` - Lookup tables
- `/projects_table` - Projects data
- `/project_assignment` - Project assignments
- `/tasks` - Task management
- `/timeentries` - Time entries
- `/update_password` - Password updates
- `/user_login` - User authentication
- `/user_routes` - User management

### Step 6: Monitor Logs

- Local API logs appear in the terminal running `sam local start-api`
- Check for any errors or debug messages
- Functions will log to CloudWatch in AWS when using real credentials

### Step 7: Stop Local Server

Press `Ctrl+C` in the terminal to stop the local API server.

### Alternative: Test Individual Functions

To test a specific function without the full API Gateway:
```bash
# Create a test event file (event.json)
{
  "httpMethod": "GET",
  "path": "/approvals",
  "headers": {
    "origin": "http://localhost:3000"
  },
  "requestContext": {
    "authorizer": {
      "claims": {
        "sub": "test-user-id"
      }
    }
  }
}

# Invoke function locally
sam local invoke ApprovalsFunction --event event.json --env-vars env.local.json --profile timesheets-dev --region us-east-1
```

### Troubleshooting Local Setup

1. **Docker not running**: Ensure Docker Desktop is started
2. **Credentials issues**: Verify `aws sts get-caller-identity --profile timesheets-dev`
3. **Table not found**: Check table names in `env.local.json` match your dev tables
4. **Port conflicts**: Change port with `--port 8080`
5. **Build issues**: Clear build cache with `sam build --cache-dir .aws-sam/cache`
6. **Permission errors**: Ensure your AWS profile has DynamoDB read/write permissions

### Local vs Production Differences

- **Local**: Uses real AWS services via credentials
- **Production**: Uses IAM roles attached to Lambda functions
- **Local**: Debug logging enabled via env vars
- **Production**: Logging controlled by Lambda configuration
- **Local**: No cold starts (functions stay warm)
- **Production**: Subject to Lambda cold start delays

### Best Practices

1. Always test locally before pushing to main
2. Use descriptive commit messages to trigger deployments
3. Monitor CloudWatch logs after deployment
4. Keep `env.local.json` out of version control (add to `.gitignore`)
5. Use the same table structure in dev and prod environments