# Week 1 — Daily Log

## Day 1

**Goal for today:**
- Set up the base system and get Elasticsearch and Kibana installed and running

**What I did:**
1. Updated the system:
   ```bash
   apt-get update && apt-get upgrade -y
   ```
2. Downloaded and installed Elasticsearch via `dpkg`:
   ```bash
   wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-9.4.4-amd64.deb
   dpkg -i elasticsearch-9.4.4-amd64.deb
   ```
3. Edited the Elasticsearch config:
   ```bash
   cd /etc/elasticsearch/
   vim elasticsearch.yml
   ```
   Done to set the network binding so Elasticsearch would be reachable from outside `localhost`, rather than only accepting local connections. Checked network interfaces along the way with `ip a` to confirm the correct address to bind to.
4. Enabled and started Elasticsearch:
   ```bash
   systemctl daemon-reload
   systemctl enable elasticsearch.service
   systemctl start elasticsearch.service
   systemctl status elasticsearch.service
   ```

**Problems hit / next steps:**
- Elasticsearch installation and startup went cleanly — Kibana setup carried into day 2 below.

---

## Day 2

**Goal for today:**
- Get Kibana running and connected to Elasticsearch

**What I did:**
1. Downloaded and installed Kibana via `dpkg`:
   ```bash
   wget https://artifacts.elastic.co/downloads/kibana/kibana-9.4.4-amd64.deb
   dpkg -i kibana-9.4.4-amd64.deb
   ```
2. Edited the Kibana config:
   ```bash
   vim /etc/kibana/kibana.yml
   ```
   Done to set the network binding so Kibana would accept connections from outside `localhost` instead of only the loopback interface.
3. Enabled and started Kibana:
   ```bash
   systemctl daemon-reload
   systemctl enable kibana.service
   systemctl start kibana.service
   systemctl status kibana.service
   ```
4. Generated a Kibana enrollment token using `elasticsearch-create-enrollment-token --scope kibana` — needed so Kibana can securely connect to and authenticate with Elasticsearch, since security features (TLS, authentication) are enabled by default
5. Generated the encryption keys needed for the keystore:
   ```bash
   ./kibana-encryption-keys generate
   ```
6. Added those keys to the Kibana keystore:
   ```bash
   ./kibana-keystore add xpack.encryptedSavedObjects.encryptionKey
   ./kibana-keystore add xpack.reporting.encryptionKey
   ./kibana-keystore add xpack.security.encryptionKey
   ```
   Needed for Kibana to encrypt saved objects, reports, and session/security data at rest.
7. Restarted Kibana to apply the new keystore values
8. Edited `/etc/kibana/kibana.yml` again to diagnose why Kibana kept crashing on startup
9. Checked service status for both Kibana and Elasticsearch
10. Pulled detailed logs with `journalctl` to find the real error
11. Confirmed listening ports with `ss -tlnp | grep 5601`
12. Checked network interfaces with `ip a`
13. Reviewed `/etc/elasticsearch/elasticsearch.yml` to confirm its network settings matched what Kibana needed
14. Added a UFW rule to allow port 5601, so the host firewall wouldn't block browser access
15. Verified firewall status with `ufw status verbose`
16. Tested connectivity with `curl -v` against both `localhost` and the VM's actual address
17. Restarted Kibana again after the config fix

![Initial Kibana dashboard](../screenshots/week-1/Screenshot%202026-07-28%20194419.png)

**Problems hit:**
- Kibana kept crash-looping (`Active: failed`, `Start request repeated too quickly`)
- `journalctl` revealed the real error: `FATAL Error: listen EADDRNOTAVAIL` — Kibana's `server.host` in `kibana.yml` was set to an IP that didn't match the VM's actual assigned IP
- After fixing that, `curl localhost:5601` still failed with "connection refused" — because Kibana was bound to the specific host IP, not `0.0.0.0`, so loopback wasn't listening
- Cloud firewall's inbound Accept rule was scoped to a private LAN IP instead of my actual public IP, so remote browser access timed out even though Kibana and UFW were both fine

**How I solved them:**
- Set `server.host: 0.0.0.0` in `kibana.yml` so Kibana listens on all interfaces instead of one static IP — confirmed via `ss -tlnp` showing `0.0.0.0:5601`
- Corrected the cloud firewall's Accept rule source to my actual public IP with a `/32` mask
- After both fixes, `curl` against the VM's address returned a full HTTP 200 response with the Kibana HTML page

