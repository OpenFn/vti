# OpenFn Lightning Workflow Design for TIOP

## Executive Summary

This document outlines the proposed OpenFn Lightning implementation to replace the current AWS Lambda-based TIOP architecture. The design maintains all existing functionality while improving maintainability, observability, and developer experience.

---

## Architecture Overview

### Workflow Structure

**Single Workflow**: "TIOP Document Processing Pipeline"
- **Trigger**: S3 file upload (via webhook or polling) OR HTTP endpoint
- **Steps**: 6 sequential steps mapping to TIOP's 5 Lambda stages
- **Adaptors Used**:
  - `@openfn/language-http` (API calls)
  - `@openfn/language-postgresql` (database queries)
  - `@openfn/language-opensearch` (event storage)
  - Custom JavaScript for validation/transformation

### High-Level Flow

```
┌─────────────────────────────────────────────────────────┐
│ Trigger: S3 Upload or HTTP POST                        │
│ Input: EPCIS XML document, filename, bucket info        │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ Step 1: Parse & Extract Metadata                       │
│ - Parse XML                                             │
│ - Extract source GLN, destination GLN, GTIN             │
│ - Count ObjectEvent and AggregationEvent                │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ Step 2: Authenticate Against Business Rules            │
│ - Query PostgreSQL for matching tiop_rule               │
│ - Validate EPCs against authorized GTINs                │
│ - Insert operation record (status: "Received")          │
│ - If invalid → Send email & fail                        │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ Step 3: Validate Against EPCIS Schema                  │
│ - Load XSD schemas                                      │
│ - Validate XML structure                                │
│ - If invalid → Send email & fail                        │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ Step 4: Transform XML to JSON                           │
│ - Call external conversion API OR use library           │
│ - Verify event counts match                             │
│ - If mismatch → Send email & fail                       │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ Step 5: Deduplicate & Load to OpenSearch               │
│ - Hash each event (SHA-256)                             │
│ - Query PostgreSQL for existing hashes                  │
│ - Filter duplicates                                     │
│ - Enrich with TIOP context                              │
│ - Bulk load to OpenSearch                               │
│ - Insert new hashes to PostgreSQL                       │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ Step 6: Route to Destination Country System            │
│ - Query router_info for destination config              │
│ - Retrieve credentials from Lightning vault             │
│ - POST XML to country endpoint                          │
│ - Insert operation record (status: "Routed")            │
│ - If error → Send email & fail                          │
└─────────────────────────────────────────────────────────┘
```

---

## Detailed Step Design

### Step 1: Parse & Extract Metadata

**Purpose**: Parse incoming XML and extract key identifiers

**Adaptor**: Custom JavaScript (using built-in XML parser)

**Input**:
```json
{
  "filename": "manufacturer/4K_events_05062024.xml",
  "bucket": "tiop-source-xml",
  "xmlContent": "<epcis:EPCISDocument>...</epcis:EPCISDocument>"
}
```

**Code**:
```javascript
// Parse XML and extract metadata
fn(state => {
  const { DOMParser } = require('xmldom');
  const { select } = require('xpath');

  const doc = new DOMParser().parseFromString(state.data.xmlContent);

  // Extract event counts
  const objectEvents = select("//ObjectEvent", doc);
  const aggregationEvents = select("//AggregationEvent", doc);

  // Extract shipping event to get source/destination
  const shippingEvents = select("//ObjectEvent[bizStep[contains(., 'shipping')]] | //AggregationEvent[bizStep[contains(., 'shipping')]]", doc);

  let sourceGLN = null;
  let destinationGLN = null;
  let billToGLN = null;
  let gtin = null;

  if (shippingEvents.length > 0) {
    const shippingEvent = shippingEvents[0];

    // Extract source (bizLocation)
    const bizLocationId = select("string(./bizLocation/id)", shippingEvent);
    if (bizLocationId) {
      // Extract GLN from URN format: urn:epc:id:sgln:0614141.00001.0
      sourceGLN = extractGLNFromUrn(bizLocationId);
    }

    // Extract destination (tiop:nts_gln)
    destinationGLN = select("string(./tiop:nts_gln)", shippingEvent);

    // Extract bill-to GLN
    billToGLN = select("string(./tiop:billto_gln)", shippingEvent);
  }

  // Extract GTIN from first EPC
  const firstEpc = select("string(.//epc)", doc);
  if (firstEpc) {
    // Extract GTIN from URN format: urn:epc:id:sgtin:0614141.012345.1234567890
    // Format: urn:epc:id:sgtin:<company>.<indicator+item>.serial
    const gtinUri = firstEpc.substring(0, firstEpc.lastIndexOf('.'));
    gtin = gtinUri;
  }

  return {
    ...state,
    metadata: {
      filename: state.data.filename,
      bucket: state.data.bucket,
      sourceGLN: sourceGLN,
      destinationGLN: destinationGLN,
      billToGLN: billToGLN,
      gtin: gtin,
      objectEventCount: objectEvents.length,
      aggregationEventCount: aggregationEvents.length
    },
    xmlContent: state.data.xmlContent
  };
});

function extractGLNFromUrn(urn) {
  // urn:epc:id:sgln:0614141.00001.0 → urn:epc:id:sgln:0614141.00001
  const parts = urn.split(':');
  const lastPart = parts[parts.length - 1];

  // Check if last segment is .0 (extension)
  if (lastPart === '0') {
    return urn; // Already valid GLN URI
  }

  // Remove serial component
  return urn.substring(0, urn.lastIndexOf('.'));
}
```

