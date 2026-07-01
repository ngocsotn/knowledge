# Active-Cold Standby DR Pattern

An Active-Cold standby is a disaster recovery topology where the backup (passive) infrastructure remains completely powered down or un-configured until a primary site outage occurs.
- **RTO (Recovery Time Objective):** High (can take hours to build, configure, and route traffic to the site).
- **RPO (Recovery Point Objective):** High (recovers data from the last daily or weekly offline storage backups).
- **Cost:** Extremely low since backup virtual hardware is not running.

## Interview Questions & Answers

### Q1: When is an Active-Cold standby topology justified?
- **Answer:** In non-critical enterprise applications, internal dev environments, or legacy archiving systems where high downtime is completely tolerable and minimizing cloud infrastructure running costs is the primary objective.
