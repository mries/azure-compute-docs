---
title: Create Linux images without a provisioning agent
description: Create generalized Linux images without a provisioning agent in Azure.
author: vamckMS
ms.service: azure-virtual-machines
ms.subservice: imaging
ms.collection: linux
ms.topic: how-to
ms.custom: devx-track-azurecli, linux-related-content
ms.date: 03/01/2026
ms.author: vakavuru
ms.reviewer: mattmcinnes
# Customer intent: As a cloud architect, I want to create generalized Linux VM images without using a provisioning agent, so that I can customize the image configuration and ensure compatibility with specific Linux distributions during deployment.
---


# Create generalized Linux images without a provisioning agent

**Applies to:** :heavy_check_mark: Linux VMs :heavy_check_mark: Flexible scale sets

When a Linux VM boots in Azure, the platform expects the operating system to:

1. Acquire network configuration using DHCP  
2. Retrieve provisioning state from the Azure WireServer  
3. Explicitly report when the VM is ready  

These steps are normally handled by **cloud-init** (recommended) or the **Azure Linux Agent (walinuxagent)**. In some scenarios, however, you may need to create Linux VM images that do not include either provisioning agent.

This article explains the minimum required interactions with Azure platform services to successfully provision a **generalized Linux VM image without a provisioning agent**

