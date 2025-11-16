pipeline {
  agent any

  environment {
    DEPLOY_DIR = 'C:\\inetpub\\wwwroot'
    GIT_CREDS  = 'github-pat'
  }

  stages {
    stage('Checkout') {
      steps {
        echo "Cloning project from GitHub with credentialsId=${env.GIT_CREDS}"
        git branch: 'master',
            url: 'https://github.com/iamanoj12/isepractice.git',
            credentialsId: "${env.GIT_CREDS}"
      }
    }

    stage('Build (none)') {
      steps {
        echo "Static site — no build step."
      }
    }

    stage('Deploy to IIS (localhost)') {
      steps {
        // Run PowerShell directly (not through 'bat') to avoid cmd/caret issues.
        powershell script: '''
Write-Output "=== Deploy to IIS: Starting ==="

$deployDir = 'C:\inetpub\wwwroot'
$src = $env:WORKSPACE

Write-Output "Source workspace: $src"
Write-Output "Target (IIS): $deployDir"

# Ensure target exists
if (-not (Test-Path $deployDir)) {
    Write-Output "Creating target folder $deployDir"
    New-Item -Path $deployDir -ItemType Directory -Force | Out-Null
} else {
    Write-Output "Target folder exists"
}

# Clean target (optional)
Write-Output "Cleaning target (remove old files)"
Get-ChildItem -Path $deployDir -Force | Where-Object { $_.Name -notin @('.', '..') } | Remove-Item -Recurse -Force -ErrorAction SilentlyContinue

# Copy files from workspace to target (try robocopy then fallback to copy-item)
Write-Output "Copying site files..."
$robocopyExe = (Get-Command robocopy.exe -ErrorAction SilentlyContinue)
if ($robocopyExe) {
    Write-Output "Using robocopy..."
    $rc = Start-Process -FilePath $robocopyExe.Path -ArgumentList @("$src","$deployDir","/MIR","/NDL","/NFL","/NJH","/NJS","/NP") -Wait -PassThru
    if ($rc.ExitCode -gt 7) {
        Write-Output "robocopy returned exit code $($rc.ExitCode) (failure) — attempting Copy-Item fallback"
        Remove-Item -Recurse -Force -Path "$deployDir\*" -ErrorAction SilentlyContinue
        Copy-Item -Path "$src\*" -Destination $deployDir -Recurse -Force
    } else {
        Write-Output "robocopy completed (exit $($rc.ExitCode))"
    }
} else {
    Write-Output "robocopy not found — using Copy-Item"
    Copy-Item -Path "$src\*" -Destination $deployDir -Recurse -Force
}

# Try to configure IIS if available
$iisAvailable = $false
try {
    Import-Module WebAdministration -ErrorAction Stop
    $iisAvailable = $true
    Write-Output "WebAdministration module loaded — IIS available."
} catch {
    Write-Output "WebAdministration module not available: $($_.Exception.Message)"
}

if (-not $iisAvailable) {
    # Try to install on systems where Install-WindowsFeature exists (best-effort)
    try {
        Write-Output "Attempting to install IIS via Install-WindowsFeature (best-effort)..."
        Install-WindowsFeature -Name Web-Server -IncludeManagementTools -ErrorAction Stop
        Import-Module WebAdministration -ErrorAction Stop
        $iisAvailable = $true
        Write-Output "IIS installed via Install-WindowsFeature."
    } catch {
        Write-Output "Install-WindowsFeature not available or failed: $($_.Exception.Message)"
    }
}

if ($iisAvailable) {
    try {
        Write-Output "Configuring Default Web Site to point to $deployDir ..."
        $site = Get-Website -Name 'Default Web Site' -ErrorAction SilentlyContinue
        if ($null -eq $site) {
            Write-Output "Default Web Site not found — creating Default Web Site bound to localhost:80"
            New-Website -Name 'Default Web Site' -Port 80 -PhysicalPath $deployDir -Force
        } else {
            Write-Output "Default Web Site exists — updating physical path"
            # Update physical path for the root virtual directory/app
            Set-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST' `
              -filter "system.applicationHost/sites/site[@name='Default Web Site']/application[@path='/']/virtualDirectory[@path='/']" `
              -name physicalPath -value $deployDir -ErrorAction SilentlyContinue
        }

        Write-Output "Starting W3SVC and Default Web Site..."
        Start-Service W3SVC -ErrorAction SilentlyContinue
        Start-Website -Name 'Default Web Site' -ErrorAction SilentlyContinue

        # Ensure index.html is in Default Documents
        $dd = Get-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST' -filter 'system.webServer/defaultDocument/files' -name '.'
        $hasIndex = $false
        foreach ($item in $dd.Collection) { if ($item.value -eq 'index.html') { $hasIndex = $true } }
        if (-not $hasIndex) {
            Write-Output "Adding index.html to default documents"
            Add-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST' -filter 'system.webServer/defaultDocument/files' -name '.' -value @{value='index.html'}
        }
    } catch {
        Write-Output "IIS config actions failed: $($_.Exception.Message)"
    }
} else {
    Write-Output "IIS not available on this machine; skipped site creation. To host on IIS, install IIS and re-run."
}

# Set permissions so IIS can read the files (best-effort)
Write-Output "Setting folder permissions for IIS_IUSRS (read/execute) ..."
try {
    icacls $deployDir /grant 'IIS_IUSRS:(OI)(CI)RX' /T | Out-Null
} catch {
    Write-Output "icacls grant failed: $($_.Exception.Message)"
}

Write-Output "=== Deploy to IIS: Finished ==="
''', returnStatus: true
      }
    }

    stage('Verify site (http://localhost)') {
      steps {
        powershell script: '''
try {
    $r = Invoke-WebRequest -UseBasicParsing -Uri 'http://localhost/index.html' -TimeoutSec 10
    if ($r.StatusCode -eq 200) {
        Write-Output "HTTP_OK"
        exit 0
    } else {
        Write-Output "HTTP_NON200:" + $r.StatusCode
        exit 2
    }
} catch {
    Write-Output "HTTP_ERROR:" + $_.Exception.Message
    exit 3
}
''', returnStatus: true
      }
    }
  }

  post {
    success {
      echo "Deployment finished. Open http://localhost/index.html on this machine."
    }
    failure {
      echo "Deployment failed — check console output above. If IIS is not installed on this host, install it and re-run, or ask me to modify the script for your environment."
    }
  }
}
