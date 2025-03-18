# Salesforce DevOps Pipeline

A comprehensive DevOps implementation for Salesforce development teams, showcasing best practices for source control, CI/CD, and deployment strategies.

## Overview

This project demonstrates a complete Salesforce DevOps pipeline that enables teams to:

- Implement Git-based source control for Salesforce metadata
- Automate testing and validation
- Deploy changes across environments (Dev, QA, UAT, Production)
- Maintain code quality and standards
- Track and audit changes

## Key Components

- **Source Control Strategy**: Git-flow implementation for Salesforce development
- **CI/CD Pipeline**: Automated build, test, and deployment using GitHub Actions
- **Environment Management**: Configuration for multiple Salesforce environments
- **Quality Gates**: Automated code quality checks and test coverage requirements
- **Deployment Automation**: Scripts for reliable, repeatable deployments

## Getting Started

See the [Setup Guide](docs/setup-guide.md) for detailed instructions on implementing this pipeline for your team.

## Documentation

- [Source Control Strategy](docs/source-control-strategy.md)
- [CI/CD Pipeline Configuration](docs/cicd-pipeline.md)
- [Environment Management](docs/environment-management.md)
- [Deployment Strategies](docs/deployment-strategies.md)
- [Best Practices](docs/best-practices.md)

### 2. Create source control strategy documentation

```markdown:docs/source-control-strategy.md
# Source Control Strategy for Salesforce Development

This document outlines our Git-based source control strategy for Salesforce development, designed to support multiple developers working on the same org while maintaining code quality and deployment reliability.

## Git-Flow Model

We implement a modified Git-Flow model tailored for Salesforce development:

### Branch Structure

- **main**: Represents the code in Production
- **develop**: Integration branch for features
- **feature/[name]**: Individual feature development
- **release/[version]**: Release preparation
- **hotfix/[name]**: Production issue fixes

### Workflow

1. **Feature Development**:
   - Create a feature branch from `develop`
   - Develop and test in isolated scratch org
   - Create pull request to merge back to `develop`

2. **Release Process**:
   - Create release branch from `develop`
   - Test in UAT environment
   - Merge to `main` when approved
   - Tag with version number

3. **Hotfix Process**:
   - Create hotfix branch from `main`
   - Fix and test in isolated environment
   - Merge to both `main` and `develop`

## Salesforce-Specific Considerations

### Metadata Management

- Use SFDX source format for better diff tracking
- Include package.xml files for deployment
- Exclude user-specific metadata

### Conflict Resolution

- Establish clear ownership of components
- Use pre-commit hooks to validate metadata
- Implement automated conflict detection

## Best Practices

- Commit early and often
- Write meaningful commit messages
- Keep feature branches short-lived
- Review code before merging
- Validate all changes in a sandbox before deployment
```

### 3. Create CI/CD pipeline configuration

```markdown:docs/cicd-pipeline.md
# CI/CD Pipeline for Salesforce

This document details our continuous integration and continuous deployment pipeline for Salesforce development.

## Pipeline Overview

Our CI/CD pipeline automates the build, test, and deployment process for Salesforce changes, ensuring consistent quality and reliable deployments.

```

Let's create a GitHub Actions workflow file:

