# TIOP Functional Analysis

## Overview

**TIOP (Traceability-Interoperability-Platform)** is a semi-centralized data-sharing mechanism for exchanging serialized pharmaceutical traceability data (EPCIS events) to meet regulatory requirements in GHSC-PSM countries (global health supply chain).

The system was created as a pilot to gather lessons learned about:
- Event data requirements and data-sharing mechanisms
- Volume of data and dynamic nature of event data
- Message syntax and structure for global health traceability

## Architecture

TIOP implements a **5-stage event-driven pipeline** using AWS Lambda functions, triggered by S3 file uploads.

### Technology Stack

- **Runtime**: Java 11 (AWS Lambda)
- **Storage**: AWS S3 (file storage), AWS RDS MySQL (metadata/business rules)
- **Repository**: OpenSearch/Elasticsearch (EPCIS event storage)
- **Infrastructure**: AWS CloudFormation, EC2, Route53, SFTP
- **Data Format**: EPCIS 1.2 XML (input) → EPCIS 2.0 JSON (storage)

## Core Functional Pipeline

### 1. Authentication (`AuthLambdaHandler`)

**Purpose**: Validate incoming documents against business rules

**Process**:
- Triggered when XML file lands in S3 bucket
- Parses EPCIS XML document
- Extracts metadata:
  - Source GLN (manufacturer location identifier)
  - Destination GLN (recipient country identifier)
  - GTIN (product identifier)
  - ObjectEvent count
  - AggregationEvent count
- Queries MySQL database for matching business rule:
  ```sql
  SELECT ti.gtin_uri FROM tiop_rule tr
  WHERE sl.gln_uri = '<source>'
  AND dl.gln = '<destination>'
  AND ts.status_description = 'Active'
  ```
- Validates all EPCs in the document match authorized GTINs
- Inserts operation record with status "Received"
- Invokes next Lambda (Validation)

**Error Handling**:
- If GLN/GTIN combination not found → Send email alert (EXC001)
- Logs error to `tiop_operation` table with status "auth failed"

**Reference**: `src/main/java/com/usaid/AuthLambdaHandler.java:39`

---

### 2. Validation (`ValidateLambdaHandler`)

**Purpose**: Perform XSD schema validation against EPCIS standards

**Process**:
- Receives S3 file location from Auth Lambda
- Loads EPCIS XSD schemas from classpath:
  - `EPCglobal-epcis-1_2.xsd`
  - `StandardBusinessDocumentHeader.xsd`
  - `EPCglobal.xsd`
  - Other supporting schemas
- Validates XML document structure and data types
- Invokes next Lambda (Transformation) if valid

**Error Handling**:
- Schema validation failures → Send email alert with error details
- Logs error to database with validation error message
- Process terminates

**Reference**: `src/main/java/com/usaid/ValidateLambdaHandler.java:54`

---

### 3. Transformation (`TransfromLambdaHandler`)

**Purpose**: Convert EPCIS XML 1.2 to EPCIS JSON 2.0

**Process**:
- Reads XML document from S3
- Calls external Document Conversion API:
  ```
  POST <conversion-api-url>
  Content-Type: application/xml
  GS1-EPC-Format: Always_GS1_Digital_Link
  ```
- Parses JSON response
- Validates event counts match between XML and JSON:
  - ObjectEvent count must match
  - AggregationEvent count must match
- Writes JSON file to destination S3 bucket
- Inserts operation record with status "Received" (for JSON)

**Error Handling**:
- HTTP 400/500 from conversion API → Email alert (EXC008/EXC009)
- Event count mismatch → Email alert (EXC007)
- Logs error to database with transformation status

**Reference**: `src/main/java/com/usaid/TransfromLambdaHandler.java:45`

---

### 4. Bulk Load (`BulkLoadLambdaHandler`)

**Purpose**: Load deduplicated events into OpenSearch repository

**Process**:
- Triggered by JSON file landing in S3
- Reads JSON document
- Extracts shipping event metadata (destination, source, GTIN)
- **Deduplication**:
  - Generates SHA-256 hash for each event (whitespace removed)
  - Queries database for existing hashes: `SELECT hash FROM event_hash`
  - Filters out duplicate events