**Still open / next steps:**
- Complete Kibana's interactive setup flow (enrollment code) in the browser
- Confirm Elasticsearch connection is fully validated inside Kibana's setup screen
- Move on to ingesting Sysmon logs
---

## Day 3

**Goal for today:**
- Set up Fleet Server and enroll a Windows host into Fleet

**Context — what Fleet Server does:**
Fleet Server acts as the central management point between Elastic Agents and Elasticsearch/Kibana. Instead of configuring and updating every agent individually, agents check in with Fleet Server to receive their policies, configuration changes, and integration updates, and Fleet Server relays their data and health status back to Kibana. This makes it possible to manage many endpoints (like this Windows host) from a single place, rather than touching each machine by hand.

**What I did:**
1. Installed Elastic Agent as a Fleet Server on the centralized Linux host using the install command generated by Kibana's "Add Fleet Server" flow, pointing it at Elasticsearch (`--fleet-server-es`), the service token, the enrollment policy, and the CA trusted fingerprint.
2. Added an inbound firewall rule on the VPC for the Fleet Server's public IP, scoped to `TCP/8220`, restricted to `64.177.122.71/32` (single host, not a subnet) rather than a wider CIDR range.
3. First enrollment attempt failed:
```
   Error: fleet-server failed: timed out waiting for Fleet Server to start after 2m0s
```
   Investigated by checking connectivity to Elasticsearch and confirming the correct architecture build (`uname -m`) was used, since the installer defaults to aarch64 downloads on some pages.
4. Downloaded the Windows x86_64 Elastic Agent build and attempted enrollment:
```powershell
   .\elastic-agent.exe install --url=https://64.177.122.71:8220 --enrollment-token=<token>
```
5. Hit `Error: already installed at: C:\Program Files\Elastic\Agent` — an older agent install was already present, so a clean uninstall was required before reinstalling.
6. Uninstall initially failed when run from the extracted install folder:
```
   Error: can only be uninstalled by executing the installed Elastic Agent at: C:\...
```
   Fixed by running the uninstall from the actual installed path, quoted because of the space in `Program Files`:
```powershell
   & "C:\Program Files\Elastic\Agent\elastic-agent.exe" uninstall
```
7. Re-ran the install command from the extracted folder, successfully enrolling the Windows host into Fleet.
8. Verified persistence of the Elastic Agent service so it survives reboots:
```powershell
   Get-Service "Elastic Agent" | Select-Object Name, Status, StartType
```
9. Noticed the enrolled host displayed as `Guest` in Fleet instead of a meaningful hostname, and renamed the Windows machine:
```powershell
   Rename-Computer -NewName "SOC-WIN-larry" -Restart
```

![Windows host added to Fleet](../screenshots/week-1/addedwindowstofleet.png)

