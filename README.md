# multi-windows-host

Harness Git Experience repo for the **sites-as-infrastructure-definitions** demo.

It answers a specific Cegid question: they run environments `testA / testB / testC / prodA`,
each with 2 Windows VMs, and want to collapse them into just **`test`** and **`prod`** — with
the *sites* becoming **infrastructure definitions** rather than environments.

This repo holds the Harness entities that prove the model, synced from project `Cegid`
(org `default`).

---

## Layout

```
.harness/
├── pipeline/
│   └── cc_multihost_deploy.yaml            # the deploy pipeline
├── infraDefinition/
│   └── testA.yaml                          # site testA — 2 Windows VMs, PDC/WinRm
├── service/
│   └── cc_multihost_demo.yaml              # WinRm service, no artifact (script-only)
└── overrides/
    └── test_cc_multihost_demo_testA.yaml   # INFRA_SERVICE_OVERRIDE for test + testA
```

Each file is a Harness entity backed by Git Experience. Editing here and pushing updates
Harness; editing in the Harness UI commits back to this branch.

> **If you move a file, update the entity's Git path in Harness too.** Harness resolves each
> entity by the `filePath` recorded on it, not by scanning `.harness/`. A rename in Git alone
> makes the entity look missing.

---

## The three-layer config model

This is the core idea. Each kind of variation has exactly one home:

| What varies | Where it lives | Mechanism |
|---|---|---|
| Which VMs, which WinRM credential | Infrastructure definition | `spec.hosts`, `spec.credentialsRef` |
| Paths, app pool, DB conn, URLs per site | Service override scoped to `env + infra` | `type: INFRA_SERVICE_OVERRIDE` + `infraIdentifier` |
| Per-individual-VM behaviour | Looping strategy in the pipeline | `repeat: items: <+stage.output.hosts>` + `<+instance.*>` |

Override precedence: **infra-scoped > env-scoped > service default**.

The demo makes this visible on purpose: the service default sets `siteName=DEFAULT_UNSET`.
If the deploy logs print `testA`, override resolution worked. If they print `DEFAULT_UNSET`,
the override did not resolve.

**Two environments, not twelve.** Adding `testD` = one infra definition + one override. It does
*not* mean a new environment with its own RBAC, approvals, and freeze windows.

---

## Entities

### `pipeline/cc_multihost_deploy.yaml`

One stage `deploy_to_site` → one **Command** step (`onDelegate: false`) with three PowerShell
command units, all running **on the target host**:

1. **Show Resolved Config** — prints the override-resolved values, proving precedence
2. **Write Release Marker** — writes `release.txt` + an IIS landing page
3. **Local Health Check** — `GET http://localhost/` from the box itself

Nothing runs delegate-side, so the pipeline doesn't depend on the delegate having any
particular shell.

To let one pipeline target any site, change the hardcoded environment/infra to `<+input>`:

```yaml
environment:
  environmentRef: <+input>
  infrastructureDefinitions: <+input>
```

### `infraDefinition/testA.yaml`

PDC / WinRm, inline `hosts` list of both VM public IPs, `environmentRef: test`,
`credentialsRef: winrm_cegid_demo`.

For real use, give each site its **own** `credentialsRef` (`winrm_testb`, `winrm_testc`,
`winrm_proda`) — so testA's service account never touches testB's boxes.

### `service/cc_multihost_demo.yaml`

WinRm type, **no artifact** — script-only, so the demo is standalone and doesn't need JFrog.
Declares `deployPath`, `appPool`, `siteName`, `healthPort` with sentinel defaults.

Every variable the pipeline references must be declared here. A variable that exists only in
the override fails at expression-resolution time (`Unresolved expression:
serviceVariables.X`) before any step runs.

### `overrides/test_cc_multihost_demo_testA.yaml`

Scoped to `test` + `testA`. This is the only place the site's real values live:

| Variable | Value |
|---|---|
| `deployPath` | `C:\inetpub\CC_A` |
| `appPool` | `ContinuousConversation_A` |
| `siteName` | `testA` |
| `healthPort` | `80` |

---

## Infrastructure (AWS us-east-1)

| Name | Instance ID | Public IP | Hostname |
|---|---|---|---|
| cegid-testa-vm1 | `i-0d660066a81475474` | 3.95.222.70 | EC2AMAZ-2JLFLCF |
| cegid-testa-vm2 | `i-051da831488b097c7` | 3.88.22.13 | EC2AMAZ-F2JTQ6L |

Windows Server 2022 (`ami-040a155879de85e73`), t3.medium, 50 GB gp3. Public IPs are **not**
Elastic — they change on every stop/start, and `infraDefinition/testA.yaml` must be updated
to match.

