# IAM-lifecycle-hire-to-exit-Security-Architect
IAM lifecycle hire to exit Security Architect


A worked example of the identity lifecycle on AWS, following one employee, Sarwat,
through joining, daily work, elevated access, a team transfer, and resignation.

Why the lifecycle ???
Most cloud breaches involve stolen credentials, over-privileged accounts, or access that
nobody removed after a person changed roles or left. Good IAM design answers four
questions every time: Who is this?   Can they prove it? What can they do? Can we prove what happened? 


The scenario
Example Corp runs a multi-account AWS Organization. Sarwat Kadri joins the Security
Operations team as a Cloud Network Engineer, reporting to her manager, RL. Six
months later she moves to the Cloud Security team. Later still she resigns 


Design choices behind the example

HR is the single source of truth for who works at the company.
One identity provider (Microsoft Entra ID in this example, Okta works the same way) handles login and MFA.
AWS IAM Identity Center is the only way people reach AWS accounts. There are no
IAM users for people.
Service control policies (SCPs) set organisation-wide guardrails.
CloudTrail, Config, Access Analyzer and Security Hub provide the evidence.

Flow Chart is this way ( Need to Write)

Stage 1: Joiner

HR creates Sarwat's record. Every access decision that follows is derived from these
attributes, so poor HR data quality is itself a security risk

The HR hire event triggers the lifecycle process. It generates a unique username, checks
for duplicates, and creates Sarwat's identity in the identity provider in a staged state (no
login yet). Attribute-based rules place her in birthright groups only

>>all-employees
>>dept-networkops
>>aws-networkops-readonly
>
>>No administrator or sensitive access is granted automatically


Provisioning into AWS with SCIM

The identity provider pushes users and groups to IAM Identity Center using SCIM, so no
one creates accounts by hand. The group aws-netops-readonly is mapped to a
permission set called NetworkReadOnly in the accounts she needs.

Day one: credentials and device
a. Sarwat receives an activation link, verified out of band by HR or her manager.
b. She registers a passkey or FIDO2 security key as her primary factor, plus a backup
factor.
c. Her laptop is enrolled in device management and marked compliant.
d. A conditional access policy requires MFA and a compliant device for the AWS sign-in
application


1.4 First sign-in
Sarwat opens the AWS access portal, which redirects her to the identity provider. After
MFA and the device check, the identity provider sends a signed SAML assertion to
Identity Center. She picks an account and role, and AWS issues temporary credentials for
that role session.
For command-line work she uses SSO, not access keys:

Illustrative output:
{
"UserId": "AROAEXAMPLEID:sarwat.Kadri@example.com",
"Account": "111122223333",
"Arn": "arn:aws:sts::111122223333:assumedrole/AWSReservedSSO_SecOpsReadOnly_0123456789abcdef/sarwat.kadri@example.com"
}
The session name is her email, so every CloudTrail event can be traced back to her as a
person. No password or long-lived key exists anywhere in AWS for her


Stage 2: Day-to-day work with least privilege

Role-based baseline
NetOpsReadOnly lets her view findings, logs and configuration but not change workloads.
She can do most of her job without ever holding write access.

Attribute-based refinement
Where teams share accounts, tags narrow access further. Identity Center can pass
attributes such as department into the session, and policies compare them with resource
tags:
{
"Version": "2012-10-17",
"Statement": [
{
"Effect": "Allow",
"Action": ["ec2:StartInstances", "ec2:StopInstances"],
"Resource": "*",
"Condition": {
"StringEquals": {
"aws:ResourceTag/department": "${aws:PrincipalTag/department}"
}
}
}
]
}

Guardrails nobody can bypass
SCPs apply to every account in an organisational unit, including to administrators. A
typical example denies turning off the security services:
{
"Version": "2012-10-17",
"Statement": [
{
"Sid": "ProtectSecurityServices",
"Effect": "Deny",
"Action": [
"cloudtrail:StopLogging",
"cloudtrail:DeleteTrail",
"config:StopConfigurationRecorder",
"guardduty:DeleteDetector"
],
"Resource": "*",
"Condition": {
"ArnNotLike": { "aws:PrincipalArn":
"arn:aws:iam::*:role/SecurityBreakGlass" }
}
}
]
}
SCPs never grant access by themselves, they only set the maximum. They should be
tested on a sandbox unit first, because a bad SCP can lock people out.


Stage 3:Elevated access for an incident
A production incident needs Sarwat to collect evidence from a log cluster that her readonly role cannot touch.
1. She raises an access request in the governance portal, naming the account, the
permission set and the reason.
2. The system checks segregation of duties rules, for example that she cannot approve
her own request.
3. RL and the application owner approve.
4. A temporary assignment grants a higher-privilege permission set (for example
IncidentResponder ) for four hours, with MFA step-up at the identity provider.

5. The assignment expires automatically, and CloudTrail records everyAPI call under her
session name.
There are several ways to build the temporary assignment, including approval workflows
built on Identity Center's APIs. AWS publishes an open-source sample for temporary
elevated access, and you should review it against your own requirements before adopting
it.
Principle: standing privilege is a risk, and time-bound privilege is a control.


Stage 4: Sarwat builds a pipeline (workload identity)
Sarwat sets up a deployment pipeline for a security tool. Instead of creating an IAM user
with access keys, she uses OIDC federation so the pipeline gets temporary credentials.
The role's trust policy pins both the audience and the exact repository and branch:


Stage 5: Mover (six months later)
Sarwat transfers to Cloud Security under a new manager. HR updates her department
and manager, which triggers the mover process.
1. The lifecycle system compares the old and new birthright access.

2. It removes dept-Netops and aws-Netops-readonly , then adds dept-secnet and
aws-sec-engineers .

3.SCIM updates reach Identity Center, which removes the OLD
assignments and adds the NEW permission set.

4.Her department attribute changes, so tag-based access follows
5.Any elevated access is flagged to the new manager for re-certification.
6. IAM Access Analyzer's unused access findings help trim permissions she never used


Stage 6: Leaver (Sarwat resigns)
HR records the termination, and the process runs in order.

Stage 7: Continuous governance



