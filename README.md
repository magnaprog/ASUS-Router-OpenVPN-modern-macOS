# ASUS Router OpenVPN Server and macOS Tunnelblick Client Setup Guide

This guide will help you set up an OpenVPN server on your ASUS router and configure your macOS to connect to it using Tunnelblick.

---

## 1. ASUS Router Configuration (OpenVPN Server Setup)

### Step 1: Install EasyRSA on macOS

1. Open the Terminal on your Mac.
2. Install EasyRSA using Homebrew:
   ```bash
   brew install easy-rsa
   ```
   Note: If you encounter any issues, you may need to update Homebrew first.

3. Verify the installation:
   ```bash
   easyrsa --version
   ```
   You should see output similar to:
   ```
   EasyRSA Version Information
   Version:     3.2.1
   Generated:   Fri Sep 13 13:04:18 CDT 2024
   SSL Lib:     OpenSSL 3.3.2 3 Sep 2024 (Library: OpenSSL 3.3.2 3 Sep 2024)
   Git Commit:  3f60a68702713161ab44f9dd80ce01f588ca49ac
   Source Repo: https://github.com/OpenVPN/easy-rsa
   ```

4. Create a directory for your VPN setup:
   ```bash
   mkdir ~/easyrsa-vpn
   cd ~/easyrsa-vpn
   ```

5. Initialize the PKI (Public Key Infrastructure) directory:
   ```bash
   easyrsa init-pki
   ```
   You should see a message indicating that the PKI initialization is complete.

### Step 2: Create a Certificate Authority (CA)

1. Build the CA:
   ```bash
   easyrsa build-ca
   ```

2. You will be prompted to enter a CA Key Passphrase. Choose a strong passphrase and remember it, as you'll need it to sign certificates later.

3. When prompted for the Common Name, you can use the default or enter a custom name (e.g., use your username for instance).

4. After completion, you should see a message indicating that the CA creation is complete and the location of your new CA certificate (typically at `/opt/homebrew/etc/pki/ca.crt`).

### Step 3: Create Certificates for the Server and Clients

#### Server Certificate
1. Generate a certificate for the VPN server:
   ```bash
   easyrsa gen-req server nopass
   ```
2. Sign the server's certificate request:
   ```bash
   easyrsa sign-req server server
   ```

#### Client Certificate
1. Generate a certificate request for a client:
   ```bash
   easyrsa gen-req client1 nopass
   ```
2. Sign the client certificate request:
   ```bash
   easyrsa sign-req client client1
   ```

#### Diffie-Hellman Parameters
Generate Diffie-Hellman parameters:
```bash
easyrsa gen-dh
```

#### Generate a TLS Key
For added security, generate a TLS-Auth key:
```bash
openvpn --genkey --secret ta.key
```

### Step 4: Export Files for the ASUS Router

Prepare the following files for upload to your ASUS router:
- CA Certificate: `pki/ca.crt`
- Server Certificate: `pki/issued/server.crt`
- Server Key: `pki/private/server.key`
- DH Parameters: `pki/dh.pem`
- TLS Key: `ta.key`

### Step 5: Enable OpenVPN Server
1. Log in to your ASUS router's web interface via `http://router.asus.com` or your router's IP address.
2. Navigate to **VPN** > **VPN Server** > **OpenVPN**.
3. Enable the **OpenVPN Server** by toggling the switch to **ON**.

### Step 6: Upload Certificates and Keys
1. Upload the following files to the respective fields:
   - **CA Certificate**: `ca.crt`
   - **Server Certificate**: `server.crt`
   - **Server Key**: `server.key`
   - **DH Parameters**: `dh.pem`
   - **TLS Key**: `ta.key` (optional but recommended)

### Step 7: General Settings
- **Client will use VPN to access**: **LAN only**

![ASUS Router General Settings](screenshots/ASUS%20Router%20General%20Settings.png)

### Step 8: Advanced Settings
- **Interface Type**: **TUN**
- **Protocol**: **UDP** (you can choose TCP if you need more reliability at the expense of speed)
- **Server Port**: **1194**
- **Authorization Mode**: **TLS**
- **HMAC Authentication**: **SHA256**
- **VPN Subnet/Netmask**: Keep default (e.g., `10.8.0.0/255.255.255.0`)
- **Advertise DNS to clients**: **Yes**
  - Ensures VPN clients use the router's DNS server.