**Problems hit:**
- Fleet Server timed out waiting to start on first attempt — likely a networking/reachability issue to Elasticsearch, or an architecture mismatch in the downloaded binary
- Windows enrollment initially pointed at the wrong port (443 instead of Fleet Server's actual listening port, 8220)
- Uninstall/reinstall was blocked because Elastic Agent enforces that uninstall must be run from its actual installed path, not the extracted install directory
- Enrolled Windows host showed up in Fleet as `Guest` rather than its intended name

**How I solved them:**
- Confirmed system architecture with `uname -m` and matched the correct Elastic Agent build
- Corrected the Fleet Server URL to use port `8220` for Windows enrollment
- Ran uninstall directly from `C:\Program Files\Elastic\Agent\elastic-agent.exe`, quoting the path due to the space in `Program Files`
- Renamed the Windows host to `SOC-WIN-larry` via `Rename-Computer -Restart` to correct the displayed hostname in Fleet

**Still open / next steps:**
- Confirm Fleet now displays the updated hostname `SOC-WIN-larry` after reboot, and unenroll/re-enroll if the name doesn't refresh automatically
- Verify Windows host is actively shipping logs/data into Elasticsearch via Fleet
- Begin mapping Windows event/Sysmon data ingestion once the host is confirmed healthy in Fleet
---

## Day 4

**Goal for today:**
- Install Sysmon on the Windows host to enable detailed process/network/registry event logging for ingestion into Elastic

**What I did:**
1. Downloaded Sysmon from the official Sysinternals page:
   https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon
2. Downloaded a community-maintained Sysmon configuration file (`sysmonconfig.xml`) from Olaf Hartong's `sysmon-modular` repo, which provides a much more comprehensive event-filtering ruleset than Sysmon's bare defaults:
   https://github.com/olafhartong/sysmon-modular

   ![Olaf sysmon-modular config](../screenshots/week-1/olaf_sysmonconfig.png)

3. Extracted the Sysmon download into `C:\Users\Administrator\Downloads\Sysmon`, confirmed contents:
```powershell
   dir
```
   Files present: `Eula.txt`, `Sysmon.exe`, `Sysmon64.exe`, `Sysmon64a.exe`, `sysmonconfig.xml`
4. Installed Sysmon using the downloaded config file, from an elevated PowerShell:
```powershell
   .\Sysmon64.exe -i sysmonconfig.xml
```
5. Confirmed successful install/start from the command output:
```
   Loading configuration file with schema version 4.90
   Sysmon schema version: 4.91
   Configuration file validated.
   Sysmon64 installed.
   SysmonDrv installed.
   Starting SysmonDrv.
   SysmonDrv started.
   Starting Sysmon64..
   Sysmon64 started.
```
6. Verified Sysmon was actively generating events by checking Event Viewer:
   `Applications and Services Logs → Microsoft → Windows → Sysmon → Operational`

   ![Sysmon events in Event Viewer](../screenshots/week-1/sysmonineventviwer.png)

**Still open / next steps:**
- Confirm Elastic Agent's Windows/Sysmon integration is picking up and shipping these events into Elasticsearch
- Validate Sysmon events are visible/searchable in Kibana, then revisit the Sysmon App for Splunk exercises as a comparison point between the two SIEM stacks
## Day 5

**Goal for today:**
- Ingest Windows Sysmon logs into Elastic via the Custom Windows Event Logs integration, and add Microsoft Defender event collection through the same integration

**What I did:**
1. In Kibana, navigated to **Integrations → Add integration** and added the **Custom Windows Event Logs** integration to the agent policy for `SOC-WIN-larry`

   ![Adding Sysmon to the integration](../screenshots/week-1/addingsysmontointegration.png)

2. Selected **Logs** as the data stream type for the policy (as opposed to Metrics or Traces)
3. Left the log channel name at its default for Sysmon, so events were pulled from the standard Sysmon Operational log
4. Configured the integration's advanced settings:
   - **Event ID**: comma-separated list of included/excluded event IDs (accepts single IDs like `4624`, ranges like `4700-4800`, or exclusions like `-4735`)
   - **Ignore events older than**: `72h`, so the agent skips backfilling anything older than 3 days
   - **Language ID**: left at `0` (system default / en-US)
5. Added a second instance of the **Custom Windows Event Logs** integration (same policy), again leaving the log channel name at its default — this time for Microsoft Defender — and set the specific Defender event IDs I wanted collected: `1116, 1117, 5001`
   - `1116` — Defender detected malware
   - `1117` — Defender took action on a threat
   - `5001` — Real-time protection was disabled

   ![Adding event IDs with Defender](../screenshots/week-1/Addingeventidswithdefender.png)

6. Saved and deployed the updated policy so Elastic Agent on the Windows host picked up both log sources
7. Confirmed the agent applied the new policy:
```powershell
   Get-Service "Elastic Agent" | Select-Object Name, Status, StartType
```
8. Verified Sysmon and Defender events were both flowing into Elasticsearch by checking **Discover** in Kibana, filtered to each respective data stream

**Still open / next steps:**
- Build a Kibana dashboard combining Sysmon and Defender data for the Windows host
- Compare Defender's native alerting (event IDs 1116/1117) against raw Sysmon events for the same malicious activity
- Tune the Event ID filter lists further if noise becomes an issue
## Day 6

**Goal for today:**
- Enroll a second Linux host (`SOC-Linux-larry`) into Fleet as a standard agent, to broaden log collection beyond the Windows endpoint

**What I did:**
1. Confirmed system architecture before downloading the agent:
   ```bash
   uname -m
   ```
   Returned `x86_64`, confirming the correct Elastic Agent build to download in Kibana's "Add agent" flow
2. Downloaded, extracted, and attempted to install/enroll the agent:
   ```bash
   curl -L -O https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-9.4.4-linux-x86_64.tar.gz
   tar xzvf elastic-agent-9.4.4-linux-x86_64.tar.gz
   cd elastic-agent-9.4.4-linux-x86_64
   sudo ./elastic-agent install --url=https://64.177.122.71:8220 --enrollment-token=<token>
   ```
3. First enrollment attempt failed and retried repeatedly with:
   ```
   Error detected: fail to execute request to fleet-server: x509: certificate signed by unknown authority, will retry in a moment.
   ```
4. Suspended the stuck process with `Ctrl+Z` to investigate — this only suspends the job rather than killing it
5. Re-ran the install with `--insecure` to bypass TLS verification against Fleet Server's self-signed cert:
   ```bash
   sudo ./elastic-agent install --url=https://64.177.122.71:8220 --enrollment-token=<token> --insecure
   ```
6. Hit `Error: already installed at: /opt/Elastic/Agent` — the suspended job from step 4 had already completed the install in the background
7. Cleared the stuck job and cleanly reinstalled:
   ```bash
   jobs
   kill -9 %1
   sudo /opt/Elastic/Agent/elastic-agent uninstall
   sudo ./elastic-agent install --url=https://64.177.122.71:8220 --enrollment-token=<token> --insecure
   ```
8. Agent enrolled successfully into Fleet:

   ![Linux agent added to Fleet](../screenshots/week-1/addedlinuxagenttofleet.png)

9. Agent showed **Degraded** health after enrolling:
   ```
   Recoverable: Elasticsearch request failed: dial tcp 216.128.156.245:9200: i/o timeout
   ```
10. Ruled out UFW as the cause — confirmed port 9200 already allowed inbound from anywhere on the Elasticsearch host:
    ```bash
    sudo ufw status verbose
    ```
11. Checked the Vultr cloud firewall group on the Elasticsearch host and found only three IPs allowed (each scoped `/32`) — none matched the Linux agent's own public IP
12. Identified the Linux agent's public IP:
    ```bash
    curl ifconfig.me
    ```
    Returned `66.42.117.183` — not present in the Elasticsearch host's firewall group
13. Added an Accept rule for `66.42.117.183/32` on TCP/9200 to the Elasticsearch host's Vultr firewall group
14. Re-tested connectivity from the agent:
    ```bash
    curl -v -k https://216.128.156.245:9200
    ```
    Result: full TLS handshake succeeded against the self-signed `SOC-ELK` cert, and Elasticsearch responded with `HTTP/1.1 401 Unauthorized` — confirming the network path was open (401 is expected without credentials; the important part was getting a response instead of a timeout)
15. Fleet briefly showed a transient **Unhealthy** state after the fix, but direct CLI status on the host confirmed everything was actually healthy:
    ```bash
    sudo /opt/Elastic/Agent/elastic-agent status --output=full
    ```
    All inputs and outputs (fleet, filestream-monitoring, log-default, system/metrics-default) reported `HEALTHY`
16. Refreshed the Fleet Agents page, which then correctly displayed `SOC-Linux-larry` as **Healthy**
17. Confirmed all three hosts (`SOC-ELK`, `SOC-WIN-larry`, `SOC-Linux-larry`) were actively sending logs into the Elastic stack:

    ![Kibana dashboard showing three hosts sending logs](../screenshots/week-1/elasticdashboardshowingweaddedthreeservers.png)

**Problems hit:**
- `x509: certificate signed by unknown authority` — Fleet Server uses a self-signed cert, which the agent doesn't trust by default
- `Ctrl+Z` suspended rather than killed the enrollment process, leaving a background install that blocked the next attempt with `Error: already installed`
- Agent enrolled but reported `Degraded`/`i/o timeout` reaching Elasticsearch on port 9200 — turned out to be the Vultr cloud firewall group missing an inbound rule for the Linux agent's specific public IP (UFW and Elasticsearch's own binding were both already correct)
- Fleet's UI briefly showed `Unhealthy` even after the fix — a stale status snapshot that cleared once the agent completed another check-in cycle

