# BitChat Deployment Guide

This guide explains how to "host" and deploy BitChat in various contexts. Since BitChat is a native iOS/macOS application with a decentralized architecture, "hosting" has different meanings depending on your goal.

## Table of Contents

- [Understanding BitChat Architecture](#understanding-bitchat-architecture)
- [Running BitChat Locally](#running-bitchat-locally)
- [Distributing to Users](#distributing-to-users)
- [Running Your Own Nostr Relay (Optional)](#running-your-own-nostr-relay-optional)
- [Enterprise Deployment](#enterprise-deployment)

---

## Understanding BitChat Architecture

BitChat is **not a traditional server-client application**. It's a decentralized, peer-to-peer messaging app with two transport layers:

1. **Bluetooth Mesh Network**: Completely offline, device-to-device communication
   - No servers required
   - Works in disaster scenarios, protests, remote areas
   - Messages relay through nearby devices

2. **Nostr Protocol**: Internet-based communication using public relays
   - Connects to existing Nostr relay network (290+ relays worldwide)
   - Location-based chat rooms using geohash
   - No central servers owned by BitChat

**What you can "host":**
- ✅ Build and run the app locally for development/testing
- ✅ Distribute the app via App Store or TestFlight
- ✅ Run your own Nostr relay (optional, for privacy/control)
- ❌ There's no "BitChat server" to deploy (it's decentralized)

---

## Running BitChat Locally

### Prerequisites

- **macOS 13.0+** (Ventura) or **iOS 16.0+**
- **Xcode 15.0+** - [Download from App Store](https://apps.apple.com/us/app/xcode/id497799835)
- **Apple Developer Account** (free tier works for local development)
- **Git** - Pre-installed on macOS
- **Homebrew** (optional, for `just` command runner) - [Install Homebrew](https://brew.sh/)

### Quick Start

#### Option 1: Using `just` (Recommended for macOS)

```bash
# Clone the repository
git clone https://github.com/JEROME-PRAKASH-L/bitchat.git
cd bitchat

# Install just command runner
brew install just

# Build and run on macOS
just run

# Clean up afterwards
just clean
```

#### Option 2: Using Xcode

```bash
# Clone the repository
git clone https://github.com/JEROME-PRAKASH-L/bitchat.git
cd bitchat

# Configure local settings
cp Configs/Local.xcconfig.example Configs/Local.xcconfig

# Edit Local.xcconfig and add your Team ID
# Find your Team ID: https://stackoverflow.com/a/18727947
# Replace ABC123 with your actual Team ID
sed -i '' 's/ABC123/YOUR_TEAM_ID_HERE/' Configs/Local.xcconfig

# Open in Xcode
open bitchat.xcodeproj
```

**Important Configuration Steps:**

1. **Update Team ID**: Edit `Configs/Local.xcconfig` with your Apple Developer Team ID
2. **Update Entitlements** (for device deployment):
   - Search for `group.chat.bitchat` in the project
   - Replace with `group.<your_bundle_id>` (e.g., `group.chat.bitchat.ABC123`)
   - Update in all `.entitlements` files

3. **Select Target**:
   - For macOS: Select "bitchat (macOS)" scheme
   - For iOS: Select "bitchat (iOS)" scheme and choose a physical device (Bluetooth requires real hardware)

4. **Build and Run**: Press `Cmd+R` or click the Run button

### Development Tips

```bash
# Quick development build without cleanup
just dev-run

# Check prerequisites
just check

# View available commands
just

# Build without running
just build

# Show app information
just info
```

---

## Distributing to Users

### 1. TestFlight Distribution (Beta Testing)

TestFlight allows you to distribute beta versions to up to 10,000 external testers.

#### Steps:

1. **Configure Release Build**:
   ```bash
   # Ensure Release.xcconfig is properly configured
   cat Configs/Release.xcconfig
   ```

2. **Archive the App**:
   - Open project in Xcode
   - Select "Any iOS Device" as destination
   - Choose **Product > Archive**
   - Wait for archive to complete

3. **Upload to App Store Connect**:
   - In the Xcode Organizer, click **Distribute App**
   - Select **App Store Connect**
   - Follow the wizard to upload

4. **Configure TestFlight**:
   - Go to [App Store Connect](https://appstoreconnect.apple.com/)
   - Select your app > TestFlight tab
   - Add internal/external testers
   - Submit for TestFlight review (external testers only)

5. **Share with Testers**:
   - Testers install TestFlight from App Store
   - Send invitation link or add by email
   - Testers download your app via TestFlight

### 2. App Store Distribution (Production)

#### Prerequisites:

- **Paid Apple Developer Account** ($99/year)
- App Store listing prepared (screenshots, description, etc.)
- App reviewed and approved by Apple

#### Steps:

1. **Prepare App Store Listing**:
   - Create app in [App Store Connect](https://appstoreconnect.apple.com/)
   - Fill in metadata (name, description, keywords, screenshots)
   - Set pricing and availability

2. **Archive and Upload** (same as TestFlight):
   - Product > Archive
   - Distribute App > App Store Connect

3. **Submit for Review**:
   - In App Store Connect, add build to your app version
   - Fill in "What's New" and other required fields
   - Click **Submit for Review**

4. **Review Process**:
   - Apple reviews app (typically 24-48 hours)
   - Address any feedback/rejections
   - Once approved, app is live on App Store

#### App Store Guidelines for BitChat:

- Emphasize privacy and security features
- Clearly explain Bluetooth permissions
- Include privacy policy (already exists at PRIVACY_POLICY.md)
- Ensure compliance with encryption export regulations
- Include demo mode or test account information

### 3. Enterprise Distribution (Internal)

For distributing to your organization without App Store:

#### Prerequisites:

- **Apple Developer Enterprise Program** ($299/year)
- Cannot be used for public distribution

#### Steps:

1. **Create Enterprise Distribution Certificate**:
   - Apple Developer Portal > Certificates, IDs & Profiles
   - Create new certificate for "In-House and Ad Hoc"

2. **Archive with Enterprise Profile**:
   - Xcode > Product > Archive
   - Distribute App > Enterprise
   - Select distribution certificate

3. **Host the IPA file**:
   ```bash
   # You'll need to host the .ipa and manifest.plist on HTTPS server
   # Create manifest.plist with your app details
   ```

4. **Install on Devices**:
   - Users visit your enterprise distribution URL
   - Tap to install (requires trust of enterprise certificate)

### 4. Ad Hoc Distribution (Limited Devices)

For testing on specific devices without TestFlight:

1. **Register Device UDIDs**:
   - Apple Developer Portal > Devices
   - Add each tester's device UDID

2. **Create Ad Hoc Provisioning Profile**:
   - Include registered devices in profile

3. **Archive and Export**:
   - Product > Archive
   - Distribute App > Ad Hoc
   - Select devices

4. **Distribute IPA**:
   - Share .ipa file via email, cloud storage, etc.
   - Testers install via Xcode or tools like Apple Configurator

---

## Running Your Own Nostr Relay (Optional)

BitChat connects to public Nostr relays for internet-based messaging. You can optionally run your own relay for:
- Privacy and control over your data
- Custom relay policies
- Reduced dependence on public infrastructure
- Lower latency for your community

### Popular Nostr Relay Implementations

#### 1. **nostr-rs-relay** (Rust)

Fast and efficient Rust implementation.

```bash
# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Clone and build
git clone https://github.com/scsibug/nostr-rs-relay.git
cd nostr-rs-relay
cargo build --release

# Configure
cp config.toml.example config.toml
# Edit config.toml with your settings

# Run
./target/release/nostr-rs-relay

# Access at ws://localhost:8080 (or configured port)
```

#### 2. **strfry** (C++)

High-performance C++ relay.

```bash
# Install dependencies (Ubuntu/Debian)
sudo apt-get install git g++ make pkg-config libtool ca-certificates \
    libssl-dev libsqlite3-dev liblmdb-dev libsecp256k1-dev

# Clone and build
git clone https://github.com/hoytech/strfry.git
cd strfry
git submodule update --init
make setup-golpe
make

# Run
./strfry relay

# Access at ws://localhost:7777
```

#### 3. **relay.py** (Python)

Simple Python implementation, good for learning.

```bash
# Install
pip install nostr-relay

# Run
nostr-relay --host 0.0.0.0 --port 8008

# Access at ws://localhost:8008
```

### Configuring BitChat to Use Your Relay

BitChat automatically connects to the relay network defined in `relays/online_relays_gps.csv`. To add your relay:

1. **Edit the relay list** (requires rebuilding):
   ```bash
   cd bitchat
   # Add your relay to relays/online_relays_gps.csv
   echo "wss://your-relay.example.com,40.7128,-74.0060" >> relays/online_relays_gps.csv
   ```

2. **Rebuild the app** with your relay included

3. **Or modify at runtime** (advanced):
   - BitChat connects to multiple relays
   - Your relay will be used if it's in the list
   - Consider forking and maintaining your own relay list

### Hosting Your Relay

#### Cloud Providers:

**DigitalOcean / Linode / Vultr:**
```bash
# 1. Create a VPS ($5-10/month minimum)
# 2. Install relay software (see above)
# 3. Configure firewall to allow WebSocket port
sudo ufw allow 8080/tcp
# 4. Set up HTTPS with Let's Encrypt
sudo apt install certbot
sudo certbot certonly --standalone -d relay.example.com
# 5. Configure reverse proxy (nginx or caddy)
```

**Example nginx configuration:**
```nginx
server {
    listen 443 ssl http2;
    server_name relay.example.com;

    ssl_certificate /etc/letsencrypt/live/relay.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/relay.example.com/privkey.pem;

    location / {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
```

**Docker Deployment:**
```bash
# Example for nostr-rs-relay
docker run -d \
  --name nostr-relay \
  -p 8080:8080 \
  -v $(pwd)/data:/usr/src/app/db \
  scsibug/nostr-rs-relay:latest
```

### Relay Considerations

- **Storage**: Relays can grow large over time (plan for 10GB+ per month depending on usage)
- **Bandwidth**: Expect significant traffic for popular relays
- **Moderation**: Consider implementing spam filtering and content policies
- **Performance**: Use SSD storage and sufficient RAM (2GB+ recommended)
- **Security**: Keep relay software updated, use HTTPS/WSS, implement rate limiting

### Testing Your Relay

```bash
# Test WebSocket connection
npm install -g wscat
wscat -c wss://your-relay.example.com

# Send a Nostr event
# (Nostr protocol details: https://github.com/nostr-protocol/nips)
```

---

## Enterprise Deployment

### Deploying BitChat Within an Organization

#### Scenario 1: Corporate Emergency Communication

**Setup:**
1. Deploy BitChat via MDM (Mobile Device Management)
2. Pre-configure relay list with corporate relay
3. Distribute via Enterprise App Store or MDM

**Configuration:**
```bash
# Create custom configuration
# Modify relay list to include only corporate relays
# Package with enterprise provisioning profile
```

#### Scenario 2: Event/Conference Communication

**Setup:**
1. Set up temporary relay for event
2. Create custom build with event relay
3. Distribute via QR code or TestFlight

**Example:**
```bash
# Event organizer deploys relay at event location
docker run -d -p 8080:8080 strfry-relay

# Attendees connect to local relay
# No internet required if using Bluetooth mesh
```

#### Scenario 3: Government/Military Deployment

**Requirements:**
- Air-gapped deployment (Bluetooth only)
- No external relay connections
- Enhanced security review

**Configuration:**
1. Disable Nostr transport entirely
2. Deploy Bluetooth-only version
3. Strict device enrollment and auditing

### Custom Deployment Scripts

```bash
#!/bin/bash
# deploy-bitchat-enterprise.sh

# 1. Configure build for enterprise
cp Configs/Release.xcconfig Configs/Enterprise.xcconfig
echo "RELAY_URL = wss://internal-relay.corp.com" >> Configs/Enterprise.xcconfig

# 2. Build with enterprise profile
xcodebuild -project bitchat.xcodeproj \
  -scheme "bitchat (iOS)" \
  -configuration Enterprise \
  -archivePath ./build/BitChat.xcarchive \
  archive

# 3. Export IPA
xcodebuild -exportArchive \
  -archivePath ./build/BitChat.xcarchive \
  -exportPath ./build/output \
  -exportOptionsPlist ExportOptions.plist

# 4. Deploy to MDM or distribution server
echo "IPA created at: ./build/output/bitchat.ipa"
```

---

## Troubleshooting Deployment

### Common Issues

**1. Code Signing Errors**
```bash
# Error: No signing identity found
# Solution: Ensure you're logged into Xcode with your Apple ID
# Xcode > Preferences > Accounts > Add Apple ID
```

**2. Entitlements Errors**
```bash
# Error: App Groups not configured
# Solution: Update all instances of group.chat.bitchat with your bundle ID
grep -r "group.chat.bitchat" . --include="*.entitlements"
# Replace each occurrence with your group ID
```

**3. Bluetooth Permissions**
```bash
# Error: Bluetooth not authorized
# Solution: Ensure Info.plist contains:
# - NSBluetoothAlwaysUsageDescription
# - NSBluetoothPeripheralUsageDescription
```

**4. Relay Connection Issues**
```bash
# Check relay connectivity
wscat -c wss://relay.example.com

# Verify relay is in relay list
grep "relay.example.com" relays/online_relays_gps.csv
```

**5. Build Failures**
```bash
# Clean build folder
just clean
# Or in Xcode: Product > Clean Build Folder (Cmd+Shift+K)

# Reset package cache
rm -rf ~/Library/Developer/Xcode/DerivedData
```

---

## Security Considerations

### Before Deploying:

1. **Review Security Audit**: See `docs/privacy-assessment.md`
2. **Understand Encryption**: Read `docs/NOISE_PROTOCOL_DOCUMENTATION.md`
3. **Privacy Policy**: Ensure compliance with `PRIVACY_POLICY.md`
4. **Code Review**: Review security-critical code:
   - Noise protocol implementation
   - Key management
   - Message encryption/decryption

### Production Checklist:

- [ ] All API keys and secrets removed from source
- [ ] Debug logging disabled in release builds
- [ ] Certificate pinning enabled (if using custom relays)
- [ ] App Transport Security properly configured
- [ ] Crash reporting configured (but privacy-preserving)
- [ ] Emergency wipe feature tested
- [ ] Backup/restore functionality tested

---

## Monitoring and Maintenance

### Relay Monitoring

```bash
# Monitor relay logs
tail -f /var/log/nostr-relay.log

# Check relay status
curl -I https://your-relay.example.com

# Monitor connections (for nostr-rs-relay)
# Check metrics endpoint if enabled
curl http://localhost:8080/metrics
```

### App Analytics (Privacy-Preserving)

- Use privacy-preserving analytics (no user tracking)
- Monitor crash reports via App Store Connect
- Track adoption via App Store metrics only

---

## Additional Resources

### Official Resources:
- [BitChat Website](http://bitchat.free)
- [App Store Listing](https://apps.apple.com/us/app/bitchat-mesh/id6748219622)
- [GitHub Repository](https://github.com/JEROME-PRAKASH-L/bitchat)

### Nostr Resources:
- [Nostr Protocol NIPs](https://github.com/nostr-protocol/nips)
- [Nostr Relay List](https://nostr.watch/)
- [Nostr Implementation Guide](https://github.com/nostr-protocol/nostr)

### Apple Developer Resources:
- [App Store Connect](https://appstoreconnect.apple.com/)
- [Apple Developer Portal](https://developer.apple.com/)
- [TestFlight Documentation](https://developer.apple.com/testflight/)
- [Enterprise Distribution Guide](https://developer.apple.com/programs/enterprise/)

### Community:
- Join Nostr network to discuss BitChat
- Report issues on GitHub
- Contribute to documentation

---

## Conclusion

"Hosting" BitChat means understanding its decentralized architecture and choosing the right deployment method for your needs:

- **For Development**: Use `just run` or Xcode
- **For Testing**: Use TestFlight
- **For Public Distribution**: Use App Store
- **For Organizations**: Use Enterprise Distribution
- **For Privacy**: Run your own Nostr relay

BitChat requires no central server, making it resilient and censorship-resistant. The app connects to a decentralized network of Nostr relays and uses peer-to-peer Bluetooth for offline communication.

For questions or support, please open an issue on GitHub or join the Nostr network.
