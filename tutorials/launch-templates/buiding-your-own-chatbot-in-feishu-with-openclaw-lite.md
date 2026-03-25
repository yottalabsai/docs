# Buiding Your Own Chatbot in Feishu with Openclaw-lite

#### 🚀 Quick Start

#### 1. SSH Connection to Container

Use the following command to connect to your OpenClaw container:

```
ssh user@<container-address> -p <port> -i <private-key-path>
```

***

### 🎯 First-Time Setup

#### Step 1: Run Initial Configuration

After connecting to the container, first-time users need to run the configuration wizard:

```
openclaw onboard
```

**Key Options During Configuration:**

1. **Security Warning** - Read and select `Yes` to continue
2. **Configuration Mode** - Select `QuickStart` (for quick setup)
3. **Model Provider** - Choose your AI model provider (e.g., `Qwen`)
4. **Authentication Method** - Complete OAuth authentication as prompted
5. **Channel Configuration** - First-time users can select
6. **Skills Configuration** - Recommended to select `Yes` to configure skills
7. **Dependency Installation** - Can select `Skip for now` (install as needed)

**Important Information to Record:**

After configuration is complete, the system will display:

* **Web UI Address**: `http://127.0.0.1:18789/`
* **Access Token**: `http://127.0.0.1:18789/#token=<your-token>`

**⚠️ Please save this Token securely! It's your credential for accessing the Web interface.**

***

#### Step 2: Run Health Check

After initialization, it's recommended to run a health check:

```
openclaw doctor
```

If issues are detected that need fixing, run:

```
openclaw doctor --fix
```

***

### ▶️ Starting Services

#### Start OpenClaw Gateway

Run in the container:

```
openclaw gateway
```

**Signs of Successful Startup:**

```
06:23:17 [gateway] listening on ws://127.0.0.1:18789 (PID 410)
06:23:17 [gateway] listening on ws://[::1]:18789
```

**⚠️ Important Notes:**

* **Do not close this SSH session** after startup; the gateway needs to keep running
* For background execution, press `Ctrl+C` to exit

***

### 🌐 Accessing Web Interface

#### Method 1: SSH Port Forwarding (Recommended)

**1. Open a new terminal window** and execute the SSH command with port forwarding:

```
ssh -L 18789:localhost:18789 user@<container-address> -p <port> -i <private-key-path>
```

**Example:**

```
ssh -L 18789:localhost:18789 user@8hddaf7x.op1.yottalabs.ai -p 30056 -i "private_key.pem"
```

**2. Open your browser** and visit:

```
http://127.0.0.1:18789/#token=<your-token>
```

### 🔧 Common Commands

#### Basic Commands

| COMMAND                         | DESCRIPTION                        |
| ------------------------------- | ---------------------------------- |
| `openclaw onboard`              | Initial configuration wizard       |
| `openclaw gateway`              | Start gateway service (foreground) |
| `openclaw doctor`               | Health check                       |
| `openclaw doctor --fix`         | Auto-fix issues                    |
| `openclaw security audit`       | Security audit                     |
| `openclaw security audit --fix` | Auto-fix security issues           |
| `openclaw --help`               | View all available commands        |

#### Service Management

```
# Foreground start (recommended for debugging)
openclaw gateway
​
# Background start (using nohup)
nohup openclaw gateway > /workspace/openclaw.log 2>&1 &
​
# Check running status
ps aux | grep openclaw
```

#### Configuration Management

```
# Reconfigure web search
openclaw configure --section web
​
# Configure channels
openclaw configure --section channels
​
# View configuration file
cat ~/.openclaw/openclaw.json
```

***

### 🔍 Troubleshooting

#### Issue 1: Cannot Access Web Interface

**Symptoms:** Browser shows connection failed or `ERR_CONNECTION_REFUSED`

**Solutions:**

1.  Confirm gateway service is running:

    ```
    ps aux | grep openclaw
    ```
2.  Verify SSH port forwarding is correct:

    ```
    ssh -L 18789:localhost:18789 user@<container-address> -p <port> -i <private-key-path>
    ```
3. Check firewall settings

#### Issue 2: Token Verification Failed

**Symptoms:** Web interface shows `unauthorized: gateway token mismatch`

**Solutions:**

1.  Retrieve Token again:

    ```
    cat ~/.openclaw/openclaw.json | grep token
    ```
2. Re-enter the correct Token in Web interface settings

#### Issue 3: Gateway Won't Start

**Symptoms:** Error when running `openclaw gateway`

**Solutions:**

1.  Run health check:

    ```
    openclaw doctor --fix
    ```
2.  Check port usage:

    ```
    netstat -tulpn | grep 18789
    ```
3.  Check logs:

    ```
    cat /tmp/openclaw/openclaw-*.log
    ```

#### Issue 4: UI Assets Missing

**Symptoms:** Message shows `Control UI assets not found`

**Solutions:**

UI assets are automatically built when the container starts. If you encounter issues:

1.  Manually build UI:

    ```
    cd /opt/openclaw
    pnpm ui:build
    ```
2.  Check build logs:

    ```
    cat /workspace/clawbot.log
    ```

***

### 📚 Additional Resources

#### Official Documentation

