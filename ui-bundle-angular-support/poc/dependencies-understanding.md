# Dependencies Understanding — Angular UI Bundle

## Package Overview

```
@salesforce/vite-plugin-ui-bundle   ← build/dev time only (Vite plugin)
@salesforce/ui-bundle               ← runtime, browser (REST helpers)
@salesforce/sdk-data                ← runtime, browser (fetch + GraphQL)
```

---

## `@salesforce/vite-plugin-ui-bundle` — Build Time Only

Lives in `webapps/packages/vite-plugin-ui-bundle`. Used in `vite.config.ts`.
Never ships to the browser.

Has two sub-plugins:

| Sub-plugin | What it does |
|---|---|
| `corePlugin` | Proxies `/services/*` → Salesforce org in dev. Injects `SFDC_ENV` (basePath, apiPath) into `index.html` in dev — on org, LWR server does this. Framework-agnostic. |
| `designPlugin` | Instruments React components for the visual design editor. React-specific. |

```ts
// vite.config.ts
import salesforce from '@salesforce/vite-plugin-ui-bundle';
plugins: [angular(), salesforce()]   // salesforce() = corePlugin + designPlugin
```

---

## `@salesforce/sdk-data` — Runtime: fetch + GraphQL

The low-level Salesforce data SDK. Two methods:

### `sdk.fetch()` — Raw REST calls

```ts
import { createDataSDK } from '@salesforce/sdk-data';

const sdk = await createDataSDK();
const response = await sdk.fetch('/services/data/v65.0/chatter/users/me');
const user = await response.json();
```

### `sdk.graphql()` — GraphQL queries

```ts
const data = await createDataSDK();
const result = await data.graphql({
  query: `
    query GetContacts {
      uiapi {
        query {
          Contact(first: 20) {
            edges { node { Id Name { value } Email { value } } }
          }
        }
      }
    }
  `
});
console.log(result.data.uiapi.query.Contact.edges);
```

Hits: `POST /services/data/v{version}/graphql`

> **Signature:** `graphql({ query, variables, operationName })` — named object, not positional.

---

## `@salesforce/ui-bundle` — Runtime: Convenience Helpers

Pre-built wrappers on top of `sdk-data`. Use these instead of writing boilerplate REST calls.

### Get / Create / Update / Delete Records (UI API)

```ts
import { getRecord, createRecord, updateRecord, deleteRecord } from '@salesforce/ui-bundle';

// Get a record
const record = await getRecord('001xxxxxxxxxxxxxxx', { fields: 'Contact.Name,Contact.Email' });
console.log(record.fields.Name.value);  // "John Smith"

// Create a record
const newRecord = await createRecord('Contact', { LastName: 'Smith', Email: 'smith@example.com' });

// Update a record
await updateRecord('001xxx', { Email: 'new@example.com' });

// Delete a record
await deleteRecord('001xxx');
```

Hits: `/services/data/v{version}/ui-api/records/...`

### Get Current Logged-In User

```ts
import { getCurrentUser } from '@salesforce/ui-bundle';

const user = await getCurrentUser();
console.log(user.name);  // "Gulshan Kumar"
console.log(user.id);    // "005..."
```

Hits: `/services/data/v{version}/chatter/users/me`

### Raw UI API Client

```ts
import { uiApiClient } from '@salesforce/ui-bundle';

// Any UI API endpoint
const res = await uiApiClient.get('/object-info/Contact');
const data = await res.json();
```

---

## How They Relate

```
@salesforce/ui-bundle
  └── internally uses createDataSDK().fetch()
        └── @salesforce/sdk-data
              └── WebAppDataSDK.fetch()
                    └── corePlugin proxy (dev) → Salesforce org
```

`@salesforce/ui-bundle` is a **convenience layer on top of `@salesforce/sdk-data`**.
For GraphQL you go to `sdk-data` directly — `ui-bundle` has no GraphQL method.

---

## When to Use Which

| Use case | Package | Method |
|---|---|---|
| GraphQL query (SOQL-like) | `@salesforce/sdk-data` | `data.graphql({ query, variables })` |
| Get a Salesforce record | `@salesforce/ui-bundle` | `getRecord(id, { fields })` |
| Create a record | `@salesforce/ui-bundle` | `createRecord(apiName, fields)` |
| Update a record | `@salesforce/ui-bundle` | `updateRecord(id, fields)` |
| Delete a record | `@salesforce/ui-bundle` | `deleteRecord(id)` |
| Get current user | `@salesforce/ui-bundle` | `getCurrentUser()` |
| Raw REST call | `@salesforce/sdk-data` | `sdk.fetch('/services/...')` |
| Load ui-bundle.json manifest | `@salesforce/ui-bundle` | `loadManifest(path)` |

---

## In the Angular Template

The template uses `sdk-data` for GraphQL (contact page):

```ts
// src/api/graphql-client.ts
import { createDataSDK } from '@salesforce/sdk-data';

export async function executeGraphQL<TData, TVariables>(
  query: string,
  variables?: TVariables
): Promise<TData> {
  const data = await createDataSDK();
  const response = await (data as any).graphql({ query, variables });
  if (response?.errors?.length) {
    throw new Error(response.errors.map((e: any) => e.message).join('; '));
  }
  return response.data;
}
```

To use `@salesforce/ui-bundle` helpers in Angular, inject them in a service:

```ts
// src/app/services/salesforce.service.ts
import { Injectable } from '@angular/core';
import { getRecord, getCurrentUser } from '@salesforce/ui-bundle';

@Injectable({ providedIn: 'root' })
export class SalesforceService {
  async getContact(id: string) {
    return getRecord(id, { fields: 'Contact.Name,Contact.Email,Contact.Phone' });
  }

  async getCurrentUser() {
    return getCurrentUser();
  }
}
```

---

## Repos

| Package | Repo | Local path |
|---|---|---|
| `@salesforce/vite-plugin-ui-bundle` | `salesforce-experience-platform-emu/webapps` | `webapps/packages/vite-plugin-ui-bundle` |
| `@salesforce/ui-bundle` | same | `webapps/packages/ui-bundle` |
| `@salesforce/sdk-data` | same | `webapps/packages/sdk/sdk-data` |