```yaml:config/github-actions-workflow.yml
name: Salesforce CI/CD Pipeline

on:
  push:
    branches: [ develop, main, 'release/*', 'hotfix/*' ]
  pull_request:
    branches: [ develop, main ]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Install Salesforce CLI
        run: |
          wget https://developer.salesforce.com/media/salesforce-cli/sfdx/channels/stable/sfdx-linux-x64.tar.xz
          mkdir ~/sfdx
          tar xJf sfdx-linux-x64.tar.xz -C ~/sfdx --strip-components 1
          echo "$HOME/sfdx/bin" >> $GITHUB_PATH
      
      - name: Authenticate to Dev Hub
        run: |
          echo "${{ secrets.SFDX_AUTH_URL }}" > ./authurl
          sfdx auth:sfdxurl:store -f ./authurl -a DevHub
      
      - name: Create Scratch Org
        run: sfdx force:org:create -f config/project-scratch-def.json -a scratch-org -s -d 1
      
      - name: Push Source
        run: sfdx force:source:push -u scratch-org
      
      - name: Run Apex Tests
        run: sfdx force:apex:test:run -u scratch-org -c -r human -w 10
      
      - name: Run Static Analysis
        run: sfdx scanner:run -t ./src -f json -o ./scan-results.json
      
      - name: Delete Scratch Org
        if: always()
        run: sfdx force:org:delete -u scratch-org -p

  deploy-to-qa:
    needs: validate
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Install Salesforce CLI
        run: |
          wget https://developer.salesforce.com/media/salesforce-cli/sfdx/channels/stable/sfdx-linux-x64.tar.xz
          mkdir ~/sfdx
          tar xJf sfdx-linux-x64.tar.xz -C ~/sfdx --strip-components 1
          echo "$HOME/sfdx/bin" >> $GITHUB_PATH
      
      - name: Authenticate to QA Org
        run: |
          echo "${{ secrets.QA_AUTH_URL }}" > ./authurl
          sfdx auth:sfdxurl:store -f ./authurl -a QAOrg
      
      - name: Deploy to QA
        run: sfdx force:source:deploy -p src -u QAOrg -l RunLocalTests

  deploy-to-uat:
    needs: validate
    if: startsWith(github.ref, 'refs/heads/release/')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Install Salesforce CLI
        run: |
          wget https://developer.salesforce.com/media/salesforce-cli/sfdx/channels/stable/sfdx-linux-x64.tar.xz
          mkdir ~/sfdx
          tar xJf sfdx-linux-x64.tar.xz -C ~/sfdx --strip-components 1
          echo "$HOME/sfdx/bin" >> $GITHUB_PATH
      
      - name: Authenticate to UAT Org
        run: |
          echo "${{ secrets.UAT_AUTH_URL }}" > ./authurl
          sfdx auth:sfdxurl:store -f ./authurl -a UATOrg
      
      - name: Deploy to UAT
        run: sfdx force:source:deploy -p src -u UATOrg -l RunLocalTests

  deploy-to-prod:
    needs: validate
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v2
      - name: Install Salesforce CLI
        run: |
          wget https://developer.salesforce.com/media/salesforce-cli/sfdx/channels/stable/sfdx-linux-x64.tar.xz
          mkdir ~/sfdx
          tar xJf sfdx-linux-x64.tar.xz -C ~/sfdx --strip-components 1
          echo "$HOME/sfdx/bin" >> $GITHUB_PATH
      
      - name: Authenticate to Production Org
        run: |
          echo "${{ secrets.PROD_AUTH_URL }}" > ./authurl
          sfdx auth:sfdxurl:store -f ./authurl -a ProdOrg
      
      - name: Deploy to Production
        run: sfdx force:source:deploy -p src -u ProdOrg -l RunLocalTests
```

### 4. Create deployment strategies documentation

```markdown:docs/deployment-strategies.md
# Salesforce Deployment Strategies

This document outlines our deployment strategies for Salesforce changes across different environments.

## Environment Strategy

We maintain the following environments:

1. **Developer Scratch Orgs**: Isolated development environments
2. **Integration/Dev Sandbox**: First integration point for features
3. **QA Sandbox**: Testing environment for quality assurance
4. **UAT Sandbox**: User acceptance testing environment
5. **Production**: Live environment

## Deployment Approaches

### Continuous Deployment to Development

- Automated deployments to Dev environment upon successful PR merges to `develop` branch
- Runs all tests in local namespace
- Validates code quality and test coverage

### Scheduled Deployments to QA

- Automated deployments to QA environment on a scheduled basis
- Runs all tests to ensure org-wide compatibility
- Generates deployment validation report

### Release-Based Deployments to UAT

- Deployments to UAT triggered by release branch creation
- Complete validation of all components
- User acceptance testing performed before production approval

### Production Deployments

- Change set or direct deployment from version control
- Scheduled during maintenance windows
- Full regression testing before and after deployment
- Rollback plan for each deployment

## Rollback Strategy

1. **Quick Rollback**: For critical issues, deploy previous version
2. **Component Rollback**: For isolated issues, deploy specific component fixes
3. **Data Rollback**: For data-related issues, restore from backup

## Deployment Validation

Before any deployment:

1. Run static code analysis
2. Validate deployment with `--checkonly` flag
3. Generate and review deployment plan
4. Ensure test coverage meets minimum requirements (>85%)
5. Verify all critical tests pass

## Post-Deployment Verification

After deployment:

1. Run smoke tests to verify critical functionality
2. Monitor error logs for new exceptions
3. Verify integration points are functioning
4. Document deployment results
```

### 5. Create environment management documentation

