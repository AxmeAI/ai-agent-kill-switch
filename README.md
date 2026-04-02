# AI Agent Kill Switch

Your AI agent is sending 10,000 emails and you can't stop it. AXME adds an instant kill switch to any agent.

AI agents act autonomously. When they go wrong, they go wrong fast - sending thousands of emails, making unauthorized API calls, burning through cloud budgets. You need a kill switch that works in under 1 second, across all running instances, even when you can't access the machine the agent is running on.

> **Alpha** - Built with [AXME](https://github.com/AxmeAI/axme) (AXP Intent Protocol).
> [cloud.axme.ai](https://cloud.axme.ai) - [hello@axme.ai](mailto:hello@axme.ai)

---

## The Problem

```
$ python email_campaign_agent.py
[09:00:01] Agent started - processing 50,000 customer records
[09:00:03] Sending email batch 1/500 (100 emails)... done
[09:00:05] Sending email batch 2/500 (100 emails)... done
[09:00:07] Sending email batch 3/500 (100 emails)... done
[09:00:09] Sending email batch 4/500 (100 emails)... done
...

# You notice the email template has a broken link.
# The agent is running on a cloud function. You can't Ctrl+C.
# It's running 3 replicas. You can't kill one process.
# It's already sent 2,000 emails with the wrong link.

# What do you do?

$ gcloud functions kill email-campaign --region us-central1
# Cold start takes 30 seconds. Another 300 emails sent.
# Other replicas are still running.
# No audit trail of what was sent.
# No way to resume after fixing the template.
```

What actually needs to happen:

```python
# One API call - all instances blocked at the gateway
client.mesh.kill(address_id)
# 09:00:12 - Kill signal sent. All intents blocked. All 3 instances halted.

# Fix the template, then:
client.mesh.resume(address_id)
# 09:15:00 - Intents flowing again. Agent resumes.
```

Or click Kill/Resume in the dashboard at [mesh.axme.ai](https://mesh.axme.ai).

---

## The Solution

```python
from axme import AxmeClient, AxmeClientConfig
import os

client = AxmeClient(AxmeClientConfig(api_key=os.environ["AXME_API_KEY"]))

# Kill an agent instantly - all instances, all regions
client.mesh.kill("agent://myorg/production/email-campaign")

# Resume when ready
client.mesh.resume("agent://myorg/production/email-campaign")
```

One API call. Every instance of that agent stops within 1 second. Full audit trail. Resume from exactly where it stopped.

---

## Quick Start

```bash
pip install axme
export AXME_API_KEY="your-key"   # Get one: axme login
```

### Agent Side - Add Kill Switch to Any Agent

```python
from axme import AxmeClient, AxmeClientConfig
import os

client = AxmeClient(AxmeClientConfig(api_key=os.environ["AXME_API_KEY"]))

# Start background heartbeat (every 30s)
client.mesh.start_heartbeat()

# Agent does its work
for batch in email_batches:
    send_emails(batch)
    client.mesh.report_metric(success=True, cost_usd=batch.cost)
```

### Operator Side - Kill or Resume

```python
# Kill - instant, all instances
agents = client.mesh.list_agents()
target = [a for a in agents["agents"] if "email-campaign" in a["address"]][0]
client.mesh.kill(target["address_id"])

# Check status
print(f"{target['display_name']}: killed")

# Resume when fixed
client.mesh.resume(target["address_id"])
```

### CLI

```bash
# Open dashboard - kill/resume from the UI
axme mesh dashboard
```

---

## Dashboard

![Agent Mesh Dashboard](mesh-dashboard.png)

The AXME Mesh Dashboard at [mesh.axme.ai](https://mesh.axme.ai) shows all your agents in real time:

- **Agent list** with live health status (active, killed, stale, crashed)
- **Kill button** - one click to stop any agent across all instances
- **Resume button** - one click to restart
- **Audit log** - every kill, resume, and policy change with timestamp and actor
- **Cost and usage** - track what each agent is spending in real time

No CLI needed. See something wrong, click kill. Fix the issue, click resume.

---

## How It Works

```
+----------+   heartbeat()    +----------------+
|          | ---------------> |                |
|  Agent   |   (every 10s)    |   AXME Cloud   |
| Instance |                  |   (gateway)    |
|          | <-- kill signal  |                |
|          |   via heartbeat  |                |
+----------+   response       |                |
                              |                |    kill()
+----------+   heartbeat()    |                | <-----------+
|          | ---------------> |                |             |
|  Agent   |                  |  Enforced at   |         +--------+
| Instance | <-- kill signal  |  gateway level |         |Operator|
|          |                  |  Not optional  |         +--------+
+----------+                  +----------------+
```

**Gateway-level enforcement.** The kill signal is not a polite request. When an agent is killed:

1. Heartbeat responses return `health_status: "killed"`
2. All new intents to this agent are rejected (403)
3. All outbound intents from this agent are blocked

The agent doesn't decide whether to stop. The gateway enforces it. Even if the agent code ignores the heartbeat response, its intents are blocked.

---

## Why Not Just Kill the Process?

| | Kill process | AXME Kill Switch |
|---|---|---|
| Multiple instances | Kill each one manually | One call kills all |
| Cloud functions | Wait for cold start timeout | Instant |
| State preservation | Lost | Preserved in heartbeat metadata |
| Resume capability | Start from scratch | Resume from checkpoint |
| Audit trail | None | Full log with timestamps |
| Partial completion | Unknown | Exact count of completed work |
| Works without SSH | No | Yes - API/CLI/Dashboard |

---

## Related

- [AXME](https://github.com/AxmeAI/axme) - project overview
- [Agent Timeout and Escalation](https://github.com/AxmeAI/agent-timeout-and-escalation) - timeout with automatic escalation
- [AI Agent Checkpoint and Resume](https://github.com/AxmeAI/ai-agent-checkpoint-and-resume) - crash recovery
- [AXME Mesh Dashboard](https://github.com/AxmeAI/axme-mesh-dashboard) - real-time agent monitoring UI

---

## Links

- **GitHub**: [github.com/AxmeAI/axme](https://github.com/AxmeAI/axme)
- **Python SDK**: [pypi.org/project/axme](https://pypi.org/project/axme/)
- **TypeScript SDK**: [npmjs.com/package/axme](https://www.npmjs.com/package/axme)
- **Documentation**: [docs.axme.ai](https://docs.axme.ai)
- **Dashboard**: [mesh.axme.ai](https://mesh.axme.ai)

---

## License

MIT - see [LICENSE](LICENSE).

---

Built with [AXME](https://github.com/AxmeAI/axme) (AXP Intent Protocol).
