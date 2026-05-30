---
title: "Asterisk PJSIP and Yealink SIP phone voice control"
related:
  - docs: /voice_control/voice_remote_cloud_assistant/
    title: Creating a Cloud assistant
  - docs: /voice_control/voice_remote_local_assistant/
    title: Creating a local assistant
  - docs: /voice_control/assist_create_open_ai_personality/
    title: Creating an assistant personality with AI
  - url: https://www.yealink.com/en/product/ip-phone-t42s
    title: Yealink T42S IP Phone
  - url: https://www.asterisk.org
    title: Asterisk PJSIP Documentation
---

This tutorial will guide you to set up a modern SIP phone to work with Home Assistant using Asterisk as a PJSIP bridge. Pick up the phone to talk to your smart home and issue commands and get responses.

## Required material

- Home Assistant 2023.5 or later, installed with Home Assistant Operating System. If you do not have Home Assistant installed yet, refer to the [installation page](/installation/) for instructions.
- A Yealink SIP phone (T42S or similar model)
- An Asterisk server (22.x or later) with PJSIP module support, connected to your network. I used https://community-scripts.org/categories?category=miscellaneous&preview=asterisk
- Network connectivity between all devices on port 5060 (SIP)
- [Cloud assistant pipeline](/voice_control/voice_remote_cloud_assistant/) or a manually configured [local assistant pipeline](/voice_control/voice_remote_local_assistant/)

## Architecture Overview

This setup uses three components:

1. **Yealink SIP Phone** - A modern digital SIP endpoint registered with Asterisk
2. **Asterisk PJSIP Server** - Acts as a SIP bridge/proxy between the phone and Home Assistant
3. **Home Assistant** - Receives the SIP call and handles voice commands via Assist

When you pick up the Yealink handset, it automatically dials a hotline extension which routes to Home Assistant.

## Setting up Asterisk

### Prerequisites

- Asterisk 22.x or later installed with PJSIP modules
- PJSIP modules loaded: `res_pjsip`, `chan_pjsip`, `res_pjsip_registrar`, `res_pjsip_outbound_registration`
- SSH access to the Asterisk server

### Get the Configuration Files