**Output State**:
```json
{
  "metadata": {
    "filename": "manufacturer/4K_events_05062024.xml",
    "bucket": "tiop-source-xml",
    "sourceGLN": "urn:epc:id:sgln:0614141.00001",
    "destinationGLN": "6151100444677",
    "billToGLN": "0614141888885",
    "gtin": "urn:epc:id:sgtin:0614141.012345",
    "objectEventCount": 2000,
    "aggregationEventCount": 2000
  },
  "xmlContent": "..."
}
```

**Error Handling**:
- XML parsing errors → Fail with descriptive error
- Missing required fields → Fail with validation error

---

### Step 2: Authenticate Against Business Rules

**Purpose**: Validate source/destination/product combination exists in database

**Adaptor**: `@openfn/language-postgresql`

**Code**:
```javascript
// Query for authorized GTINs
query({
  query: `
    SELECT ti.gtin_uri
    FROM tiop_rule tr
    INNER JOIN tiop_status ts ON tr.status_id = ts.status_id
    INNER JOIN location sl ON tr.source_location_id = sl.location_id
    INNER JOIN location dl ON tr.destination_location_id = dl.location_id
    INNER JOIN trade_item ti ON tr.item_id = ti.item_id
    WHERE ts.status_description = 'Active'
      AND sl.current_indicator = 'A'
      AND dl.current_indicator = 'A'
      AND ti.current_indicator = 'A'
      AND sl.gln_uri = $1
      AND dl.gln = $2
  `,
  params: [
    state => state.metadata.sourceGLN,
    state => state.metadata.destinationGLN
  ]
});

// Validate GTINs
fn(state => {
  const authorizedGtins = state.data.map(row => row.gtin_uri);

  if (authorizedGtins.length === 0) {
    throw new Error(
      `[EXC001] Manufacture GLN uri [${state.metadata.sourceGLN}], ` +
      `recipient country GLN [${state.metadata.destinationGLN}], and ` +
      `GTIN uri [${state.metadata.gtin}] combination does not exist in TIOP business rules`
    );
  }

  // Parse XML and validate all EPCs
  const { DOMParser } = require('xmldom');
  const { select } = require('xpath');

  const doc = new DOMParser().parseFromString(state.xmlContent);
  const allEpcs = select(".//epc/text()", doc).map(node => {
    const epcUri = node.nodeValue;
    // Remove serial component to get GTIN URI
    return epcUri.substring(0, epcUri.lastIndexOf('.'));
  });

  const uniqueGtins = [...new Set(allEpcs)];

  // Check all GTINs are authorized
  for (const gtinUri of uniqueGtins) {
    if (!authorizedGtins.includes(gtinUri)) {
      throw new Error(
        `[EXC001] EPC with GTIN [${gtinUri}] is not authorized for ` +
        `source [${state.metadata.sourceGLN}] to destination [${state.metadata.destinationGLN}]`
      );
    }
  }

  return {
    ...state,
    authorizedGtins: authorizedGtins
  };
});

// Insert operation record
query({
  query: `
    INSERT INTO tiopdb.tiop_operation (
      event_type_id, source_partner_id, destination_partner_id,
      source_location_id, destination_location_id, item_id, rule_id,
      status_id, document_name, object_event_count, aggregation_event_count,
      exception_detail, create_date, creator_id, last_modified_date,
      last_modified_by, current_indicator, ods_text
    )
    VALUES (
      NULL,
      (SELECT DISTINCT stp.partner_id FROM location sl
       INNER JOIN trading_partner stp ON sl.partner_id = stp.partner_id
       WHERE sl.current_indicator = 'A' AND stp.current_indicator = 'A'
       AND gln_uri = $1),
      (SELECT DISTINCT dtp.partner_id FROM location dl
       INNER JOIN trading_partner dtp ON dl.partner_id = dtp.partner_id
       WHERE dl.current_indicator = 'A' AND dtp.current_indicator = 'A'
       AND gln = $2),
      (SELECT DISTINCT sl.location_id FROM location sl
       INNER JOIN trading_partner stp ON sl.partner_id = stp.partner_id
       WHERE sl.current_indicator = 'A' AND stp.current_indicator = 'A'
       AND gln_uri = $1),
      (SELECT DISTINCT dl.location_id FROM location dl
       INNER JOIN trading_partner dtp ON dl.partner_id = dtp.partner_id
       WHERE dl.current_indicator = 'A' AND dtp.current_indicator = 'A'
       AND gln = $2),
      (SELECT DISTINCT ti.item_id FROM trade_item ti
       WHERE ti.current_indicator = 'A' AND gtin_uri = $3),
      (SELECT tr.rule_id FROM tiop_rule tr
       INNER JOIN tiop_status ts ON tr.status_id = ts.status_id
       INNER JOIN location sl ON tr.source_location_id = sl.location_id
       INNER JOIN location dl ON tr.destination_location_id = dl.location_id
       INNER JOIN trade_item ti ON tr.item_id = ti.item_id
       WHERE ts.status_description = 'Active'
       AND sl.current_indicator = 'A' AND dl.current_indicator = 'A'
       AND ti.current_indicator = 'A'
       AND sl.gln_uri = $1 AND dl.gln = $2 AND ti.gtin_uri = $3),
      6, -- Received status
      $4, $5, $6, NULL,
      NOW(), 'openfn_auth', NOW(), 'openfn_auth', 'A', ''
    )
  `,
  params: [
    state => state.metadata.sourceGLN,
    state => state.metadata.destinationGLN,
    state => state.metadata.gtin,
    state => state.metadata.filename,
    state => state.metadata.objectEventCount,
    state => state.metadata.aggregationEventCount
  ]
});
```

