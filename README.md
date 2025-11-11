# Asterisk PBX Setup with Docker

A complete Asterisk PBX setup running in Docker for local network SIP communication on macOS.

## Overview

This setup provides a fully functional Asterisk PBX server running in a Docker container, configured for SIP communication on your local network. It includes:

- PJSIP configuration for modern SIP handling
- RTP media configuration for audio transmission
- Example SIP endpoints (1001, 1002) for testing
- Pre-configured dialplan with test extensions
- CDR logging
- Sound files for IVR and announcements

## Prerequisites

- Docker Desktop installed on macOS
- Your Mac's local network IP address (e.g., `192.168.0.104`)
- SIP client software (e.g., Zoiper, Linphone, or any SIP phone)

## Setup Flow

### 1. Find Your Mac's IP Address

```bash
# Find your local IP address
ifconfig | grep "inet " | grep -v 127.0.0.1
# Or use:
ipconfig getifaddr en0
```

Note this IP address - you'll need it for configuration.

### 2. Update Configuration Files

#### Update `asterisk_conf/pjsip.conf`

Edit the transport section to use your Mac's IP address:


```ini
[transport-udp]
type=transport
protocol=udp
bind=0.0.0.0:5060

; ADVERTISE YOUR MAC'S LAN IP (change to your real IP)
external_signaling_address=192.168.0.104
external_media_address=192.168.0.104
```

Also update the `media_address` in endpoint defaults:

```ini
[endpoint-defaults](!)
...
media_address=192.168.0.104
```

#### Update `asterisk_conf/asterisk.conf`

Ensure the `externip` matches your Mac's IP:

```ini
[options]
externip = 192.168.0.104
localnet = 192.168.0.0/255.255.255.0
```

### 3. Start the Container

```bash
# Start Asterisk
docker-compose up -d

# Check if it's running
docker-compose ps

# View logs
docker-compose logs -f asterisk
```

### 4. Verify Asterisk is Running

```bash
# Connect to Asterisk CLI
docker-compose exec asterisk asterisk -rvvv

# Or run a single command
docker-compose exec asterisk asterisk -rx "core show version"
```

### 5. Configure SIP Clients

Use the following credentials for SIP endpoints:

