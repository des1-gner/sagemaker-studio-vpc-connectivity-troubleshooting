# Troubleshooting SageMaker Studio connectivity in a VPC-only domain

A general guide for diagnosing connectivity failures from Amazon SageMaker Studio
when the domain runs in **VPC-only** mode (`AppNetworkAccessType: VpcOnly`) and the
VPC has no direct internet egress (for example, all traffic leaves via a Transit
Gateway to on-premises, behind a corporate proxy).

It covers two common, often-conflated problems:

1. A Studio app (e.g. the managed **MLflow App**) times out on TCP connect.
2. A **local IDE (VS Code) to Studio Space** remote connection hangs or fails.

No account IDs, resource IDs, or environment-specific values appear here. Replace
the angle-bracket placeholders with your own.

---

## Background: why VPC-only changes everything

In a VPC-only domain, Studio apps and Spaces have **no automatic path to the public
internet**. They can only reach what the VPC's routing and endpoints allow. Two
patterns are common:

- **Interface/Gateway VPC endpoints** keep AWS service traffic private inside AWS.
- **Centralised egress** sends `0.0.0.0/0` to a Transit Gateway, then out via
  Direct Connect / VPN to on-premises, where a proxy or firewall controls what
  reaches the internet.

A connection fails when the thing being reached (a public service endpoint, a
download CDN, an AWS API) has **no working path** under that model, either no VPC
endpoint for it, or the on-premises proxy is not permitting it.

The single most useful first question: **does the endpoint I am trying to reach
resolve to a public address or a private one, and does this VPC have a working path
to that address?**

---

## Step 0: Separate the problems before diagnosing

Different symptoms are different flows. Do not assume one root cause covers all of
them. For each failure, note:

- The **exact endpoint/hostname** being contacted.
- The **Region** it is in (traffic for one Region should not route through an
  endpoint in another).
- Whether the hostname resolves to a **public** or **private** IP.
- The **exact error** and which **stage** it fails at (DNS, TCP connect, TLS,
  application).

A control-plane API call (e.g. `api.sagemaker.<region>.amazonaws.com`) and a
data/app endpoint (e.g. an MLflow App URL) are separate flows and can fail for
different reasons.

---

## Step 1: Confirm the domain and VPC setup

```bash
# Is the domain VPC-only? Which VPC and subnets?
aws sagemaker describe-domain --domain-id <domain-id> \
  --query '{Access:AppNetworkAccessType,VpcId:VpcId,Subnets:SubnetIds,SG:DefaultUserSettings.SecurityGroups}'
```

If `AppNetworkAccessType` is `VpcOnly`, there is no service-managed internet path;
everything depends on your VPC.

---

## Step 2: Trace the egress path

```bash
# What do the Studio subnets route 0.0.0.0/0 to?
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=<vpc-id>" \
  --query 'RouteTables[].Routes[].{Dest:DestinationCidrBlock,GW:GatewayId,NAT:NatGatewayId,TGW:TransitGatewayId,VPCE:VpcEndpointId}'

# Is there a NAT gateway at all?
aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=<vpc-id>" \
  --query 'NatGateways[].{Id:NatGatewayId,State:State,Subnet:SubnetId}'

# Which interface/gateway VPC endpoints exist in this VPC?
aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=<vpc-id>" \
  --query 'VpcEndpoints[].{Id:VpcEndpointId,Type:VpcEndpointType,Service:ServiceName,State:State}'
```

Interpret `0.0.0.0/0`:

- **To a NAT gateway** that sits in a public subnet with an internet gateway route:
  the VPC has its own internet egress.
- **To a Transit Gateway**: traffic leaves to wherever the TGW forwards it
  (another VPC, or on-premises via Direct Connect / VPN). Reachability then depends
  on that downstream path and any proxy/firewall there. Follow the TGW route table
  to see the next hop.
- **No `0.0.0.0/0` at all**: no internet egress; only VPC endpoints work.

If the endpoint you need resolves to a **public** IP and there is no NAT/IGW, the
traffic can only succeed if the TGW/on-premises path forwards it to that public
address.

---

## Issue A: MLflow App TCP connection timeout

**Symptom.** From a Studio notebook, DNS for the MLflow App hostname
(`*.mlflow.sagemaker.<region>.app.aws`) resolves to public IPs, but TCP connect to
port 443 times out.

```bash
# From the Studio terminal
HOST="<your-mlflow-app-host>.mlflow.sagemaker.<region>.app.aws"
getent ahosts "$HOST"                      # DNS resolves? to public IPs?
curl -vI --connect-timeout 10 "https://$HOST/"          # with any proxy env applied
curl -vI --noproxy '*' --connect-timeout 10 "https://$HOST/"   # proxy bypassed
```

Read the result:

- **DNS resolves, TCP times out** → a reachability problem, not a DNS problem.
- **`--noproxy '*'` still times out** → the proxy is not the thing carrying the
  traffic; the VPC itself has no working egress to those public IPs. Go back to
  Step 2.
- **Works only with the proxy applied** → the proxy is the egress path; make sure
  it permits the MLflow and SageMaker endpoints (see below).

### Two ways to fix it

**Option 1 — reach it over your existing egress (proxy / on-premises).**
If your model is centralised egress via TGW to an on-premises proxy, the proxy must
allow the SageMaker and MLflow destinations. Because the MLflow App resolves to AWS
service IPs that can change, prefer allowing by **domain** rather than pinning IPs:

```
*.sagemaker.<region>.amazonaws.com
api.sagemaker.<region>.amazonaws.com
*.mlflow.sagemaker.<region>.app.aws
```