> [!NOTE]
> For more information about supported provisioning agents, see the documentation for [walinuxagent](https://github.com/Azure/WALinuxAgent) and [cloud-init](https://github.com/canonical/cloud-init).


This approach is typically used in the following scenarios:
- Your Linux distribution or version does not support cloud-init or the Azure Linux Agent.
- You require custom VM properties to be set, such as the hostname.


This article shows how you can set up your VM image to satisfy the Azure platform requirements and set the hostname, without installing a provisioning agent.

> [!IMPORTANT]
>
> Before proceeding, determine whether you actually need a generalized image.
>
> If you do **not** require hostname changes, SSH key injection, or any first‑boot configuration, use a **specialized image** instead. Specialized images avoid provisioning logic entirely and are simpler to maintain.

## Networking and reporting ready


In order for a Linux VM to communicate with Azure platform services, a DHCP-enabled network configuration is required.  The client is used to retrieve a host IP, DNS resolution, and route management from the virtual network. Most distros ship with these utilities out-of-the-box. Tools that are tested on Azure by Linux distro vendors include `dhclient`, `network-manager`, `systemd-networkd` and others.

> [!NOTE]
>
> Generalized images created without a provisioning agent currently **require DHCP**. Static network configurations are not supported and will prevent the VM from provisioning successfully.

After networking has been set up and configured, the VM must explicitly report a "Ready" state to Azure




## Choose the right Linux image provisioning approach

| Scenario / Requirement | cloud-init (recommended) | Agentless generalized image ⚠️ **Advanced** | Specialized image |
|------------------------|--------------------------|-----------------------------|-------------------|
| Default Azure VM provisioning | ✅ Yes | ❌ No | ❌ No |
| Supported and maintained by Azure | ✅ Yes | ❌ No (self-supported) | ✅ Yes |
| Requires first-boot customization | ✅ Yes | ✅ Yes (custom logic required) | ❌ No |
| Linux distro supports cloud-init or Azure Linux Agent | ✅ Yes | ❌ No | ✅ Yes |
| Minimal OS or unsupported distro | ❌ No | ✅ Yes | ✅ Yes |
| Requires VM generalization | ✅ Yes | ✅ Yes | ❌ No |
| Ongoing maintenance effort | Low | High | Low |
| Recommended for most customers | ✅ Yes | ❌ No (advanced only) | ✅ Yes |

**Guidance:**
- Use **cloud-init** for most Linux workloads in Azure.
- Use an **agentless generalized image** only for advanced or unsupported scenarios.
- Use a **specialized image** when no provisioning logic is required.





## Configure an Image for Agentless Provisioning (Advanced Scenario)

> [!IMPORTANT]
> This scenario is provided for demonstration and educational purposes.  
> Agentless provisioning is intended for **advanced use cases** and **non-standard configurations**. Support is provided on a **best-effort basis only**, and this approach is **not recommended for general production workloads**.

The following demo illustrates the **minimum platform interactions** required to provision a Linux VM **without a provisioning agent**. It is intended for experienced users who need to understand low-level provisioning behavior or explore specialized scenarios.

This example uses an existing Marketplace image with the Azure Linux Agent removed and custom logic added solely to report provisioning readiness.


### Create the resource group and base VM:

```azurecli
$ az group create --location eastus --name demo1
```

Create the base VM:

```azurecli
$ az vm create \
    --resource-group demo1 \
    --name demo1 \
    --location eastus \
    --ssh-key-value <ssh_pub_key_path> \
    --public-ip-address-dns-name demo1 \
    --image "debian:debian-10:10:latest"
```

### Remove the image provisioning Agent

Once the VM is provisioning, you can connect to it via SSH and remove the Linux Agent:

```bash
$ sudo apt purge -y waagent
$ sudo rm -rf /var/lib/waagent /etc/waagent.conf /var/log/waagent.log
```

### Add required code to the VM

Also inside the VM, because we've removed the Azure Linux Agent we need to provide a mechanism to report ready.

#### Python script

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

"""
Azure provisioning helper:
- Compute and set a clean hostname (no CLI override).
- Remove legacy GUID suffix (any hyphen/space variant) if present.
- Persist hostname robustly (hostnamectl with retry; fallback to /bin/hostname + /etc/hostname).
- Ensure /etc/hosts 127.0.1.1 mapping.
- Optionally fetch GoalState from Wireserver and POST Ready health (best-effort).
- Write detailed logs to /var/log/azure-provisioning.log
"""

import http.client
import json
import logging
import os
import re
import shutil
import socket
import subprocess
import sys
import time
from typing import Optional, Tuple
from urllib.request import Request, urlopen
from urllib.error import URLError, HTTPError
import xml.etree.ElementTree as ET

# -----------------------------
# Configuration
# -----------------------------
LOGFILE = "/var/log/azure-provisioning.log"

# Azure Wireserver (fabric)
WIRESERVER_IP = "168.63.129.16"
GOALSTATE_PATH = "/machine?comp=goalstate"
HEALTH_PATH = "/machine?comp=health"
WS_HEADERS = {
    "x-ms-version": "2012-11-30",
    "x-ms-agent-name": "custom-provisioning"
    # Content-Type set at request time
}

# Azure IMDS (instance metadata)
IMDS_URL = "http://169.254.169.254/metadata/instance/compute?api-version=2021-08-01&format=json"
IMDS_HEADERS = {"Metadata": "true"}

# Legacy GUID (remove when suffix of hostname). Accepts hyphen/space/no-separator variants.
LEGACY_GUID_CANON = "82bc1ec8-d567-4e3f-903d-a30ef1981555"
# Match end-of-string in flexible ways (hyphens/spaces optional):
LEGACY_GUID_STRIP_REGEX = re.compile(
    r"(?i)[-\s]*82bc1ec8[-\s]*d567[-\s]*4e3f[-\s]*903d[-\s]*a30ef1981555\s*$"
)

# Hostname retry/backoff when systemd DBus not yet ready
HOSTNAMECTL_RETRIES = 6
HOSTNAMECTL_BASE_SLEEP = 2.0  # seconds

# -----------------------------
# Logging
# -----------------------------
os.makedirs(os.path.dirname(LOGFILE), exist_ok=True)
logging.basicConfig(
    filename=LOGFILE,
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(message)s"
)
log = logging.getLogger("azure-provisioning")


# -----------------------------
# Helpers
# -----------------------------
def run(cmd: list[str]) -> Tuple[int, str, str]:
    """Run a command, return (rc, stdout, stderr)."""
    try:
        p = subprocess.run(cmd, check=False, capture_output=True, text=True)
        return p.returncode, p.stdout.strip(), p.stderr.strip()
    except Exception as e:
        log.exception("Command raised exception: %s", cmd)
        return 1, "", str(e)


def sanitize_hostname(name: str) -> str:
    """
    RFC 952/1123 single-label sanitization:
    - Lowercase, keep only [a-z0-9-]
    - Collapse multiple hyphens
    - Trim leading/trailing hyphens
    - No dots; max 63 chars
    """
    if not name:
        return "linux-vm"
    h = name.lower()
    # Replace any non [a-z0-9-] (including '.') with hyphen
    h = re.sub(r"[^a-z0-9-]", "-", h)
    # Collapse multiple hyphens
    h = re.sub(r"-{2,}", "-", h)
    # Trim leading/trailing hyphens
    h = h.strip("-")
    if not h:
        h = "linux-vm"
    if len(h) > 63:
        h = h[:63]
    return h


def strip_legacy_guid_suffix(name: str) -> str:
    """
    Remove the legacy GUID suffix (accepting hyphen/space/no-separator variants) if present at end.
    """
    if not name:
        return name
    stripped = LEGACY_GUID_STRIP_REGEX.sub("", name)
    # Clean any trailing spaces/hyphens left by the removal
    stripped = stripped.rstrip(" -")
    return stripped


def get_imds_vm_name(timeout: float = 2.0) -> Optional[str]:
    """Fetch compute.name from IMDS; return None if unavailable."""
    try:
        req = Request(IMDS_URL, headers=IMDS_HEADERS)
        with urlopen(req, timeout=timeout) as r:
            if r.status != 200:
                log.warning("IMDS HTTP %s", r.status)
                return None
            data = json.loads(r.read().decode("utf-8", errors="ignore"))
        # Field can be 'name' or 'computerName' depending on API version/behavior
        vm_name = data.get("name") or data.get("computerName")
        if not vm_name:
            log.warning("IMDS returned no name field")
        return vm_name
    except (URLError, HTTPError, TimeoutError) as e:
        log.warning("IMDS not reachable: %s", e)
        return None
    except Exception as e:
        log.exception("IMDS unexpected error: %s", e)
        return None


def write_etc_hostname(hostname: str) -> None:
    """Write /etc/hostname."""
    with open("/etc/hostname", "w", encoding="utf-8") as f:
        f.write(hostname + "\n")


def ensure_hosts_mapping(hostname: str) -> None:
    """
    Ensure /etc/hosts includes:
      127.0.1.1  <short> <full>
    Keep '127.0.0.1 localhost' intact and remove previous 127.0.1.1 lines.
    """
    try:
        short = hostname.split(".")[0]
        alias_line = f"127.0.1.1\t{short} {hostname}\n"

        path = "/etc/hosts"
        if os.path.exists(path):
            with open(path, "r", encoding="utf-8") as f:
                lines = f.readlines()
        else:
            lines = ["127.0.0.1\tlocalhost\n"]

        kept: list[str] = []
        for ln in lines:
            s = ln.strip()
            # Keep localhost and all others except existing 127.0.1.1 mappings
            if s.startswith("127.0.1.1"):
                continue
            kept.append(ln)

        kept.append(alias_line)
        with open(path, "w", encoding="utf-8") as f:
            f.writelines(kept)
        log.info("Updated /etc/hosts: %s", alias_line.strip())
    except Exception as e:
        log.warning("Failed to update /etc/hosts: %s", e)


def persist_hostname(hostname: str) -> None:
    """
    Persist hostname robustly:
    - Try hostnamectl with retries (for early-boot DBus availability)
    - Fallback to /bin/hostname
    - Always write /etc/hostname and ensure hosts mapping
    """
    desired = hostname
    last_err = ""
    used_hostnamectl = False

    hostnamectl = shutil.which("hostnamectl")
    if hostnamectl:
        for attempt in range(1, HOSTNAMECTL_RETRIES + 1):
            rc, out, err = run([hostnamectl, "set-hostname", desired])
            if rc == 0:
                used_hostnamectl = True
                log.info("hostnamectl succeeded on attempt %d", attempt)
                break
            last_err = err or out
            # If it's a DBus timing issue, retry; otherwise no point in waiting
            if "Failed to connect to bus" in last_err or "No such file or directory" in last_err:
                sleep_s = HOSTNAMECTL_BASE_SLEEP * (2 ** (attempt - 1))
                log.warning("hostnamectl attempt %d failed: %s (retrying in %.1fs)",
                            attempt, last_err, sleep_s)
                time.sleep(sleep_s)
            else:
                log.warning("hostnamectl attempt %d failed (non-DBus): %s", attempt, last_err)
                break

    if not used_hostnamectl:
        # Fallback to /bin/hostname
        hostname_bin = shutil.which("hostname") or "/bin/hostname"
        rc, out, err = run([hostname_bin, desired])
        if rc != 0:
            raise RuntimeError(f"hostname fallback failed: rc={rc}, err={err or out or ''}")

    # Persist on disk and add /etc/hosts mapping (ignore failures gracefully)
    try:
        write_etc_hostname(desired)
    except Exception as e:
        log.exception("Failed writing /etc/hostname: %s", e)
        # do not raise; we still continue

    ensure_hosts_mapping(desired)
    log.info("Hostname persistence complete. desired=%s current=%s", desired, socket.gethostname())


def ws_get_goalstate(conn: http.client.HTTPConnection) -> Optional[ET.Element]:
    """GET goalstate XML from wireserver; return Element or None on failure."""
    try:
        conn.request("GET", GOALSTATE_PATH, headers=WS_HEADERS)
        resp = conn.getresponse()
        if resp.status != 200:
            log.error("Failed to retrieve GoalState: HTTP %s %s", resp.status, resp.reason)
            return None
        xml_text = resp.read().decode("utf-8", errors="replace")
        root = ET.fromstring(xml_text)
        log.info("GoalState retrieved successfully.")
        return root
    except ET.ParseError as e:
        log.error("Failed to parse GoalState XML: %s", e)
        return None
    except Exception as e:
        log.warning("GoalState retrieval error (best-effort): %s", e)
        return None


def ws_parse_goalstate(root: ET.Element) -> Tuple[Optional[str], Optional[str], Optional[str]]:
    """Extract (container_id, instance_id, incarnation) from GoalState XML."""
    if root is None:
        return None, None, None
    container_id = root.findtext("./Container/ContainerId")
    instance_id = root.findtext("./Container/RoleInstanceList/Role/InstanceId")
    incarnation = root.findtext("./Incarnation")
    log.info("Parsed GoalState - ContainerId=%s InstanceId=%s Incarnation=%s",
             container_id, instance_id, incarnation)
    return container_id, instance_id, incarnation


def ws_build_health_xml(container_id: str, instance_id: str, incarnation: str) -> str:
    """Build Ready Health XML payload."""
    health = ET.Element("GoalState")
    inc_el = ET.SubElement(health, "GoalStateIncarnation")
    inc_el.text = str(incarnation or "")
    cont_el = ET.SubElement(health, "Container")
    cid_el = ET.SubElement(cont_el, "ContainerId")
    cid_el.text = str(container_id or "")
    rlist_el = ET.SubElement(cont_el, "RoleInstanceList")
    role_el = ET.SubElement(rlist_el, "Role")
    iid_el = ET.SubElement(role_el, "InstanceId")
    iid_el.text = str(instance_id or "")
    h2 = ET.SubElement(health, "Health")
    st = ET.SubElement(h2, "State")
    st.text = "Ready"
    return ET.tostring(health, encoding="unicode", method="xml")


def ws_post_health(conn: http.client.HTTPConnection, xml_payload: str) -> None:
    """POST health payload to wireserver; raises on HTTP error."""
    headers = dict(WS_HEADERS)
    headers["Content-Type"] = "text/xml; charset=utf-8"
    conn.request("POST", HEALTH_PATH, body=xml_payload.encode("utf-8"), headers=headers)
    resp = conn.getresponse()
    # Drain body
    _ = resp.read()
    if resp.status >= 300:
        raise RuntimeError(f"Health POST failed: HTTP {resp.status} {resp.reason}")
    log.info("Wireserver health response: %s %s", resp.status, resp.reason)


def decide_hostname(instance_id: Optional[str]) -> str:
    """
    Build the desired hostname (no GUID append). Precedence:
    1) IMDS name
    2) GoalState instanceId
    3) current hostname
    4) 'linux-vm-<machineid_tail>' or time
    Then strip any legacy GUID suffix and sanitize.
    """
    base = get_imds_vm_name()
    if base:
        log.info("IMDS hostname: %s", base)
    if not base and instance_id:
        base = instance_id
        log.info("Using GoalState InstanceId as hostname base: %s", base)
    if not base:
        try:
            base = socket.gethostname()
            log.info("Using current hostname as base: %s", base)
        except Exception:
            base = ""

    if not base:
        # Last resort fallback on /etc/machine-id tail or epoch time
        try:
            with open("/etc/machine-id", "r", encoding="utf-8") as f:
                mid = f.read().strip()
            tail = mid[-8:] if mid else str(int(time.time()))
        except Exception:
            tail = str(int(time.time()))
        base = f"linux-vm-{tail}"
        log.info("Using fallback base: %s", base)

    # Strip legacy GUID suffix (any variant) if present
    stripped = strip_legacy_guid_suffix(base)
    if stripped != base:
        log.info("Stripped legacy GUID suffix. Before: %s  After: %s", base, stripped)

    # Sanitize to single-label hostname
    desired = sanitize_hostname(stripped)
    log.info("Sanitized hostname: %s", desired)
    return desired


def main() -> int:
    if os.geteuid() != 0:
        print("ERROR: run as root (sudo).", file=sys.stderr)
        return 1

    log.info("Azure provisioning script starting.")

    # Try GoalState (best effort)
    container_id = instance_id = incarnation = None
    try:
        conn = http.client.HTTPConnection(WIRESERVER_IP, timeout=5)
        gs_root = ws_get_goalstate(conn)
        container_id, instance_id, incarnation = ws_parse_goalstate(gs_root) if gs_root is not None else (None, None, None)
    except Exception as e:
        log.warning("Wireserver connection error (best-effort): %s", e)
        conn = None

    # Decide and persist hostname (robust)
    try:
        desired = decide_hostname(instance_id)
        current = ""
        try:
            current = socket.gethostname()
        except Exception:
            pass

        if not current or current != desired:
            log.info("Setting hostname from '%s' to '%s'", current, desired)
            persist_hostname(desired)
        else:
            log.info("Hostname already correct: %s", current)
    except Exception as e:
        log.exception("Failed to set hostname: %s", e)
        # Keep going; health post is independent

    # If we have GoalState fields, report Ready health (best effort)
    try:
        if conn and container_id and instance_id and incarnation:
            payload = ws_build_health_xml(container_id, instance_id, incarnation)
            log.info("Sending health report to Wireserver.")
            ws_post_health(conn, payload)
        else:
            log.warning("Skipping health POST (missing GoalState fields or connection).")
    except Exception as e:
        log.warning("Health POST failed (continuing): %s", e)
    finally:
        try:
            if conn:
                conn.close()
        except Exception:
            pass

    log.info("Provisioning (hostname + health) completed.")
    return 0


if __name__ == "__main__":
    try:
        sys.exit(main())
    except Exception as e:
        log.exception("Unhandled exception: %s", e)
        sys.exit(1)


```

#### Bash script

```
#!/usr/bin/env bash
set -euo pipefail

LOGFILE="/var/log/azure-provisioning.log"

IMDS_NAME_URL="http://169.254.169.254/metadata/instance/compute/name?api-version=2021-08-01&format=text"
GOALSTATE_URL="http://168.63.129.16/machine/?comp=goalstate"
HEALTH_URL="http://168.63.129.16/machine?comp=health"

MAX_ATTEMPTS=5
SLEEP_SECONDS=5

mkdir -p "$(dirname "$LOGFILE")"
exec >>"$LOGFILE" 2>&1

ts() { date -Is; }
log() { echo "[$(ts)] $*" >&2; }

sanitize_hostname() {
  local raw="${1,,}"
  raw="$(echo -n "$raw" | tr -c 'a-z0-9-' '-')"
  raw="$(sed -E 's/-+/-/g; s/^-+//; s/-+$//' <<<"$raw")"
  [[ -z "$raw" ]] && raw="linux-vm"
  echo "${raw:0:63}"
}

current_hostname() { hostname 2>/dev/null || true; }

# ---------- HTTP helper (captures body + status code) ----------
curl_with_status() {
  # usage: curl_with_status <url> [curl args...]
  local url="$1"; shift
  local out rc
  local start elapsed
  start="$(date +%s%3N)"

  set +e
  out="$(curl -sS --noproxy '*' -w $'\n%{http_code}' "$@" "$url")"
  rc=$?
  set -e

  elapsed=$(( $(date +%s%3N) - start ))

  if [[ $rc -ne 0 ]]; then
    # echo status=000 to make it obvious
    printf '%s\n' "" "000" "$elapsed" "$rc"
    return 0
  fi

  # last line is status code
  local http
  http="$(printf '%s\n' "$out" | tail -n1)"
  local body
  body="$(printf '%s\n' "$out" | sed '$d')"

  printf '%s\n' "$body" "$http" "$elapsed" "0"
}

fetch_hostname_from_imds() {
  log "IMDS: attempting to fetch VM name"
  local body http ms rc
  readarray -t r < <(curl_with_status "$IMDS_NAME_URL" -H "Metadata:true" --connect-timeout 1 --max-time 2)
  body="${r[0]}"
  http="${r[1]}"
  ms="${r[2]}"
  rc="${r[3]}"

  if [[ "$rc" != "0" ]]; then
    log "IMDS: curl failed (rc=$rc)"
    return 1
  fi
  log "IMDS: HTTP $http (${ms} ms)"

  [[ "$http" == "200" ]] || return 1
  printf '%s\n' "$body" | head -n1
}

determine_hostname() {
  local candidate
  if candidate="$(fetch_hostname_from_imds)"; then
    log "IMDS hostname: $candidate"
    printf '%s\n' "$(sanitize_hostname "$candidate")"
    return 0
  fi

  candidate="$(current_hostname)"
  if [[ -n "${candidate:-}" ]]; then
    log "Using current hostname: $candidate"
    printf '%s\n' "$(sanitize_hostname "$candidate")"
    return 0
  fi

  local mid=""
  [[ -r /etc/machine-id ]] && mid="$(tail -c 8 /etc/machine-id 2>/dev/null | tr -d '\n' || true)"
  [[ -z "$mid" ]] && mid="$(date +%s)"
  candidate="linux-vm-$mid"
  log "Fallback hostname: $candidate"
  printf '%s\n' "$(sanitize_hostname "$candidate")"
}

set_hostname_persist() {
  local hn="$1"
  local short="${hn%%.*}"

  if command -v hostnamectl >/dev/null 2>&1; then
    hostnamectl set-hostname "$hn" || { log "ERROR: hostnamectl failed"; return 1; }
  else
    echo "$hn" > /etc/hostname || { log "ERROR: writing /etc/hostname failed"; return 1; }
    command -v hostname >/dev/null 2>&1 && hostname "$hn" || true
  fi

  echo "$hn" > /etc/hostname || true

  if [[ -w /etc/hosts ]]; then
    local tmp; tmp="$(mktemp)"
    awk '{ if ($1 == "127.0.1.1") next; print }' /etc/hosts > "$tmp"
    if [[ "$hn" == *.* ]]; then
      echo "127.0.1.1 ${short} ${hn}" >> "$tmp"
    else
      echo "127.0.1.1 ${short}" >> "$tmp"
    fi
    cp "$tmp" /etc/hosts && rm -f "$tmp"
  else
    log "WARN: /etc/hosts not writable; skipping update"
  fi
}

test_wireserver_tcp() {
  log "WireServer: testing TCP connectivity to 168.63.129.16:80"
  if timeout 2 bash -c 'cat </dev/null >/dev/tcp/168.63.129.16/80' 2>/dev/null; then
    log "WireServer: TCP/80 reachable"
  else
    log "WARN: WireServer TCP/80 NOT reachable"
  fi
}

fetch_goalstate() {
  log "WireServer: GET goalstate"

  local tmp body http rc start elapsed
  tmp="$(mktemp)"

  start="$(date +%s%3N)"
  set +e
  http="$(curl -sS --noproxy '*' \
    -H "x-ms-agent-name: azure-vm-register" \
    -H "Content-Type: text/xml;charset=utf-8" \
    -H "x-ms-version: 2012-11-30" \
    --connect-timeout 2 --max-time 5 \
    -w "%{http_code}" \
    -o "$tmp" \
    "$GOALSTATE_URL")"
  rc=$?
  set -e
  elapsed=$(( $(date +%s%3N) - start ))

  if [[ $rc -ne 0 ]]; then
    log "WireServer: goalstate curl failed (curl rc=$rc)"
    rm -f "$tmp"
    return 1
  fi

  log "WireServer: goalstate HTTP $http (${elapsed} ms)"

  if [[ "$http" != "200" ]]; then
    log "WireServer: goalstate returned non-200"
    rm -f "$tmp"
    return 1
  fi

  cat "$tmp"
  rm -f "$tmp"
}

parse_goalstate_fields() {
  local xml; xml="$(cat)"

  local container incarnation instance
  container="$(grep -m1 -oE '<ContainerId>[^<]+' <<<"$xml" | sed 's/<ContainerId>//' || true)"
  incarnation="$(grep -m1 -oE '<Incarnation>[^<]+' <<<"$xml" | sed 's/<Incarnation>//' || true)"
  instance="$(awk '
    /<RoleInstanceList>/{f=1}
    f && /<InstanceId>/{gsub(/.*<InstanceId>|<\/InstanceId>.*/,""); print; exit}
  ' <<<"$xml" || true)"

  [[ -z "$instance" ]] && instance="$(grep -m1 -oE '<InstanceId>[^<]+' <<<"$xml" | sed 's/<InstanceId>//' || true)"

  log "WireServer: parsed ContainerId=${container:-<empty>}"
  log "WireServer: parsed InstanceId=${instance:-<empty>}"
  log "WireServer: parsed Incarnation=${incarnation:-<empty>}"

  [[ -n "$container" && -n "$instance" && -n "$incarnation" ]] || return 1
  printf '%s\n' "$container" "$instance" "$incarnation"
}

build_ready_xml() {
  local container_id="$1"
  local instance_id="$2"
  local incarnation="$3"

  # NOTE: terminator EOF MUST be flush-left, no spaces.
  cat <<EOF
<?xml version="1.0" encoding="utf-8"?>
<Health xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema">
  <GoalStateIncarnation>${incarnation}</GoalStateIncarnation>
  <Container>
    <ContainerId>${container_id}</ContainerId>
    <RoleInstanceList>
      <Role>
        <InstanceId>${instance_id}</InstanceId>
        <Health><State>Ready</State></Health>
      </Role>
    </RoleInstanceList>
  </Container>
</Health>
EOF
}

post_health_ready() {
  test_wireserver_tcp

  local attempt=1
  while [[ "$attempt" -le "$MAX_ATTEMPTS" ]]; do
    log "WireServer: health Ready attempt $attempt/$MAX_ATTEMPTS"

    local goalstate fields ready_xml body http ms rc
    if ! goalstate="$(fetch_goalstate)"; then
      log "WARN: goalstate fetch failed; retrying in $SLEEP_SECONDS s"
      sleep "$SLEEP_SECONDS"
      attempt=$((attempt+1))
      continue
    fi

    if ! readarray -t fields < <(printf '%s' "$goalstate" | parse_goalstate_fields); then
      log "WARN: goalstate parse failed; retrying in $SLEEP_SECONDS s"
      sleep "$SLEEP_SECONDS"
      attempt=$((attempt+1))
      continue
    fi

    ready_xml="$(build_ready_xml "${fields[0]}" "${fields[1]}" "${fields[2]}")"
    log "WireServer: POST health Ready"

    readarray -t r < <(curl_with_status "$HEALTH_URL" \
      -X POST \
      -H "x-ms-agent-name: azure-vm-register" \
      -H "Content-Type: text/xml;charset=utf-8" \
      -H "x-ms-version: 2012-11-30" \
      --connect-timeout 2 --max-time 5 \
      --data-binary "$ready_xml")
    body="${r[0]}"
    http="${r[1]}"
    ms="${r[2]}"
    rc="${r[3]}"

    if [[ "$rc" == "0" && "$http" == "200" ]]; then
      log "WireServer: health POST HTTP 200 (${ms} ms)"
      return 0
    fi

    # If it failed, dump a small snippet of response body for debugging
    if [[ -n "${body:-}" ]]; then
      log "WireServer: health response body (first 300 chars): $(printf '%s' "$body" | tr '\n' ' ' | head -c 300)"
    fi

    log "WARN: WireServer health POST failed (rc=$rc http=$http); retrying in $SLEEP_SECONDS s"
    sleep "$SLEEP_SECONDS"
    attempt=$((attempt+1))
  done

  log "ERROR: Failed to report provisioning health Ready after $MAX_ATTEMPTS attempts"
  return 1
}

main() {
  log "Azure provisioning script started"

  local desired current
  desired="$(determine_hostname | tr -d '\r\n')"
  current="$(current_hostname || true)"

  if [[ -n "$desired" && "$desired" != "$current" ]]; then
    log "Setting hostname: $current -> $desired"
    set_hostname_persist "$desired"
  else
    log "Hostname already set to: $current"
  fi

  post_health_ready
  log "Script completed successfully"
}

if [[ "${1:-}" == "--check" ]]; then
  # syntax check mode (useful in CI or manual verification)
  log "Running bash syntax check passed (this script is executing)."
  exit 0
fi

main
```

#### Generic steps (if not using Python or Bash)

If your VM doesn't have Python installed or available, you can programmatically reproduce this above script logic with the following steps:

1. Retrieve the `ContainerId`, `InstanceId`, and `Incarnation` by parsing the response from the WireServer: `curl -X GET -H 'x-ms-version: 2012-11-30' http://168.63.129.16/machine?comp=goalstate`.

2. Construct the following XML data, injecting the parsed `ContainerId`, `InstanceId`, and `Incarnation` from the above step:
   ```xml
   <Health>
     <GoalStateIncarnation>INCARNATION</GoalStateIncarnation>
     <Container>
       <ContainerId>CONTAINER_ID</ContainerId>
       <RoleInstanceList>
         <Role>
           <InstanceId>INSTANCE_ID</InstanceId>
           <Health>
             <State>Ready</State>
           </Health>
         </Role>
       </RoleInstanceList>
     </Container>
   </Health>
   ```

3. Post this data to WireServer: `curl -X POST -H 'x-ms-version: 2012-11-30' -H "x-ms-agent-name: WALinuxAgent" -H "Content-Type: text/xml;charset=utf-8" -d "$REPORT_READY_XML" http://168.63.129.16/machine?comp=health`

### Automating running the code at first boot

This demo uses systemd, which is the most common init system in modern Linux distros. So the easiest and most native way to ensure this report ready mechanism runs at the right time is to create a systemd service unit. You can add the following unit file to `/etc/systemd/system` (this example names the unit file `azure-provisioning.service`):

- Python
```[Unit]
Description=Early Azure Goal State Sender (No-Agent Provisioning)
DefaultDependencies=no
#After=local-fs.target network-pre.target
After=network-online.target dbus.service systemd-hostnamed.service
Wants=network-online.target dbus.service systemd-hostnamed.service
#Before=sysinit.target

[Service]
Type=oneshot
User=root
RemainAfterExit=yes
TimeoutSec=300
StandardOutput=journal
StandardError=journal
ExecStart=/usr/bin/bash /usr/local/azure-provisioning.py

[Install]
WantedBy=sysinit.target

```
- Bash
```
[Unit]
Description=Early Azure Goal State Sender (No-Agent Provisioning)
DefaultDependencies=no
#After=local-fs.target network-pre.target
After=network-online.target dbus.service systemd-hostnamed.service
Wants=network-online.target dbus.service systemd-hostnamed.service
#Before=sysinit.target

[Service]
Type=oneshot
User=root
RemainAfterExit=yes
TimeoutSec=300
StandardOutput=journal
StandardError=journal
ExecStart=/usr/bin/bash /usr/local/azure-provisioning.sh

[Install]
WantedBy=sysinit.target
```



This systemd service does three things for basic provisioning:

1. Reports ready to Azure (to indicate that it came up successfully).
1. Renames the VM based off of the user-supplied VM name by pulling this data from [Azure Instance Metadata Service (IMDS)](./instance-metadata-service.md). **Note** IMDS also provides other [instance metadata](./instance-metadata-service.md#access-azure-instance-metadata-service), such as SSH Public Keys, so you can set more than the hostname.
1. Disables itself so that it only runs on first boot and not on subsequent reboots.

With the unit on the filesystem, run the following to enable it:

```bash
$ sudo systemctl enable azure-provisioning.service
```

Now the VM is ready to be generalized and have an image created from it.

#### Completing the preparation of the image

Back on your development machine, run the following to prepare for image creation from the base VM:

```azurecli
$ az vm deallocate --resource-group demo1 --name demo1
$ az vm generalize --resource-group demo1 --name demo1
```

And create the image from this VM:

```azurecli
$ az image create \
    --resource-group demo1 \
    --source demo1 \
    --location eastus \
    --name demo1img
```

Now we're ready to create a new VM from the image. This can also be used to create multiple VMs:

```azurecli
$ IMAGE_ID=$(az image show -g demo1 -n demo1img --query id -o tsv)
$ az vm create \
    --resource-group demo12 \
    --name demo12 \
    --location eastus \
    --ssh-key-value <ssh_pub_key_path> \
    --public-ip-address-dns-name demo12 \
    --image "$IMAGE_ID"
    --enable-agent false
```

> [!NOTE]
>
> It is important to set `--enable-agent` to `false` because walinuxagent doesn't exist on this VM that is going to be created from the image.

The VM should be provisioned successfully. After Logging into the newly provisioning VM, you should be able to see the output of the report ready systemd service:

```bash
$ sudo journalctl -u azure-provisioning.service
-- Logs begin at Thu 2020-06-11 20:28:45 UTC, end at Thu 2020-06-11 20:31:24 UTC. --
Jun 11 20:28:49 thstringnopa systemd[1]: Starting Azure Provisioning...
Jun 11 20:28:54 thstringnopa python3[320]: Retrieving goal state from the Wireserver
Jun 11 20:28:54 thstringnopa python3[320]: ContainerId: 7b324f53-983a-43bc-b919-1775d6077608
Jun 11 20:28:54 thstringnopa python3[320]: InstanceId: fbb84507-46cd-4f4e-bd78-a2edaa9d059b._thstringnopa2
Jun 11 20:28:54 thstringnopa python3[320]: Sending the following data to Wireserver:
Jun 11 20:28:54 thstringnopa python3[320]: <Health><GoalStateIncarnation>1</GoalStateIncarnation><Container><ContainerId>7b324f53-983a-43bc-b919-1775d6077608</ContainerId><RoleInstanceList><Role><InstanceId>fbb84507-46cd-4f4e-bd78-a2edaa9d059b._thstringnopa2</InstanceId><Health><State>Ready</State></Health></Role></RoleInstanceList></Container></Health>
Jun 11 20:28:54 thstringnopa python3[320]: Response: 200 OK
Jun 11 20:28:56 thstringnopa bash[472]:   % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
Jun 11 20:28:56 thstringnopa bash[472]:                                  Dload  Upload   Total   Spent    Left  Speed
Jun 11 20:28:56 thstringnopa bash[472]: [158B blob data]
Jun 11 20:28:56 thstringnopa2 systemctl[475]: Removed /etc/systemd/system/multi-user.target.wants/azure-provisioning.service.
Jun 11 20:28:56 thstringnopa2 systemd[1]: azure-provisioning.service: Succeeded.
Jun 11 20:28:56 thstringnopa2 systemd[1]: Started Azure Provisioning.
```

## Support

If you implement your own provisioning code/agent, then you own the support of this code, Microsoft support will only investigate issues relating to the provisioning interfaces not being available. We're continually making improvements and changes in this area, so you must monitor for changes in cloud-init and Azure Linux Agent for provisioning API changes.

## Next steps

For more information, see [Linux provisioning](provisioning.md).
