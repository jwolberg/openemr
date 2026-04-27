Overview: Perform an component audits of the codebase across 5 segments:
1. Architecture -   document how the system is organized, where data lives, how the layers interact, and what the integration points are for adding new capabilities. 

2. Security audit — identify authentication and authorization risks, data exposure vectors, PHI handling issues, and any HIPAA-relevant gaps in the current system. 

3. Performance audit — understand where the system is slow, what the bottlenecks are, how the data is structured, and what constraints will affect response latency. 

4. Data Quality audit — find how complete, consistent, and reliable that data actually is. Missing fields, inconsistent formatting, duplicate records, and stale data all become agent failure modes. 

5. Compliance & Regulatory audit —  HIPAA logging requirements, data retention policies, breach notification obligations, and BAA (Business Associate Agreement) implications of sending PHI to an LLM provider. 


additionally, include in output:

-  Intgration: Recommended integration points for an AI Clinical Co-Pilot
- Risks: Risk table: severity, finding, evidence, affected files, recommendation
- Open: Open questions / assumptions

Do not propose new features yet. This is audit only. 
Prefer concrete file references and code-path observations over generic advice.


output file: docs/AUDIT_RAW.md




Examine these folders/files:
- apis/
- oauth2/
- gacl/
- interface/
- library/
- src/
- portal/
- sites/default/
- sql/
- db/
- controllers/
- templates/
- config/
- docker/
- _rest_routes.inc.php
- index.php
- controller.php
- bootstrap.php
- composer.json / composer.lock
- package.json / package-lock.json

Ignore generated/vendor/test/build/documentation folders unless needed to validate a finding:
- vendor/
- node_modules/
- tests/
- ci/
- .github/
- Documentation/
- public/assets/
- sites/default/documents/
- sites/default/cache/
- sites/default/logs/



1. Architecture

Start with:

index.php
bootstrap.php
controller.php
_rest_routes.inc.php
src/
library/
interface/
apis/
portal/

Goal: understand request flow, session initialization, routing, service layer, database access, and API boundaries.


2. Security / auth / authorization

Focus on:

interface/login/
oauth2/
gacl/
apis/
portal/
library/
src/
sites/default/

Look for:

session handling
password/auth flow
role checks
ACL enforcement
API token validation
OAuth scopes
CSRF protection
escaping/sanitization
direct file access
PHI exposure
config secrets
patient portal isolation



3. PHI and compliance

Focus on:

interface/patient_file/
interface/forms/
interface/reports/
interface/billing/
portal/
apis/
src/
sql/
sites/default/

Look for:

where patient data is read/written
where encounter notes live
where documents/uploads live
audit logging behavior
export/download paths
data retention assumptions
LLM/third-party data transfer risks



4. Performance

Focus on:

sql/
db/
src/
library/
apis/
interface/reports/
interface/patient_file/
interface/billing/

Look for:

large joins
missing indexes
N+1 query patterns
report queries
patient search
encounter retrieval
API response bottlenecks
document/file access
agent latency constraints



5. Data quality

Focus on:

sql/
db/
interface/patient_file/
interface/forms/
src/
apis/

Look for:

required vs optional fields
duplicate patient risks
inconsistent date formats
stale encounter data
free-text note structure
coding systems
missing demographics
unreliable medication/problem/allergy data

