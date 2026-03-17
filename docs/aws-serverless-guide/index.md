# Building a Serverless Backend with AWS CDK: The CrocsList Architecture and Implementation Guide

**Author:** Patricia Lo | **Date:** Dec 2025

**Audience:** Backend engineers, cloud architects, and DevOps practitioners with intermediate AWS experience

**Documentation Type:** Architecture and Implementation Guide

---

## Introduction

Serverless architectures on AWS removes the requirement to manage server infrastructure, allowing teams to focus on application logic rather than infrastructure operations. However, managing the dozens of interconnected resources that make up a serverless backend — Lambda functions, API Gateway endpoints, DynamoDB tables, S3 buckets, Cognito user pools — can quickly become unwieldy without a disciplined approach to infrastructure definition.

This guide walks through the architecture and implementation of a production serverless backend for a marketplace application, built entirely with the **AWS Cloud Development Kit (CDK)** in TypeScript. The serverless backend supports:

- **Listing management** — full CRUD operations with image uploads
- **Search** — filtered, sortable queries across listings
- **Buyer/Seller interaction** — messaging between buyers and sellers (not yet fully live due to the exclusion of websockets)
- **Favourites** — user-scoped bookmarking of listings
- **Role-based access** — separate authentication flows for standard users and administrators
- **Admin dashboard** — view site live stats and CRUD operations for reported listings and users
- **Reporting System** — user reported listings will be sent to the admin for review and reported user will be notified
- **Map services** — geocoded user and listing information to display pins on a map

Rather than assembling these features ad hoc, the serverless backend uses the AWS CDK's construct model to encapsulate each feature as a self-contained, composable unit of infrastructure. The result is a codebase where adding a new feature means writing a single construct that declares its own Lambda functions, API routes, and database permissions — then plugging it into the stack.

### Document Scope

This document describes the serverless backend architecture and infrastructure
implementation of the CrocsList marketplace application.

It focuses on:
- infrastructure design decisions
- service integration patterns
- deployment and operational workflows

Frontend implementation details and UI behavior are intentionally out of scope.

### Terminology

- **Construct** — A reusable CDK component encapsulating AWS resources.
- **Feature Construct** — A project-specific construct representing one application domain.
- **Protected Route** — API Gateway endpoint requiring Cognito authentication.

---

## Architecture Overview

The serverless backend follows a standard three-tier serverless pattern on AWS:

**Figure 1 — High-level serverless architecture**
```
┌───────────────────────────────────────────────────────────┐
│                      Client Tier                          │
│       React SPA - Separate User & Admin Dashboards        │
└──────────────────────────┬────────────────────────────────┘
                           │ HTTPS + JWT (Cognito idToken)
                           ▼
┌───────────────────────────────────────────────────────────┐
│                      API Tier                             │
│                                                           │
│   ┌─────────────────────────────────────────────────┐     │
│   │            Amazon API Gateway (REST)            │     │
│   │                                                 │     │
│   │  /public/*     Unauthenticated (search, browse) │     │
│   │  /user/*       Cognito User Pool authorizer     │     │
│   │  /admin/*      Cognito Admin Pool authorizer    │     │
│   └──────────────────────┬──────────────────────────┘     │
│                          │  Lambda integration            │
│                          ▼                                │
|         ┌──────────┐   ┌──────────┐   ┌──────────┐        |
|         |  Mapping |   |  Admin   |   | Reporting|        |
|         |  Lambda  |   |  Lambda  |   |  Lambda  |        |
|         └────┬─────┘   └────┬─────┘   └────┬─────┘        |
│ ┌──────────┐ | ┌──────────┐ | ┌──────────┐ | ┌──────────┐ │
│ │ Listings │ | │  Search  │ | │   Chat   │ | │Favourites│ |
│ │  Lambda  │ | │  Lambda  │ | │  Lambda  │ | │  Lambda  │ |
│ └────┬─────┘ | └────┬─────┘ | └────┬─────┘ | └────┬─────┘ │
└──────┼───────┼──────┼───────┼──────┼───────┼──────┼───────┘
       │       │      │       │      │       │      │       
       ▼       ▼      ▼       ▼      ▼       ▼      ▼       
┌───────────────────────────────────────────────────────────┐
│                     Data Tier                             │
│                                                           │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│   │   DynamoDB   │  │      S3      │  │   Cognito    │    │
│   │              │  │              │  │              │    │
│   │ · Listings   │  │ Listing      │  │ User Pool    │    │
│   │ · LiveChat   │  │ Photos       │  │ Admin Pool   │    │
│   │ · Favourites │  │              │  │              │    │
│   │ · Users      │  │              │  │              │    │
│   └──────────────┘  └──────────────┘  └──────────────┘    │
└───────────────────────────────────────────────────────────┘
```