**How I solved them:**
- Used `--insecure` on the install command to bypass cert validation for this lab environment (in production, would use `--certificate-authorities` or a CA fingerprint instead, as done for the Windows Fleet Server enrollment on Day 3)
- Killed the suspended background job with `kill -9 %1`, then cleanly uninstalled via `/opt/Elastic/Agent/elastic-agent uninstall` before reinstalling
- Diagnosed the timeout methodically: ruled out UFW first (already open), then found and fixed the actual cause — the agent's own public IP (`66.42.117.183/32`) was missing from the Elasticsearch host's Vultr firewall group
- Confirmed the fix using a manual `curl` test (TLS handshake + 401 response, instead of a timeout) before trusting Fleet's own status indicator, then cross-checked with `elastic-agent status --output=full` directly on the host to bypass a temporarily stale Fleet UI reading

**Still open / next steps:**
- Decide what integrations/log sources to add for this Linux host, now that it's confirmed healthy (currently only running the base System integration)
- Consider standardizing on `--certificate-authorities`/CA fingerprint instead of `--insecure` across agents, for consistency with the Windows Fleet Server setup
- Build a Kibana dashboard incorporating this Linux host alongside the existing Windows/Sysmon/Defender data
- Look into scoping the Vultr firewall's port 9200 rule down to just port 9200 (not a full `1:65535` range) for tighter security, and consider narrowing the existing 5601 rule the same way
## Day 7

