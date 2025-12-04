---
layout: default
title: Home
---

# Dash Marketing API Documentation

## Welcome to Dash Marketing

Dash Marketing provides a comprehensive platform for managing your locations, collecting leads, tracking analytics, and building custom landing pages. Our API enables you to integrate Dash Marketing's powerful features directly into your applications, websites, and third-party tools.

## Getting Started

### Base URL

All API requests should be made to:

```
https://api.dashmarketing.io
```

### Authentication

All API endpoints require authentication using an API key. Include your API key in the request headers:

```
X-API-KEY: your_api_key_here
```

### Obtaining an API Key

**Option 1: Request from Dash Marketing**
Contact your Dash Marketing representative at support@dashmarketing.io to request an API key for your organization.

**Option 2: Create Your Own in Dash Core**
1. Log in to Dash Core
2. Navigate to **Organization Settings** > **API Keys**
3. Click **Create New API Key**
4. Copy and store the key securely (it will only be shown once)

### Security Best Practices

- Never expose your API key in client-side code or public repositories
- Store API keys in environment variables
- Use separate keys for different integrations
- Rotate keys periodically
- Revoke compromised keys immediately

---

## Available API Endpoints

### Lead Management

#### [Lead Submission API →](./LEAD_SUBMISSION_API.md)

Submit leads to your Dash Core locations programmatically. Perfect for:
- Website contact forms
- Landing page forms
- Third-party integrations (Zapier, Make, etc.)
- Custom applications
- Marketing automation platforms

**Key Features:**
- Simple REST API
- Custom metadata support for form fields
- Rate limiting and CAPTCHA protection
- Real-time lead submission
- Automatic lead tracking and analytics

**Quick Example:**

```bash
curl -X POST https://api.dashmarketing.io/lead/Submit \
  -H "Content-Type: application/json" \
  -H "X-API-KEY: your_api_key_here" \
  -d '{
    "email": "customer@example.com",
    "firstName": "John",
    "lastName": "Doe",
    "locationUuid": "your-location-uuid",
    "source": "Website",
    "metadata": {
      "customField": "value"
    }
  }'
```

[View Full Documentation →](./LEAD_SUBMISSION_API.md)

---

## Coming Soon

Additional API endpoints are in development and will be documented here as they become available:

### Contact Management API
Manage contacts across your organization, update contact information, and track customer interactions.

### Location API
Retrieve location information, search locations, and manage location data.

### Analytics API
Access lead analytics, conversion metrics, and performance data for your locations.

### Page Management API
Create and manage landing pages programmatically.

---

## API Standards

### Request Format

All requests must include:
- `Content-Type: application/json` header
- `X-API-KEY` header with your API key
- Valid JSON body for POST/PUT requests

### Response Format

All responses return JSON with the following structure:

**Success Response:**
```json
{
  "success": true,
  "message": "Operation completed successfully",
  "data": { ... },
  "timestamp": "2025-12-04T10:30:00.000Z"
}
```

**Error Response:**
```json
{
  "success": false,
  "message": "Error description",
  "errors": ["Error detail 1", "Error detail 2"],
  "timestamp": "2025-12-04T10:30:00.000Z"
}
```

### HTTP Status Codes

| Code | Description |
|------|-------------|
| 200 | Success |
| 400 | Bad Request - Invalid parameters or validation error |
| 401 | Unauthorized - Missing or invalid authentication |
| 403 | Forbidden - Valid authentication but insufficient permissions |
| 404 | Not Found - Resource doesn't exist |
| 429 | Too Many Requests - Rate limit exceeded |
| 500 | Internal Server Error - Something went wrong on our end |

### Rate Limiting

All API endpoints are subject to rate limiting to ensure fair usage:

- **Default:** 5 requests per 60 seconds per IP address
- Rate limits apply per endpoint
- Headers indicate rate limit status:
  - `X-RateLimit-Limit` - Maximum requests allowed
  - `X-RateLimit-Remaining` - Requests remaining
  - `X-RateLimit-Reset` - Time when limit resets (Unix timestamp)

