nama: RDP

pada:
  pengiriman_alur_kerja:

Pekerjaan:
  RDP aman:
    berjalan di: windows-latest
    waktu habis-menit: 3600

    tangga:
      - nama: Konfigurasi Pengaturan RDP Inti
        jalankan: |
          # Aktifkan Remote Desktop dan nonaktifkan Otentikasi Tingkat Jaringan (jika diperlukan)
          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' `
                             -Nama "fDenyTSConnections" -Nilai 0 -Paksa
          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' `
                             -Nama "UserAuthentication" -Nilai 0 -Paksa
          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' `
                             -Nama "SecurityLayer" -Nilai 0 -Paksa

          # Hapus aturan yang sudah ada dengan nama yang sama untuk menghindari duplikasi
          netsh advfirewall firewall delete rule name="RDP-Tailscale"
          
          # Untuk pengujian, izinkan semua koneksi masuk pada port 3389
          `netsh advfirewall firewall add rule name="RDP-Tailscale" `
            dir=in action=allow protocol=TCP localport=3389

          # (Opsional) Mulai ulang layanan Remote Desktop untuk memastikan perubahan berlaku
          Restart-Service -Name TermService -Force

      - nama: Membuat Pengguna RDP dengan Kata Sandi Aman
        jalankan: |
          Add-Type -AssemblyName System.Security
          $charSet = @{
              Atas = [char[]](65..90) # AZ
              Lebih rendah = [char[]](97..122) # az
              Angka = [char[]](48..57) # 0-9
              Spesial = ([char[]](33..47) + [char[]](58..64) +
                         [char[]](91..96) + [char[]](123..126)) # Karakter khusus
          }
          $rawPassword = @()
          $rawPassword += $charSet.Upper | Get-Random -Count 4
          $rawPassword += $charSet.Lower | Get-Random -Count 4
          $rawPassword += $charSet.Number | Get-Random -Count 4
          $rawPassword += $charSet.Special | Get-Random -Count 4
          $password = -join ($rawPassword | Sort-Object { Get-Random })
          $securePass = ConvertTo-SecureString $password -AsPlainText -Force
          New-LocalUser -Name "Skyro" -Password $securePass -AccountNeverExpires
          Add-LocalGroupMember -Group "Administrators" -Member "Skyro"
          Add-LocalGroupMember -Group "Remote Desktop Users" -Member "Skyro"
          
          echo "RDP_CREDS=Pengguna: Skyro | Kata Sandi: $password" >> $env:GITHUB_ENV
          
          jika (-bukan (Get-LocalUser -Name "Skyro")) {
              Write-Error "Pembuatan pengguna gagal"
              keluar 1
          }

      - nama: Instal Tailscale
        jalankan: |
          $tsUrl = "https://pkgs.tailscale.com/stable/tailscale-setup-1.82.0-amd64.msi"
          $installerPath = "$env:TEMP\tailscale.msi"
          
          Invoke-WebRequest -Uri $tsUrl -OutFile $installerPath
          Mulai-Proses msiexec.exe -ArgumentList "/i", "`"$installerPath`"", "/quiet", "/norestart" -Wait
          Hapus Item $installerPath -Paksa

      - nama: Membangun Koneksi Tailscale
        jalankan: |
          # Jalankan Tailscale dengan kunci otentikasi yang diberikan dan tetapkan nama host yang unik
          & "$env:ProgramFiles\Tailscale\tailscale.exe" up --authkey=${{ secrets.TAILSCALE_AUTH_KEY }} --hostname=gh-runner-$env:GITHUB_RUN_ID
          
          # Tunggu Tailscale menetapkan IP
          $tsIP = $null
          $percobaan ulang = 0
          selama (-tidak $tsIP -dan $retries -lt 10) {
              $tsIP = & "$env:ProgramFiles\Tailscale\tailscale.exe" ip -4
              Mulai-Tidur -Detik 5
              $percobaan ulang++
          }
          
          jika (-bukan $tsIP) {
              Write-Error "IP Tailscale tidak ditetapkan. Keluar."
              keluar 1
          }
          echo "TAILSCALE_IP=$tsIP" >> $env:GITHUB_ENV
      
      - nama: Verifikasi Aksesibilitas RDP
        jalankan: |
          Write-Host "Alamat IP Tailscale: $env:TAILSCALE_IP"
          
          # Uji konektivitas menggunakan Test-NetConnection terhadap IP Tailscale pada port 3389
          $testResult = Test-NetConnection -ComputerName $env:TAILSCALE_IP -Port 3389
          jika (-not $testResult.TcpTestSucceeded) {
              Write-Error "Koneksi TCP ke port RDP 3389 gagal"
              keluar 1
          }
          Write-Host "Konektivitas TCP berhasil!"

      - nama: Pertahankan Koneksi
        jalankan: |
          Write-Host "`n=== AKSES RDP ==="
          Write-Host "Alamat: $env:TAILSCALE_IP"
          Write-Host "Nama Pengguna: Skyro"
          Write-Host "Kata Sandi: $(echo $env:RDP_CREDS)"
          Tulis-Host "==================`n"
          
          # Pertahankan runner tetap aktif tanpa batas waktu (atau hingga dibatalkan secara manual)
          selama ($true) {
              Write-Host "[$(Get-Date)] RDP Aktif - Gunakan Ctrl+C di alur kerja untuk mengakhiri"
              Mulai-Tidur -Detik 300
          }