**Error Handling**:
```javascript
// Catch block (configured in workflow settings)
fn(state => {
  // Send email notification
  const errorMessage = state.error.message;

  return {
    ...state,
    emailNotification: {
      subject: `[${process.env.ENV}] File Processing Issue: [${state.metadata.filename}] - Attention Needed`,
      body: `
        <h4>An issue [EXC001] encountered while processing the file ${state.metadata.filename}
        which was received on ${new Date().toLocaleDateString()}.</h4>
        <h4>Details of the Issue:</h4>
        <p>${errorMessage}</p>
        <p>TIOP operation team</p>
      `
    }
  };
});

// Send email using HTTP adaptor
post(
  process.env.SMTP_API_URL,
  {
    body: state => ({
      to: process.env.TO_EMAIL,
      from: process.env.FROM_EMAIL,
      subject: state.emailNotification.subject,
      html: state.emailNotification.body
    }),
    headers: {
      'Authorization': state => `Bearer ${state.configuration.smtpToken}`
    }
  }
);

// Rethrow error to fail workflow
fn(state => {
  throw state.error;
});
```

---

### Step 3: Validate Against EPCIS Schema

**Purpose**: Validate XML against EPCIS 1.2 XSD schemas

**Adaptor**: Custom JavaScript (using `libxmljs2` or similar)

**Code**:
```javascript
fn(state => {
  const libxmljs = require('libxmljs2');
  const fs = require('fs');
  const path = require('path');

  // Load XSD schemas (these would be uploaded as workflow resources)
  const xsdPath = path.join(__dirname, 'schemas', 'EPCglobal-epcis-1_2.xsd');
  const xsdDoc = libxmljs.parseXml(fs.readFileSync(xsdPath, 'utf8'));

  // Parse XML
  const xmlDoc = libxmljs.parseXml(state.xmlContent);

  // Validate
  const isValid = xmlDoc.validate(xsdDoc);

  if (!isValid) {
    const errors = xmlDoc.validationErrors.map(e => e.message).join('; ');

    throw new Error(
      `[EXC002] XML Schema validation failed: ${errors}`
    );
  }

  console.log('XML validation successful');
  return state;
});
```

**Alternative Approach** (if XSD validation is complex):
```javascript
// Call external validation service
post(
  process.env.VALIDATION_API_URL,
  {
    body: state => ({
      xml: state.xmlContent,
      schema: 'EPCIS_1_2'
    }),
    headers: {
      'Content-Type': 'application/json'
    }
  }
);

fn(state => {
  if (!state.data.valid) {
    throw new Error(`[EXC002] ${state.data.errors.join('; ')}`);
  }
  return state;
});
```

**Error Handling**: Same pattern as Step 2 (email + fail)

---

### Step 4: Transform XML to JSON

**Purpose**: Convert EPCIS XML 1.2 to JSON 2.0

**Adaptor**: `@openfn/language-http`

**Code**:
```javascript
// Call external transformation API
post(
  process.env.XML_TO_JSON_CONVERSION_URL,
  {
    body: state => state.xmlContent,
    headers: {
      'Content-Type': 'application/xml',
      'GS1-EPC-Format': 'Always_GS1_Digital_Link'
    }
  }
);

// Validate transformation result
fn(state => {
  const jsonResponse = state.data;

  // Check for error response
  if (jsonResponse.status) {
    const status = jsonResponse.status;
    const detail = jsonResponse.detail || jsonResponse.message;

    const errorCode = status === 400 ? 'EXC008' : 'EXC009';
    throw new Error(`[${errorCode}] HTTP ${status} response from Document Conversion API: ${detail}`);
  }

  // Validate event counts
  const epcisBody = jsonResponse.epcisBody;
  if (!epcisBody) {
    throw new Error('[EXC007] Invalid JSON response: missing epcisBody');
  }

  const eventList = epcisBody.eventList;
  let jsonObjEventCount = 0;
  let jsonAggEventCount = 0;

  for (const event of eventList) {
    if (event.type === 'ObjectEvent') {
      jsonObjEventCount++;
    } else if (event.type === 'AggregationEvent') {
      jsonAggEventCount++;
    }
  }

  // Verify counts match
  if (jsonObjEventCount !== state.metadata.objectEventCount ||
      jsonAggEventCount !== state.metadata.aggregationEventCount) {
    throw new Error(
      `[EXC007] Event counts mismatch between original XML 1.2 version document ` +
      `(${state.metadata.objectEventCount} Object, ${state.metadata.aggregationEventCount} Aggregation) ` +
      `and converted JSON 2.0 document (${jsonObjEventCount} Object, ${jsonAggEventCount} Aggregation)`
    );
  }

  console.log(`Transformation successful: ${jsonObjEventCount} ObjectEvents, ${jsonAggEventCount} AggregationEvents`);

  return {
    ...state,
    jsonContent: jsonResponse
  };
});

// Insert transformation record
query({
  query: `
    INSERT INTO tiopdb.tiop_operation (
      event_type_id, source_partner_id, destination_partner_id,
      source_location_id, destination_location_id, item_id, rule_id,
      status_id, document_name, object_event_count, aggregation_event_count,
      exception_detail, create_date, creator_id, last_modified_date,
      last_modified_by, current_indicator, ods_text
    )
    VALUES (
      NULL,
      (SELECT DISTINCT stp.partner_id FROM location sl
       INNER JOIN trading_partner stp ON sl.partner_id = stp.partner_id
       WHERE sl.current_indicator = 'A' AND stp.current_indicator = 'A'
       AND gln_uri = $1),
      (SELECT DISTINCT dtp.partner_id FROM location dl
       INNER JOIN trading_partner dtp ON dl.partner_id = dtp.partner_id
       WHERE dl.current_indicator = 'A' AND dtp.current_indicator = 'A'
       AND gln = $2),
      (SELECT DISTINCT sl.location_id FROM location sl
       INNER JOIN trading_partner stp ON sl.partner_id = stp.partner_id
       WHERE sl.current_indicator = 'A' AND stp.current_indicator = 'A'
       AND gln_uri = $1),
      (SELECT DISTINCT dl.location_id FROM location dl
       INNER JOIN trading_partner dtp ON dl.partner_id = dtp.partner_id
       WHERE dl.current_indicator = 'A' AND dtp.current_indicator = 'A'
       AND gln = $2),
      (SELECT DISTINCT ti.item_id FROM trade_item ti
       WHERE ti.current_indicator = 'A' AND gtin_uri = $3),
      (SELECT tr.rule_id FROM tiop_rule tr
       INNER JOIN tiop_status ts ON tr.status_id = ts.status_id
       INNER JOIN location sl ON tr.source_location_id = sl.location_id
       INNER JOIN location dl ON tr.destination_location_id = dl.location_id
       INNER JOIN trade_item ti ON tr.item_id = ti.item_id
       WHERE ts.status_description = 'Active'
       AND sl.current_indicator = 'A' AND dl.current_indicator = 'A'
       AND ti.current_indicator = 'A'
       AND sl.gln_uri = $1 AND dl.gln = $2 AND ti.gtin_uri = $3),
      7, -- Transformed status
      $4, $5, $6, NULL,
      NOW(), 'openfn_transform', NOW(), 'openfn_transform', 'A', ''
    )
  `,
  params: [
    state => state.metadata.sourceGLN,
    state => state.metadata.destinationGLN,
    state => state.metadata.gtin,
    state => state.metadata.filename.replace('.xml', '.json'),
    state => state.metadata.objectEventCount,
    state => state.metadata.aggregationEventCount
  ]
});
```

