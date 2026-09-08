# AD Self-Service Demo: Unlock Account, with Approval

A reusable Ansible Automation Platform demo built for **Senior Active Directory
engineers with zero AAP experience**. It answers one question in about five
minutes: *"what does AAP actually give me that I don't already have?"*

## The story

A user calls the helpdesk: "I'm locked out." Today that's a ticket, a queue,
and someone with domain admin rights typing `Unlock-ADAccount` by hand.

With this demo:

1. Someone (helpdesk, the user's manager, or the AD engineer themselves)
   opens the **"AD Self-Service - Unlock Account"** workflow in AAP and
   types in the locked-out username.
2. The engineer's manager gets an **approval request** - nothing runs until
   they click Approve.
3. AAP unlocks the account using the **official, Red Hat-certified
   `microsoft.ad` collection** - the same module family used for
   production AD management, not a toy script.
4. Every step - who asked, who approved, what ran, what changed - is in the
   AAP job history. No ticket, no shared credentials, no "who unlocked
   this and why" mystery three weeks later.

The value pitch for this audience: **you keep control (approval, RBAC,
credential vaulting), you get delegation (a manager or helpdesk can trigger
it without a domain admin password), and you get an audit trail for free.**

## What's actually in here

- `microsoft.ad` (certified) - unlocks the account (`microsoft.ad.user`,
  `account_locked: false`)
- `ansible.windows` (certified) - WinRM connectivity
- `microsoft.mecm` (certified) - included and wired in as a **real,
  ready-to-go** job template (`MECM - Trigger Policy Refresh`) for shops
  that also run Microsoft Endpoint Configuration Manager. Not live in this
  demo environment (see below) but not a stub either - it's the real
  module against the real argument spec.
- Everything is config-as-code (`playbooks/files/config_as_code/`), applied
  by one job template, using the same `infra.aap_configuration` pattern as
  the rest of this org's AAP demos.
- All collections resolve through the **Red Hat Automation Hub**
  certified/validated content endpoints already configured on this
  controller's Default organization - not the public community Galaxy.

## Repo layout

```
playbooks/
  main.yml                 Setup - AD Demo - CAC: applies everything below
  bootstrap_dc.yml         One-time lab setup (see "Setting this up")
  unlock_user.yml          The actual demo payload
  mecm_policy_refresh.yml  Optional MECM node (see "Going live with MECM")
  files/config_as_code/    Inventory, hosts, groups, project, job templates,
                            workflow + survey + approval definitions
collections/requirements.yml
```

## Setting this up

### 1. Provision a target Windows Server

This demo was built against a Windows Server 2022 EC2 instance with WinRM
enabled via the official Ansible bootstrap script
([`ConfigureRemotingForAnsible.ps1`](https://github.com/ansible/ansible-documentation/blob/devel/examples/scripts/ConfigureRemotingForAnsible.ps1))
passed as EC2 user-data at launch. AWS's stock Windows AMIs (EC2Launch v2)
do **not** enable WinRM by default - you must supply that script yourself.

### 2. Create the machine credential (not in git, on purpose)

The Windows host's local Administrator password is a real secret for a real
machine. It is **intentionally not stored in this repository**, encrypted
or otherwise - same reason you wouldn't commit a production database
password. Create it directly on the controller:

- Name: `AD Demo - Windows DC`
- Type: `Machine`
- Username / Password: the instance's Administrator credentials

### 3. Run `Bootstrap - AD Demo DC` once

Promotes the host to a new AD DS forest, creates a demo user, and **really**
locks that account out (via genuine failed logons against a lowered lockout
threshold - not a fake flag) so there's something real to unlock live.
Needs these extra vars at launch (also not stored in git):

```yaml
ad_domain_name: ad-demo.local
ad_domain_netbios_name: ADDEMO
ad_safe_mode_password: <choose one>
ad_demo_user_sam: jsmith
ad_demo_user_firstname: Jane
ad_demo_user_surname: Smith
ad_demo_user_password: <choose one>
```

### 4. Run `Setup - AD Demo - CAC`

Applies the inventory, hosts, project, job templates, and the workflow with
its survey and approval node.

### 5. Run the demo

Launch **AD Self-Service - Unlock Account**, type in `jsmith`, approve it,
watch it unlock.

## Going live with MECM

`MECM - Trigger Policy Refresh` uses `microsoft.mecm.site_ps_drive` and
`microsoft.mecm.client_action` (`RequestMachinePolicyNow`) against a real
Configuration Manager Primary Site Server - it has to run on that site
server directly, since the module drives the local Configuration Manager
PowerShell module. To wire it into the live workflow as a node after the
unlock succeeds:

1. Add a host for your MECM Primary Site Server to the `mecm_site_server`
   group in `controller_host_groups.yml`.
2. Set `mecm_site_code`, `mecm_provider_host`, and `mecm_device_name` as
   extra vars on the job template (or a workflow survey question).
3. Add a `node301` entry to `controller_templates_workflow.yml` under
   `node201`'s `success_nodes`, pointing at `MECM - Trigger Policy Refresh`.

Standing up a full MECM/SCCM site (SQL Server, Primary Site, WSUS, ADK) was
out of scope for this demo - see the project notes if you want to take that
on separately.

## Security notes

- WinRM (5985/5986) and RDP (3389) are open broadly on this instance's
  security group for demo convenience in an ephemeral sandbox account.
  Tighten this before using the pattern anywhere that isn't disposable.
- Rotate the Windows Administrator password and any AAP credentials tied to
  this environment once the demo is retired.