- **Log verbosity**: **3**

![ASUS Router Advanced Settings](screenshots/ASUS%20Router%20Advanced%20Settings.png)

### Step 9: Custom Configuration
Add the following lines to the **Custom Configuration** section:
```
#push "redirect-gateway def1"
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 8.8.4.4"
persist-key
persist-tun
```

Let's break down these settings:

- `#push "redirect-gateway def1"`: This line is commented out. When uncommented, it forces all client traffic through the VPN. We don't need this for now because:
  1. It may cause connectivity issues if not all services are accessible through the VPN.
  2. It allows for split-tunneling, where only specific traffic goes through the VPN while other traffic uses the regular internet connection.
  3. It provides more flexibility in how you use the VPN connection.

  You might want to uncomment this line if:
  - You want to ensure all internet traffic is routed through the VPN for maximum privacy.
  - You're using the VPN to access region-restricted content and need all traffic to appear from the VPN's location.
  - You're in a situation where you need to hide all your internet activity from your local network.

- `push "dhcp-option DNS 8.8.8.8"` and `push "dhcp-option DNS 8.8.4.4"`: These lines push Google's public DNS servers to the clients.
- `persist-key` and `persist-tun`: These options keep the key and tunnel device open across restarts.

### Step 10: Export OVPN Configuration File
1. Click **Apply** to save your changes.
2. After setting up the VPN server, click **Export OpenVPN configuration file** to download the `.ovpn` file.
3. This `.ovpn` file will be used to configure clients (e.g., your Mac).

### Step 11: Modify the OVPN Configuration File

1. After exporting the OpenVPN configuration file from your ASUS router, open the exported .ovpn file in a text editor.

2. Locate the `<cert>` and `<key>` sections in the .ovpn file.

3. Replace the placeholder text in the `<cert>` section with the contents of `client1.crt`.

4. Replace the placeholder text in the `<key>` section with the contents of `client1.key`.

5. Save the modified .ovpn file. This file now contains all the necessary certificate and key information for the client.

### Step 12: Import the Modified OVPN File into Tunnelblick

1. Double-click the modified .ovpn file to import it into Tunnelblick.

2. Follow the prompts to add the new configuration to Tunnelblick.

By following these steps, you'll create a unique client certificate and key, and incorporate them into the OVPN configuration file. This ensures that each client has its own secure credentials for connecting to the VPN.

---

## 2. macOS Configuration (Tunnelblick VPN Client Setup)