- Enriches events with TIOP context:
  ```json
  {
    "tiop:psa": "<billto_gln>",
    "@context": [{"tiop": "https://ref.opentiop.org/epcis/"}]
  }
  ```
- Bulk loads to OpenSearch using `_bulk` API:
  ```
  POST <opensearch-url>/_bulk
  {"index": {"_index": "epcis_index"}}
  {event-json}
  ```
- Stores new event hashes in MySQL `event_hash` table
- Moves processed XML and JSON files to "processed" S3 buckets
- Deletes source files

**Error Handling**:
- OpenSearch errors → Email alert (EXC012), log to database
- Failed events removed from hash insert
- File move failures → Email alert (EXC013)

**Reference**: `src/main/java/com/usaid/BulkLoadLambdaHandler.java:49`

---

### 5. Routing (`RouterLambdaHandler`)

**Purpose**: Route processed documents to destination country systems

**Process**:
- Queries database for destination routing configuration:
  ```sql
  SELECT routing_secret_name FROM router_info
  WHERE country_gln = '<destination>'
  ```
- Retrieves country-specific credentials from AWS Secrets Manager
- Reads original XML from S3
- POSTs XML to country endpoint:
  ```
  POST <country-api-url>
  Content-Type: application/xml
  Authorization: Bearer <token>
  ```
- Inserts operation record with status "Routed"

**Error Handling**:
- HTTP 401 → Email alert with authentication error
- HTTP 400/500 → Email alert (EXC014) with error details
- Missing routing config → Email alert (EXC010)

**Reference**: `src/main/java/com/usaid/RouterLambdaHandler.java:34`

---

## Data Model

### Core Tables (MySQL)

1. **`trading_partner`**
   - Partners: manufacturers, procurement agents, countries
   - Hierarchical (parent-child relationships)

2. **`location`**
   - Physical locations (warehouses, facilities)
   - Identified by GLN (Global Location Number)
   - Linked to trading partners

3. **`trade_item`**
   - Products/pharmaceuticals
   - Identified by GTIN (Global Trade Item Number)
   - Includes: description, strength, dosage form, ATC code

4. **`tiop_rule`**
   - Defines authorized data flows
   - Links: source_location → destination_location → item
   - Status: Active/Inactive

5. **`tiop_operation`**
   - Audit trail for all document processing
   - Tracks: event counts, status, errors, timestamps
   - Status values:
     - 3: Auth failed
     - 5: Transformation failed
     - 6: Received (Auth)
     - 7: Received (Transformation)
     - 11: Bulkload failed
     - (Routing status)

6. **`event_hash`**
   - Stores SHA-256 hashes for deduplication
   - Prevents duplicate events in OpenSearch

7. **`router_info`**
   - Country routing configuration
   - Links GLN to AWS Secrets Manager secret name

**Reference**: `db/tiop_ddl.sql`

---

## OpenSearch Repository

### Index: `epcis_index`

**Structure**:
```json
{
  "type": "ObjectEvent | AggregationEvent",
  "eventTime": "2024-01-01T12:00:00Z",
  "bizStep": "urn:epcglobal:cbv:bizstep:shipping",
  "bizLocation": {
    "id": "urn:epc:id:sgln:..."
  },
  "epcList": ["urn:epc:id:sgtin:..."],
  "tiop:nts_gln": "0614141999996",
  "tiop:billto_gln": "0614141888885",
  "tiop:psa": "0614141888885",
  "@context": [{
    "tiop": "https://ref.opentiop.org/epcis/"
  }]
}
```

**Key Fields**:
- Standard EPCIS 2.0 fields (type, eventTime, bizStep, etc.)
- Custom TIOP extensions:
  - `tiop:nts_gln`: Destination country GLN
  - `tiop:billto_gln`: Procurement service agent GLN
  - `tiop:psa`: PSA identifier

**Reference**: `epcis-repo-scripts/epcis-repo-index.json`

---

## Error Handling & Notifications

### Email Notifications

All stages send standardized error emails on failures:

**Format**:
```
Subject: [ENV] File Processing Issue: [filename] - Attention Needed

Body:
An issue [EXC###] encountered while processing the file <filename>
which was received on <date>.

Details of the Issue:
<error message>

TIOP operation team
```

