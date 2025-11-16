From folder
docker build -t esedocker .
docker ps
docker run -d --name esedocker -p 7070:70 esedocker
docker ps
docker stop esedocker
docker rm esedocker
docker run -d --name esedocker -p 7070:80 esedocker



make file extension through
New-Item -Path . -Name "Dockerfile" -ItemType "file"


from github
docker build -t esedocker https://github.com/Subit418/esedocker.git
docker run -d --name esedocker -p 7070:70 esedocker


to test through python
# Stop anything on port 80 if present (careful)
# Serve files from C:\inetpub\wwwroot
cd C:\inetpub\wwwroot
python -m http.server 80



to install iis to run localhost 
dism /online /enable-feature /featurename:IIS-WebServer /all
dism /online /enable-feature /featurename:IIS-WebServerManagementTools /all
Get-Service W3SVC





gitbash push commands
 cd desktop
 cd esejenkins
 git init
 git add .
 git remote add origin https://github.com/Subit418/esejenkins.git
 git commit -m "ese"
 git push -u origin master







jenkinfile without scm
pipeline {
    agent any

    environment {
        // Deploy under a subfolder
        DEPLOY_DIR = 'C:\\inetpub\\wwwroot\\withoutscm'
        SITE_URL = 'http://localhost/withoutscm/index.html'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Cloning project from GitHub...'
                git branch: 'master', url: 'https://github.com/iamanoj12/withoutscm.git'
            }
        }

        stage('Build') {
            steps {
                echo 'Static portfolio site – no build step needed.'
                bat 'dir /B /S'
            }
        }

        stage('Deploy (robocopy)') {
            steps {
                echo "Deploying workspace to: ${env.DEPLOY_DIR}"
                // Use robocopy to mirror workspace to destination, excluding .git
                bat """
                REM Create target folder if missing
                if not exist "${DEPLOY_DIR}" mkdir "${DEPLOY_DIR}"

                REM Mirror workspace -> target, exclude .git and Jenkins specific files
                robocopy "%WORKSPACE%" "${DEPLOY_DIR}" /MIR /XD .git .svn .jenkins /NFL /NDL /NJH /NJS /nc /ns /np

                REM (Optional) If robocopy fails with exit codes >1, treat as failure - robocopy returns codes, we normalize them
                if %ERRORLEVEL% GEQ 8 exit /b %ERRORLEVEL%
                """
            }
        }

        stage('IIS: ensure running & permissions') {
            steps {
                echo 'Ensure IIS running, enable static content, grant folder read permissions'
                bat """
                REM Start W3SVC if not running
                sc query W3SVC | find "RUNNING" >nul || (echo "Starting W3SVC" & net start W3SVC)

                REM Enable Static Content feature (no-op if already enabled)
                dism /online /enable-feature /featurename:IIS-StaticContent /all > "%TEMP%\\dism_static.txt" 2>&1

                REM Ensure Default Web Site is started (appcmd requires admin)
                "%windir%\\system32\\inetsrv\\appcmd.exe" start site "Default Web Site" 2>nul || echo "appcmd start site returned non-zero"

                REM Grant read permission recursively to IIS_IUSRS (AppPool identities use it by default)
                icacls "${DEPLOY_DIR}" /grant "IIS_IUSRS:(OI)(CI)R" /T
                """
            }
        }

        stage('Verify via local HTTP request') {
            steps {
                echo 'Attempt a local HTTP request to verify files are served'
                // Use PowerShell Invoke-WebRequest for clearer status output
                powershell '''
                try {
                    $resp = Invoke-WebRequest -Uri "${env:SITE_URL}" -UseBasicParsing -TimeoutSec 10 -ErrorAction Stop
                    Write-Host "HTTP Status:" $resp.StatusCode
                    Write-Host "Content-length:" ($resp.RawContentLength)
                    # Optionally show a snippet of HTML
                    $body = $resp.Content
                    if ($body.Length -gt 200) { $body = $body.Substring(0,200) + "...(truncated)" }
                    Write-Host "Body snippet:`n$body"
                } catch {
                    Write-Host "Local HTTP request failed: $($_.Exception.Message)"
                    exit 1
                }
                '''
            }
        }

        stage('Final note') {
            steps {
                echo 'If verification succeeded, open in browser: http://<server-ip-or-hostname>/withoutscm/index.html'
                echo 'If verification failed, check firewall, bindings, and IIS logs.'
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline finished successfully!'
            echo "👉 Open in browser (from server): ${env.SITE_URL}"
            echo "👉 Open in browser (from other machine): http://<server-ip-or-hostname>/withoutscm/index.html"
        }
        failure {
            echo '❌ Pipeline failed! See pipeline console output for diagnostics.'
        }
    }
}