```markdown:docs/environment-management.md
# Salesforce Environment Management

This document outlines our approach to managing multiple Salesforce environments throughout the development lifecycle.

## Environment Types

### Developer Environments

- **Scratch Orgs**: Ephemeral environments for feature development
- **Developer Sandboxes**: Personal developer environments for longer-term work

### Integration Environments

- **Development Integration Org**: First integration point for features
- **Build Verification Org**: Automated testing environment for CI/CD

### Testing Environments

- **QA Sandbox**: Full or partial copy sandbox for testing
- **UAT Sandbox**: Full copy sandbox for user acceptance testing

### Production

- **Production Org**: Live environment

## Environment Configuration

We use configuration files to maintain consistency across environments:

- **scratch-org-def.json**: Defines scratch org configuration
- **environment-config.json**: Environment-specific settings
- **data-plan.json**: Test data configuration

## Environment Refresh Strategy

- **Developer Sandboxes**: Refreshed on-demand
- **QA Sandbox**: Refreshed quarterly
- **UAT Sandbox**: Refreshed before major releases

## Data Management

- Use SFDX data plan for consistent test data
- Mask sensitive data in non-production environments
- Maintain reference data across all environments

## Access Control

- Implement least-privilege access model
- Use permission sets for functional access
- Document required permissions for each role

## Environment Monitoring

- Monitor governor limits across environments
- Track deployment success rates
- Monitor test coverage over time
```

### 6. Create a sample SFDX project configuration

```json:config/project-scratch-def.json
{
  "orgName": "DevOps Pipeline Demo",
  "edition": "Developer",
  "features": ["EnableSetPasswordInApi", "API"],
  "settings": {
    "lightningExperienceSettings": {
      "enableS1DesktopEnabled": true
    },
    "securitySettings": {
      "passwordPolicies": {
        "enableSetPasswordInApi": true
      }
    },
    "mobileSettings": {
      "enableS1EncryptedStoragePref2": false
    }
  }
}
```

### 7. Create a sample sfdx-project.json file

```json:sfdx-project.json
{
  "packageDirectories": [
    {
      "path": "src",
      "default": true
    }
  ],
  "namespace": "",
  "sfdcLoginUrl": "https://login.salesforce.com",
  "sourceApiVersion": "55.0",
  "plugins": {
    "dependencies": [
      {
        "name": "sfdx-scanner",
        "version": "latest"
      }
    ]
  }
}
```

### 8. Create deployment scripts

```bash:scripts/validate-deployment.sh
#!/bin/bash

# Script to validate deployment to target org
# Usage: ./validate-deployment.sh <target-org> <source-path>

TARGET_ORG=$1
SOURCE_PATH=$2

if [ -z "$TARGET_ORG" ] || [ -z "$SOURCE_PATH" ]; then
  echo "Usage: ./validate-deployment.sh <target-org> <source-path>"
  exit 1
fi

echo "Validating deployment to $TARGET_ORG..."

# Run deployment check
sfdx force:source:deploy --checkonly --testlevel RunLocalTests --sourcepath $SOURCE_PATH --targetusername $TARGET_ORG --json > deployment_check.json

# Check result
RESULT=$(cat deployment_check.json | jq -r '.status')

if [ "$RESULT" == "0" ]; then
  echo "Validation successful!"
  exit 0
else
  echo "Validation failed. See deployment_check.json for details."
  exit 1
fi
```

```bash:scripts/deploy-to-org.sh
#!/bin/bash

# Script to deploy to target org
# Usage: ./deploy-to-org.sh <target-org> <source-path>

TARGET_ORG=$1
SOURCE_PATH=$2

if [ -z "$TARGET_ORG" ] || [ -z "$SOURCE_PATH" ]; then
  echo "Usage: ./deploy-to-org.sh <target-org> <source-path>"
  exit 1
fi

echo "Deploying to $TARGET_ORG..."

# Run deployment
sfdx force:source:deploy --testlevel RunLocalTests --sourcepath $SOURCE_PATH --targetusername $TARGET_ORG --json > deployment.json

# Check result
RESULT=$(cat deployment.json | jq -r '.status')

if [ "$RESULT" == "0" ]; then
  echo "Deployment successful!"
  exit 0
else
  echo "Deployment failed. See deployment.json for details."
  exit 1
fi
```

### 9. Create best practices documentation

```markdown:docs/best-practices.md
# Salesforce DevOps Best Practices

This document outlines our recommended best practices for Salesforce development and deployment.

## Source Control

- **Commit Frequently**: Make small, focused commits
- **Meaningful Messages**: Write descriptive commit messages
- **Branch Strategy**: Follow the Git-flow model
- **Pull Requests**: Require code reviews for all changes
- **No Direct Commits**: Never commit directly to main branches

## Development

- **Scratch Orgs**: Use scratch orgs for feature development
- **Coding Standards**: Follow Apex and Lightning Web Component best practices
- **Test Coverage**: Maintain >85% test coverage for all Apex classes
- **Documentation**: Document classes, methods, and complex logic
- **Modular Design**: Build reusable components and avoid duplication

## Testing

- **Test-Driven Development**: Write tests before implementation
- **Comprehensive Testing**: Test positive, negative, and bulk scenarios
- **Integration Tests**: Test integrations with external