Each VM is bootstrapped by userdata that installs IIS + ASP.NET, creates a self-signed cert
and a WinRM **HTTPS listener on 5986**, enables Negotiate/Basic auth, opens the firewall, and
creates `C:\inetpub\CC_A`. The script ends with `<persist>true</persist>`, so **a reboot
re-runs the whole bootstrap** — that's the repair mechanism when a VM's WinRM config is wrong.

Neither VM has an SSM instance profile, so `aws ssm send-command` is not available for in-guest
fixes. Options are reboot or RDP.

---

## Adding a site

1. **Infra definition** — copy `infraDefinition/testA.yaml`, change `identifier`, `hosts`,
   and `credentialsRef`. For `prodA`, also change `environmentRef: production`.
2. **Override** — copy `overrides/test_cc_multihost_demo_testA.yaml`, scope it to the new
   `environmentRef` + `infraIdentifier`, set the site's paths and app pool.
3. **WinRM credential** — create in the Harness UI (see gotcha 6).
4. Nothing in the pipeline changes.

An alternative to inline `hosts` is a **PDC connector** with JSON `hostAttributes` plus
`hostFilter: {type: HostAttributes, matchCriteria: ALL|ANY}` — better when host lists are
large or shared across infra definitions. There's also `hostGroups` (per-group
`credentialRef`) behind feature flag `CDS_ENABLE_INFRA_HOST_GROUPS`, but it accepts fixed
values only.

---

## Gotchas — every one of these cost a failed run

1. **Command steps do NOT auto-fan-out across hosts.** An explicit looping strategy is
   mandatory:
   ```yaml
   strategy:
     repeat:
       items: <+stage.output.hosts>
   ```
   Without it: `Invalid argument(s): Host information is missing in Command Step. Please make
   sure the looping strategy (repeat) is provided.`

2. **Never pin WinRM deployments to the Windows delegate.** Adding
   `delegateSelectors: [cegid-windows-ci]` — on the step *or* the infra definition — fails the
   stage: `Non active delegates, Delegate(s) don't have selectors [cegid-windows-ci],
   COMMAND_TASK_NG Task type not supported by delegate(s)`. The Windows delegate is for **CI
   builds**; WinRM `COMMAND_TASK_NG` is dispatched from **Linux** delegates, which connect
   outbound to the Windows hosts on 5986. Seeing a Linux delegate name in a WinRM failure is
   normal and is *not* the bug.

3. **`onDelegate: true` + PowerShell is broken.** A delegate-side PowerShell ShellScript step
   with no selector lands on a random Linux delegate and dies with
   `Failed to execute command [/bin/sh, -c, pwsh -Version]`. Keep work on the target hosts.

4. **`<+infra.identifier>` resolves to `null`.** Use `<+env.identifier>` plus
   `<+instance.hostName>` instead.

5. **Service override create needs a JSON object, not a YAML string.** Passing `yaml: "..."`
   fails with `Override spec is not provided in request`. Correct shape:
   ```json
   {"spec": {"variables": [{"name": "siteName", "type": "String", "value": "testA"}]}}
   ```

6. **The MCP `harness_create` enum has no `secret` type.** WinRM credentials and
   site-specific secrets must be created in the UI or via the REST API.

7. **Port-check tooling matters.** `/dev/tcp` gives false negatives — it reported 5986 closed
   on healthy hosts. Use `nc -z -v -w 10 <ip> 5986`.

8. **`curl --ntlm` against `/wsman` is not a valid credential test.** It returns 401 even for a
   host Harness authenticates to successfully. Don't use it to conclude creds are broken.

---

## Troubleshooting a per-host WinRM auth failure

**Symptom:** one host in the loop succeeds, another fails with
`Authorization Error: Invalid credentials. Check AuthenticationScheme, username and password`.

Both hosts share one `credentialsRef`, so a *single*-host failure means that host's bootstrap
didn't finish configuring the password / WinRM auth. It is not a credential problem.

```bash
# 1. Confirm userdata actually ran (look for "Windows is Ready to use")
aws ec2 get-console-output --instance-id <id> --output text --query Output | tail -40

# 2. Reboot — <persist>true</persist> re-runs the full bootstrap
aws ec2 reboot-instances --instance-ids <id>

# 3. Wait ~4-5 min, then confirm the listener is back
nc -z -v -w 10 <ip> 5986
```

Then re-run the pipeline.

---

## Links

- [Pipeline in Harness](https://app.harness.io/ng/account/T_JG6UCfQcye3MFhGUx3tw/all/orgs/default/projects/Cegid/pipelines/cc_multihost_deploy/pipeline-studio)
- Account `T_JG6UCfQcye3MFhGUx3tw` · Org `default` · Project `Cegid`