Apply proxy settings consistently for Studio apps. A Lifecycle Configuration is a
clean way to set them on app start, see the sample script in References.

**Option 2 — make it private with an interface VPC endpoint (no internet needed).**
The managed MLflow offering supports AWS PrivateLink. Note the newer **MLflow App**
and the older **MLflow tracking server** use **different** interface endpoint
service names:

| Offering | Interface endpoint service name |
|---|---|
| MLflow App | `aws.sagemaker.<region>.mlflow` |
| MLflow tracking server | `aws.sagemaker.<region>.experiments` |

You also need the SageMaker API interface endpoint, and you must **enable private
DNS** on the endpoint so the app hostname resolves to the private endpoint inside
your VPC instead of the public IPs. Confirm the VPC has `enableDnsSupport` and
`enableDnsHostnames` set to `true`, and that the endpoint security group allows
inbound/outbound HTTPS (443). See References.

---

## Issue B: Local VS Code to Studio Space connection hangs/fails

**Symptom.** Opening the Space in VS Code (Remote-SSH) either errors with "Could not
establish connection to ..." or spins for a long time on the remote user before
timing out.

**Mechanism.** The remote IDE connection to a Space runs as an **SSH tunnel over AWS
Systems Manager (SSM)**. It needs:

- IAM permission to start the session (e.g. `sagemaker:StartSession`).
- A network path from the Space's VPC to the **SSM** endpoints and the **SageMaker
  API**.
- For the IDE to bring up its remote server, a way to fetch the server/extensions
  (from the marketplace CDN) — which a no-internet VPC cannot reach directly.

### Capture the client-side logs first

In VS Code:

1. Open the Command Palette (`Ctrl+Shift+P` / `Cmd+Shift+P`).
2. Run `Remote-SSH: Settings` and set the log level to `trace`.
3. Retry the connection, then open the **Output** panel and select the
   **Remote-SSH** channel (or run `Remote-SSH: Open Log`).

Identify the stall stage:

- Stuck on the **SSH host setup / SSM session** (before any "server" line) → the
  tunnel is not establishing. Check SSM reachability, IAM, and the network path.
- Reaches the host, then hangs on **"Downloading/Installing VS Code server"** or
  **"Acquiring lock"** → the remote server fetch is blocked by lack of egress to the
  CDN.

### Fixes for a no-internet VPC

Two supported options (VS Code):

1. **HTTP proxy with controlled allow-listing** — point Remote-SSH at your proxy
   (`remote.SSH.httpProxy` / `httpsProxy`) and have the proxy allow the IDE server
   and extension CDNs.
2. **Pre-packaged remote server and extensions** — bake the `.vscode-server`
   directory into the environment (e.g. via a Lifecycle Configuration) so the remote
   side never needs to download it.

Note on IDE choice: VS Code and Kiro support the proxy path in private subnets;
Cursor is not supported for connecting to Spaces in subnets without outbound
internet.

See References for the setup steps.

---

## Issue C: Lifecycle Configuration (LCC) failures

If a previously working setup breaks after an LCC change (common when the LCC
configures the proxy or pre-packages the IDE server):

1. **Check the LCC's CloudWatch logs** to see where the script fails at launch.
2. **Isolate it** — launch a space/app with **no LCC attached**. If that launches
   cleanly, the LCC is the cause; compare it against a known-good version.
3. Roll the LCC forward/back and re-test, verifying the proxy/env settings it sets
   match what the current network path requires.

---

## General checklist

- [ ] Confirm the domain is VPC-only and note the VPC/subnets/security groups.
- [ ] Map `0.0.0.0/0`: NAT/IGW, Transit Gateway, or none. Follow TGW to its next hop.
- [ ] For each failing endpoint: public vs private resolution, and Region.
- [ ] Reproduce with `curl` including a `--noproxy '*'` run to isolate the proxy.
- [ ] Decide the model: reach public endpoints via proxy/on-prem, OR go private with
      VPC endpoints.
- [ ] For private MLflow: correct service name for your offering (App vs tracking
      server) + SageMaker API endpoint + private DNS enabled + DNS attributes on +
      SG allows 443.
- [ ] For VS Code→Space: capture Remote-SSH trace logs, identify the stall stage,
      then use proxy allow-listing or pre-packaged server.
- [ ] If an LCC changed: check its CloudWatch logs and test with no LCC attached.
- [ ] Network-path changes (proxy allow-list, routes, endpoints) usually require
      your networking team; AWS service-side config does not control the on-prem hop.

---

## References

- Connect to SageMaker AI MLflow through an interface VPC endpoint (service names,
  private DNS, create steps):
  https://docs.aws.amazon.com/sagemaker/latest/dg/mlflow-interface-endpoint-create.html
- MLflow interface VPC endpoint overview:
  https://docs.aws.amazon.com/sagemaker/latest/dg/mlflow-interface-endpoint.html
- Remote access to Studio Spaces (SSH over SSM):
  https://docs.aws.amazon.com/sagemaker/latest/dg/vscode-access.html
- Connect a remote IDE to Spaces in subnets without internet access (proxy
  allow-listing, pre-packaged server):
  https://docs.aws.amazon.com/sagemaker/latest/dg/remote-access-local-ide-setup-vpc-no-internet.html
- SageMaker Studio Lifecycle Configuration examples (includes a set-proxy-settings
  sample): https://github.com/aws-samples/sagemaker-studio-lifecycle-config-examples
- SageMaker Studio + MLflow integration samples:
  https://github.com/aws-samples/sagemaker-studio-mlflow-integration

---

*This guide is generic. Validate every command and setting against the current AWS
documentation and your own environment before relying on it.*