**Error Handling**: Same pattern (email + fail with error code)

---

### Step 5: Deduplicate & Load to OpenSearch

**Purpose**: Hash events, filter duplicates, enrich, and bulk load

**Adaptor**: `@openfn/language-postgresql` + `@openfn/language-opensearch`

**Code**:
```javascript
// Query existing hashes
query({
  query: 'SELECT hash FROM tiopdb.event_hash',
  callback: (state, rows) => {
    const existingHashes = new Set(rows.map(r => r.hash));
    return { ...state, existingHashes };
  }
});

// Process events: hash, dedupe, enrich
fn(state => {
  const crypto = require('crypto');

  const eventList = state.jsonContent.epcisBody.eventList;
  const newHashes = [];
  const enrichedEvents = [];

  for (const event of eventList) {
    // Generate hash (whitespace removed)
    const eventStr = JSON.stringify(event).replace(/\s+/g, '');
    const hash = crypto.createHash('sha256').update(eventStr).digest('hex');

    // Skip duplicates
    if (state.existingHashes.has(hash)) {
      console.log('Duplicate event detected, skipping');
      continue;
    }

    // Enrich event with TIOP context
    const enrichedEvent = {
      ...event,
      'tiop:psa': state.metadata.billToGLN,
      '@context': event['@context'] || []
    };

    // Add TIOP namespace if not present
    if (!enrichedEvent['@context'].some(ctx => ctx.tiop)) {
      enrichedEvent['@context'].push({
        'tiop': 'https://ref.opentiop.org/epcis/'
      });
    }

    enrichedEvents.push(enrichedEvent);
    newHashes.push(hash);
  }

  console.log(`Filtered ${eventList.length - enrichedEvents.length} duplicate events`);
  console.log(`Preparing to load ${enrichedEvents.length} new events`);

  return {
    ...state,
    enrichedEvents,
    newHashes
  };
});

// Bulk load to OpenSearch
fn(state => {
  const { Client } = require('@opensearch-project/opensearch');

  const client = new Client({
    node: process.env.OPENSEARCH_URL,
    auth: {
      username: process.env.OPENSEARCH_USER,
      password: process.env.OPENSEARCH_PASSWORD
    }
  });

  if (state.enrichedEvents.length === 0) {
    console.log('No new events to load');
    return state;
  }

  // Build bulk request body
  const bulkBody = [];
  for (const event of state.enrichedEvents) {
    bulkBody.push({ index: { _index: 'epcis_index' } });
    bulkBody.push(event);
  }

  // Execute bulk insert
  return client.bulk({ body: bulkBody })
    .then(response => {
      // Check for errors
      if (response.body.errors) {
        const failedItems = response.body.items.filter(item => item.index.status !== 200);
        console.log(`${failedItems.length} events failed to index`);

        // Remove failed hashes
        const failedIndexes = failedItems.map(item =>
          response.body.items.indexOf(item)
        );

        const successfulHashes = state.newHashes.filter((hash, idx) =>
          !failedIndexes.includes(idx)
        );

        return {
          ...state,
          newHashes: successfulHashes,
          opensearchResponse: response.body
        };
      }

      console.log(`Successfully indexed ${state.enrichedEvents.length} events`);
      return {
        ...state,
        opensearchResponse: response.body
      };
    });
});

// Insert new hashes to database
fn(state => {
  if (state.newHashes.length === 0) {
    console.log('No new hashes to insert');
    return state;
  }

  const values = state.newHashes.map(hash =>
    `('${hash}', NOW())`
  ).join(',');

  return {
    ...state,
    hashInsertQuery: `INSERT INTO tiopdb.event_hash(hash, create_date) VALUES ${values}`
  };
});

query({
  query: state => state.hashInsertQuery
});
```