### Step 1: Install Tunnelblick
1. Download and install [Tunnelblick](https://tunnelblick.net/).
2. Launch Tunnelblick after installation.

### Step 2: Import the OVPN Configuration
1. Open Tunnelblick and click **"I have configuration files"**.
2. Drag and drop the `.ovpn` file you exported from your ASUS router into Tunnelblick.
3. Tunnelblick will prompt to install the configuration for **All Users** or **Only Me** — choose based on your preference.

### Step 3: Configure DNS and Routing in Tunnelblick
1. After importing the `.ovpn` file, open **VPN Details** in Tunnelblick.
2. Go to the **Settings** tab for your VPN profile.
3. Make sure the following options are set:
   - **Set DNS/WINS**: **Tunnelblick's default setting**.
   - **Flush DNS cache**: **Checked** (to avoid DNS leaks).
   - **Disable IPv6 unless the VPN server is accessed using IPv6**: **Checked** (prevents IPv6 leaks).
4. Optionally, enable the **Reconnect on disconnect** option to automatically reconnect the VPN.

![Tunnelblick Settings](screenshots/Tunnelblick%20Settings.png)

### Step 4: Connect to the VPN
1. In Tunnelblick, select the VPN profile you just imported and click **Connect**.
2. Tunnelblick will ask for your credentials if required (username and password).

### Step 5: Flush DNS Cache (if needed)
After connecting to the VPN, you may need to flush your DNS cache to ensure that your computer uses the VPN's DNS servers. This is especially important if you're experiencing issues with accessing certain websites or if you want to make sure you're using the VPN's DNS settings.

To flush the DNS cache on macOS, open Terminal and run the following command:

```bash
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
```

You'll be prompted to enter your macOS user password. After running this command, your DNS cache will be cleared, and the DNS resolver will be restarted.

When to use this command:
- After connecting to the VPN for the first time
- If you're having trouble accessing certain websites while connected to the VPN
- If you've made changes to your network or VPN settings and want to ensure they take effect immediately

Note: You may need to run this command each time you connect to the VPN, especially if you're experiencing DNS-related issues.

---

## 3. Testing the VPN Connection

### Step 1: Verify Public IP
After connecting to the VPN, visit [WhatIsMyIP.com](https://www.whatismyip.com/) or similar services to verify that your public IP address has changed to your VPN server's public IP (your home IP address).

### Step 2: Check DNS Leaks
Go to [DNSLeakTest.com](https://www.dnsleaktest.com/) and perform a DNS leak test. Ensure that DNS queries are being routed through the VPN and the DNS server provided by your router (e.g., `192.168.1.1`) is being used.

---

## 4. Optional: ASUS Router VPN Client Setup (If Required)

If you ever want your ASUS router to act as a VPN client (connecting to an external VPN service like NordVPN, ExpressVPN, etc.), follow these steps:

### Step 1: Upload OVPN File to ASUS Router
1. Go to **VPN** > **VPN Client** in the ASUS router interface.
2. Click **Add Profile**.
3. Select **OpenVPN**.
4. Upload the `.ovpn` file provided by your VPN provider (this is different from your server setup).
5. Enter any additional required credentials (username, password).

### Step 2: Start VPN Client
1. After the profile is added, click **Activate** to connect the ASUS router as a VPN client.

---

## 5. Troubleshooting

### Issue: Public IP Doesn’t Change After Connecting
- Note that the `redirect-gateway def1` directive is commented out in the custom configuration. If you want all traffic to go through the VPN, uncomment this line.
- If you want selective routing, you may need to add specific `route` commands to your OpenVPN configuration.

### Issue: DNS Configuration
- The current setup uses Google's public DNS servers. If you want to use your router's DNS or other servers, modify the `push "dhcp-option DNS"` lines in the custom configuration.

### Issue: Can’t Connect to the VPN
- Ensure that the firewall on your router allows traffic on the OpenVPN port (e.g., `1194`).
- Check that your ISP isn’t blocking VPN traffic or UDP ports.

---

## 6. iOS and Android Configuration

### iOS Setup

#### Step 1: Install OpenVPN Connect
1. Download and install [OpenVPN Connect](https://apps.apple.com/us/app/openvpn-connect/id590379981) from the App Store.

#### Step 2: Import the OVPN Configuration
1. Email the `.ovpn` file to yourself or use a cloud storage service to transfer it to your iOS device.
2. Tap on the `.ovpn` file and select "Open in OpenVPN".
3. In the OpenVPN app, tap the "+" icon to add the profile.
4. Review the profile details and tap "Add" to import it.

#### Step 3: Connect to the VPN
1. In the OpenVPN app, tap on the newly added profile.
2. Slide the "Connection" toggle to connect to the VPN.
3. If prompted, enter your username and password.

### Android Setup

#### Step 1: Install OpenVPN Connect
1. Download and install [OpenVPN Connect](https://play.google.com/store/apps/details?id=net.openvpn.openvpn) from the Google Play Store.

#### Step 2: Import the OVPN Configuration
1. Transfer the `.ovpn` file to your Android device using email, cloud storage, or direct file transfer.
2. Open the OpenVPN Connect app and tap the "+" icon.
3. Select "Import" and choose the `.ovpn` file from your device.
4. Review the profile details and tap "Import" to add it.

#### Step 3: Connect to the VPN
1. In the OpenVPN app, tap on the imported profile.
2. Tap the "Connect" button to establish the VPN connection.
3. If required, enter your username and password.

---

By following this guide, you can re-establish the OpenVPN server on your ASUS router and configure your Mac as a VPN client whenever needed. This process will route your internet traffic through your home network securely, ensuring both privacy and access to local resources.