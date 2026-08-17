# Detailed Lab Notes — Analyzing Network Traffic with VPC Flow Logs

## Task 1 — Configure a Custom Network with VPC Flow Logs

### Task 1A — Created the VPC and Enabled Flow Logs

**Action:** Created a custom VPC called `vpc-net` with a subnet named `vpc-subnet` and enabled VPC Flow Logs on the subnet.

**Configuration**
- Network: `vpc-net`
- Subnet: `vpc-subnet`
- CIDR: `10.1.3.0/24`
- Flow Logs: Enabled

**Observation:** The subnet was created with the expected `10.1.3.0/24` range and showed a VPC Flow Logs configuration attached to it.

![VPC subnet with Flow Logs enabled](../evidence/01-vpc-subnet-flow-logs-enabled.jpg)

**What I learned:** VPC Flow Logs have to be enabled for the subnet I want to monitor. Once enabled, Google Cloud can start recording network flow information for traffic using that subnet.

**Why it mattered:** Without Flow Logs, the network would still work, but I would have much less visibility into the traffic moving to and from resources in it.

### Task 1B — Created the HTTP and SSH Firewall Rule

**Action:** Created an ingress firewall rule called `allow-http-ssh` for `vpc-net`.

**Configuration**
- Rule: `allow-http-ssh`
- Network: `vpc-net`
- Direction: Ingress
- Target tag: `http-server`
- Source range: `0.0.0.0/0`
- TCP 22: SSH
- TCP 80: HTTP
- Action: Allow

**Observation:** The rule applied to instances with the `http-server` network tag and allowed HTTP and SSH traffic from any IPv4 address.

![HTTP and SSH firewall rule](../evidence/02-allow-http-ssh-firewall-rule.jpg)

**What I learned:** The network tag is what ties the firewall rule to the VM.

**Why it mattered:** I needed HTTP traffic to reach the Apache server so I could generate real network activity and then find that traffic in the VPC Flow Logs.

---

## Task 2 — Create an Apache Web Server

### Task 2A — Created `web-server`

**Action:** Created a Compute Engine VM called `web-server` and attached it to the custom network and subnet from Task 1.

**Configuration**
- VM: `web-server`
- Network: `vpc-net`
- Internal IP: `10.1.3.2`
- Network tag: `http-server`

**Observation:** The VM was running on `vpc-net` with the `http-server` tag applied.

![Running web-server VM](../evidence/03-web-server-vm.jpg)

**What I learned:** The network tag connected the VM to the firewall rule, while placing it on the Flow Logs-enabled subnet put its traffic inside the logging scope.

**Why it mattered:** This tied the server, firewall rule, and logging setup together before I started generating traffic.

### Task 2B — Installed Apache and Verified the Web Server

**Action:** Connected to `web-server` over SSH, installed Apache, and replaced the default page with a simple `Hello World!` page.

**Observation:** Opening the external IP in a browser loaded the custom page successfully.

![Apache Hello World page](../evidence/04-apache-hello-world.jpg)

**What I learned:** The successful page load confirmed Apache was running and HTTP traffic was reaching the VM through the firewall rule.

**Why it mattered:** I now had a working HTTP service that I could use to generate traffic and then look for that traffic in VPC Flow Logs.

---

## Task 3 — Verified My Web Traffic in VPC Flow Logs

**Action:** Generated HTTP traffic to `web-server`, then filtered VPC Flow Logs using the source IP from my browser session.

**Observation:** Logs Explorer returned a matching connection to the web server. The expanded record showed the source and destination IPs, ports, and protocol.

![VPC Flow Log showing browser traffic](../evidence/05-vpc-flow-log-browser-traffic.jpg)

**What I learned:** I was able to take traffic I knew I generated from my browser and find the matching connection inside VPC Flow Logs.

**Why it mattered:** This showed me how Flow Logs can be used to trace actual network activity and see who connected to a resource, which port they used, and where the traffic was going.

---

## Task 4 — Export VPC Flow Logs to BigQuery

### Task 4A — Created the BigQuery Log Sink

**Action:** Created a Log Router sink called `bq_vpcflows` and configured it to send VPC Flow Logs to a BigQuery dataset.

**Observation:** The sink was enabled and pointed to the `bq_vpcflows` dataset. The inclusion filter targeted the VPC Flow Logs.

![Log Router sink](../evidence/06-log-router-sink-to-bigquery.jpg)

**What I learned:** The Log Router sink is what moves matching log entries out of Cloud Logging and into BigQuery.

**Why it mattered:** This connected the logging side of the lab to the analysis side.

### Task 4B — Verified the BigQuery Destination

**Action:** Opened BigQuery to make sure the exported flow logs were arriving.

**Observation:** The `bq_vpcflows` dataset contained a `compute_googleapis_com_vpc_flows...` table.

![BigQuery VPC Flow Logs table](../evidence/07-bigquery-vpc-flow-table.jpg)

**What I learned:** Seeing the table confirmed that the sink was working and the data was reaching BigQuery.

### Task 4C — Generated More Traffic

**Action:** Used Cloud Shell to send 50 HTTP requests to the server.

```bash
export MY_SERVER=<EXTERNAL_IP>
for ((i=1;i<=50;i++)); do curl $MY_SERVER; done
```

**Observation:** The server returned the `Hello World!` response for the requests.

![Repeated HTTP requests](../evidence/08-generate-http-traffic-cloud-shell.jpg)

**What I learned:** Generating known traffic gave me a predictable set of requests to look for later.

**Why it mattered:** It let me test the path from traffic generation to logging and then into BigQuery.

### Task 4D — Queried the Flow Logs in BigQuery

**Action:** Ran a SQL query against the exported VPC Flow Logs.

**Observation:** The query returned VPC name, subnet, source and destination addresses, ports, protocol, and total bytes sent.

![BigQuery VPC Flow Log query results](../evidence/09-bigquery-flow-log-query-results.jpg)

**What I learned:** BigQuery let me turn raw flow-log records into a table that was much easier to compare and analyze.

**Why it mattered:** Instead of opening individual log records, I could use SQL to see which systems were communicating, which ports they were using, and how much traffic was being sent.

---

## Task 5 — Review Flow Log Aggregation

**Action:** Opened the VPC Flow Logs management view for `vpc-subnet` to work with the aggregation and sampling settings.

![VPC Flow Logs management view](../evidence/10-vpc-flow-logs-management.jpg)

**What I learned:** More detailed logging gives me more visibility, but it also creates more data. Aggregation and sampling can reduce log volume.

**Why it mattered:** Logging is a tradeoff between visibility and storage/analysis cost.

**Evidence note:** I did not preserve a separate screenshot of the final Advanced Settings values after the aggregation changes.