**Alternative OpenSearch Approach** (using HTTP adaptor if no OpenSearch adaptor):
```javascript
post(
  process.env.OPENSEARCH_URL + '/_bulk',
  {
    body: state => {
      let bulkBody = '';
      for (const event of state.enrichedEvents) {
        bulkBody += JSON.stringify({ index: { _index: 'epcis_index' } }) + '\n';
        bulkBody += JSON.stringify(event) + '\n';
      }
      return bulkBody;
    },
    headers: {
      'Content-Type': 'application/x-ndjson',
      'Authorization': state => {
        const auth = Buffer.from(
          `${state.configuration.opensearchUser}:${state.configuration.opensearchPassword}`
        ).toString('base64');
        return `Basic ${auth}`;
      }
    }
  }
);
```

**Error Handling**:
- OpenSearch errors → Email (EXC012) + fail
- Database errors → Email + fail

---

### Step 6: Route to Destination Country System

**Purpose**: POST XML document to destination country API

**Adaptor**: `@openfn/language-postgresql` + `@openfn/language-http`

**Code**:
```javascript
// Query routing configuration
query({
  query: `
    SELECT routing_secret_name, api_url
    FROM router_info
    WHERE country_gln = $1
      AND active = true
  `,
  params: [state => state.metadata.destinationGLN],
  callback: (state, rows) => {
    if (rows.length === 0) {
      throw new Error(
        `[EXC010] An error occurred while routing the EPCIS document. ` +
        `Routing record does not exist for recipient GLN [${state.metadata.destinationGLN}].`
      );
    }

    return {
      ...state,
      routingConfig: rows[0]
    };
  }
});

// Retrieve credentials from Lightning credential vault
// (Credentials would be pre-configured in Lightning with names matching routing_secret_name)
fn(state => {
  const credentialName = state.routingConfig.routing_secret_name;

  // Access credential (Lightning makes credentials available in state.configuration)
  // For this to work, the credential must be added to the workflow's allowed credentials
  const bearerToken = state.configuration[credentialName + '_bearer_token'];

  if (!bearerToken) {
    throw new Error(
      `[EXC010] Missing credentials for destination [${state.metadata.destinationGLN}]. ` +
      `Credential name: ${credentialName}`
    );
  }

  return {
    ...state,
    destinationBearerToken: bearerToken
  };
});

// POST XML to country endpoint
post(
  state => state.routingConfig.api_url,
  {
    body: state => state.xmlContent,
    headers: {
      'Content-Type': 'application/xml',
      'Authorization': state => `Bearer ${state.destinationBearerToken}`
    }
  }
);

// Handle response
fn(state => {
  const status = state.data.status || 200;

  if (status !== 200) {
    const errorMsg = state.data.error || state.data.message || 'Unknown error';
    throw new Error(
      `[EXC014] Routing failed with HTTP ${status}: ${errorMsg}`
    );
  }

  console.log('Successfully routed document to country system');
  return state;
});

// Insert routing record
query({
  query: `
    INSERT INTO tiopdb.tiop_operation (
      event_type_id, source_partner_id, destination_partner_id,
      source_location_id, destination_location_id, item_id, rule_id,
      status_id, document_name, object_event_count, aggregation_event_count,
      exception_detail, create_date, creator_id, last_modified_date,
      last_modified_by, current_indicator, ods_text
    )
    VALUES (
      NULL,
      (SELECT DISTINCT stp.partner_id FROM location sl
       INNER JOIN trading_partner stp ON sl.partner_id = stp.partner_id
       WHERE sl.current_indicator = 'A' AND stp.current_indicator = 'A'
       AND gln_uri = $1),
      (SELECT DISTINCT dtp.partner_id FROM location dl
       INNER JOIN trading_partner dtp ON dl.partner_id = dtp.partner_id
       WHERE dl.current_indicator = 'A' AND dtp.current_indicator = 'A'
       AND gln = $2),
      (SELECT DISTINCT sl.location_id FROM location sl
       INNER JOIN trading_partner stp ON sl.partner_id = stp.partner_id
       WHERE sl.current_indicator = 'A' AND stp.current_indicator = 'A'
       AND gln_uri = $1),
      (SELECT DISTINCT dl.location_id FROM location dl
       INNER JOIN trading_partner dtp ON dl.partner_id = dtp.partner_id
       WHERE dl.current_indicator = 'A' AND dtp.current_indicator = 'A'
       AND gln = $2),
      (SELECT DISTINCT ti.item_id FROM trade_item ti
       WHERE ti.current_indicator = 'A' AND gtin_uri = $3),
      (SELECT tr.rule_id FROM tiop_rule tr
       INNER JOIN tiop_status ts ON tr.status_id = ts.status_id
       INNER JOIN location sl ON tr.source_location_id = sl.location_id
       INNER JOIN location dl ON tr.destination_location_id = dl.location_id
       INNER JOIN trade_item ti ON tr.item_id = ti.item_id
       WHERE ts.status_description = 'Active'
       AND sl.current_indicator = 'A' AND dl.current_indicator = 'A'
       AND ti.current_indicator = 'A'
       AND sl.gln_uri = $1 AND dl.gln = $2 AND ti.gtin_uri = $3),
      8, -- Routed status
      $4, $5, $6, NULL,
      NOW(), 'openfn_router', NOW(), 'openfn_router', 'A', ''
    )
  `,
  params: [
    state => state.metadata.sourceGLN,
    state => state.metadata.destinationGLN,
    state => state.metadata.gtin,
    state => state.metadata.filename,
    state => state.metadata.objectEventCount,
    state => state.metadata.aggregationEventCount
  ]
});
```

**Error Handling**: Same pattern (email + fail with error code)