**Key design decisions:**

| Decision | Rationale |
|---|---|
| Single CDK stack | Keeps all resources in one CloudFormation stack for atomic deployments and simplified teardown. |
| Feature-scoped constructs | Each domain feature (listings, chat, etc.) owns its Lambda functions, routes, and permissions. |
| Python Lambda runtime | Lightweight cold starts and rapid iteration for data-oriented handlers. |
| Cognito authorizers on API Gateway | Token validation happens at the gateway level, before Lambda invocation, reducing compute cost. |
| DynamoDB on-demand capacity | Eliminates capacity planning for unpredictable marketplace traffic patterns. |

---

## Prerequisites

Before working with this codebase, ensure the following tools are installed and configured:

| Tool | Version | Purpose |
|---|---|---|
| Node.js | 18.x or later | CDK CLI and TypeScript compilation |
| AWS CDK CLI | 2.x | Infrastructure synthesis and deployment |
| AWS CLI | 2.x | Credential management and SSO login |
| Python | 3.11 | Lambda function runtime |
| TypeScript | 5.x | CDK infrastructure code |

You will also need an AWS account with appropriate IAM permissions and configured CLI credentials:

```bash
aws configure sso
aws sso login --profile <your-profile>
```

---

## Project Structure

The serverless backend follows a separation between **infrastructure definitions** (TypeScript, in `lib/`) and **runtime code** (Python, in `lambdas/`):

```
backend/
├── bin/
│   └── backend.ts              # CDK app entry point
├── lib/
│   ├── stacks/
│   │   └── backend-stack.ts    # Root stack — wires all constructs together
│   ├── ApiGateway/
│   │   ├── gateway.ts          # REST API and resource tree
│   │   └── endpoints.ts        # Route path constants
│   ├── Cognito/
│   │   └── cognito.ts          # User pool imports and authorizer setup
│   ├── DynamoDb/
│   │   └── tables.ts           # Table definitions and indexes
│   ├── S3/
│   │   └── listingPhotos.ts    # Photo upload bucket
│   └── features/
│       ├── UserListingCRUD.ts   # Listing create/update/delete + photo upload
│       ├── SearchListing.ts     # Search and read operations
│       ├── LiveChat.ts          # Messaging system
│       └── FavouritesCRUD.ts    # Bookmark management
├── lambdas/
│   ├── userListingCRUD/         # Python handlers for listing mutations
│   ├── searchListings/          # Python handlers for queries
│   ├── liveChat/                # Python handlers for chat
│   ├── mapping/                 # Python handlers for maps
│   ├── favourites/              # Python handlers for favourites
│   └── adminDashboard/          # Python handlers for admin and reporting
├── cdk.json
├── tsconfig.json
└── package.json
```

The `lib/` directory describes what infrastructure exists, while `lambdas/` contains what that infrastructure does at runtime.

---

## Infrastructure as Code with AWS CDK

### Why CDK Over CloudFormation or Terraform

AWS CDK allows you to define cloud infrastructure using general-purpose programming languages. Compared to writing raw CloudFormation YAML or Terraform HCL, CDK offers three practical advantages for this project:

1. **Type safety.** TypeScript catches misconfigured resources at compile time — for example, passing a DynamoDB table where an S3 bucket is expected.
2. **Abstraction via constructs.** Repeated patterns (like attaching a Cognito authorizer to every protected route) can be encapsulated in reusable methods.
3. **Same-language tooling.** Infrastructure and application configuration live in the same TypeScript project, sharing types, constants, and IDE support.

### The Construct Pattern

CDK organizes infrastructure into **constructs** — composable classes that represent one or more AWS resources. This project uses three levels of abstraction:

| Level | Example | What it represents |
|---|---|---|
| L1 (CFN Resources) | `CfnBucket` | A 1:1 mapping to a CloudFormation resource. |
| L2 (Curated Constructs) | `dynamodb.Table`, `lambda.Function` | An AWS resource with sensible defaults and convenience methods (e.g., `.grantReadData()`). |
| L3 (Patterns / Custom) | `ListingsFeatureConstruct` | A project-specific grouping of multiple L2 constructs into a single logical unit. |

The feature constructs in `lib/features/` are **L3 constructs** — each one bundles Lambda functions, API Gateway routes, and IAM permissions into a cohesive module that the root stack instantiates with a single call.

### Stack Composition

The `ProductionStack` class serves as the composition root. It instantiates shared infrastructure (Cognito, API Gateway, DynamoDB tables, S3) and passes references into each feature construct:

```typescript
export class ProductionStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const cognito   = new CognitoConstruct(this, "Cognito", { /* pool ARNs */ });
    const gateway   = new CRUDGatewayConstruct(this, "CrudGateway");
    const tables    = new DynamoTablesConstruct(this, "DynamoTables", userTableArn);
    const photoBucket = new ListingPhotosBucketConstruct(this, "ListingPhotosBucket");

    new ListingsFeatureConstruct(this, "ListingFeature", {
      tables, auth: cognito, api: gateway, listingPhotos: photoBucket,
    });

    new SearchListingsFeatureConstruct(this, "SearchListingsFeature", {
      api: gateway, tables, auth: cognito,
    });

    new LiveChatFeatureConstruct(this, "LiveChatFeature", {
      api: gateway, tables, auth: cognito,
    });

    new FavouritesFeatureConstruct(this, "FavouritesFature", {
      api: gateway, tables, auth: cognito,
    });
  }
}
```

Each feature construct receives only the shared resources it needs through its props interface. This dependency injection pattern makes it straightforward to add new features: define a construct, declare its dependencies, and plug it into the stack.

---

## Data Layer: DynamoDB Table Design

### Table Schemas

The application uses four DynamoDB tables, each optimized for a specific access pattern:

| Table | Partition Key | Sort Key | Purpose |
|---|---|---|---|
| **Users** | `id` (String) | — | User profiles (imported from external stack) |
| **Listings** | `listing_id` (String) | — | Marketplace item listings |
| **LiveChat** | `chatId` (String) | `timestamp` (String) | Chat message history |
| **Favourites** | `user_id` (String) | `listing_id` (String) | User bookmark associations |

Table names include the AWS account ID and region to prevent collisions across environments:

```typescript
const tableName = `Listings-${cdk.Stack.of(this).account}-${cdk.Stack.of(this).region}`;
```

### Access Patterns and Indexing

The Listings table includes a **Global Secondary Index** (GSI) to support a common query: "retrieve all listings owned by a specific user."

```typescript
this.listingsTable.addGlobalSecondaryIndex({
  indexName: "owner-id-index",
  partitionKey: { name: "owner_id", type: dynamodb.AttributeType.STRING },
  projectionType: dynamodb.ProjectionType.ALL,
});
```

The following table maps each access pattern to the key structure that serves it:

| Access Pattern | Table | Key Condition |
|---|---|---|
| Get listing by ID | Listings | `listing_id = :id` |
| Get all listings by owner | Listings (GSI) | `owner_id = :uid` |
| Search listings by keyword | Listings | Full table scan with filter |
| Get messages in a conversation | LiveChat | `chatId = :cid` (ordered by `timestamp`) |
| Check if user favourited a listing | Favourites | `user_id = :uid AND listing_id = :lid` |
| Get all favourites for a user | Favourites | `user_id = :uid` |

The **LiveChat** table's composite key (`chatId` + `timestamp`) allows efficient retrieval of a conversation's messages in chronological order using a single query operation, avoiding the need for client-side sorting.

The **Favourites** table uses a composite key (`user_id` + `listing_id`) to support both existence checks (is this listing favourited?) and range queries (all favourites for a user) without additional indexes.

---

## API Layer: API Gateway Configuration

### Resource Hierarchy

The API Gateway is organized into four top-level resource groups, each serving a different authorization context:

