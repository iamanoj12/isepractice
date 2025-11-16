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
        script {
          // Copy files to C:\inetpub\wwwroot and configure IIS so site is reachable at http://localhost
          bat '''
powershell -NoProfile -ExecutionPolicy Bypass -Command ^
  "Write-Output '=== Deploy to IIS: Starting ==='; ^
   $deployDir = 'C:\\inetpub\\wwwroot'; $src = '${WORKSPACE}'; ^
   Write-Output ('Source workspace: ' + $src); Write-Output ('Target (IIS): ' + $deployDir); ^
   # Ensure target exists
   if (-not (Test-Path $deployDir)) { Write-Output 'Creating target folder'; New-Item -Path $deployDir -ItemType Directory -Force | Out-Null } else { Write-Output 'Target folder exists' } ; ^
   # Clean target (optional) - comment out if you don't want to remove existing files
   Write-Output 'Cleaning target (remove old files)' ; ^
   Get-ChildItem -Path $deployDir -Force | Where-Object { $_.Name -notin @('.', '..') } | Remove-Item -Recurse -Force -ErrorAction SilentlyContinue ; ^
   # Copy files from workspace to target (recursively)
   Write-Output 'Copying site files...' ; ^
   robocopy \"$src\" \"$deployDir\" /MIR /NDL /NFL /NJH /NJS /NP || { Write-Output 'robocopy returned non-zero; trying xcopy fallback'; cmd /c \"xcopy /E /I /Y \\\"$src\\\" \\\"$deployDir\\\"\" } ; ^
   # Ensure IIS service is running
   Write-Output 'Ensuring W3SVC (IIS) service is present and running...' ; ^
   try { if ((Get-Service -Name W3SVC -ErrorAction SilentlyContinue) -eq $null) { Write-Output 'W3SVC service not found' } } catch { Write-Output 'Get-Service failed: ' + $_.Exception.Message } ; ^
   # Try to install / enable IIS if missing (best-effort; may require admin privileges) 
   $iisInstalled = $false ; 
   try { Import-Module WebAdministration -ErrorAction Stop; $iisInstalled = $true; Write-Output 'WebAdministration module available (IIS likely installed).' } catch { Write-Output 'WebAdministration module not available: ' + $_.Exception.Message } ; ^
   if (-not $iisInstalled) { 
     Write-Output 'Attempting to install IIS (Server) via Install-WindowsFeature - best-effort'; 
     try { Install-WindowsFeature -Name Web-Server -IncludeManagementTools -ErrorAction Stop; Import-Module WebAdministration; $iisInstalled = $true; Write-Output 'IIS installed via Install-WindowsFeature' } catch { Write-Output 'Install-WindowsFeature failed or not available: ' + $_.Exception.Message } 
   } ; ^
   # If IIS module available, configure site
   if ($iisInstalled) { 
     Write-Output 'Configuring IIS site...'; 
     try {
       # Ensure Default Web Site exists and points to $deployDir
       $site = Get-Website -Name 'Default Web Site' -ErrorAction SilentlyContinue
       if ($null -eq $site) {
         Write-Output 'Default Web Site not found — creating a new Default Web Site bound to localhost:80'
         New-Website -Name 'Default Web Site' -Port 80 -PhysicalPath $deployDir -Force
       } else {
         Write-Output 'Default Web Site exists — ensuring physical path points to deploy dir'
         # Update the physical path of root application
         Set-ItemProperty \"IIS:\\Sites\\Default Web Site\" -Name physicalPath -Value $deployDir -ErrorAction SilentlyContinue || Write-Output 'Could not set physicalPath via Set-ItemProperty; trying Set-WebConfigurationProperty'
         # Also set physicalPath for the root application explicitly
         Set-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST' -filter \"system.applicationHost/sites/site[@name='Default Web Site']/application[@path='/']/virtualDirectory[@path='/']\" -name physicalPath -value $deployDir -ErrorAction SilentlyContinue
       }
       # Start site & ensure started
       Start-Service W3SVC -ErrorAction SilentlyContinue
       Start-Website -Name 'Default Web Site' -ErrorAction SilentlyContinue
       Write-Output 'Default Web Site started.'
       # Ensure index.html is in default documents
       $dd = Get-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST' -filter 'system.webServer/defaultDocument/files' -name '.' 
       $hasIndex = $false
       foreach ($item in $dd.Collection) { if ($item.value -eq 'index.html') { $hasIndex = $true } }
       if (-not $hasIndex) { Write-Output 'Adding index.html to default documents'; Add-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST' -filter 'system.webServer/defaultDocument/files' -name '.' -value @{value='index.html'} }
     } catch { Write-Output 'IIS configuration/website actions failed: ' + $_.Exception.Message }
   } else {
     Write-Output 'IIS not available on this machine; skipped site creation. If you want IIS-based hosting, install IIS manually or run on a Windows Server with IIS features.'
   }
   # Fix filesystem permissions so IIS can read (grant Read/Execute to IIS_IUSRS and DefaultAppPool identity)
   Write-Output 'Setting folder permissions for IIS_IUSRS and DefaultAppPool (best-effort)...'
   try { icacls $deployDir /grant 'IIS_IUSRS:(OI)(CI)RX' /T } catch { Write-Output 'icacls failed: ' + $_.Exception.Message }
   # Also attempt to grant to IIS AppPool identity
   try { icacls $deployDir /grant 'IIS AppPool\\DefaultAppPool:(OI)(CI)M' /T } catch { Write-Output 'grant to IIS AppPool identity failed (not critical): ' + $_.Exception.Message }
   Write-Output '=== Deploy to IIS: Finished ==='
  "
'''
        }
      }
    }

    stage('Verify site (http://localhost)') {
      steps {
        script {
          // Do a simple HTTP check to localhost
          bat '''
powershell -NoProfile -Command ^
  "try { $r = Invoke-WebRequest -UseBasicParsing -Uri 'http://localhost/index.html' -TimeoutSec 10; if ($r.StatusCode -eq 200) { Write-Output 'HTTP_OK' ; exit 0 } else { Write-Output 'HTTP_NON200:' + $r.StatusCode ; exit 2 } } catch { Write-Output 'HTTP_ERROR:' + $_.Exception.Message ; exit 3 }"
'''
        }
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