---

## Configuration

### Environment Variables

Configure these in Lightning's project settings:

```bash
# Database
DATABASE_URL=postgresql://user:pass@host:5432/tiopdb

# OpenSearch
OPENSEARCH_URL=https://opensearch.example.com
OPENSEARCH_USER=admin
OPENSEARCH_PASSWORD=secret

# Email/SMTP
SMTP_API_URL=https://api.sendgrid.com/v3/mail/send
FROM_EMAIL=tiop-notifications@example.com
TO_EMAIL=tiop-operations@example.com

# External APIs
XML_TO_JSON_CONVERSION_URL=https://converter.example.com/xml-to-json
VALIDATION_API_URL=https://validator.example.com/validate

# Environment
ENV=dev  # or 'prod'
```

### Credentials

Configure these in Lightning's credential vault:

1. **postgresql_credentials**
   - Type: PostgreSQL
   - Host, Port, Database, User, Password

2. **opensearch_credentials**
   - Type: Custom
   - Fields: `opensearchUser`, `opensearchPassword`

3. **smtp_credentials**
   - Type: Custom
   - Fields: `smtpToken`

4. **Country Routing Credentials** (one per country)
   - Type: Custom
   - Name: `{routing_secret_name}_bearer_token`
   - Fields: `bearer_token`

### Workflow Resources

Upload these files to workflow resources:

```
schemas/
  ├── EPCglobal-epcis-1_2.xsd
  ├── StandardBusinessDocumentHeader.xsd
  ├── EPCglobal.xsd
  ├── DocumentIdentification.xsd
  ├── BusinessScope.xsd
  ├── Manifest.xsd
  ├── BasicTypes.xsd
  └── Partner.xsd
```

---

## Error Handling Strategy

### Global Error Handler

Configure a catch step at the workflow level:

```javascript
// Error handler (runs on any step failure)
fn(state => {
  const error = state.error;
  const errorCode = error.message.match(/\[EXC\d+\]/)?.[0] || '[EXC999]';

  console.error(`Workflow failed at step: ${state.currentStep}`);
  console.error(`Error: ${error.message}`);

  // Extract metadata (if available)
  const filename = state.metadata?.filename || 'unknown';
  const stage = state.currentStep || 'unknown';

  return {
    ...state,
    failureEmail: {
      to: process.env.TO_EMAIL,
      from: process.env.FROM_EMAIL,
      subject: `[${process.env.ENV}] File Processing Issue: [${filename}] - Attention Needed`,
      html: `
        <h4>An issue ${errorCode} encountered while processing the file ${filename}
        which was received on ${new Date().toLocaleDateString()}.</h4>
        <h4>Details of the Issue:</h4>
        <p>Stage: ${stage}</p>
        <p>Error: ${error.message}</p>
        <p>Stack: ${error.stack}</p>
        <p>TIOP operation team</p>
      `
    }
  };
});

// Send failure email
post(
  process.env.SMTP_API_URL,
  {
    body: state => state.failureEmail,
    headers: {
      'Authorization': state => `Bearer ${state.configuration.smtpToken}`
    }
  }
);

// Log error to database (if we have metadata)
query({
  query: `
    INSERT INTO tiopdb.tiop_operation (
      event_type_id, source_partner_id, destination_partner_id,
      source_location_id, destination_location_id, item_id, rule_id,
      status_id, document_name, object_event_count, aggregation_event_count,
      exception_detail, create_date, creator_id, last_modified_date,
      last_modified_by, current_indicator, ods_text
    )
    VALUES (
      NULL,
      (SELECT DISTINCT stp.partner_id FROM location sl
       INNER JOIN trading_partner stp ON sl.partner_id = stp.partner_id
       WHERE sl.current_indicator = 'A' AND stp.current_indicator = 'A'
       AND gln_uri = $1),
      (SELECT DISTINCT dtp.partner_id FROM location dl
       INNER JOIN trading_partner dtp ON dl.partner_id = dtp.partner_id
       WHERE dl.current_indicator = 'A' AND dtp.current_indicator = 'A'
       AND gln = $2),
      (SELECT DISTINCT sl.location_id FROM location sl
       INNER JOIN trading_partner stp ON sl.partner_id = stp.partner_id
       WHERE sl.current_indicator = 'A' AND stp.current_indicator = 'A'
       AND gln_uri = $1),
      (SELECT DISTINCT dl.location_id FROM location dl
       INNER JOIN trading_partner dtp ON dl.partner_id = dtp.partner_id
       WHERE dl.current_indicator = 'A' AND dtp.current_indicator = 'A'
       AND gln = $2),
      (SELECT DISTINCT ti.item_id FROM trade_item ti
       WHERE ti.current_indicator = 'A' AND gtin_uri = $3),
      NULL, -- rule_id
      99, -- Error status
      $4, $5, $6, $7,
      NOW(), 'openfn_error', NOW(), 'openfn_error', 'A', ''
    )
  `,
  params: [
    state => state.metadata?.sourceGLN || 'unknown',
    state => state.metadata?.destinationGLN || 'unknown',
    state => state.metadata?.gtin || 'unknown',
    state => state.metadata?.filename || 'unknown',
    state => state.metadata?.objectEventCount || 0,
    state => state.metadata?.aggregationEventCount || 0,
    state => state.error.message.substring(0, 500) // Truncate if too long
  ]
});
```

### Retry Strategy

Configure automatic retries in workflow settings:

