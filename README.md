name: RDP GAMING - FIXED VERSION

on:
  workflow_dispatch:

jobs:
  gaming-rdp:
    runs-on: windows-latest
    timeout-minutes: 4320
    
    steps:
      - name: Enable Top Performance Mode
        run: |
          powercfg /setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c
          sc config wuauserv start= disabled
          sc stop wuauserv

      - name: Configure Core RDP Settings
        run: |
          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0 -Force
          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name "UserAuthentication" -Value 0 -Force
          netsh advfirewall firewall delete rule name="RDP-Tailscale"
          netsh advfirewall firewall add rule name="RDP-Tailscale" dir=in action=allow protocol=TCP localport=3389
          Restart-Service -Name TermService -Force

      - name: Create RDP User
        run: |
          $password = "Admin123!"
          $securePass = ConvertTo-SecureString $password -AsPlainText -Force
          try { Remove-LocalUser -Name "RDP" -ErrorAction SilentlyContinue } catch {}
          New-LocalUser -Name "RDP" -Password $securePass -AccountNeverExpires
          Add-LocalGroupMember -Group "Administrators" -Member "RDP"
          Add-LocalGroupMember -Group "Remote Desktop Users" -Member "RDP"
          echo "RDP_CREDS=User: RDP | Password: $password" >> $env:GITHUB_ENV

      - name: Install Tailscale
        run: |
          $tsUrl = "https://pkgs.tailscale.com/stable/tailscale-setup-1.82.0-amd64.msi"
          $installerPath = "$env:TEMP\tailscale.msi"
          Invoke-WebRequest -Uri $tsUrl -OutFile $installerPath
          Start-Process msiexec.exe -ArgumentList "/i", "`"$installerPath`"", "/quiet", "/norestart" -Wait

      - name: Connect Tailscale
        run: |
          & "$env:ProgramFiles\Tailscale\tailscale.exe" up --authkey=${{ secrets.TAILSCALE_AUTH_KEY }} --hostname=gh-runner-$env:GITHUB_RUN_ID
          Start-Sleep -Seconds 10
          $tsIP = & "$env:ProgramFiles\Tailscale\tailscale.exe" ip -4
          echo "TAILSCALE_IP=$tsIP" >> $env:GITHUB_ENV

      - name: Install Gaming Essentials
        run: |
          # Install Chocolatey
          Set-ExecutionPolicy Bypass -Scope Process -Force
          [System.Net.ServicePointManager]::SecurityProtocol = 3072
          if (-not (Get-Command choco -ErrorAction SilentlyContinue)) {
              iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
          }
          
          $env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")
          
          # Install core requirements (pake ignore checksums biar ga error)
          choco install vcredist-all -y --ignore-checksums
          choco install directx -y --ignore-checksums
          choco install dotnetfx -y --ignore-checksums
          choco install firefox -y --ignore-checksums

      - name: Install Browsers & Game Launchers
        run: |
          # Chrome via winget
          $env:WINGET_ACCEPT_PACKAGE_AGREEMENTS = "true"
          winget install Google.Chrome --silent --accept-package-agreements --accept-source-agreements || Write-Host "Chrome install skipped" -ForegroundColor Yellow
          
          # Steam
          winget install Valve.Steam --silent --accept-package-agreements || Write-Host "Steam install skipped" -ForegroundColor Yellow
          
          # Discord
          winget install Discord.Discord --silent --accept-package-agreements || Write-Host "Discord install skipped" -ForegroundColor Yellow

      - name: Install Roblox (Pakai Mirror)
        run: |
          Write-Host "Downloading Roblox..." -ForegroundColor Cyan
          
          # Buat folder
          New-Item -ItemType Directory -Path C:\Games -Force -ErrorAction SilentlyContinue
          
          # Coba download dari beberapa sumber
          $urls = @(
              "https://setup.rbxcdn.com/RobloxStudio.exe",
              "https://roblox.schorsch.group/RobloxStudio.exe"
          )
          
          $downloaded = $false
          foreach ($url in $urls) {
              try {
                  Write-Host "Trying $url..." -ForegroundColor Yellow
                  Invoke-WebRequest -Uri $url -OutFile "C:\Games\RobloxStudio.exe" -ErrorAction Stop
                  $downloaded = $true
                  break
              } catch {
                  Write-Host "Failed, trying next..." -ForegroundColor Red
              }
          }
          
          if ($downloaded) {
              Write-Host "Installing Roblox..." -ForegroundColor Cyan
              Start-Process -Wait -FilePath "C:\Games\RobloxStudio.exe" -ArgumentList "/S"
              Write-Host "Roblox installed!" -ForegroundColor Green
          } else {
              Write-Host "Roblox download failed - install manual nanti" -ForegroundColor Yellow
          }

      - name: Optimize Storage (No Error)
        run: |
          Write-Host "Cleaning temporary files..." -ForegroundColor Cyan
          
          # Hapus temp dengan pengecekan
          if (Test-Path "$env:TEMP") {
              Remove-Item "$env:TEMP\*" -Recurse -Force -ErrorAction SilentlyContinue
              Write-Host "Temp folder cleaned" -ForegroundColor Green
          }
          
          if (Test-Path "C:\Windows\Temp") {
              Remove-Item "C:\Windows\Temp\*" -Recurse -Force -ErrorAction SilentlyContinue
              Write-Host "Windows Temp cleaned" -ForegroundColor Green
          }
          
          # Disable hibernation
          powercfg -h off
          
          # Show free space
          $disk = Get-PSDrive C -ErrorAction SilentlyContinue
          if ($disk) {
              $freeGB = [math]::Round($disk.Free/1GB, 2)
              $totalGB = [math]::Round(($disk.Used + $disk.Free)/1GB, 2)
              Write-Host "`n💾 Free space: $freeGB GB / $totalGB GB" -ForegroundColor Green
          }

      - name: Display Connection Info
        run: |
          Clear-Host
          $tsIP = $env:TAILSCALE_IP
          
          Write-Host "`n" ("="*60) "`n" -ForegroundColor Cyan
          Write-Host "           🎮 RDP GAMING - READY 🎮" -ForegroundColor Yellow
          Write-Host "`n" ("="*60) "`n" -ForegroundColor Cyan
          
          Write-Host "📡 CONNECTION:" -ForegroundColor Green
          Write-Host "   IP: $tsIP : 3389" -ForegroundColor White
          Write-Host "`n"
          
          Write-Host "🔐 LOGIN:" -ForegroundColor Green
          Write-Host "   User: RDP" -ForegroundColor White
          Write-Host "   Pass: Admin123!" -ForegroundColor White
          Write-Host "`n"
          
          Write-Host "⚠️ JANGAN TUTUP TAB INI!" -ForegroundColor Red
          Write-Host "`n" ("="*60) "`n" -ForegroundColor Cyan

      - name: Keep Alive
        run: |
          while ($true) {
              $uptime = (Get-Date) - (Get-Process -Id $pid).StartTime
              Write-Host "[$(Get-Date)] 🟢 Uptime: $($uptime.ToString('dd\:hh\:mm\:ss'))"
              Start-Sleep -Seconds 300
          }
