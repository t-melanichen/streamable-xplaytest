# API Contracts: Instantly Shareable Playtest

S2S API contracts for the PlayTest ↔ Partner Registry ↔ GSSV integration.

---

## 1. PlayTest → Partner Registry: Create Offering

**When**: Playtest created with `cloudStreamingEnabled: true`

```http
POST /v1/offerings
Authorization: Bearer {s2s-token}
Content-Type: application/json
MS-CV: {correlation-vector}
```

### Request Body
```json
{
  "offeringId": "xpt-{sellerId}-{playtestId}",
  "displayName": "Playtest: {gameName}",
  "description": "Private playtest offering for {gameName}",
  "type": "playtest",
  "auth": {
    "type": "dnaGroup",
    "dnaGroupId": "{playtestAudienceDnaGroupId}"
  },
  "regions": ["EastUS", "WestUS", "WestEurope"],
  "sessionLimits": {
    "maxConcurrentSessions": 50
  },
  "serviceLevel": "standard",
  "expiration": "2026-09-01T00:00:00Z",
  "metadata": {
    "sellerId": "{sellerId}",
    "partnerCenterProductId": "{productId}",
    "playtestId": "{playtestId}",
    "source": "xplaytest"
  }
}
```

### Response (201 Created)
```json
{
  "offeringId": "xpt-{sellerId}-{playtestId}",
  "dedicatedFqdn": "xpt-{sellerId}-{playtestId}.gssv-play-prod.xboxlive.com",
  "status": "active",
  "createdAt": "2026-06-15T10:00:00Z"
}
```

---

## 2. PlayTest → Partner Registry: Delete Offering

**When**: Playtest deleted

```http
DELETE /v1/offerings/{offeringId}
Authorization: Bearer {s2s-token}
MS-CV: {correlation-vector}
```

### Response (204 No Content)
Empty body. Offering fully cleaned up.

---

## 3. PlayTest → Partner Registry: Update Offering (DNA Group Change)

**When**: Playtest audience/DNA group is modified

```http
PATCH /v1/offerings/{offeringId}
Authorization: Bearer {s2s-token}
Content-Type: application/json
MS-CV: {correlation-vector}
```

### Request Body
```json
{
  "auth": {
    "type": "dnaGroup",
    "dnaGroupId": "{newDnaGroupId}"
  }
}
```

### Response (200 OK)
```json
{
  "offeringId": "xpt-{sellerId}-{playtestId}",
  "auth": {
    "type": "dnaGroup",
    "dnaGroupId": "{newDnaGroupId}"
  },
  "updatedAt": "2026-06-16T14:00:00Z"
}
```

---

## 4. PlayTest → Partner Registry: Update Offering Title (New Build)

**When**: Build ingested into GSSV successfully

```http
PUT /v1/offerings/{offeringId}/titles
Authorization: Bearer {s2s-token}
Content-Type: application/json
MS-CV: {correlation-vector}
```

### Request Body
```json
{
  "titles": [
    {
      "titleId": "{titleId}",
      "productId": "{partnerCenterProductId}",
      "buildId": "{ingestedBuildId}",
      "platform": "XboxOneAndSeriesX",
      "inputTypes": ["controller", "touch"],
      "availability": {
        "startDate": "2026-06-15T00:00:00Z",
        "endDate": "2026-09-01T00:00:00Z"
      }
    }
  ]
}
```

### Response (200 OK)
```json
{
  "offeringId": "xpt-{sellerId}-{playtestId}",
  "titles": [
    {
      "titleId": "{titleId}",
      "buildId": "{ingestedBuildId}",
      "status": "active"
    }
  ],
  "updatedAt": "2026-06-17T09:00:00Z"
}
```

---

## 5. PlayTest → GSSV: Ingest Build

**When**: `XPackagePlaytestPublishWorkflowJobStatusTopicProcessor` fires for a cloud-enabled playtest

```http
POST /v1/builds/ingest
Authorization: Bearer {s2s-token}
Content-Type: application/json
MS-CV: {correlation-vector}
```

### Request Body
```json
{
  "packageId": "{xpackageId}",
  "partnerCenterProductId": "{productId}",
  "offeringId": "xpt-{sellerId}-{playtestId}",
  "buildSource": "xplaytest",
  "metadata": {
    "playtestId": "{playtestId}",
    "sellerId": "{sellerId}",
    "buildVersion": "1.0.23.456"
  }
}
```

### Response (202 Accepted)
```json
{
  "ingestionId": "{guid}",
  "status": "queued",
  "estimatedCompletionMinutes": 5
}
```

> **Note**: Build ingestion is async. PlayTest should poll or receive a callback when ingestion completes before updating the offering title.

---

## 6. GSSV → Bayside: Get Offerings (existing, needs DNA group support)

**When**: Bayside checks what offerings a user has access to

```http
GET /offerings
Authorization: Bearer {user-token}
X-XBL-Contract-Version: 4
```

### Response (200 OK) — with playtest offerings included
```json
{
  "offerings": [
    {
      "offeringId": "xpt-{sellerId}-{playtestId}",
      "displayName": "Playtest: {gameName}",
      "type": "playtest",
      "dedicatedFqdn": "xpt-{sellerId}-{playtestId}.gssv-play-prod.xboxlive.com",
      "titles": [
        {
          "titleId": "{titleId}",
          "productId": "{productId}",
          "name": "{gameName}",
          "imageUrl": "https://...",
          "buildId": "{buildId}"
        }
      ]
    }
  ]
}
```

> **Known Issue**: DNA group-authed offerings currently don't appear in this response. This needs a GSSV-side fix (P0 item #6).

---

## 7. PlayTestFD Response Extension

The PlayTestFD response model should include streaming info when available:

### Extended PlaytestModel (new fields)
```json
{
  "playtestId": "guid",
  "name": "My Game Nightly Build",
  "status": "published",
  "cloudStreamingEnabled": true,
  "streaming": {
    "offeringId": "xpt-{sellerId}-{playtestId}",
    "streamingLink": "https://xbox.com/play/launch/xpt-{sellerId}-{playtestId}",
    "lastBuildStatus": "ready",
    "lastBuildIngestedAt": "2026-06-17T09:05:00Z"
  }
}
```

---

## Authentication Notes

### S2S Auth (PlayTest → Partner Registry / GSSV)
- Use Azure AD app-to-app tokens (existing Xbox S2S pattern)
- PlayTest service identity needs permissions on Partner Registry and GSSV
- Follows existing `ServiceClient` auth configuration in Xbox.Xbet.Service

### User Auth (Bayside)
- Xbox Live token with XBL user claims
- DNA group membership checked server-side by GSSV
- No client-side group check (prevents spoofing)