**Extension 1001:**
- Username: `1001`
- Password: `p1001`
- Server: `192.168.0.104` (your Mac's IP)
- Port: `5060`
- Transport: `UDP`

**Extension 1002:**
- Username: `1002`
- Password: `p1002`
- Server: `192.168.0.104` (your Mac's IP)
- Port: `5060`
- Transport: `UDP`

### 6. Test Registration

```bash
# Check registered endpoints
docker-compose exec asterisk asterisk -rx "pjsip show endpoints"

# Check registrations
docker-compose exec asterisk asterisk -rx "pjsip show registrations"
```

## Configuration Details

### Port Mappings

- **5060/UDP & TCP**: SIP signaling
- **10000-10100/UDP**: RTP media ports

### Key Configuration Files

- `asterisk_conf/pjsip.conf`: PJSIP transport and endpoint configuration
- `asterisk_conf/extensions.conf`: Dialplan configuration
- `asterisk_conf/rtp.conf`: RTP media settings
- `asterisk_conf/asterisk.conf`: Core Asterisk settings

### Default Extensions

- **600**: Interactive IVR menu
- **601**: Dial tone playback
- **1001**: Dial extension 1001
- **1002**: Dial extension 1002

## Audio Diagnosis Process

When experiencing audio issues (no sound, one-way audio), follow this systematic diagnosis:

### Step 1: Verify SIP Registration

```bash
# Check if endpoints are registered
docker-compose exec asterisk asterisk -rx "pjsip show endpoints"

# Expected output should show endpoints as "Available"
```

**If not registered:**
- Check SIP client credentials match `pjsip.conf`
- Verify firewall allows UDP port 5060
- Check logs: `docker-compose logs asterisk | grep -i register`

### Step 2: Check Call Establishment

```bash
# Monitor Asterisk logs during a call
docker-compose logs -f asterisk

# Look for:
# - "Channel PJSIP/1001-xxxxx answered"
# - "Locally RTP bridged"
```

**If call doesn't connect:**
- Verify dialplan context matches endpoint context (`local`)
- Check extensions.conf routing
- Verify both endpoints are registered

### Step 3: Verify SDP (Session Description Protocol)

The SDP in SIP INVITE/200 OK messages must show the correct IP address.

```bash
# Enable verbose SIP logging
docker-compose exec asterisk asterisk -rx "pjsip set logger on"

# Make a call and check logs for SDP:
docker-compose logs asterisk | grep -A 10 "Content-Type: application/sdp"
```

**Check the `c=` line in SDP:**
```
c=IN IP4 192.168.0.104  ← Should be your Mac's IP, NOT container IP
```

**If SDP shows container IP (e.g., 172.21.0.2):**
- Verify `external_media_address` in `pjsip.conf` transport section
- Verify `media_address` in endpoint defaults
- Ensure Docker bridge networks are NOT in `local_net` (only your LAN subnet)
- Restart container: `docker-compose restart asterisk`

### Step 4: Check RTP Configuration

```bash
# Verify RTP settings
docker-compose exec asterisk asterisk -rx "rtp show settings"

# Check RTP statistics during a call
docker-compose exec asterisk asterisk -rx "rtp set debug on"
```

**Verify in `rtp.conf`:**
- `rtpbindaddr=0.0.0.0` (binds to all interfaces)
- `strictrtp=no` (helps with Docker NAT)
- Port range matches docker-compose.yml (10000-10100)

### Step 5: Monitor RTP Traffic

```bash
# Enable RTP debugging
docker-compose exec asterisk asterisk -rx "rtp set debug on"

# Make a call and watch for:
# - "Strict RTP learning after remote address set to: X.X.X.X:PORT"
# - "Strict RTP switching source address" (should NOT switch to Docker IP)
```

**Common issues:**
- RTP switching to Docker bridge IP (192.168.65.1 or 172.21.0.x) → Fix SDP IP
- No RTP packets received → Check firewall for UDP 10000-10100
- RTP learning wrong address → Ensure `strictrtp=no` or fix NAT traversal

### Step 6: Verify Network Connectivity

```bash
# From your Mac, test if ports are accessible
nc -u -v 192.168.0.104 5060  # SIP port
nc -u -v 192.168.0.104 10000 # RTP port

# Check if container can receive traffic
docker-compose exec asterisk netstat -ulnp | grep 5060
docker-compose exec asterisk netstat -ulnp | grep 10000
```

### Step 7: Check Firewall Rules

```bash
# macOS firewall might block UDP ports
# Check System Preferences > Security & Privacy > Firewall

# Temporarily disable firewall for testing
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setglobalstate off
```

### Step 8: Review Asterisk Logs

```bash
# Check for RTP/media errors
docker-compose logs asterisk | grep -i rtp
docker-compose logs asterisk | grep -i media
docker-compose logs asterisk | grep -i "no.*audio"

# Check messages.log
tail -f asterisk_logs/messages.log
```

## Common Audio Issues and Solutions

### Issue: No Audio in Either Direction

**Symptoms:**
- Call connects but no sound
- SDP shows container IP instead of host IP

**Solution:**
1. Verify `external_media_address` in `pjsip.conf` transport
2. Verify `media_address` in endpoint defaults
3. Remove Docker bridge networks from `local_net`
4. Restart container: `docker-compose restart asterisk`

### Issue: One-Way Audio

**Symptoms:**
- Audio works in one direction only
- RTP switching to Docker bridge IP in logs

**Solution:**
1. Check `strictrtp=no` in `rtp.conf`
2. Verify `rtp_symmetric=yes` in endpoint config
3. Ensure `direct_media=no` (forces media through Asterisk)
4. Check firewall allows bidirectional UDP traffic

### Issue: Audio Cuts Out or Choppy

**Symptoms:**
- Audio starts then stops
- Intermittent audio quality

**Solution:**
1. Check network latency: `ping 192.168.0.104`
2. Verify RTP port range is sufficient (10000-10100)
3. Check for packet loss
4. Verify no other services using RTP ports

### Issue: Registration Fails

**Symptoms:**
- SIP client shows "Registration Failed"
- Logs show "Failed to authenticate"

**Solution:**
1. Verify username/password in `pjsip.conf` matches client
2. Check auth object name matches endpoint auth reference
3. Verify endpoint context matches dialplan context
4. Check SIP client is using correct server IP and port

## Useful Commands

```bash
# Reload PJSIP configuration
docker-compose exec asterisk asterisk -rx "pjsip reload"

# Show all PJSIP endpoints
docker-compose exec asterisk asterisk -rx "pjsip show endpoints"

# Show active channels
docker-compose exec asterisk asterisk -rx "core show channels"

# Show RTP statistics
docker-compose exec asterisk asterisk -rx "rtp show stats"

# Enable verbose debugging
docker-compose exec asterisk asterisk -rx "core set verbose 5"
docker-compose exec asterisk asterisk -rx "core set debug 5"

# View CDR records
cat asterisk_logs/cdr-csv/Master.csv
```

## Stopping and Cleanup

```bash
# Stop Asterisk
docker-compose down

# Stop and remove volumes (keeps configs)
docker-compose down -v

# View logs
docker-compose logs asterisk
```

## File Structure

```
asterisk_setup/
├── docker-compose.yml          # Docker service definition
├── asterisk_conf/              # Asterisk configuration files
│   ├── pjsip.conf             # PJSIP transport and endpoints
│   ├── extensions.conf        # Dialplan
│   ├── rtp.conf               # RTP media settings
│   └── asterisk.conf          # Core settings
├── asterisk_conf_sounds/      # Sound files (GSM format)
└── asterisk_logs/             # Log files and CDR
    ├── messages.log           # Asterisk messages
    └── cdr-csv/               # Call Detail Records
```

## Notes

- This setup uses Docker bridge networking (not host mode) for better isolation
- RTP ports 10000-10100 must be accessible from your network
- The configuration is optimized for local network use
- For production use, consider security hardening and proper firewall rules

## Troubleshooting Quick Reference

| Symptom | Check | Fix |
|---------|-------|-----|
| No registration | Credentials, firewall | Verify pjsip.conf auth |
| Call connects, no audio | SDP IP address | Fix external_media_address |
| One-way audio | RTP source address | Check strictrtp, rtp_symmetric |
| Choppy audio | Network, ports | Check latency, port range |
| Container won't start | Port conflicts | Check if ports in use |

## Support

For issues:
1. Check logs: `docker-compose logs asterisk`
2. Review this diagnosis process
3. Verify all IP addresses match your network
4. Ensure firewall allows required ports