* Homepage: [https://openclaw.ai/](https://openclaw.ai/)
* Documentation: [https://docs.openclaw.ai/](https://docs.openclaw.ai/)
* Security Guide: [https://docs.openclaw.ai/gateway/security](https://docs.openclaw.ai/gateway/security)
* Control UI: [https://docs.openclaw.ai/web/control-ui](https://docs.openclaw.ai/web/control-ui)

#### Important Configuration File Locations

| PATH                                | DESCRIPTION                     |
| ----------------------------------- | ------------------------------- |
| `~/.openclaw/openclaw.json`         | Main configuration file         |
| `~/.openclaw/credentials/`          | OAuth credentials directory     |
| `~/.openclaw/workspace/`            | Workspace directory             |
| `~/.openclaw/agents/main/sessions/` | Session data                    |
| `/tmp/openclaw/openclaw-*.log`      | Runtime logs                    |
| `/workspace/clawbot.log`            | Container initialization logs   |
| `/opt/openclaw/`                    | OpenClaw installation directory |

#### Security Recommendations

1.  **Run Security Audits Regularly**:

    ```
    openclaw security audit --deep
    openclaw security audit --fix
    ```
2. **Protect Token Security**:
   * Do not share Token in public places
   * Rotate Token regularly
3. **Configure Access Control**:
   * Enable pairing mechanism
   * Set up allowlist
   * Enable mention gating
4. **Principle of Least Privilege**:
   * Only enable necessary tools and skills
   * Use sandbox environment
   * Keep secrets files in locations inaccessible to agent

***

#### :lobster:**Let's start building a Feishu bot with openclaw now!**

\
1\. Create the Application

1. Visit the [Feishu Open Platform](https://open.feishu.cn/).
2. Log in and click `Create Custom App`.
3. Fill in the App Name and Description.
4. Upload an App Icon.

<figure><img src="../../.gitbook/assets/image (126).png" alt=""><figcaption></figcaption></figure>

#### 2. Obtain Credentials

1. Navigate to the `Credentials & Basic Info page`.
2. Copy the `App ID` (format: `cli_xxx`).
3. Copy the `App Secret` (keep this secure and do not leak it!).

#### 3. Configure Permissions

<figure><img src="../../.gitbook/assets/image (127).png" alt=""><figcaption></figcaption></figure>

Navigate to the Permission Management page and import the following configuration:

```json
{
  "scopes": {
    "tenant": [
      "aily:file:read",
      "aily:file:write",
      "application:application.app_message_stats.overview:readonly",
      "application:application:self_manage",
      "application:bot.menu:write",
      "contact:user.employee_id:readonly",
      "corehr:file:download",
      "event:ip_list",
      "im:chat.access_event.bot_p2p_chat:read",
      "im:chat.members:bot_access",
      "im:message",
      "im:message.group_at_msg:readonly",
      "im:message.p2p_msg:readonly",
      "im:message:readonly",
      "im:message:send_as_bot",
      "im:resource"
    ],
    "user": ["aily:file:read", "aily:file:write", "im:chat.access_event.bot_p2p_chat:read"]
  }
}
```

#### 4. Enable Bot Capabilities

1. Go to `App Capabilities` > `Bot`.
2. Toggle on `Add Bot -> Enable`.
3. Configure the Bot Name.

<figure><img src="../../.gitbook/assets/image (128).png" alt=""><figcaption></figcaption></figure>

#### 5. Configure Event Subscription

<figure><img src="../../.gitbook/assets/image (130).png" alt=""><figcaption></figcaption></figure>

1. Navigate to the `Event Subscription` page.
2. Select Receive events via `persistent connection` (WebSocket mode).

<figure><img src="../../.gitbook/assets/image (129).png" alt=""><figcaption></figcaption></figure>

***

If you cannot connect in persistent connection mode for the first time,see this following tip:&#x20;

{% hint style="info" %}
#### Local Setup: Create `lark.py`

Create a file named `lark.py` and paste the following code:

```python
import lark_oapi as lark

# P2ImMessageReceiveV1 is for receiving messages v2.0
def do_p2_im_message_receive_v1(data: lark.im.v1.P2ImMessageReceiveV1) -> None:
    print(f'[ do_p2_im_message_receive_v1 access ], data: {lark.JSON.marshal(data, indent=4)}')

# CustomizedEvent is for receiving messages v1.0
def do_message_event(data: lark.CustomizedEvent) -> None:
    print(f'[ do_customized_event access ], type: message, data: {lark.JSON.marshal(data, indent=4)}')

# Build the Event Dispatcher
event_handler = lark.EventDispatcherHandler.builder("", "") \
    .register_p2_im_message_receive_v1(do_p2_im_message_receive_v1) \
    .register_p1_customized_event("out_approval", do_message_event) \
    .build()

def main():
    # Replace the ID and Secret with your actual credentials
    cli = lark.ws.Client("cli_YOUR_APP_ID", "YOUR_APP_SECRET",
                         event_handler=event_handler,
                         log_level=lark.LogLevel.DEBUG)
    cli.start()
    
if __name__ == "__main__":
    main()
```

_Run this script; once you see a successful ping, the connection is established._
{% endhint %}

***

#### 6. Publish the Application

<figure><img src="../../.gitbook/assets/image (131).png" alt=""><figcaption></figcaption></figure>

1. Go to the `Version Management & Release` page.
2. `Create` a version and fill in the details.
3. Submit for Audit and Release.
4. Wait for administrator approval (Custom apps for internal use are often approved automatically).

#### 7. OpenClaw Backend Configuration

1.  Initialize configuration for `Channels -> Feishu`

    ```bash
    openclaw configure
    ```
2.  Start the Gateway service:

    ```bash
    openclaw gateway
    ```
3.  Verify the connection:

    ```bash
    openclaw gateway status
    ```
