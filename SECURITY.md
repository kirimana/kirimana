# Security Policy

## Supported versions

Kirimana is in private preview (v0.9). Security fixes apply to the current preview release and to v1.0 once shipped.

## Reporting a vulnerability

If you find a security issue — in our code, our documentation, or in this repository's infrastructure — please report it privately, **not** via public issue.

**Email:** `security@kirimana.io`

We aim to respond within 48 hours on business days. We follow a coordinated-disclosure model: we will work with you on a remediation timeline (default 90 days) before public disclosure, and we will credit you in the security advisory unless you prefer otherwise.

## Scope

The following are in scope:

- Vulnerabilities in Kirimana code (when source is published)
- Vulnerabilities in adapter implementations we maintain
- Issues that allow contracts or audit records to be tampered with
- Issues that bypass the AI Gateway trust ladder

The following are out of scope:

- Issues in third-party platforms we integrate with (please report to the platform vendor)
- Issues affecting Kirimana running on unsupported configurations

## Security updates

During preview, security fixes ship as new tagged versions on a "best speed" basis. From v1.0, we will publish security advisories via GitHub Security Advisories and the changelog.