**Error Codes**:
- EXC001: Business rule validation failure (GLN/GTIN combo not found)
- EXC007: Event count mismatch (XML vs JSON)
- EXC008: HTTP 400 from conversion API
- EXC009: HTTP 500 from conversion API
- EXC010: Missing routing configuration
- EXC011: Bulkload exception
- EXC012: OpenSearch error response
- EXC013: File move/delete failure
- EXC014: Routing API error

### Database Logging

Every error logs to `tiop_operation` table with:
- Event type, source/destination partners, locations
- Item (GTIN)
- Rule ID (if matched)
- Status (failed stage)
- Document name
- Event counts
- Exception detail (error message)
- Timestamps and creator IDs

---

## Key Functional Requirements

### 1. Data Ingestion
- Accept EPCIS 1.2 XML documents via SFTP → S3

### 2. Authorization
- Validate source, destination, and product combinations
- Enforce business rules for data sharing

### 3. Data Quality
- Schema validation against EPCIS standards
- Event count verification across transformations

### 4. Standardization
- Convert legacy XML 1.2 to modern JSON 2.0 format
- Enrich with custom TIOP context

### 5. Deduplication
- Prevent duplicate events using cryptographic hashing
- Maintain hash registry in database

### 6. Centralized Repository
- Store all events in searchable OpenSearch index
- Support queries by GLN, GTIN, date, bizStep, etc.

### 7. Distribution
- Route documents to destination country systems
- Support country-specific authentication

### 8. Audit & Compliance
- Complete audit trail of all operations
- Error tracking and notification
- Operation metadata (timestamps, counts, status)

### 9. Monitoring & Alerting
- Email notifications for all failures
- Detailed error messages with context
- Error categorization (auth, validation, transformation, etc.)

---

## Integration Points

### Input
- **SFTP Server** → S3 bucket (source XML files)
- Trading partners upload EPCIS XML documents

### Output
- **Country Systems**: HTTP POST of XML documents
- **OpenSearch Dashboard**: Query/visualization interface
- **Email**: SMTP for error notifications

### External Dependencies
- **Document Conversion API**: Proprietary XML→JSON conversion service
- **AWS Secrets Manager**: Credential storage
- **AWS RDS**: Business rules and metadata
- **OpenSearch**: Event repository

---

## Limitations & Considerations

### Current Architecture Limitations

1. **Tight AWS Coupling**: Heavy use of AWS-specific services (Lambda, S3, Secrets Manager)
2. **External Transformation Dependency**: Relies on external API for XML→JSON conversion
3. **Cold Start Latency**: Lambda cold starts can delay processing
4. **Limited Retry Logic**: Basic error handling, no sophisticated retry strategies
5. **Manual Infrastructure**: Requires CloudFormation deployment and management
6. **Deduplication Scalability**: Full hash table scan for every document
7. **No Real-time Visibility**: Limited monitoring of in-flight operations

### Data Volume Considerations

Based on the code:
- Processes documents with thousands of events (e.g., "4K_events_05062024.xml")
- Hash table grows indefinitely (no TTL or cleanup)
- Bulk API calls can be large (no batching limits observed)

---

## Deployment Architecture

```
┌─────────────┐
│ SFTP Server │
└──────┬──────┘
       │
       ▼
┌─────────────────┐
│  S3 (XML files) │
└────────┬────────┘
         │ Trigger
         ▼
┌──────────────────┐
│  1. Auth Lambda  │◄──── MySQL (business rules)
└────────┬─────────┘
         │ Invoke
         ▼
┌──────────────────┐
│  2. Validate     │◄──── XSD Schemas
│     Lambda       │
└────────┬─────────┘
         │ Invoke
         ▼
┌──────────────────┐
│  3. Transform    │◄──── Conversion API
│     Lambda       │
└────────┬─────────┘
         │ Write
         ▼
┌──────────────────┐
│ S3 (JSON files)  │
└────────┬─────────┘
         │ Trigger
         ▼
┌──────────────────┐
│  4. BulkLoad     │◄──── MySQL (event_hash)
│     Lambda       │────► OpenSearch
└────────┬─────────┘
         │ Invoke
         ▼
┌──────────────────┐
│  5. Router       │◄──── Secrets Manager
│     Lambda       │────► Country APIs
└──────────────────┘
```

---

## Comparison: TIOP vs OpenFn Lightning