- **Step 1-3**: No retries (validation failures shouldn't retry)
- **Step 4**: Retry 3 times with exponential backoff (external API)
- **Step 5**: Retry 2 times with 5s delay (database/OpenSearch transient errors)
- **Step 6**: Retry 3 times with exponential backoff (external API)

---

## Trigger Configuration

### Option 1: HTTP Endpoint Trigger

**Pros**: Simple, real-time, no polling
**Cons**: Requires webhook integration from S3 or SFTP

```javascript
// Workflow triggered by HTTP POST
// Expected payload:
{
  "filename": "manufacturer/document.xml",
  "bucket": "tiop-source-xml",
  "xmlContent": "<epcis:EPCISDocument>...</epcis:EPCISDocument>"
}
```

**S3 Event Notification Setup**:
- Configure S3 bucket to POST to Lightning webhook URL on `s3:ObjectCreated:*`
- Or use AWS Lambda trigger → HTTP POST to Lightning

### Option 2: S3 Polling Trigger

**Pros**: No webhook setup needed
**Cons**: Not real-time (polls every N minutes)

Use OpenFn's built-in S3 polling:
- Configure S3 credentials
- Set bucket name
- Set polling interval (e.g., 1 minute)
- File pattern: `*.xml`

### Option 3: Message Queue Trigger

**Pros**: Reliable, decoupled, handles bursts
**Cons**: Additional infrastructure (SQS/RabbitMQ)

- S3 → SNS → SQS → Lightning
- Lightning polls SQS queue

---

## Migration Strategy

### Phase 1: Parallel Operation

1. Deploy OpenFn workflow alongside existing Lambda functions
2. Configure to process copies of files (duplicate to separate S3 bucket)
3. Compare results:
   - OpenSearch indices
   - Database operation records
   - Email notifications
4. Tune performance and error handling

### Phase 2: Gradual Cutover

1. Route 10% of traffic to OpenFn (use S3 bucket prefix routing)
2. Monitor for errors and performance
3. Increase to 50%, then 100%
4. Keep Lambda functions on standby

### Phase 3: Decommission

1. Remove Lambda functions
2. Clean up CloudFormation stacks
3. Update documentation
4. Train team on Lightning interface

---

## Performance Considerations

### Throughput

**Current TIOP**:
- Lambda concurrency: Up to 1000 (AWS limit)
- Processing time: ~10-30s per document (including cold starts)

**OpenFn Lightning**:
- Concurrent workflows: Based on subscription tier
- Processing time: ~5-15s per document (no cold starts)
- Can handle 100+ concurrent executions

### Optimization Tips

1. **Reduce Database Queries**: Cache authorized GTINs in workflow state
2. **Batch Hash Lookups**: Query hashes in batches of 1000
3. **Parallel Routing**: If multiple destinations, use parallel branches
4. **Incremental Hashing**: Store hash incrementally rather than full scan

### Resource Usage

- **Memory**: ~256MB per workflow run (vs 1024MB Lambda)
- **CPU**: Minimal (mostly I/O bound)
- **Storage**: State size ~2-5MB per run (XML + metadata)

---

## Monitoring & Observability

### Built-in Lightning Features

1. **Work Order Dashboard**
   - View all workflow runs
   - Filter by status, date, duration
   - Search by filename

2. **Inspector**
   - Step-by-step execution view
   - Input/output state at each step
   - Error messages and stack traces

3. **Logs**
   - Structured logs with context
   - Search and filter
   - Export capability

### Custom Monitoring

Add monitoring steps:

```javascript
// After each major step
fn(state => {
  // Log metrics to monitoring service
  return post(
    'https://monitoring.example.com/metrics',
    {
      body: {
        workflow: 'tiop-pipeline',
        step: state.currentStep,
        duration: Date.now() - state.startTime,
        filename: state.metadata.filename,
        eventCount: state.metadata.objectEventCount + state.metadata.aggregationEventCount
      }
    }
  );
});
```

### Alerts

Configure alerts in Lightning:
- Workflow failure rate > 5%
- Average duration > 30s
- No successful runs in 1 hour

---

## Testing Strategy

### Unit Testing

Test individual steps:

```javascript
// test/steps/parse-metadata.test.js
const { parseMetadata } = require('../steps/parse-metadata');

describe('parseMetadata', () => {
  it('should extract source GLN from shipping event', () => {
    const state = {
      data: {
        xmlContent: '<EPCISDocument>...</EPCISDocument>'
      }
    };

    const result = parseMetadata(state);

    expect(result.metadata.sourceGLN).toBe('urn:epc:id:sgln:0614141.00001');
  });
});
```

### Integration Testing

Use Lightning's built-in test runner:

1. Create test fixtures (sample XML files)
2. Mock external dependencies (conversion API, country endpoints)
3. Run workflow with test data
4. Assert:
   - Database records created
   - OpenSearch documents indexed
   - Emails sent (to test inbox)

### Load Testing

1. Generate 100 sample EPCIS documents
2. Submit in parallel
3. Measure:
   - Throughput (documents/minute)
   - Error rate
   - Average duration
   - Database connection pool usage

---

## Cost Comparison

### Current TIOP (AWS Lambda)

**Monthly costs** (estimated for 10,000 documents/month):
- Lambda invocations: $50
- Lambda execution time: $100
- RDS (db.t3.medium): $70
- OpenSearch (3 nodes, t3.medium): $200
- S3 storage: $10
- Data transfer: $20
- **Total**: ~$450/month

### OpenFn Lightning

**Monthly costs**:
- Lightning subscription (Business tier): $500/month (includes 100k workflow runs)
- RDS: $70 (unchanged)
- OpenSearch: $200 (unchanged)
- S3 storage: $10 (unchanged)
- **Total**: ~$780/month

**Difference**: +$330/month (+73%)

**Value proposition**:
- Simplified operations (no Lambda management)
- Better observability and debugging
- Faster development cycles
- Included support

---

## Security Considerations

### Credentials

- All credentials stored in Lightning's encrypted vault
- No hardcoded secrets in workflow code
- Credentials scoped to workflow (not accessible from other workflows)

### Data Protection

- XML/JSON content never persisted to Lightning database (only in workflow state during execution)
- State is encrypted at rest
- TLS for all API communications

### Access Control

- Role-based access to workflows
- Audit log of all workflow edits
- API key authentication for triggers

### Compliance

- SOC 2 Type II certified (Lightning platform)
- GDPR compliant
- Data residency options available

---

## Support & Maintenance

### Documentation

1. Workflow documentation (in Lightning)
   - High-level flow diagram
   - Step-by-step explanations
   - Credential requirements
   - Error code reference

2. Runbook
   - Common errors and solutions
   - Rollback procedures
   - Performance tuning

3. Architecture diagrams
   - System integration map
   - Data flow diagram

### Team Training

1. **Developers** (4 hours)
   - OpenFn workflow basics
   - JavaScript for data transformation
   - Debugging with Inspector
   - Version control and deployments

2. **Operations** (2 hours)
   - Monitoring dashboards
   - Handling failed workflows
   - Managing credentials
   - Reviewing logs

3. **Business Users** (1 hour)
   - Viewing workflow status
   - Understanding error notifications
   - Requesting new integrations

---

## Appendices

### A. Error Code Reference

| Code | Stage | Description | Resolution |
|------|-------|-------------|------------|
| EXC001 | Auth | GLN/GTIN combination not in business rules | Add rule to database |
| EXC002 | Validate | XSD schema validation failure | Fix XML structure |
| EXC007 | Transform | Event count mismatch | Check conversion API |
| EXC008 | Transform | HTTP 400 from conversion API | Fix XML format |
| EXC009 | Transform | HTTP 500 from conversion API | Retry or escalate |
| EXC010 | Route | Missing routing configuration | Add to router_info table |
| EXC012 | BulkLoad | OpenSearch indexing error | Check index settings |
| EXC014 | Route | Country endpoint error | Check credentials/API |

### B. Database Schema Changes

Add status for OpenFn stages:

```sql
-- Add new status values
INSERT INTO tiop_status (status_id, status_description, create_date, creator_id, last_modified_date, last_modified_by, current_indicator)
VALUES
  (99, 'Workflow Error', NOW(), 'system', NOW(), 'system', 'A');

-- Add index for performance
CREATE INDEX idx_event_hash_hash ON event_hash(hash);
CREATE INDEX idx_tiop_operation_filename ON tiop_operation(document_name);
CREATE INDEX idx_tiop_operation_status ON tiop_operation(status_id, create_date);
```

### C. Adaptor Dependencies

```json
{
  "dependencies": {
    "@openfn/language-http": "^6.0.0",
    "@openfn/language-postgresql": "^4.0.0",
    "xmldom": "^0.6.0",
    "xpath": "^0.0.32",
    "libxmljs2": "^0.33.0",
    "@opensearch-project/opensearch": "^2.0.0"
  }
}
```

### D. Sample Workflow JSON

```json
{
  "name": "TIOP Document Processing Pipeline",
  "trigger": {
    "type": "webhook",
    "path": "/tiop/process"
  },
  "steps": [
    {
      "id": "parse-metadata",
      "name": "Parse & Extract Metadata",
      "adaptor": "@openfn/language-common",
      "code": "fn(state => { /* Step 1 code */ })"
    },
    {
      "id": "authenticate",
      "name": "Authenticate Against Business Rules",
      "adaptor": "@openfn/language-postgresql",
      "credentials": "postgresql_credentials",
      "code": "query({ /* Step 2 code */ })"
    },
    {
      "id": "validate",
      "name": "Validate Against EPCIS Schema",
      "adaptor": "@openfn/language-common",
      "code": "fn(state => { /* Step 3 code */ })"
    },
    {
      "id": "transform",
      "name": "Transform XML to JSON",
      "adaptor": "@openfn/language-http",
      "code": "post(/* Step 4 code */)"
    },
    {
      "id": "bulkload",
      "name": "Deduplicate & Load to OpenSearch",
      "adaptor": "@openfn/language-postgresql",
      "credentials": "postgresql_credentials",
      "code": "query({ /* Step 5 code */ })"
    },
    {
      "id": "route",
      "name": "Route to Destination Country System",
      "adaptor": "@openfn/language-http",
      "code": "post(/* Step 6 code */)"
    }
  ],
  "catch": {
    "name": "Error Handler",
    "adaptor": "@openfn/language-http",
    "code": "fn(state => { /* Error handling code */ })"
  }
}
```

---

## Conclusion

This OpenFn Lightning workflow design provides a complete replacement for the TIOP Lambda-based architecture with:

✅ **Functional Parity**: All 5 stages implemented with same logic
✅ **Improved DX**: Visual workflow editor, better debugging tools
✅ **Better Observability**: Built-in monitoring, logs, and Inspector
✅ **Simpler Operations**: No Lambda packaging, deployment, or infrastructure management
✅ **Enhanced Error Handling**: Automatic retries, comprehensive error tracking
✅ **Maintainability**: JavaScript vs Java, hot-swappable workflows

**Recommendation**: Proceed with migration using the phased approach outlined in the Migration Strategy section.

**Next Steps**:
1. Set up OpenFn Lightning project
2. Configure credentials and environment variables
3. Implement Step 1 (Parse Metadata) and test with sample data
4. Incrementally add remaining steps
5. Run parallel with existing system for validation
6. Gradual traffic cutover
