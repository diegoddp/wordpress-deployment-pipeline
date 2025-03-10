# WordPress Sites Deployment with Azure DevOps YAML Pipelines

This repository contains the YAML pipeline configuration for deploying WordPress sites using Azure DevOps. The deployment process has been migrated from classic pipelines to YAML for better version control and flexibility.

## Pipeline Overview
The pipeline is divided into three main stages:

1. Build Package: This stage builds the WordPress artifact and performs necessary checks.
2. Deploy to UAT: This stage deploys the built artifact to the UAT environment.
3. Deploy to Production: This stage deploys the built artifact to the production environment.

## Pipeline Configuration
### Trigger
The pipeline is triggered on changes to the master branch.

````

trigger:
- master

````
**Pool**
The pipeline uses the latest Ubuntu image.

````
pool:
  vmImage: ubuntu-latest
````

**Variables**

Environment variables are grouped and managed securely.

````
variables:
- group: 'Environment Variables'


````
**Stages**

**Build Package**
This stage builds the WordPress artifact and performs checks for unauthorized email addresses and disallowed plugins.

````
stages:
- stage: Build
  displayName: 'Build Package'
  jobs:
  - job: BuildJob
    displayName: 'Build WordPress Artifact'
    steps:
    - task: UsePythonVersion@0
      inputs:
        versionSpec: '3.x'
      displayName: 'Use Python 3.x'
    
    - script: echo "Building WordPress project..."
      displayName: 'Build Step'
      
    - powershell: |
        # Define the path to the directory containing your .sql files
        $DbFile = Join-Path -Path $(System.DefaultWorkingDirectory) -ChildPath "db"
        
        # Regular expression pattern to match email addresses
        $emailPattern = "[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}"
        
        # Known image extensions
        $imageExtensions = @(".webp", ".png", ".jpg", ".jpeg")  # Add more as needed
        
        # Get a list of .sql files in the specified directory
        $sqlFiles = Get-ChildItem -Path $DbFile -Filter "*.sql" -File
        
        # Initialize an array to store unauthorized email addresses
        $unauthorizedEmails = @()
        
        foreach ($file in $sqlFiles) {
            try {
                # Read the content of the file
                $fileContent = Get-Content -Path $file.FullName -Raw
        
                # Search for email addresses
                $matches = [regex]::Matches($fileContent, $emailPattern)
        
                if ($matches.Count -gt 0) {
                    foreach ($match in $matches) {
                        $email = $match.Value.ToLower()  # Convert to lowercase
                        $endsWithImageExtension = $imageExtensions | Where-Object { $email.EndsWith($_) }
        
                        if (-not $endsWithImageExtension -and $email -notin @("user@example.com", "user2@example.com")) {
                            $unauthorizedEmails += $email
                        }
                    }
                }
            } catch {
                Write-Host "Error reading $($file.Name): $_"
                # Let's also fail the pipeline if there's an error
                throw "Pipeline failed due to an error reading $($file.Name): $_"
            }
        }
        
        # Print unauthorized email addresses
        if ($unauthorizedEmails.Count -gt 0) {
            Write-Host "Unauthorized email addresses found:"
            foreach ($unauthEmail in $unauthorizedEmails) {
                Write-Host "  - $unauthEmail"
            }
            # Fail the pipeline explicitly
            throw "Pipeline failed due to unauthorized email addresses."
        } else {
            Write-Host "No unauthorized email addresses found."
        
      displayName: 'Detect PII Data'
  
    - powershell: |
        # Debugging statements
        Write-Host "Checking if 'updraft' folder exists..."
        $updraftFolder = Join-Path -Path $(System.DefaultWorkingDirectory) -ChildPath "wordpress\wp-content\updraft"
        if (Test-Path $updraftFolder -PathType Container) {
            Write-Host "'updraft' folder found. Removing..."
            Remove-Item -Path $updraftFolder -Recurse -Force
            Write-Host "'updraft' folder removed successfully."
        } else {
            Write-Host "'updraft' folder not found. No action required."
        }
        
        Write-Host "Checking if 'updraft*' plugins exist..."
        $updraftPlugins = Get-ChildItem -Path "$(System.DefaultWorkingDirectory)\wordpress\wp-content\plugins" -Filter "updraft*" -Directory
        if ($updraftPlugins.Count -gt 0) {
            Write-Host "'updraft' plugins found. Removing..."
            $updraftPlugins | ForEach-Object {
                Remove-Item -Path $_.FullName -Recurse -Force
            }
            Write-Host "'updraft' plugins removed successfully."
        } else {
            Write-Host "'updraft' plugins not found. No action required."
        }
        
        Write-Host "Checking if 'blackhole-bad-bots' plugin exists..."
        $blackHoleBots = Join-Path -Path $(System.DefaultWorkingDirectory) -ChildPath "wordpress\wp-content\plugins\blackhole-bad-bots"
        if (Test-Path $blackHoleBots -PathType Container) {
            Write-Host "'blackhole-bad-bots' plugin found. Removing..."
            Remove-Item -Path $blackHoleBots -Recurse -Force
            Write-Host "'blackhole-bad-bots' plugin removed successfully."
        } else {
            Write-Host "'blackhole-bad-bots' plugin not found. No action required."
      displayName: 'Remove Disallowed Plugins'

      
    - powershell: |
        # Define the path to the WordPress folder
        $wordpressFolderPath = Join-Path -Path $(System.DefaultWorkingDirectory) -ChildPath "wordpress"
        
        # Check if .sql files exist and remove them
        $sqlFiles = Get-ChildItem -Path $wordpressFolderPath -Filter "*.sql" -File
        if ($sqlFiles.Count -gt 0) {
        Write-Host "SQL files found in the WordPress folder. Removing..."
            $sqlFiles | ForEach-Object {
                Remove-Item -Path $_.FullName -Force
            }
            Write-Host "SQL files removed successfully."
        } else {
            Write-Host "No SQL files found in the WordPress folder. No action required."
        
      displayName: 'Delete any .sql in the root'

    - task: ArchiveFiles@2
      displayName: 'Archive Files'
      inputs:
        rootFolderOrFile: wordpress
        archiveFile: '$(Build.ArtifactStagingDirectory)/wordpress.zip'
        includeRootFolder: false

    - task: CopyFiles@2
      displayName: 'Copy DB'
      inputs:
        SourceFolder: '$(build.sourcesdirectory)/db'
        Contents: '**/*.sql'
        TargetFolder: '$(build.ArtifactStagingDirectory)/db/'

    - task: PublishBuildArtifacts@1
      displayName: 'Publish Artifact: drop'
      inputs:
        PathtoPublish: '$(Build.ArtifactStagingDirectory)'
        ArtifactName: 'drop'
````

**Deploy to UAT**

This stage deploys the built artifact to the UAT environment and resets the database.
````
- stage: DeployToUAT
  displayName: 'Deploy to UAT'
  dependsOn: Build
  jobs:
  - deployment: UATDeploy
    environment: 'uat' # Azure DevOps environment name
    displayName: 'Deploy to UAT'
    strategy:
      runOnce:
        deploy:
          steps:

          - task: AzureWebApp@1
            displayName: 'Deploy to UAT'
            inputs:
              azureSubscription: '$(azureSubsc)' # Azure subscription for staging
              appType: 'webAppLinux'
              appName: '$(appservice_name)'
              package: '$(Pipeline.Workspace)/drop/wordpress.zip'
              runtimeStack: 'PHP|8.2'
            enabled: true
          
          - script: |
              mysql --host=$(sqlhost).mysql.database.azure.com --user=$(administratorLogin) --password=$(administratorLoginPassword) --ssl-mode=REQUIRED --silent -e "DROP DATABASE IF EXISTS $(databases_mysql_name_uat); CREATE DATABASE $(databases_mysql_name_uat);"
            displayName: 'Reset Database'
            enabled: true

          - script: |
              mysql --host=$(sqlhost).mysql.database.azure.com --user=$(administratorLogin) --password=$(administratorLoginPassword) --ssl-mode=REQUIRED $(databases_mysql_name_uat) < "$(Pipeline.Workspace)/drop/db/blankdb.sql"
            displayName: 'Deploy Database'
````
**Deploy to Production**

This stage deploys the built artifact to the production environment and resets the database.

````
- stage: DeployToProduction
  displayName: 'Deploy to Production'
  dependsOn: DeployToUAT
  jobs:
  - deployment: ProductionDeploy
    environment: prd
    displayName: 'Production Deployment'
    strategy:
      runOnce:
        deploy:
          steps:

          - task: AzureRmWebAppDeployment@4
            displayName: 'Azure App Service Deploy: $(appservice_name)'
            inputs:
              azureSubscription: '$(azureSubsc)'
              appType: webAppLinux
              WebAppName: '$(appservice_name)'
              deployToSlotOrASE: true
              ResourceGroupName: '$(ResourceGroupName)'
              SlotName: prd
              package: '$(Pipeline.Workspace)/drop/wordpress.zip'

          - script: |
              mysql --host=$(sqlhost).mysql.database.azure.com --user=$(administratorLogin) --password=$(administratorLoginPassword) --ssl-mode=REQUIRED --silent -e "DROP DATABASE IF EXISTS $(databases_mysql_name_prd); CREATE DATABASE $(databases_mysql_name_prd);"
            displayName: 'Reset Database'

          - script: |
              mysql --host=$(sqlhost).mysql.database.azure.com --user=$(administratorLogin) --password=$(administratorLoginPassword) --ssl-mode=REQUIRED $(databases_mysql_name_prd) < "$(Pipeline.Workspace)/drop/db/blankdb.sql"
            displayName: 'Deploy Database'
````

**Security Considerations**

Ensure that sensitive information such as client IDs, client secrets, passwords, and database names are managed securely through environment variables or other secure methods.