| Capability | TIOP | OpenFn Lightning | Assessment |
|------------|------|------------------|------------|
| **Data Reception** | S3 file drops (XML) via SFTP | HTTP webhooks, polling, message queues, S3 triggers | ✅ Equivalent - Lightning has broader input options |
| **Authentication/Authorization** | Database rule validation (GLN/GTIN combos) | OAuth, API keys, custom auth logic in jobs | ✅ Can implement custom validation logic |
| **Schema Validation** | XSD validation for EPCIS 1.2 | Custom validation in jobs (JSON Schema, custom code) | ✅ Can implement with libraries |
| **Data Transformation** | XML→JSON via external API | Built-in adaptors + custom JavaScript | ✅ Better - no external API needed, can use libraries |
| **Deduplication** | SHA-256 hashing in MySQL | Can implement in workflow state or external DB | ✅ Can implement, more flexible |
| **Data Storage/Repository** | OpenSearch/Elasticsearch | Connects to any DB, can use OpenSearch adaptor | ✅ Equivalent |
| **Routing/Distribution** | HTTP POST to country endpoints | HTTP adaptors to multiple destinations | ✅ Equivalent with better error handling |
| **Error Handling** | Email notifications + DB logging | Built-in error handling, notifications, retry logic | ✅ Better - more robust and configurable |
| **Audit Trail** | MySQL operation tracking | Full run history, logs, state management | ✅ Better - built-in versioning |
| **Infrastructure Management** | AWS Lambda, S3, RDS, OpenSearch, CloudFormation | Managed cloud service or self-hosted | ✅ Simpler ops - less infrastructure to manage |
| **Business Logic Updates** | Code changes + Lambda redeployment | Visual workflow editor, hot-swap jobs | ✅ Better - faster iteration |
| **Observability** | CloudWatch logs | Built-in Inspector, logs, work order history | ✅ Better - integrated debugging tools |
| **Credential Management** | AWS Secrets Manager | Built-in credential vault | ✅ Equivalent |
| **Scalability** | Lambda auto-scaling (with cold starts) | Platform auto-scaling | ✅ Better - no cold starts |
| **Cost Model** | Pay per Lambda invocation + storage | Subscription based on usage tier | ⚠️ Depends on volume |

### Key Advantages of Using OpenFn Lightning

1. **No Custom Code Required**: The 1,500+ lines of Java Lambda code can be replaced with OpenFn workflows using pre-built adaptors
2. **Better Orchestration**: Lightning's workflow engine handles the 5-stage pipeline natively with proper error handling, retries, and branching
3. **Faster Development**: Visual workflow builder and JavaScript vs. Java compilation/deployment
4. **Real-time Processing**: Can process documents immediately on upload vs. Lambda cold starts (can add seconds)
5. **Better Observability**: Built-in monitoring, logs, Inspector for debugging, work order history
6. **Credential Security**: Built-in credential storage with role-based access
7. **Flexible Triggers**: Can support multiple input methods (S3, HTTP, polling, webhooks)
8. **Version Control**: Built-in workflow versioning and rollback
9. **Testing**: Built-in workflow testing and simulation tools
10. **Lower Maintenance**: Less infrastructure to manage (no Lambda packaging, deployment, VPC configuration)

### Considerations for Migration

**Advantages of Current TIOP Architecture**:
- Already deployed and operational
- Team familiar with Java/AWS ecosystem
- Existing CloudFormation infrastructure
- Proven at current scale

**Migration Complexity**:
- Moderate: Workflow logic is straightforward, but will need to replicate:
  - XSD validation (can use XML validation libraries)
  - Hash-based deduplication logic
  - Database queries for business rules
  - Email notification formatting

**Recommended Approach**:
- Run parallel systems initially
- Migrate one stage at a time
- Start with Router (simplest) to prove concept
- Move Auth/Validation/Transform together
- BulkLoad last (most critical)

---

## Conclusion

TIOP implements a solid, functional pipeline for pharmaceutical traceability data sharing. The core requirements—authentication, validation, transformation, deduplication, storage, and routing—are well-defined and could be effectively implemented in OpenFn Lightning with several advantages:

- Reduced operational complexity
- Better developer experience
- Improved error handling and observability
- More flexible integration options
- Easier maintenance and updates

The main challenge would be replicating the EPCIS XSD validation and ensuring the deduplication logic performs well at scale.
