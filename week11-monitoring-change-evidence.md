# Week 11: Monitoring and Change Evidence

## HarborTech Ticket Summary
Ticket TKT-2026-0011 comes from Riverside Goods. The client reports that its website slowed down after a change window and that the web tier Auto Scaling group no longer matches its approved capacity. No approved change request exists for the window, and nobody remembers making a change. My task was to reconstruct what happened from monitoring and audit evidence instead of memory, and to avoid naming a person or a cause the records do not support.

## Client Impact
The web tier went from three desired instances to one, and the maximum size dropped from four to one. This removed both redundancy and the ability to scale out. Latency rose about eightfold, from 0.3 s to 2.4 s, so customers see a slower storefront and are more likely to abandon pages or time out. Because the group can no longer grow, a traffic spike could cause an outage.

An unapproved change with no owner is also a security and audit concern. The client cannot show who changed production or why. The latency alarm fired only about four minutes after the change, so detection was fast. The problem is that no one knew a change had occurred.

## AWS Services Involved
- Amazon CloudWatch
- CloudWatch Metrics (TargetResponseTime)
- CloudWatch Logs (log group for the web application)
- CloudWatch Alarms (rg-web-latency-high)
- AWS CloudTrail
- AWS Config
- Amazon Athena (a candidate tool for querying CloudTrail history at larger scale; not required for this small packet)
- Monitoring
- Audit Evidence
- Configuration History

## Virtualization Connection
In a virtualized, software-defined environment, servers, scaling limits, and networks are settings that can be changed with one API call in seconds, with no physical trace. Monitoring and audit services make that invisible layer visible. CloudWatch shows how the resources behave. CloudTrail records which management API calls were made and by what identity. AWS Config records what the resource configuration was before and after. Together they allow a team to reconstruct a change that happened in seconds.

## Evidence Reviewed
- CloudWatch metrics: TargetResponseTime rose from 0.3 s to 2.4 s at 14:07:00.
- CloudWatch alarm history: rg-web-latency-high went from OK to ALARM at 14:09:00 (threshold above 1 s for 2 datapoints).
- Application logs: at 14:07:30 the web app log group showed a WARN for an upstream response of 2.38 s and one healthy target.
- CloudTrail management events: an UpdateAutoScalingGroup call at 14:05:12 with requested desiredCapacity 1 and maxSize 1.
- AWS Config resource history: the web Auto Scaling group changed from DesiredCapacity 3 to 1 and MaxSize 4 to 1, recorded at 14:06:03.
- Event timestamps: all five records sit within about four minutes. I am assuming they share one time zone.
- User or role identity: the call was made from an assumed-role session, which is a role, not a named person.
- API actions: UpdateAutoScalingGroup only.
- Resource state before and after the change: from Config (above).
- Athena query results: none were provided in this packet.
- Incident timeline: reconstructed below.

## Operational Analysis
**Incident window:** 14:05:12 (API call) through at least 14:09:00 (alarm). The packet has no later record, so the end of the incident is not established.

**Reconstructed timeline (event order):**
1. 14:05:12 - CloudTrail records UpdateAutoScalingGroup from an assumed-role session of a web operations role, with requested desiredCapacity 1 and maxSize 1.
2. 14:06:03 - AWS Config records the web Auto Scaling group changing from DesiredCapacity 3 to 1 and MaxSize 4 to 1.
3. 14:07:00 - CloudWatch shows TargetResponseTime rising from 0.3 s to 2.4 s.
4. 14:07:30 - Application log shows an upstream response of 2.38 s and only one healthy target.
5. 14:09:00 - The latency alarm goes from OK to ALARM.

**Findings (directly supported by evidence):**
- Finding: The Auto Scaling group configuration changed from desired 3 / max 4 to desired 1 / max 1. Config shows the before and after state, and CloudTrail shows a matching request.
- Finding: The API request was made by an assumed-role session of a web operations role, not by an identified individual.
- Finding: The config change preceded the latency rise, the slow-response log entry, and the alarm, in that order.
- Finding: Latency rose from 0.3 s to 2.4 s and breached the 1 s alarm threshold. Only one target was healthy at 14:07:30.
- Finding: No approved change request is on file for this window.

**Assumptions (reasonable but not confirmed):**
- Assumption: The reduced capacity caused the slowdown. The timing and the single healthy target fit, but no record proves causation (traffic, a deployment, or a dependency could also contribute).
- Assumption: All sources report in the same time zone and their clocks are in sync.
- Assumption: The CloudTrail call is the same change Config recorded. The values and timing match, but the packet does not link them by request ID.

**Unresolved questions:**
- Which person or automation used the role session? This needs the role-assumption event, the identity provider or SSO logs, and the session's source identity.
- Was the change deliberate, an automation or script error, or unauthorized?
- Was it authorized through some channel other than a change request?
- Did the alarm clear, and has capacity been restored? The packet shows no events after 14:09.
- The trail covers management events only, so it cannot show activity inside the application (data events).

## Recommendation
- Verify first: pull the role-assumption event for the session and review the identity provider logs to learn who or what used the role. Check whether any change record or maintenance window covers 14:05.
- Keep monitoring the alarm and the latency metric.
- Ask an authorized engineer or analyst to review the Auto Scaling configuration. Restoring the approved capacity (desired 3, max 4) is a production change, so an authorized person should decide and perform it.
- Keep the CloudTrail, Config, log, and alarm records intact as evidence.
- Within my authority as a junior operations intern, I recommend and document. I do not make the production change myself.

## Escalation Notes
Escalate to the on-call cloud analyst or engineer and to the operations manager:
- Restoring the Auto Scaling configuration, since this is a production change that needs approval.
- The unapproved change itself, since it has no change request.

Escalate to security if the role-session investigation shows the role was used unexpectedly, or if the session cannot be tied to a legitimate user or process. I am not accusing anyone. The evidence shows a role session, not a person, and attributing the action is a decision for security and management.

## Lessons Learned
- Start with a timeline and let the evidence align before naming a cause. A single suspicious timestamp is not proof.
- Each source answers a different question. Metrics and alarms show impact and timing, logs show application behavior, CloudTrail shows who or what made the API call, and Config shows what changed from what to what.
- Correlation is not causation. The evidence supports "the configuration changed, then performance degraded," but the cause remains an assumption until verified.
- Roles hide people. A role session needs additional records to connect it to a real user or process.
- Audit coverage has limits. A trail of management events cannot answer every question.
- Label every statement as a finding, assumption, recommendation, or escalation.

## Professional Vocabulary
- **Monitoring:** continuously collecting and watching data about the health and performance of systems.
- **Metric:** a numeric measurement sampled over time, such as response time.
- **Log:** a timestamped text record of events produced by an application or system.
- **Alarm:** a rule that watches a metric and changes state (such as OK to ALARM) when a threshold is crossed.
- **CloudWatch:** the AWS service for metrics, logs, and alarms.
- **CloudTrail:** the AWS service that records API calls and management activity, including who or what made them.
- **AWS Config:** the AWS service that records resource configurations and how they change over time.
- **Audit Trail:** a chronological record that lets you show who did what and when.
- **Configuration History:** the sequence of before and after states of a resource.
- **Incident Timeline:** the ordered list of events in an incident, built from evidence.
- **Evidence Correlation:** lining up records from separate sources by time and resource to see what they show together.
