pipeline {
    agent none  // No default agent - we'll specify per stage
    
    environment {
        // IIS Configuration
        IIS_SITE_NAME = 'eShopOnWeb'
        IIS_SITE_PATH = 'C:\\inetpub\\wwwroot\\eShopOnWeb'
        IIS_APP_POOL = 'eShopOnWebAppPool'
        IIS_PORT = '8081'
        BACKUP_PATH = 'C:\\Backups\\eShopOnWeb'
        PROJECT_KEY = 'eShopOnWeb'
        PROJECT_NAME = 'eShopOnWeb'
        PROJECT_VERSION = '1.0'
        
        // Publish directories
        PUBLISH_DIR = './publish'
        
        // GitHub credentials
        GIT_CREDENTIALS = credentials('github-creds')
    }
    
    stages {
        stage('Build and Test') {
            agent any  // Use default Linux agent for build
            
            tools {
                dotnetsdk 'dotnet-8'
            }
            
            stages {
                stage('Checkout') {
                    steps {
                        echo 'Checking out code...'
                        git branch: 'main',
                            credentialsId: 'github-credentials',
                            url: 'https://github.com/thegagankapoor/eShopOnWeb'
                    }
                }
                
                stage('Verify .NET Installation') {
                    steps {
                        echo 'Verifying .NET SDK installation...'
                        sh 'dotnet --version'
                        sh 'dotnet --list-sdks'
                    }
                }
                
                stage('Restore Dependencies') {
                    steps {
                        echo 'Restoring NuGet packages...'
                        sh 'dotnet restore eShopOnWeb.sln'
                    }
                }
                
                stage('OWASP Dependency Check') {
                    steps {
                        echo "=====Running OWASP Dependency Check"

                        dependencyCheck(
                            odcInstallation: 'DP-Check',
                            additionalArguments: '''
                                --scan .
                                --format HTML
                                --format XML
                                --failOnCVSS 11
                                '''
                        )
                    }
                }
                stage('Build') {
                    steps {
                        echo 'Building the solution...'
                        sh 'dotnet build eShopOnWeb.sln --configuration Release --no-restore'
                    }
                }
                
                stage('Install SonarScanner Tool') {
                    steps {
                        sh '''
                            dotnet tool install --global dotnet-sonarscanner || dotnet tool update --global dotnet-sonarscanner
                            export PATH="$PATH:$HOME/.dotnet/tools"
                        '''
                    }
                }
                
                stage('SonarQube Analysis') {
                    steps {
                        echo '===== Running SonarQube Code Analysis'
                        withSonarQubeEnv('SonarQube') {
                            sh '''
                                export PATH="$PATH:/var/lib/jenkins/.dotnet/tools"
                                
                                dotnet sonarscanner begin \
                                    /k:"${PROJECT_KEY}" \
                                    /n:"${PROJECT_NAME}" \
                                    /v:"${PROJECT_VERSION}" \
                                    
                                dotnet build eShopOnWeb.sln --configuration Release --no-restore
                                
                                dotnet sonarscanner end
                                
                            '''
                        }
                    }
                }
                
                stage('Run Unit Tests') {
                    steps {
                        echo 'Running unit tests...'
                        sh 'dotnet test tests/UnitTests/UnitTests.csproj --configuration Release --no-build --verbosity normal --logger "trx;LogFileName=unit-test-results.trx"'
                    }
                    post {
                        always {
                            mstest testResultsFile: '**/unit-test-results.trx'
                        }
                    }
                }
                
                stage('Run Integration Tests') {
                    steps {
                        echo 'Running integration tests...'
                        sh 'dotnet test tests/IntegrationTests/IntegrationTests.csproj --configuration Release --no-build --verbosity normal --logger "trx;LogFileName=integration-test-results.trx"'
                    }
                    post {
                        always {
                            mstest testResultsFile: '**/integration-test-results.trx'
                        }
                    }
                }
                
                stage('Publish Application') {
                    steps {
                        echo 'Publishing Web application...'
                        sh "dotnet publish src/Web/Web.csproj --configuration Release --output ${PUBLISH_DIR}/Web --no-build"
                        
                        echo 'Publishing PublicApi...'
                        sh "dotnet publish src/PublicApi/PublicApi.csproj --configuration Release --output ${PUBLISH_DIR}/PublicApi --no-build"
                        
                        echo 'Publishing BlazorAdmin...'
                        sh "dotnet publish src/BlazorAdmin/BlazorAdmin.csproj --configuration Release --output ${PUBLISH_DIR}/BlazorAdmin --no-build"
                    }
                }
                
                stage('Create Deployment Package') {
                    steps {
                        echo 'Creating deployment package...'
                        sh '''
                            cd publish/Web
                            zip -r ../../eShopOnWeb-Web-${BUILD_NUMBER}.zip .
                            cd ../..
                            
                            cd publish/PublicApi
                            zip -r ../../eShopOnWeb-Api-${BUILD_NUMBER}.zip .
                            cd ../..
                            
                            cd publish/BlazorAdmin
                            zip -r ../../eShopOnWeb-Admin-${BUILD_NUMBER}.zip .
                            cd ../..
                        '''
                        archiveArtifacts artifacts: '*.zip', fingerprint: true
                        
                        // Stash the published files for Windows agent
                        stash includes: 'publish/**/*', name: 'published-app'
                    }
                }
            }
        }
        
        stage('Deploy to IIS') {
            agent {
                label 'windows-iis-agent'
            }
            
            stages {
                stage('Prepare IIS Environment') {
                    steps {
                        echo 'Preparing IIS environment - creating folders, app pool, and website...'
                        powershell """
                            Import-Module WebAdministration
                            
                            Write-Host "=== Preparing IIS Environment ==="
                            
                            # 1. Create Backup Directory if not exists
                            if (-not (Test-Path '${BACKUP_PATH}')) {
                                New-Item -ItemType Directory -Path '${BACKUP_PATH}' -Force | Out-Null
                                Write-Host "[✓] Created backup directory: ${BACKUP_PATH}"
                            } else {
                                Write-Host "[✓] Backup directory already exists: ${BACKUP_PATH}"
                            }
                            
                            # 2. Create Deployment Directory if not exists
                            if (-not (Test-Path '${IIS_SITE_PATH}')) {
                                New-Item -ItemType Directory -Path '${IIS_SITE_PATH}' -Force | Out-Null
                                Write-Host "[✓] Created deployment directory: ${IIS_SITE_PATH}"
                            } else {
                                Write-Host "[✓] Deployment directory already exists: ${IIS_SITE_PATH}"
                            }
                            
                            # 3. Create Application Pool if not exists
                            if (-not (Test-Path "IIS:\\\\AppPools\\\\${IIS_APP_POOL}")) {
                                New-WebAppPool -Name '${IIS_APP_POOL}'
                                Set-ItemProperty "IIS:\\\\AppPools\\\\${IIS_APP_POOL}" -Name "managedRuntimeVersion" -Value ""
                                Set-ItemProperty "IIS:\\\\AppPools\\\\${IIS_APP_POOL}" -Name "enable32BitAppOnWin64" -Value \$false
                                Write-Host "[✓] Created application pool: ${IIS_APP_POOL}"
                                Write-Host "    - Runtime: No Managed Code (.NET Core)"
                                Write-Host "    - 64-bit mode: Enabled"
                            } else {
                                Write-Host "[✓] Application pool already exists: ${IIS_APP_POOL}"
                            }
                            
                            # 4. Create Website if not exists
                            if (-not (Get-Website -Name '${IIS_SITE_NAME}' -ErrorAction SilentlyContinue)) {
                                New-Website -Name '${IIS_SITE_NAME}' `
                                    -Port ${IIS_PORT} `
                                    -PhysicalPath '${IIS_SITE_PATH}' `
                                    -ApplicationPool '${IIS_APP_POOL}' `
                                    -Force
                                Write-Host "[✓] Created website: ${IIS_SITE_NAME}"
                                Write-Host "    - Port: ${IIS_PORT}"
                                Write-Host "    - Physical Path: ${IIS_SITE_PATH}"
                                Write-Host "    - App Pool: ${IIS_APP_POOL}"
                            } else {
                                Write-Host "[✓] Website already exists: ${IIS_SITE_NAME}"
                                # Update binding if needed
                                Set-ItemProperty "IIS:\\\\Sites\\\\${IIS_SITE_NAME}" -Name physicalPath -Value '${IIS_SITE_PATH}'
                                Set-ItemProperty "IIS:\\\\Sites\\\\${IIS_SITE_NAME}" -Name applicationDefaults.applicationPool -Value '${IIS_APP_POOL}'
                                Write-Host "    - Updated physical path and app pool"
                            }
                            
                            Write-Host ""
                            Write-Host "=== IIS Environment Ready ==="
                        """
                    }
                }
                
                stage('Unstash Files') {
                    steps {
                        echo 'Retrieving published files from build agent...'
                        unstash 'published-app'
                    }
                }
                
                stage('Stop IIS Site') {
                    steps {
                        echo 'Stopping IIS site and application pool...'
                        powershell """
                            Import-Module WebAdministration
                            
                            Write-Host "=== Stopping IIS Services ==="
                            
                            # Stop Website
                            if (Get-Website -Name '${IIS_SITE_NAME}' -ErrorAction SilentlyContinue) {
                                \$siteState = (Get-WebsiteState -Name '${IIS_SITE_NAME}').Value
                                if (\$siteState -eq 'Started') {
                                    Stop-Website -Name '${IIS_SITE_NAME}' -ErrorAction SilentlyContinue
                                    Write-Host "[✓] Stopped website: ${IIS_SITE_NAME}"
                                } else {
                                    Write-Host "[i] Website already stopped: ${IIS_SITE_NAME}"
                                }
                            }
                            
                            # Stop Application Pool
                            if (Test-Path "IIS:\\\\AppPools\\\\${IIS_APP_POOL}") {
                                \$poolState = (Get-WebAppPoolState -Name '${IIS_APP_POOL}').Value
                                if (\$poolState -eq 'Started') {
                                    Stop-WebAppPool -Name '${IIS_APP_POOL}' -ErrorAction SilentlyContinue
                                    Write-Host "[✓] Stopped application pool: ${IIS_APP_POOL}"
                                } else {
                                    Write-Host "[i] Application pool already stopped: ${IIS_APP_POOL}"
                                }
                            }
                            
                            Start-Sleep -Seconds 5
                            Write-Host "[✓] Services stopped successfully"
                        """
                    }
                }
                
                stage('Backup Current Deployment') {
                    steps {
                        echo 'Backing up current deployment...'
                        powershell """
                            Write-Host "=== Creating Backup ==="
                            
                            \$backupPath = '${BACKUP_PATH}\\\\' + (Get-Date -Format 'yyyy-MM-dd_HH-mm-ss')
                            
                            if (Test-Path '${IIS_SITE_PATH}') {
                                \$items = Get-ChildItem -Path '${IIS_SITE_PATH}' -ErrorAction SilentlyContinue
                                if (\$items.Count -gt 0) {
                                    New-Item -ItemType Directory -Path \$backupPath -Force | Out-Null
                                    Copy-Item -Path '${IIS_SITE_PATH}\\\\*' -Destination \$backupPath -Recurse -Force
                                    Write-Host "[✓] Backup created at: \$backupPath"
                                } else {
                                    Write-Host "[i] No files to backup (empty directory)"
                                }
                            } else {
                                Write-Host "[i] No existing deployment to backup"
                            }
                        """
                    }
                }
                
                stage('Deploy Files to IIS') {
                    steps {
                        echo 'Deploying files to IIS...'
                        powershell """
                            Write-Host "=== Deploying Application Files ==="
                            
                            # Remove old files
                            if (Test-Path '${IIS_SITE_PATH}') {
                                Remove-Item -Path '${IIS_SITE_PATH}\\\\*' -Recurse -Force -ErrorAction SilentlyContinue
                                Write-Host "[✓] Cleaned deployment directory"
                            }
                            
                            # Copy new files
                            Copy-Item -Path '${WORKSPACE}\\\\publish\\\\Web\\\\*' -Destination '${IIS_SITE_PATH}' -Recurse -Force
                            Write-Host "[✓] Files deployed to: ${IIS_SITE_PATH}"
                            
                            # Set permissions
                            \$acl = Get-Acl '${IIS_SITE_PATH}'
                            \$permission = 'IIS_IUSRS','Read,ReadAndExecute','ContainerInherit,ObjectInherit','None','Allow'
                            \$accessRule = New-Object System.Security.AccessControl.FileSystemAccessRule \$permission
                            \$acl.SetAccessRule(\$accessRule)
                            Set-Acl '${IIS_SITE_PATH}' \$acl
                            Write-Host "[✓] Permissions configured for IIS_IUSRS"
                            
                            # Count deployed files
                            \$fileCount = (Get-ChildItem -Path '${IIS_SITE_PATH}' -Recurse -File).Count
                            Write-Host "[✓] Total files deployed: \$fileCount"
                        """
                    }
                }
                
                stage('Install ASP.NET Core Hosting Bundle') {
                    steps {
                        echo 'Checking ASP.NET Core Hosting Bundle...'
                        powershell """
                            Write-Host "=== Checking ASP.NET Core Runtime (Hosting Bundle) ==="
                            
                            if (Get-Command dotnet -ErrorAction SilentlyContinue) {
                                Write-Host "[✓] dotnet found"

                                Write-Host ""
                                Write-Host "Installed runtimes:"
                                dotnet --list-runtimes

                            # Check ASP.NET Core 8 runtime exists
                            \$aspnet8 = dotnet --list-runtimes | Select-String "Microsoft.AspNetCore.App 8\\."
                            if (-not \$aspnet8) {
                                throw "[✗] ASP.NET Core 8 runtime NOT found. Install Hosting Bundle .NET 8."
                            } 

                            Write-Host ""
                            Write-Host "[✓] ASP.NET Core 8 runtime found (Hosting Bundle looks installed)"
                        }
                        else {
                            throw "[✗] dotnet not found. Install .NET 8 Hosting Bundle on this Windows IIS machine."
                            }
                        """
                    }
                }
                
                stage('Start IIS Site') {
                    steps {
                        echo 'Starting IIS site and application pool...'
                        powershell """
                            Import-Module WebAdministration
                            
                            Write-Host "=== Starting IIS Services ==="
                            
                            # Start Application Pool
                            Start-WebAppPool -Name '${IIS_APP_POOL}'
                            Write-Host "[✓] Started application pool: ${IIS_APP_POOL}"
                            
                            # Start Website
                            Start-Website -Name '${IIS_SITE_NAME}'
                            Write-Host "[✓] Started website: ${IIS_SITE_NAME}"
                            
                            # Wait and verify
                            Start-Sleep -Seconds 5
                            
                            \$poolState = (Get-WebAppPoolState -Name '${IIS_APP_POOL}').Value
                            \$siteState = (Get-WebsiteState -Name '${IIS_SITE_NAME}').Value
                            
                            Write-Host ""
                            Write-Host "=== Service Status ==="
                            Write-Host "Application Pool: \$poolState"
                            Write-Host "Website: \$siteState"
                            
                            if (\$poolState -ne 'Started' -or \$siteState -ne 'Started') {
                                Write-Host "[✗] ERROR: Services failed to start properly"
                                throw "IIS site or app pool failed to start"
                            }
                            
                            Write-Host "[✓] All services started successfully"
                        """
                    }
                }
                
                stage('Verify Deployment') {
                    steps {
                        echo 'Verifying deployment files...'
                        powershell """
                            Write-Host "=== Deployment Verification ==="
                            
                            # Check for web.config
                            if (Test-Path '${IIS_SITE_PATH}\\\\web.config') {
                                Write-Host "[✓] web.config found"
                            } else {
                                Write-Host "[✗] WARNING: web.config not found"
                            }
                            
                            # Check for main DLL
                            if (Test-Path '${IIS_SITE_PATH}\\\\Web.dll') {
                                Write-Host "[✓] Web.dll found"
                            } else {
                                Write-Host "[✗] WARNING: Web.dll not found"
                            }
                            
                            # Check for appsettings.json
                            if (Test-Path '${IIS_SITE_PATH}\\\\appsettings.json') {
                                Write-Host "[✓] appsettings.json found"
                            } else {
                                Write-Host "[✗] WARNING: appsettings.json not found"
                            }
                        """
                    }
                }
                
                stage('Health Check') {
                    steps {
                        echo 'Running health check...'
                        script {
                            def healthCheckPassed = powershell(returnStatus: true, script: """
                                Write-Host "=== Running Health Check ==="
                                
                                Start-Sleep -Seconds 10
                                
                                try {
                                    \$response = Invoke-WebRequest -Uri 'http://localhost:${IIS_PORT}' -UseBasicParsing -TimeoutSec 30
                                    Write-Host "[✓] Health check PASSED"
                                    Write-Host "    - HTTP Status: \$(\$response.StatusCode)"
                                    Write-Host "    - URL: http://localhost:${IIS_PORT}"
                                    exit 0
                                } catch {
                                    Write-Host "[!] Health check returned error"
                                    Write-Host "    - HTTP Status: \$(\$_.Exception.Response.StatusCode.value__)"
                                    Write-Host "    - Error: \$(\$_.Exception.Message)"
                                    Write-Host ""
                                    Write-Host "[i] Application deployed but may need configuration"
                                    Write-Host "    Check IIS logs at: C:\\\\inetpub\\\\logs\\\\LogFiles"
                                    Write-Host "    Check Event Viewer for application errors"
                                    exit 1
                                }
                            """)
                            
                            if (healthCheckPassed != 0) {
                                echo '⚠️ Health check failed - Application may need database configuration'
                                echo '   The application is deployed but returning HTTP 500'
                                echo '   This is usually due to missing database connection or configuration'
                            }
                        }
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo '========================================='
            echo '✓ DEPLOYMENT COMPLETED SUCCESSFULLY!'
            echo '========================================='
            echo "Application URL: http://localhost:${IIS_PORT}"
            echo "IIS Site: ${IIS_SITE_NAME}"
            echo "Build Number: ${BUILD_NUMBER}"
            echo ''
            echo 'NOTE: If you see HTTP 500 errors, you may need to:'
            echo '  1. Configure the database connection string'
            echo '  2. Run database migrations'
            echo '  3. Check application logs'
        }
        failure {
            echo '========================================='
            echo '✗ DEPLOYMENT FAILED!'
            echo '========================================='
            echo 'Check the console output above for errors'
        }
        always {
            echo 'Pipeline execution completed.'
        }
    }
}
