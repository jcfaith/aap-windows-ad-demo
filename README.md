# AD Self-Service: Unlock Account, with Approval

**Ansible Automation Platform (AAP) applied to a real Active Directory
operation.** This repository contains everything needed to reproduce it in
your own environment.

No prior AAP experience required to follow along.

---

## The problem this solves

A user calls the helpdesk: "I'm locked out."

Today, that's a ticket, a queue, and eventually someone with Domain Admin
rights typing `Unlock-ADAccount` by hand - or a shared credential sitting in
a helpdesk runbook that nobody's rotated since 2019.

**With this pattern:**

1. Someone (helpdesk, the user's manager, or you) opens a form in AAP and
   types in the locked-out username. That's it - no PowerShell, no RDP,
   no domain admin login.
2. The user's manager gets an **approval request**. Nothing runs until they
   click Approve.
3. AAP unlocks the account using `microsoft.ad`, the **official
   Red Hat/Microsoft certified Ansible collection** for Active Directory -
   the same module family used for production AD management, not a
   toy script cobbled together for a one-off.
4. Every step - who asked, who approved, what actually ran, what changed -
   lands in AAP's job history automatically. No shared credentials, no
   "who unlocked this and why" mystery three weeks later.

## Why this matters

You already know how to unlock an AD account. That was never the hard part.
The hard part is everything *around* it:

| Today | With AAP |
|---|---|
| Whoever fixes it needs Domain Admin rights, or a shared service account | Only AAP's service account touches AD directly - people get delegated access to *one specific action*, not the keys to the domain |
| "Who approved this?" means checking Slack/email/a ticket, if it was documented at all | Approval is a required step in the workflow itself - it can't be skipped, and it's logged |
| Fixes happen by hand, so they're inconsistent between whoever's on call | Same certified module runs the same way every time |
| Auditors ask "show me every account unlock in Q3" and someone spends a day in ticket history | It's a filtered list in AAP's job history |
| Adding a second Windows task means writing another script, another set of creds to manage | Same platform, same credential vault, same approval pattern - the marginal cost of the next automation is small |

Account unlock is a deliberately small starting point. The pattern here -
survey → approval → certified-collection execution → audit trail - is
exactly what you'd reuse for password resets, group membership changes,
computer object cleanup, or anything else on your AD to-do list. And it's
not AD-specific - the same platform runs the same way against network
gear, cloud infrastructure, Linux fleets, and (see below) Microsoft
Endpoint Configuration Manager.

## What's actually running, technically

- **`microsoft.ad`** - the certified collection Microsoft and Red Hat
  maintain for AD management. This uses `microsoft.ad.user` with
  `account_locked: false` to unlock the account, and
  `microsoft.ad.domain` to stand up the lab domain controller.
- **`ansible.windows`** - certified collection, handles the WinRM
  connection to the Windows host.
- **`microsoft.mecm`** - also included, wired up as a real (not stubbed)
  job template for organizations that also run Microsoft Endpoint
  Configuration Manager. See "Extending to MECM" below.
- **A Workflow Job Template** - AAP's term for a chain of steps. This one
  has two: an approval step, then the actual unlock job.
- **A Survey** - the "form" the requester fills in (just a username field
  here). AAP surveys can do text, passwords, multiple choice, and more.
- **A Machine Credential** - AAP's encrypted, access-controlled way of
  storing the Windows login it uses. Nobody who triggers the workflow ever
  sees this credential - they don't need domain admin rights to trigger it.
- **Config-as-code** - the entire setup (inventory, credentials structure,
  job templates, the workflow itself) is defined as YAML in this repo and
  applied by one job template, not clicked together by hand. That's what
  makes this reusable - you're looking at the actual definition, not a
  screenshot.
- All of the above resolve through **Red Hat's Automation Hub**, the
  certified/validated content source - not the public community catalog.
  Same content model used in production.

## Reproduce this in your own environment

Any Windows Server 2022+ host reachable over WinRM works, on-prem or
cloud. These steps use AWS because that's what this build used.

### 1. Stand up a target Windows Server

Launch a Windows Server 2022 instance with WinRM enabled. AWS's stock
Windows AMIs don't turn WinRM on by default - pass the official Ansible
bootstrap script,
[`ConfigureRemotingForAnsible.ps1`](https://github.com/ansible/ansible-documentation/blob/devel/examples/scripts/ConfigureRemotingForAnsible.ps1),
as user-data (or run it manually if using an existing host).

### 2. Create a Machine credential in AAP

In AAP: Credentials → Add → type `Machine`. Point it at the Windows host's
local Administrator login. This is the one step that's deliberately *not*
config-as-code - a real password for a real machine doesn't belong in a
git repo, even encrypted.

### 3. Run `Bootstrap - AD Demo DC` (one time)

This job template (playbook: `playbooks/bootstrap_dc.yml`) promotes the
host to a new AD DS forest and creates a demo user. Supply these as extra
vars at launch time (also not stored in git):

```yaml
ad_domain_name: ad-demo.local
ad_domain_netbios_name: ADDEMO
ad_safe_mode_password: <choose one>
ad_demo_user_sam: jsmith
ad_demo_user_firstname: Jane
ad_demo_user_surname: Smith
ad_demo_user_password: <choose one>
```

To have something real to unlock afterward, lock the account out the same
way a real lockout happens - a few genuine failed logon attempts against a
lowered lockout threshold, not a fake "locked" flag. `bootstrap_dc.yml`
does this for you as part of the same run.

### 4. Run `Setup - AD Demo - CAC`

This applies everything else in `playbooks/files/config_as_code/` -
inventory, hosts, job templates, and the workflow with its survey and
approval node.

### 5. Launch the workflow

Launch **AD Self-Service - Unlock Account**, enter the locked-out username
in the survey, and approve the request. The job log confirms the account
was actually locked and is now unlocked (`changed: true` on the unlock
task - not a no-op).

## Extending to MECM

Organizations that also run Microsoft Endpoint Configuration Manager can
reach it through the same platform. `MECM - Trigger Policy Refresh` is a
real job template using `microsoft.mecm.site_ps_drive` and
`microsoft.mecm.client_action` to push an immediate machine policy refresh
- a natural follow-on after an unlock, so policy changes land without
waiting for the client's normal poll interval. It has to run directly on
the MECM Primary Site Server (that's how the module works - it drives the
Configuration Manager PowerShell module locally there).

To wire it into the live workflow:

1. Add your MECM Primary Site Server as a host in the `mecm_site_server`
   group (`controller_host_groups.yml`).
2. Set `mecm_site_code`, `mecm_provider_host`, and `mecm_device_name`.
3. Add a node to `controller_templates_workflow.yml` after the unlock step,
   pointing at `MECM - Trigger Policy Refresh`.

## A couple of practical notes

- This lab's Windows host has WinRM/RDP open broadly for convenience.
  Scope that down to specific source IPs for anything beyond a throwaway
  sandbox.
- Rotate the credentials used here (Windows Administrator, AD safe-mode
  password, demo user password) once you're done with them - none of them
  are stored in this repository, but they're real secrets while the
  environment is up.

## Questions

Talk to your Red Hat account team about extending this pattern to your own
AD environment - survey, approval, certified collection, audit trail is
the same shape you'd reuse for the next ten things on your AD/Windows
automation list.