If you need higher rate limits for your use case, contact support@dashmarketing.io.

---

## Data Types and Formats

### UUIDs

Locations, organizations, leads, and other resources are identified by UUIDs (Universally Unique Identifiers):

```
Format: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
Example: a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

### Dates and Timestamps

All dates and timestamps use ISO 8601 format in UTC:

```
Format: YYYY-MM-DDTHH:mm:ss.sssZ
Example: 2025-12-04T10:30:00.000Z
```

### Phone Numbers

Phone numbers should include country code:

```
Format: +1-555-123-4567 or +15551234567
```

### Email Addresses

Standard email format:

```
Format: user@domain.com
```

---

## SDKs and Libraries

Official SDKs are in development. In the meantime, you can use standard HTTP clients:

**JavaScript/TypeScript:**
- `fetch` API
- `axios`
- `node-fetch`

**Python:**
- `requests`
- `httpx`
- `urllib3`

**PHP:**
- `cURL`
- `Guzzle`
- `file_get_contents` with stream context

**C#/.NET:**
- `HttpClient`
- `RestSharp`

---

## Webhooks

Webhook support is coming soon. Webhooks will allow you to receive real-time notifications for events such as:

- New lead submissions
- Lead status changes
- Contact updates
- Form submissions

Stay tuned for webhook documentation.

---

## Support and Resources

### Contact Support

**Email:** support@dashmarketing.io

**Response Times:**
- Critical issues: Within 4 hours
- General inquiries: Within 24 hours

**Support Includes:**
- API integration assistance
- Troubleshooting and debugging
- Rate limit increase requests
- Feature requests and feedback


## Best Practices

### Error Handling

Always implement proper error handling:

```javascript
try {
  const response = await fetch('https://api.dashmarketing.io/lead/Submit', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-API-KEY': apiKey
    },
    body: JSON.stringify(leadData)
  });

  if (!response.ok) {
    const error = await response.json();
    console.error('API Error:', error);
    // Handle specific error cases
    if (response.status === 429) {
      // Rate limited - implement backoff
    } else if (response.status === 400) {
      // Validation error - check form data
    }
  }

  const result = await response.json();
  return result;
} catch (error) {
  console.error('Network Error:', error);
  // Handle network errors
}
```

### Retry Logic

Implement exponential backoff for transient failures:

```javascript
async function submitWithRetry(data, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await submitLead(data);
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      const delay = Math.pow(2, i) * 1000; // 1s, 2s, 4s
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

### Testing

Test your integration thoroughly:

1. **Test with valid data** - Ensure successful submissions work
2. **Test with invalid data** - Verify error handling
3. **Test rate limiting** - Ensure backoff logic works
4. **Test network failures** - Simulate connection issues
5. **Test different scenarios** - Various form fields, metadata combinations

---

## Terms of Service

By using the Dash Marketing API, you agree to:

- Use the API only for authorized purposes
- Not exceed rate limits without approval
- Not attempt to circumvent security measures
- Comply with all applicable laws and regulations
- Properly secure and protect your API keys

For full Terms of Service, visit: https://www.dashmarketing.io/terms

---

## Privacy and Data Protection

Dash Marketing is committed to protecting user data:

- All API traffic is encrypted via HTTPS/TLS
- API keys are hashed and stored securely
- Lead data is stored in compliance with GDPR and CCPA
- Data retention policies can be configured per organization

For our full Privacy Policy, visit: https://www.dashmarketing.io/privacy

---

## Feedback

We're constantly improving our API. Share your feedback:

- **Feature Requests:** support@dashmarketing.io
- **Bug Reports:** support@dashmarketing.io
- **Documentation Issues:** support@dashmarketing.io

Your input helps us build better tools for your business.

---

**Last Updated:** December 2025

**Version:** 1.0