```
/
├── /public       No authentication required
│   ├── /search
│   ├── /userListings
│   └── /listing
├── /auth         Authentication endpoints (signup, signin)
├── /user         Cognito User Pool authorizer required
│   ├── /listings
│   │   └── /photo
│   ├── /favouritedListings
│   ├── /favourites
│   └── /chat
│       ├── /sendMessage
│       ├── /getMessages
│       └── /getAllChats
└── /admin        Cognito Admin Pool authorizer required
    ├── /reported-listings
    ├── /getAllListings
    ├── /delete-listing
    └── /getUsers
```

The `CRUDGatewayConstruct` exposes each top-level resource as a public property, allowing feature constructs to attach sub-resources without needing access to the full API object:

```typescript
export class CRUDGatewayConstruct extends Construct {
  public readonly api: apigw.RestApi;
  public readonly publicResource: apigw.Resource;
  public readonly authResource: apigw.Resource;
  public readonly userResource: apigw.Resource;
  public readonly adminResource: apigw.Resource;

  constructor(scope: Construct, id: string) {
    super(scope, id);

    this.api = new apigw.RestApi(this, "CRUDApi", {
      restApiName: "CRUD API",
      deployOptions: { stageName: "prod" },
    });

    this.publicResource = this.api.root.addResource("public");
    this.userResource   = this.api.root.addResource("user");
    this.adminResource  = this.api.root.addResource("admin");
  }
}
```

Route path segments are defined as constants in a shared `endpoints.ts` file, preventing typos and enabling IDE auto-completion across feature constructs:

```typescript
export const API_ENDPOINTS = {
  user: {
    value: "user",
    chat: {
      value: "chat",
      sendMessage: "sendMessage",
      getMessages: "getMessages",
      getAllChats: "getAllChats",
    },
  },
  // ...
} as const;
```

### Route Definitions

