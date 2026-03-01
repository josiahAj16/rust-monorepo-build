# Conch Integration Guide

## Quick Integration Steps

### 1. Frontend Integration

#### Setup SDK in Your Project

```bash
npm install @conch/sdk
```

#### Import and Initialize

```typescript
import { ConchClient } from '@conch/sdk';

const client = new ConchClient({
  apiUrl: 'http://localhost:3001',  // Your backend URL
  token: 'your-jwt-token'            // From login/auth
});
```

#### Get Auth Token

```typescript
// Login to get token
const response = await fetch('http://localhost:3001/auth/login', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    email: 'user@example.com',
    password: 'password123'
  })
});

const { token } = await response.json();
const client = new ConchClient({
  apiUrl: 'http://localhost:3001',
  token
});
```

#### Create Conches

```typescript
// Create a new conch with initial state
const conch = await client.conches.create({
  state: {
    name: 'My Data',
    description: 'Example conch',
    value: 42
  }
});

console.log('Created:', conch.id, 'Era:', conch.era);
```

#### Read Conches

```typescript
// Get a specific conch
const conch = await client.conches.get('conch-uuid');
console.log('State:', conch.state);
console.log('Era:', conch.era);

// List all conches
const list = await client.conches.list();
console.log('Total:', list.conches.length);
```

#### Update Conches

```typescript
// Update state (creates new era)
const updated = await client.conches.update('conch-uuid', {
  state: {
    name: 'Updated Name',
    description: 'Updated description',
    value: 43
  }
});

console.log('New Era:', updated.era);
```

#### View History

```typescript
// Get complete lineage
const lineage = await client.conches.lineage('conch-uuid');

lineage.forEach(entry => {
  console.log(`Era ${entry.era}:`, entry.state_hash, entry.timestamp);
});
```

#### Subscribe to Updates

```typescript
// Listen for real-time updates
const unsubscribe = client.subscribe('conch-uuid', (event) => {
  console.log('Conch updated!');
  console.log('Old state:', event.old_state);
  console.log('New state:', event.new_state);
  console.log('New era:', event.era);
});

// Stop listening
// unsubscribe();
```

### 2. Graph Integration

#### Create Relationships

```typescript
// Create a link from one conch to another
const link = await client.links.create('source-uuid', {
  target_id: 'target-uuid',
  link_type: 'related',
  metadata: { context: 'information' }
});

console.log('Link created:', link.id);
```

#### Query Relationships

```typescript
// Get all directly connected conches
const neighborhood = await client.conches.neighborhood('conch-uuid');

console.log('Connected:', neighborhood.neighbors.length);

neighborhood.neighbors.forEach(({ link, conch }) => {
  console.log(`${link.link_type}: ${conch.id}`);
});
```

### 3. React Component Example

```typescript
'use client';

import { useState, useEffect } from 'react';
import { ConchClient } from '@conch/sdk';

export function ConchViewer({ conchId }: { conchId: string }) {
  const [conch, setConch] = useState(null);
  const [loading, setLoading] = useState(true);
  const [updates, setUpdates] = useState(0);

  useEffect(() => {
    const client = new ConchClient({
      apiUrl: 'http://localhost:3001',
      token: 'your-token'
    });

    // Fetch initial data
    client.conches.get(conchId).then(setConch).finally(() => setLoading(false));

    // Subscribe to updates
    const unsubscribe = client.subscribe(conchId, (event) => {
      setConch(event.new_conch);
      setUpdates(prev => prev + 1);
    });

    return unsubscribe;
  }, [conchId]);

  if (loading) return <div>Loading...</div>;
  if (!conch) return <div>Not found</div>;

  return (
    <div>
      <h2>Conch: {conch.id}</h2>
      <p>Era: {conch.era}</p>
      <p>Updates received: {updates}</p>
      <pre>{JSON.stringify(conch.state, null, 2)}</pre>
    </div>
  );
}
```

### 4. Server-side Integration (Node.js)

```typescript
import { ConchClient } from '@conch/sdk';

async function processConches() {
  const client = new ConchClient({
    apiUrl: 'http://localhost:3001',
    token: 'your-jwt-token'
  });

  // Get all conches
  const { conches } = await client.conches.list();

  for (const conch of conches) {
    // Process each conch
    console.log(`Processing ${conch.id} at era ${conch.era}`);

    // Get history
    const lineage = await client.conches.lineage(conch.id);
    console.log(`  History length: ${lineage.length}`);

    // Update with new data
    const updated = await client.conches.update(conch.id, {
      state: {
        ...conch.state,
        processed_at: new Date().toISOString()
      }
    });

    console.log(`  Updated to era ${updated.era}`);
  }
}

await processConches();
```

### 5. Rust Integration

```rust
use conch::ConchClient;
use serde_json::json;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
  let client = ConchClient::new(
    "http://localhost:3001".to_string(),
    "your-jwt-token".to_string()
  );

  // Create a conch
  let conch = client.create_conch(json!({
    "name": "Rust Conch",
    "language": "Rust"
  })).await?;

  println!("Created: {}", conch.id);

  // Update state
  let updated = client.update_conch(
    &conch.id,
    json!({
      "name": "Rust Conch",
      "language": "Rust",
      "updated": true
    })
  ).await?;

  println!("Updated to era {}", updated.era);

  // Get history
  let lineage = client.get_lineage(&conch.id).await?;
  println!("History entries: {}", lineage.len());

  Ok(())
}
```

### 6. REST API Direct Integration

If you prefer direct HTTP calls:

#### Authentication