This setup requires two configuration files. You can obtain them from the [Yealink Home Assistant Setup Guide](https://github.com/jaydenthorup/yealink_ha_setup_guide) repository:

1. Clone or download the repository:
   ```bash
   git clone https://github.com/jaydenthorup/yealink_ha_setup_guide.git
   cd yealink_ha_setup_guide
   ```

2. You'll find:
   - `configs/pjsip.conf` - PJSIP endpoint and transport configuration
   - `configs/extensions.conf` - Dialplan routing configuration

These files are pre-configured and ready to use. You only need to update the IP addresses and credentials for your environment.

### Deploy the Configuration

1. Back up your existing Asterisk configs:
   ```bash
   cp /etc/asterisk/pjsip.conf /etc/asterisk/pjsip.conf.backup
   cp /etc/asterisk/extensions.conf /etc/asterisk/extensions.conf.backup
   ```

2. Deploy the provided `pjsip.conf` and `extensions.conf` files to `/etc/asterisk/`

3. Update the Home Assistant IP address in `pjsip.conf`:
   - Find the `[homeassistant]` AOR section
   - Change `contact=sip:192.168.2.245:5060` to your Home Assistant IP
   - Find the `[homeassistant_identify]` section
   - Change `match=192.168.2.245` to your Home Assistant IP

4. Update SIP credentials in `[yealink_auth]` section if desired:
   - `username` - SIP username (default: `yealink`)
   - `password` - SIP password (default: `password123`)

5. Reload Asterisk configuration:
   ```bash
   asterisk -rx 'core reload'
   ```
   Or restart Asterisk completely:
   ```bash
   systemctl restart asterisk
   ```

### Verify Asterisk is Ready

Check that PJSIP modules are loaded:
```bash
asterisk -rx 'module show like pjsip'
```

You should see:
- `res_pjsip.so`
- `chan_pjsip.so`
- `res_pjsip_registrar.so`

## Setting up the Yealink Phone

### Access the Phone Configuration

1. Open a web browser and navigate to the phone's IP address: `http://<phone-ip>`
2. Log in with default credentials:
   - Username: `admin`
   - Password: `admin`

### Configure SIP Account

1. Go to **Account** → **Account 1**
2. Set the following values:
   - **Label**: `Home Assistant`
   - **Display Name**: `Home Assistant`
   - **Server Address**: Your Asterisk server IP (e.g., `192.168.1.100`)
   - **SIP User ID**: `yealink` (or your chosen username)
   - **Authenticate User ID**: `yealink`
   - **Authenticate Password**: `password123` (or your chosen password)
   - **Enable this account**: Yes
3. Click **Confirm** to save

The phone should register with Asterisk and display **Registered** status.

### Configure Hotline (Auto-dial)

1. Go to **Feature** → **General Settings** (menu path may vary by firmware)
2. Find the **Hotline** section:
   - **Hotline Number**: `100` (matches the extension in `extensions.conf`)
   - **Hotline Delay**: `4` seconds
3. Click **Confirm**

Now when you pick up the handset, it will automatically dial Home Assistant after 4 seconds.

## Setting up Home Assistant

### Add Voice over IP Integration

1. In Home Assistant, go to {% my config_flow_start domain="voip" title="**Settings** > **Devices & services** > **Add integration**" %}
2. Select **Voice over IP** from the list
3. The integration will create a SIP endpoint listening on port 5060

### Allow Calls from Your Phone

1. In the **Voice over IP** integration, select the **device** link
2. Under **Configuration**, enable **Allow calls** for your phone
3. (Optional) Select a specific assistant or personality to use for voice commands

### Test the Connection

1. Pick up the Yealink phone handset
2. Wait 4 seconds for auto-dial to engage
3. You should hear: *"This is your smart home speaking. Your phone is connected, but you must configure it within Home Assistant."*
4. If you hear voice output, the connection is working
5. Give a voice command, such as:
   - "Turn on the living room light"
   - "What is the temperature?"
   - "Open the garage door"

### Troubleshooting Connection Issues

**No voice heard when picking up phone:**

1. Verify the phone shows **Registered** status in its web interface
2. Check that the Asterisk server can reach Home Assistant: `ping <ha-ip>`
3. Verify Home Assistant SIP port is listening: Check the Voice over IP integration status in HA
4. Check Asterisk logs for SIP errors: `tail -f /var/log/asterisk/messages.log`
5. Restart Asterisk: `systemctl restart asterisk`

**Phone won't register:**

1. Check the SIP credentials match in both phone and `pjsip.conf`
2. Verify network connectivity between phone and Asterisk server
3. Check if firewall is blocking port 5060
4. Review Asterisk logs for authentication errors

**Call connects but audio is one-way:**

1. Verify `direct_media=no` in `pjsip.conf` (audio must route through Asterisk)
2. Check codec support: Both Yealink and Asterisk should support OPUS or ULAW
3. Verify RTP ports are not blocked by firewall (typically UDP 10000-20000)

## Give Your Voice Assistant Personality Using the OpenAI Integration

To add a custom personality to your voice assistant:

1. [Create an OpenAI personality](/voice_control/assist_create_open_ai_personality/)
2. In the **Voice over IP** integration, under **Configuration**, select the assistant you created
3. Pick up your phone and interact with your customized assistant

## Advanced Configuration

### Add Additional Phone Extensions

To add more SIP phones or extensions:

1. Create new endpoint sections in `pjsip.conf`:
   ```ini
   [phone2]
   type=endpoint
   context=from_yealink
   auth=phone2_auth
   aors=phone2
   ...
   
   [phone2_auth]
   type=auth
   username=phone2
   password=mypassword
   
   [phone2]
   type=aor
   max_contacts=1
   ```

2. Configure the new phone with its username and password

3. Reload Asterisk: `asterisk -rx 'core reload'`

### Customize Call Routing

Edit `extensions.conf` to change how calls are routed:

- Extension `100` - Hotline to Home Assistant (can be changed)
- Extension `s` - Offhook auto-dial routing (can be disabled)
- Extension `_X.` - Catch-all for other dialed extensions

### Change Hotline Delay

Adjust the delay before auto-dial in the phone's Hotline settings (typically 0-30 seconds). A 0-second delay is recommended.

## Advantages of This Approach

- **Modern SIP phones**: Works with any SIP-compatible phone (not limited to analog adapters)
- **Flexible**: Can route calls to different Home Assistant instances or other SIP services
- **Scalable**: Add multiple phones by creating new endpoints
- **No special hardware**: Uses standard SIP protocol
- **Open source**: Asterisk PJSIP is free and widely supported

## Comparison to Analog Phone Setups

**vs. Grandstream Analog Adapter:**
- Asterisk/Yealink uses modern SIP protocol vs. proprietary analog adapter
- More flexible call routing and customization
- Better audio quality with modern codecs (OPUS support)
- Can integrate with other SIP services

**vs. Home Assistant SIP directly:**
- Asterisk acts as a bridge, allowing multiple phones to register
- Better control over call routing and extension management
- Easier to add advanced features (IVR, call recording, etc.)

## Next Steps

- Explore [Home Assistant Assist](/voice_control/) for command configuration
- Set up [routines and automations](/automations/) triggered by voice commands
- Consider adding [multiple phones](/docs/configuration/voice_control/asterisk-pjsip-yealink-voice-assistant/#add-additional-phone-extensions)
- Review [Asterisk documentation](https://docs.asterisk.org) for advanced features

## Getting Help

If you encounter issues:

1. Check the [Troubleshooting](#troubleshooting-connection-issues) section above
2. Review Asterisk logs: `/var/log/asterisk/messages.log`
3. Test SIP connectivity: `asterisk -rx 'pjsip show contacts'`
4. Visit the [Home Assistant Community](https://community.home-assistant.io/) for support
5. For detailed configuration documentation, see the [Yealink Home Assistant Setup Guide](https://github.com/jaydenthorup/yealink_ha_setup_guide)

## References

- **Repository with configuration files**: https://github.com/jaydenthorup/yealink_ha_setup_guide
- **Asterisk PJSIP Documentation**: https://docs.asterisk.org/Configuration/Channel-Drivers/SIP/Configuring-res_pjsip/
- **Home Assistant Voice Control**: https://www.home-assistant.io/voice_control/
- **Yealink T42S Documentation**: https://support.yealink.com/
