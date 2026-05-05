# Office 365 Basic Hygiene Checkup

This guide covers basic email authentication hygiene for Microsoft 365 tenants that are not using a dedicated phishing gateway or the higher-end Microsoft Defender for Office 365 features.

The goal is to make your own domain harder to spoof by publishing SPF, enabling DKIM signing, and rolling out DMARC carefully. Test changes before enforcing them, especially if third-party services send mail as your domain.

## Preparation

### What You Need To Know

1. Expect the process to take at least a week if your organization uses multiple mail-sending services.
2. SPF, DKIM, and DMARC only work well when you account for every legitimate sender that uses your domain.
3. Start with monitoring, fix legitimate failures, then move toward enforcement.
4. Microsoft changes admin portals often. When possible, use Exchange Online PowerShell to retrieve tenant-specific values.

### What You Need

1. Administrative access to your DNS provider.
2. Microsoft 365 admin access.
3. A mailbox or DMARC reporting service that can receive aggregate reports.
4. A list of all services that send mail for your domain, including Microsoft 365, ticketing systems, CRMs, payroll systems, marketing platforms, websites, and scanners.

## Do Not Use a Catch-All Mailbox as a Hygiene Control

Older versions of this guide recommended creating a catch-all mailbox by changing the accepted domain to `Internal Relay` and redirecting mail that did not match a dynamic group of users. Do not use that approach as a basic hygiene control.

That design is risky because it can redirect legitimate mail for shared mailboxes, Microsoft 365 groups, distribution lists, mail contacts, resource mailboxes, aliases, and other recipient types that are not captured by a simple "all users" group. It also changes accepted-domain behavior in a way that Microsoft normally uses for split-domain or relay scenarios.

For normal Microsoft 365 tenants:

- Keep your accepted domain as `Authoritative` when all valid recipients are in Microsoft 365.
- Let Exchange reject mail to non-existent recipients.
- Create aliases for common misspellings only when there is a documented business need.
- Use message trace and user reporting to investigate suspicious mail instead of collecting all mail to invalid addresses.

Reference: [Accepted domains in Exchange](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)

## SPF

### What Is It?

Sender Policy Framework (SPF) tells receiving mail systems which servers are authorized to send mail for your domain.

SPF checks the envelope sender domain, not necessarily the visible `From` address that users see. DMARC is what ties SPF or DKIM authentication back to the visible `From` domain through alignment.

### Do I Have It?

Enter your domain into an SPF testing tool such as <https://mxtoolbox.com/spf.aspx>. Confirm that:

1. Exactly one SPF TXT record exists for the domain.
2. It includes every legitimate sender.
3. It does not exceed the SPF DNS lookup limit.
4. It ends with an enforcement rule such as `-all` or `~all`.

Microsoft recommends `-all` for Microsoft 365 domains when DKIM and DMARC are also configured, because DMARC reporting lets you validate failures and move toward enforcement.

Example for a domain that sends only from Microsoft 365:

```text
v=spf1 include:spf.protection.outlook.com -all
```

If you use other services, include them too. Example only:

```text
v=spf1 include:spf.protection.outlook.com include:mail.example-saas.com ip4:203.0.113.10 -all
```

Reference: [Set up SPF for Microsoft 365](https://learn.microsoft.com/en-us/microsoft-365/security/office-365-security/how-office-365-uses-spf-to-prevent-spoofing?view=o365-worldwide)

## DKIM

### What Is It?

DomainKeys Identified Mail (DKIM) signs outbound mail with a private key. Receiving mail systems verify the signature with public DNS records. DKIM is especially important because forwarded mail often breaks SPF, but DKIM can survive forwarding when the message is not modified.

### Do I Have It?

Check DKIM in the Microsoft Defender portal or Exchange Online PowerShell. You can also use a DKIM validation tool such as <https://mxtoolbox.com/dkim.aspx>, but the selector value must match the selector Microsoft is using for your domain.

Use PowerShell to inspect all configured domains:

```powershell
Get-DkimSigningConfig | Format-List Name,Enabled,Status,Selector1CNAME,Selector2CNAME
```

For one domain:

```powershell
Get-DkimSigningConfig -Identity widgets.com |
  Format-List Name,Enabled,Status,Selector1CNAME,Selector2CNAME
```

### How Do I Configure It?

Do not manually derive DKIM CNAME targets from examples. Microsoft introduced a newer DKIM CNAME format for new custom domains in May 2025, and existing domains can still use the older format. The safe process is to retrieve the exact `Selector1CNAME` and `Selector2CNAME` values from Microsoft 365 for each domain.

If the domain does not already have a DKIM signing config, create it first:

```powershell
New-DkimSigningConfig -DomainName widgets.com -Enabled $false
```

Then retrieve the required CNAME values:

```powershell
Get-DkimSigningConfig -Identity widgets.com |
  Format-List Selector1CNAME,Selector2CNAME
```

Create two CNAME records at your DNS provider:

```text
Host name: selector1._domainkey
Points to: <Selector1CNAME value from Microsoft 365>

Host name: selector2._domainkey
Points to: <Selector2CNAME value from Microsoft 365>
```

After DNS has propagated, enable DKIM signing:

```powershell
Set-DkimSigningConfig -Identity widgets.com -Enabled $true
```

Verify status:

```powershell
Get-DkimSigningConfig -Identity widgets.com |
  Format-List Name,Enabled,Status
```

Reference: [Configure DKIM for Microsoft 365](https://learn.microsoft.com/en-us/defender-office-365/email-authentication-dkim-configure)

## DMARC

### What Is It?

Domain-based Message Authentication, Reporting, and Conformance (DMARC) checks whether SPF or DKIM passed and aligned with the visible `From` domain. It also tells receiving systems what to do when a message fails DMARC.

A message passes DMARC if either aligned SPF or aligned DKIM passes. A message fails DMARC if both fail.

DMARC can generate a lot of aggregate reports. Use a DMARC reporting service or a mailbox with an automated parser; raw XML reports are difficult to review manually at scale.

### Do I Have It?

Check for a TXT record at:

```text
_dmarc.widgets.com
```

You can use a DMARC lookup tool such as <https://mxtoolbox.com/dmarc.aspx>.

### How Do I Configure It?

Start with monitoring:

```text
Host name: _dmarc
TXT value: v=DMARC1; p=none; pct=100; rua=mailto:dmarc-reports@widgets.com
```

Review reports, identify all legitimate sources, and fix SPF or DKIM alignment issues. Then move to quarantine:

```text
Host name: _dmarc
TXT value: v=DMARC1; p=quarantine; pct=25; rua=mailto:dmarc-reports@widgets.com
```

Increase `pct=` gradually:

```text
pct=50
pct=75
pct=100
```

After quarantine is stable, move to reject:

```text
Host name: _dmarc
TXT value: v=DMARC1; p=reject; pct=100; rua=mailto:dmarc-reports@widgets.com
```

Repeat the process for subdomains and lower-volume domains before enforcing the parent domain.

Reference: [Set up DMARC for Microsoft 365](https://learn.microsoft.com/en-us/defender-office-365/email-authentication-dmarc-configure)

## IPv6

If you send mail over IPv6, publish matching SPF `ip6:` mechanisms or service includes for those IPv6 senders. Microsoft 365 support for inbound anonymous IPv6 mail has changed over time, so review current Microsoft guidance before opening support tickets or changing connectors.

Reference: [Support for anonymous inbound email messages over IPv6](https://learn.microsoft.com/en-us/defender-office-365/mail-flow-about)