```bash
# Get token
curl -X POST http://localhost:3001/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "password123"
  }'

# Response
{
  "token": "eyJ...",
  "user_id": "uuid",
  "username": "user"
}
```

#### Create Conch

```bash
curl -X POST http://localhost:3001/conches \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJ..." \
  -d '{
    "state": {
      "name": "My Data",
      "value": 42
    }
  }'
```

#### Get Conch

```bash
curl http://localhost:3001/conches/{id} \
  -H "Authorization: Bearer eyJ..."
```

#### Update Conch

```bash
curl -X PUT http://localhost:3001/conches/{id} \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJ..." \
  -d '{
    "state": {
      "name": "Updated",
      "value": 43
    }
  }'
```

#### Delete Conch

```bash
curl -X DELETE http://localhost:3001/conches/{id} \
  -H "Authorization: Bearer eyJ..."
```

#### Get Lineage

```bash
curl http://localhost:3001/conches/{id}/lineage \
  -H "Authorization: Bearer eyJ..."
```

#### Create Link

```bash
curl -X POST http://localhost:3001/links/{source_id} \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJ..." \
  -d '{
    "target_id": "target-uuid",
    "link_type": "related",
    "metadata": {}
  }'
```

#### Get Neighborhood

```bash
curl http://localhost:3001/conches/{id}/neighborhood \
  -H "Authorization: Bearer eyJ..."
```

### 7. Error Handling

```typescript
import { ConchClient } from '@conch/sdk';

const client = new ConchClient({
  apiUrl: 'http://localhost:3001',
  token: 'token'
});

try {
  const conch = await client.conches.get('invalid-id');
} catch (error) {
  if (error.status === 404) {
    console.log('Conch not found');
  } else if (error.status === 401) {
    console.log('Unauthorized - invalid token');
  } else if (error.status === 403) {
    console.log('Forbidden - no permission');
  } else {
    console.error('Unexpected error:', error);
  }
}
```

### 8. WebSocket Integration

```typescript
const ws = new WebSocket('ws://localhost:3001/ws/conch-uuid');

ws.onopen = () => {
  console.log('Connected to conch updates');
};

ws.onmessage = (event) => {
  const update = JSON.parse(event.data);
  console.log('State changed:', update);
  console.log('New era:', update.era);
  console.log('Old state:', update.old_state);
  console.log('New state:', update.new_state);
};

ws.onerror = (error) => {
  console.error('WebSocket error:', error);
};

ws.onclose = () => {
  console.log('Disconnected from conch');
};

// Send subscription message (if needed)
ws.send(JSON.stringify({
  type: 'subscribe',
  conch_id: 'uuid'
}));

// Close connection
// ws.close();
```

### 9. Complete Example Application

```typescript
import { ConchClient } from '@conch/sdk';

async function exampleApp() {
  // 1. Get token
  const loginRes = await fetch('http://localhost:3001/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      email: 'user@example.com',
      password: 'password123'
    })
  });

  const { token } = await loginRes.json();

  // 2. Initialize client
  const client = new ConchClient({
    apiUrl: 'http://localhost:3001',
    token
  });

  // 3. Create two conches
  const doc1 = await client.conches.create({
    state: { title: 'Document 1' }
  });

  const doc2 = await client.conches.create({
    state: { title: 'Document 2' }
  });

  console.log('Created documents:', doc1.id, doc2.id);

  // 4. Create a link
  await client.links.create(doc1.id, {
    target_id: doc2.id,
    link_type: 'references',
    metadata: { context: 'related documents' }
  });

  // 5. View relationships
  const neighborhood = await client.conches.neighborhood(doc1.id);
  console.log('Document 1 connects to:', neighborhood.neighbors.length, 'conches');

  // 6. Update document
  const updated = await client.conches.update(doc1.id, {
    state: {
      title: 'Document 1 Updated',
      content: 'New content here'
    }
  });

  console.log('Updated to era:', updated.era);

  // 7. View history
  const history = await client.conches.lineage(doc1.id);
  console.log('Document has', history.length, 'versions');

  // 8. Subscribe to changes
  const unsubscribe = client.subscribe(doc1.id, (event) => {
    console.log('Document updated to era', event.era);
  });

  // 9. Make another update (will trigger subscription)
  await client.conches.update(doc1.id, {
    state: {
      title: 'Document 1 Final Update',
      content: 'Final content',
      timestamp: new Date().toISOString()
    }
  });

  // 10. Cleanup
  unsubscribe();
}

exampleApp().catch(console.error);
```

### 10. Common Patterns

#### Batch Operations

```typescript
const conches = ['id1', 'id2', 'id3'];
const results = await Promise.all(
  conches.map(id => client.conches.get(id))
);
```

#### Monitoring Changes

```typescript
const unsubscribers = [];

for (const conchId of importantConches) {
  const unsub = client.subscribe(conchId, (event) => {
    console.log(`${conchId} changed to era ${event.era}`);
    alertTeam(conchId, event);
  });
  unsubscribers.push(unsub);
}

// Cleanup when done
unsubscribers.forEach(unsub => unsub());
```

#### Building a Graph Visualization

```typescript
async function buildGraphData(rootConchId) {
  const neighborhood = await client.conches.neighborhood(rootConchId);

  const nodes = [{ id: rootConchId, label: 'Root' }];
  const edges = [];

  for (const { link, conch } of neighborhood.neighbors) {
    nodes.push({ id: conch.id, label: conch.id.substring(0, 8) });
    edges.push({
      source: rootConchId,
      target: conch.id,
      label: link.link_type
    });
  }

  return { nodes, edges };
}
```

---

**Your Conch integration is ready! Start building with immutable data.** 🚀