The complete API surface is summarized below:

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/public/search` | None | Search listings by keyword, tags, sort order |
| `GET` | `/public/userListings` | None | List all listings for a given user |
| `GET` | `/public/listing` | None | Retrieve a single listing by ID |
| `POST` | `/user/listings` | User | Create a new listing |
| `PATCH` | `/user/listings` | User | Update an existing listing |
| `DELETE` | `/user/listings` | User | Delete a listing |
| `POST` | `/user/listings/photo` | User | Generate a presigned URL for photo upload |
| `GET` | `/user/favouritedListings` | User | Retrieve listings the user has favourited |
| `GET` | `/user/favourites` | User | Check or list favourite associations |
| `POST` | `/user/favourites` | User | Add a listing to favourites |
| `DELETE` | `/user/favourites` | User | Remove a listing from favourites |
| `POST` | `/user/chat/sendMessage` | User | Send a chat message |
| `GET` | `/user/chat/getMessages` | User | Retrieve messages for a conversation |
| `GET` | `/user/chat/getAllChats` | User | List all conversations for the user |

All `/user/*` and `/admin/*` routes require a valid Cognito `idToken` in the `Authorization` header. CORS preflight responses are configured on every protected resource to support browser-based clients.

---

## Compute Layer: Lambda Functions

### Function Organization

Lambda functions are written in Python 3.11 and grouped by feature domain. Each feature construct creates one or more Lambda functions, wires them to API Gateway routes, and grants them the minimum necessary DynamoDB permissions.

| Feature | Lambda | Handler | Trigger |
|---|---|---|---|
| **Listings** | `listingCUDLambda` | `listing_cud.lambda_handler` | `POST/PATCH/DELETE /user/listings` |
| **Listings** | `listingPhotoLambda` | `listing_photo.lambda_handler` | `POST /user/listings/photo` |
| **Search** | `SearchListingsLambda` | `search.lambda_handler` | `GET /public/search` |
| **Search** | `userListingsLambda` | `get_user_listings.lambda_handler` | `GET /public/userListings` |
| **Search** | `favouritedListingsLambda` | `get_user_favourited_listings.lambda_handler` | `GET /user/favouritedListings` |
| **Search** | `GetListingByIdLambda` | `get_listing_by_id.lambda_handler` | `GET /public/listing` |
| **Chat** | `SendMessageLambda` | `sendMessage.lambda_handler` | `POST /user/chat/sendMessage` |
| **Chat** | `GetMessagesLambda` | `getMessages.lambda_handler` | `GET /user/chat/getMessages` |
| **Chat** | `getAllChatsLambda` | `getAllChats.lambda_handler` | `GET /user/chat/getAllChats` |
| **Favourites** | `FavouritesCRUDLambda` | `favourites.lambda_handler` | `GET/POST/DELETE /user/favourites` |

Table names and bucket names are passed to Lambda functions through environment variables, keeping runtime code decoupled from CDK resource identifiers:

```typescript
const listingCUDLambda = new lambda.Function(this, "listingCUDLambda", {
  runtime: lambda.Runtime.PYTHON_3_11,
  handler: "listing_cud.lambda_handler",
  code: lambda.Code.fromAsset("lambdas/userListingCRUD"),
  environment: {
    LISTINGS_TABLE: props.tables.listingsTable.tableName,
    USER_POOL_ID: props.auth.userPool.userPoolId,
  },
});
```

### IAM Permissions via CDK Grants

CDK's grant methods generate least-privilege IAM policies without requiring you to write policy JSON directly. Each Lambda function receives only the permissions it needs:

```typescript
// Read-write access for mutation operations
props.tables.listingsTable.grantReadWriteData(listingCUDLambda);

// Read-only access for query operations
props.tables.listingsTable.grantReadData(searchLambda);

// Write-only access for S3 uploads via presigned URLs
props.listingPhotos.bucket.grantPut(listingPhotoLambda);
```

This approach is both more concise and more secure than hand-crafting IAM policy documents. CDK scopes the generated policy to the specific table or bucket ARN, preventing accidental cross-resource access.

---

## Authentication: Cognito Integration

### Dual User Pool Architecture

The application maintains two separate Cognito User Pools to enforce role isolation:

| Pool | Self-Signup | Purpose |
|---|---|---|
| **User Pool** | Enabled | Standard marketplace users (buyers and sellers) |
| **Admin Pool** | Disabled | Platform administrators (invite-only) |

Separating these pools ensures that a compromised user token cannot escalate to admin privileges, even if the application logic has a vulnerability. Admin accounts are created exclusively through AWS Console or CLI, never through the application's signup flow.

In the CDK stack, both pools are imported by ARN from a separately managed deployment:

```typescript
export class CognitoConstruct extends Construct {
  public readonly userPool: cognito.IUserPool;
  public readonly adminPool: cognito.IUserPool;

  constructor(scope: Construct, id: string, props: CognitoConstructProps) {
    super(scope, id);

    this.userPool = cognito.UserPool.fromUserPoolArn(
      this, "ImportedUserPool", props.userPoolArn
    );
    this.adminPool = cognito.UserPool.fromUserPoolArn(
      this, "ImportedAdminPool", props.adminPoolArn
    );
  }
}
```

Importing pools by ARN (rather than creating them in this stack) prevents accidental deletion of user data during stack teardown.

### Authorizer Configuration

API Gateway authorizers validate Cognito `idToken` JWTs *before* invoking Lambda functions. This means unauthenticated requests never reach the compute layer, reducing both cost and attack surface.

Each feature construct creates its own authorizer instance and attaches it to protected routes through a shared helper method:

```typescript
private addMethodWithAuthorizer(
  resource: apigw.IResource,
  method: string,
  lambdaFn: lambda.IFunction,
  authorizer: apigw.IAuthorizer
) {
  resource.addMethod(method, new apigw.LambdaIntegration(lambdaFn), {
    authorizer,
    authorizationType: apigw.AuthorizationType.COGNITO,
  });
}
```

This helper encapsulates the three concerns of every protected route — Lambda integration, authorizer attachment, and auth type declaration — into a single call, reducing boilerplate and ensuring consistency.

---

## Storage: S3 for Media Assets

### Presigned URL Upload Flow

Listing images are stored in an S3 bucket with public read access enabled, allowing direct browser access to images without proxying through Lambda:

```typescript
this.bucket = new s3.Bucket(this, "ListingPhotosBucket", {
  bucketName: `listing-photos-${cdk.Stack.of(this).account}-${cdk.Stack.of(this).region}`,
  removalPolicy: cdk.RemovalPolicy.RETAIN,
  publicReadAccess: true,
  blockPublicAccess: s3.BlockPublicAccess.BLOCK_ACLS_ONLY,
  cors: [{
    allowedOrigins: ["*"],
    allowedMethods: [s3.HttpMethods.GET, s3.HttpMethods.PUT],
    allowedHeaders: ["*"],
  }],
});
```

Uploads use a **presigned URL** pattern to avoid routing file data through API Gateway and Lambda, which have payload size limits (10 MB and 6 MB respectively):
**Figure 2 — Presigned upload workflow**
```
┌───────────┐          ┌──────────────┐          ┌─────────────┐
│  Client   │  POST    │ Photo Lambda │ Presign  │     S3      │
│ (Browser) │─────────>│ (generates   │─────────>│   Bucket    │
│           │<─────────│  signed URL) │          │             │
│           │  URL     │              │          │             │
│           │──────────┼──────────────┼────PUT──>│             │
└───────────┘          └──────────────┘          └─────────────┘
```

1. The client sends a `POST` request to `/user/listings/photo` with metadata (filename, content type).
2. The Lambda function generates a time-limited presigned PUT URL for S3.
3. The client uploads the file directly to S3 using the presigned URL.

This approach offloads bandwidth from the API tier and allows uploads of any size supported by S3 (up to 5 GB for single PUT operations).

---

## Feature Constructs: Modular Implementation

Each feature construct follows a consistent internal pattern:

1. **Create a Cognito authorizer** for the feature's protected routes.
2. **Define Lambda function(s)** with the Python runtime and appropriate environment variables.
3. **Grant database permissions** using CDK's `.grantReadData()` or `.grantReadWriteData()` methods.
4. **Attach routes** to the API Gateway resource tree.
5. **Configure CORS** preflight responses on protected resources.

### Listings CRUD

The `ListingsFeatureConstruct` manages two Lambda functions:

- **`listingCUDLambda`** handles `POST`, `PATCH`, and `DELETE` operations on `/user/listings`, all routed to a single Lambda that dispatches by HTTP method.
- **`listingPhotoLambda`** generates presigned S3 upload URLs at `/user/listings/photo`.

Routing multiple HTTP methods to a single Lambda reduces cold-start surface area — instead of three separate functions, one warm container handles all mutation operations.

### Search

The `SearchListingsFeatureConstruct` creates four Lambda functions that serve different read patterns:

| Function | Route | Description |
|---|---|---|
| `SearchListingsLambda` | `GET /public/search` | Keyword search with optional tag filters and sort |
| `userListingsLambda` | `GET /public/userListings` | All listings by a specific seller |
| `favouritedListingsLambda` | `GET /user/favouritedListings` | Listings bookmarked by the authenticated user |
| `GetListingByIdLambda` | `GET /public/listing` | Single listing detail view |

Public search routes use `AuthorizationType.NONE`, allowing unauthenticated users to browse the marketplace. The favourited listings endpoint requires authentication because it queries the Favourites table using the caller's user ID.

### Chat

The `LiveChatFeatureConstruct` implements a messaging system with three operations:

- **Send message** (`POST /user/chat/sendMessage`) — writes a message item to the LiveChat table with a composite key of `chatId` (a deterministic combination of both user IDs) and `timestamp`.
- **Get messages** (`GET /user/chat/getMessages`) — queries all messages for a specific conversation, returned in chronological order by sort key.
- **Get all chats** (`GET /user/chat/getAllChats`) — retrieves the list of all conversation partners for the authenticated user.

Route path segments use the shared `API_ENDPOINTS` constants to ensure consistency between infrastructure definitions and any client-side route references:

```typescript
const chatRoot = props.api.userResource.addResource(API_ENDPOINTS.user.chat.value);
const sendMessageResource = chatRoot.addResource(API_ENDPOINTS.user.chat.sendMessage);
```

### Favourites

The `FavouritesFeatureConstruct` routes all three operations (`GET`, `POST`, `DELETE`) to a single Lambda function at `/user/favourites`. The Lambda inspects the HTTP method at runtime to determine the operation:

- **POST** — adds a `user_id` / `listing_id` pair to the Favourites table.
- **DELETE** — removes the pair.
- **GET** — checks existence of a specific pair or returns all favourites for the user.

This pattern keeps the infrastructure minimal (one Lambda, one route) while still supporting full CRUD semantics.

---

### Mapping

The mapping feature provides geocoding capabilities that convert between street addresses and geographic coordinates. Unlike the previous feature constructs, the mapping Lambdas are deployed outside the CDK stack (under `lambdas/NonCDK/mapping/`) and exposed through a separate API Gateway deployment. This separation keeps the lightweight, stateless geocoding service decoupled from the main application stack.

Two Lambda functions handle the two directions of geocoding:

| Function | Route | Description |
|---|---|---|
| `getCoords` | `POST /geocode` | Forward geocode — converts an address string to latitude/longitude |
| `getPlace` | `POST /reverse-geocode` | Reverse geocode — converts latitude/longitude to a human-readable address |

Both functions use the **AWS Location Service (Geo-Places)** API through the `boto3` `geo-places` client, avoiding the need for third-party geocoding services or API key management:

```python
client = boto3.client('geo-places', region_name='us-west-2')

# Forward geocode: address → coordinates
resp = client.geocode(QueryText=address, MaxResults=1)
lng, lat = resp['ResultItems'][0]['Position']

# Reverse geocode: coordinates → address
resp = client.reverse_geocode(QueryPosition=[lng, lat], MaxResults=1)
address = resp['ResultItems'][0]['Address']['Label']
```

Neither Lambda accesses DynamoDB directly. Instead, the geocoding results feed into the listing creation flow: when a user creates or edits a listing, the client calls the geocoding endpoint first, then includes the returned `latitude` and `longitude` in the listing payload sent to the Listings CRUD Lambda. The coordinates are stored alongside the listing in DynamoDB, enabling map-based display on the frontend.

**Figure 3 — Mapping Workflow**
```
┌───────────┐  address   ┌──────────────┐  coords   ┌───────────┐  listing + coords ┌──────────┐
│  Client   │───────────>│ getCoords    │──────────>│  Client   │──────────────────>│ Listings │
│ (Browser) │<───────────│ Lambda       │           │ (Browser) │                   │  Lambda  │
│           │  {lat,lng} │              │           │           │                   │          │
└───────────┘            └──────────────┘           └───────────┘                   └──────────┘
```

The mapping endpoints require no authentication, since geocoding results are not user-specific and carry no sensitive data.

---

### Admin and Reporting

The admin and reporting features provide platform administrators with tools to monitor marketplace activity, review reported listings, and take moderation actions. Like the mapping feature, these Lambdas are deployed outside the CDK stack (under `lambdas/NonCDK/admin-dashboard/`) and are wired to the `/admin` resource group, which is protected by the Cognito Admin Pool authorizer.

The feature spans five Lambda functions across two concerns: **admin dashboard operations** and **user-facing reporting**.

#### Admin Dashboard

| Function | Route | Auth | Description |
|---|---|---|---|
| `getAllListings` | `GET /admin/getAllListings` | Admin | Full scan of the Listings table |
| `getUsers` | `GET /admin/getUsers` | Admin | Full scan of the Users table |
| `getReportedAndDeleted` | `GET /admin/reported-listings` | Admin | Listings with reports, split by moderation status |
| `deleteListing` | `POST /admin/delete-listing` | Admin | Soft-delete a listing and notify the seller |

The `getReportedAndDeleted` function scans the Listings table and categorizes results into two groups based on their moderation state.

This gives the admin dashboard two views: listings that have been reported but are still active (requiring review), and listings that have already been removed (for audit trail purposes). Reports within each listing are sorted chronologically by `reported_at`.

#### Soft-Delete with Email Notification

The `deleteListing` Lambda implements a **soft-delete toggle** rather than permanent deletion. It flips the `is_removed` boolean on the listing and, when removing a listing, triggers an email notification to the seller via **Amazon SES**.

The seller's email address is resolved by calling a separate user-lookup endpoint (configured via the `GET_USER_ENDPOINT` environment variable), and the SES email includes the listing name and the admin's removal reason. Toggling `is_removed` back to `false` restores the listing and clears the `removed_reason` field, allowing administrators to reverse moderation decisions.

#### User Reporting

| Function | Route | Auth | Description |
|---|---|---|---|
| `reportListing` | `POST /user/report/listing` | User | Submit a report against a listing |

The reporting flow starts on the user side. Any authenticated user can report a listing by submitting a reason, which the `reportListing` Lambda appends to a `reports` list attribute on the listing item.

Using `list_append` with `if_not_exists` ensures the first report on a listing initializes the list rather than failing on a missing attribute. Each report captures who filed it, when, and why — giving administrators full context when reviewing flagged content.

#### End-to-End Moderation Flow

The reporting and admin features form a complete moderation pipeline:

**Figure 4 — Listing Reporting Workflow**
```
┌──────────┐  report   ┌───────────────┐           ┌──────────────┐
│   User   │──────────>│ reportListing │─ append ─>│   Listings   │
│          │           │   Lambda      │  report   │    Table     │
└──────────┘           └───────────────┘           └──────┬───────┘
                                                          │
┌──────────┐  review   ┌───────────────┐    scan          │
│  Admin   │<──────────│ getReported   │<─────────────────┘
│          │  listed   │ AndDeleted    │
│          │           └───────────────┘
│          │
│          │  remove   ┌───────────────┐  SES email  ┌──────────┐
│          │──────────>│ deleteListing │────────────>│  Seller  │
└──────────┘           └───────────────┘             └──────────┘
```

1. A user reports a listing via `POST /user/report/listing`.
2. The report is appended to the listing's `reports` list in DynamoDB.
3. An administrator reviews reported listings via `GET /admin/reported-listings`.
4. The administrator removes a listing via `POST /admin/delete-listing`, which soft-deletes it and sends an SES email to the seller explaining the removal.

---

## Deployment

### Synthesize and Deploy

CDK synthesizes the TypeScript constructs into a CloudFormation template, then deploys it:

```bash
cd backend
npm run build          # Compile TypeScript
npx cdk synth          # Preview the CloudFormation template
npx cdk deploy         # Deploy to AWS
```

### Verify the Deployment

After deployment, CDK outputs the API Gateway endpoint URL. Verify the stack is functional:

```bash
# Test public search endpoint (no auth required)
curl "https://<api-id>.execute-api.us-west-2.amazonaws.com/prod/public/search?name=test"
```

### Tear Down

To remove all resources created by the stack:

```bash
npx cdk destroy
```

> **Note:** S3 buckets configured with `removalPolicy: RETAIN` will persist after stack deletion to prevent accidental data loss. Delete them manually through the AWS Console if needed.

---

## Known Limitations

The current architecture intentionally prioritizes simplicity and rapid iteration.
The following limitations are known:

- Listing search relies on DynamoDB scans and may require OpenSearch for large datasets.
- Live chat uses request-based polling rather than WebSockets, limiting real-time responsiveness.
- A single CDK stack simplifies deployment but may increase deployment duration as features expand.
- Mapping and admin services are deployed outside CDK, introducing operational inconsistency.

---

## Summary

This guide covered the architecture and implementation of a serverless backend built with AWS CDK, spanning seven core AWS services:

| Service | Role in Architecture |
|---|---|
| **AWS CDK** | Infrastructure definition, deployment, and permissions management |
| **API Gateway** | Request routing, CORS, and pre-Lambda authentication |
| **Lambda** | Stateless business logic (Python 3.11) |
| **DynamoDB** | Low-latency data storage with single-digit-millisecond reads |
| **Cognito** | JWT-based authentication with role isolation |
| **S3** | Media storage with direct client uploads via presigned URLs |
| **SES** | Send notifications to Sellers, Buyers, and Admins |

The **construct-per-feature** pattern demonstrated here scales well as the application grows. Adding a new feature — say, a notification system — means creating a new construct that declares its own Lambda, routes, and permissions, then adding a single line to the root stack. No existing code needs modification.

For teams evaluating serverless on AWS, CDK offers a practical middle ground: the full power of CloudFormation with the ergonomics of a typed programming language. The patterns in this guide — feature constructs, grant-based permissions, presigned upload flows, and dual-pool authentication — provide a reusable foundation for production serverless applications.