**Goal for today:**
- Build a Kibana query to detect SSH brute-force activity, turn it into an alert rule, and visualize the failed/successful attempts on a dashboard using Maps

**What I did:**
1. Went into **Discover**, selected the relevant data view, and built the initial query to isolate SSH activity:
   ```
   system.auth.ssh.event : *
   ```
   Reviewed the results — noticed a large volume of `Failed` events hitting many different usernames (`root`, `dev`, `test`, `ubuntu`, etc.) from a small handful of source IPs in short bursts, most heavily from `45.153.34.41` (Netherlands).

   ![Initial query for making the rule](../screenshots/week-2/created_initquery_for_makingrule.png)

2. Narrowed the query to only failed authentications and used **Create rule → Elasticsearch query** from Discover to build an alert off of it:
   ```
   system.auth.ssh.event : "Failed"
   ```
3. Configured the rule's grouping/threshold to catch brute-force behavior instead of alerting on every single failed login:
   - **Group by**: `source.ip`
   - **Threshold**: alert when a group exceeds a set number of matching documents (tuned this against the volume seen from `45.153.34.41`)
   - **Time window**: a short rolling window (few minutes) so the rule reacts to bursts, not scattered one-off failures
   - Left **"Exclude matches from previous runs"** enabled so the same failed-login burst doesn't re-trigger the rule on every scheduled run
4. Set the **rule schedule** to check on a short interval so bursts get caught quickly without over-hammering Elasticsearch
5. Saved the rule — confirmed it correctly picked out the `45.153.34.41` brute-force pattern rather than firing on isolated failed logins

   ![Created SSH brute-force rule](../screenshots/week-2/createdsshbruteforcerule.png)

6. Built a **dashboard** with two Maps panels side by side — one for **SSH Failed Authentications** and one for **SSH Success** — each showing a choropleth of attempt volume by country (`source.geo.country_iso_code`), with a shared `source.geo.country_iso_code: US` filter applied at the dashboard level

   ![Dashboard of failed and success SSH](../screenshots/week-2/madeadashboardof_failedandsuccess_ssh.png)

7. Used the brute-force query data to populate the map layer and confirm the geographic spread of the attack traffic

   ![Using brute-force data on the map](../screenshots/week-2/usingbruteforcedataonmap.png)

**Problems hit:**
- Initially couldn't edit the query on the second map panel (SSH Success) — turned out both panels were pointing at the same underlying saved Maps object rather than two independent visualizations, so editing one affected both
- Had to be careful that the panel-level query (`Failed` vs `Accepted`/success value) was actually being set on the correct, independent saved object rather than a shared one

**How I solved them:**
- Went into the Maps editor for the linked panel and saved it as a new, separate visualization (rather than saving back to the shared object), then re-pointed the dashboard panel at the new one
- Set the layer query explicitly per map afterward: `system.auth.ssh.event : "Failed"` on one, the success equivalent on the other, so each panel now reflects independent data

**Still open / next steps:**
- Confirm the alert rule fires correctly against live traffic (not just backtested against existing documents) and wire up an action (e.g. webhook/email) so the alert actually notifies someone
- Tune the threshold/time window further if false positives or missed bursts show up over the next few days
- Consider adding a `user.name` grouping in addition to `source.ip` to catch spray-style attacks that rotate IPs but reuse a small set of usernames
