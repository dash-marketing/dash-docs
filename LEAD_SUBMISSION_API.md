# Lead Submission API Documentation

## Overview

The Dash Lead Submission API allows you to programmatically submit leads to your Dash Core locations. This API is designed for integrating with forms, third-party platforms (like Zapier), and custom applications.

**Base URL:** `https://api.dashmarketing.io`

**Endpoint:** `POST /lead/Submit`

---

## Authentication

All API requests require authentication using an API key. The API key must be included in the request headers.

### Header

```
X-API-KEY: your_api_key_here
```

### Obtaining an API Key

There are two ways to obtain an API key:

#### Option 1: Request from Dash Marketing

Contact your Dash Marketing representative to request an API key for your organization. They will provide you with:
- Your organization-specific API key
- Location UUIDs for your locations
- Any additional configuration or support needed

#### Option 2: Create and Manage Keys in Dash Core

If you have access to Dash Core, you can create and manage your own API keys:

1. Log in to your Dash Core account
2. Navigate to **Organization Settings** > **API Keys**
3. Click **Create New API Key**
4. Provide a name for the key (e.g., "Website Contact Form", "Zapier Integration")
5. Copy the generated key immediately - it will only be shown once
6. Store the key securely

**Security Best Practices:**
- Never share your API key publicly or commit it to version control
- Use environment variables to store API keys in your applications
- Rotate keys periodically for enhanced security
- Create separate keys for different integrations to limit exposure
- Revoke keys immediately if compromised

---

## Request

### Endpoint

```
POST /lead/Submit
```

### Headers

```
Content-Type: application/json
X-API-KEY: your_api_key_here
```

### Request Body

The request body must be a JSON object with the following fields:

#### Required Fields

| Field | Type | Description | Max Length |
|-------|------|-------------|------------|
| `email` | string | Email address of the lead | 255 |
| `locationUuid` | string | UUID of the location this lead is associated with | 36 |

#### Optional Fields

| Field | Type | Description | Max Length |
|-------|------|-------------|------------|
| `firstName` | string | Lead's first name | 100 |
| `lastName` | string | Lead's last name | 100 |
| `phone` | string | Lead's phone number (must be valid phone format) | 20 |
| `estimatedStartDate` | string | ISO 8601 date when lead wants to start service (e.g., "2025-01-15T00:00:00Z") | - |
| `source` | string | Source of the lead. Valid values: `QR`, `Website`, `Referral`, `InPerson`, `SocialMedia`, `Advertisement`, `DashPage`, `DashForm`, `Other` | 100 |
| `sourceDetails` | string | Additional details about the source (e.g., "Google Ads Campaign XYZ") | 200 |
| `notes` | string | Additional notes about the lead or their inquiry | 2000 |
| `formName` | string | Name of the form where lead was submitted (e.g., "Contact Us", "Get Quote") | 100 |
| `slug` | string | Page slug where form was submitted | 200 |
| `referrerUrl` | string | URL the lead came from | 500 |
| `metadata` | object | Additional custom fields collected on your form (see Metadata section below) | - |
| `captchaToken` | string | CAPTCHA token for bot protection (if CAPTCHA is enabled) | - |
| `captchaType` | string | Type of CAPTCHA. Valid values: `recaptcha`, `hcaptcha`, `turnstile` | - |

#### Metadata

The `metadata` field allows you to include any additional form fields that don't map to the static fields above. This is useful for custom form fields specific to your use case.

**Example metadata use cases:**
- Custom questions: "How did you hear about us?", "What services are you interested in?"
- Business-specific fields: "Property size", "How many parking spaces?", "Preferred contact method"
- Marketing data: "Campaign ID", "UTM parameters", "Promo code"

**Metadata requirements:**
- Keys must be strings with a maximum length of 100 characters
- Values can be any JSON-serializable type (string, number, boolean, array, object)
- The entire metadata object is stored as-is and can be retrieved later

---

## Response

### Success Response

**HTTP Status Code:** `200 OK`

```json
{
  "success": true,
  "message": "Lead submitted successfully",
  "leadUuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "timestamp": "2025-12-04T10:30:00.000Z"
}
```

### Error Responses

#### Validation Error

**HTTP Status Code:** `400 Bad Request`

```json
{
  "success": false,
  "message": "Validation failed",
  "errors": [
    "Email is required",
    "Invalid location UUID format"
  ],
  "timestamp": "2025-12-04T10:30:00.000Z"
}
```

#### Authentication Error

**HTTP Status Code:** `403 Forbidden`

```
Invalid or expired API Key.
```

#### Rate Limit Exceeded

**HTTP Status Code:** `429 Too Many Requests`

The API enforces rate limiting of 5 requests per 60 seconds per IP address to prevent abuse.

---

## Examples

### Example 1: Basic Lead Submission

**Request:**

```bash
curl -X POST https://api.yourdomain.com/lead/Submit \
  -H "Content-Type: application/json" \
  -H "X-API-KEY: sk_live_abc123xyz789" \
  -d '{
    "email": "john.doe@example.com",
    "firstName": "John",
    "lastName": "Doe",
    "phone": "+1-555-123-4567",
    "locationUuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "source": "Website",
    "formName": "Contact Us"
  }'
```

**Response:**

```json
{
  "success": true,
  "message": "Lead submitted successfully",
  "leadUuid": "f9e8d7c6-b5a4-3210-9876-543210fedcba",
  "timestamp": "2025-12-04T10:30:00.000Z"
}
```

### Example 2: Lead with Custom Metadata

**Request:**

```bash
curl -X POST https://api.yourdomain.com/lead/Submit \
  -H "Content-Type: application/json" \
  -H "X-API-KEY: sk_live_abc123xyz789" \
  -d '{
    "email": "jane.smith@example.com",
    "firstName": "Jane",
    "lastName": "Smith",
    "phone": "+1-555-987-6543",
    "locationUuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "source": "Website",
    "sourceDetails": "Google Ads - Spring Campaign",
    "formName": "Get a Quote",
    "notes": "Interested in weekly service",
    "estimatedStartDate": "2025-01-15T00:00:00Z",
    "metadata": {
      "propertySize": "5000 sq ft",
      "serviceType": "Commercial Cleaning",
      "preferredContactMethod": "Email",
      "numberOfEmployees": 25,
      "hearAboutUs": "Google Search",
      "utmSource": "google",
      "utmMedium": "cpc",
      "utmCampaign": "spring2025"
    }
  }'
```

**Response:**

```json
{
  "success": true,
  "message": "Lead submitted successfully",
  "leadUuid": "1a2b3c4d-5e6f-7890-abcd-ef1234567890",
  "timestamp": "2025-12-04T10:30:00.000Z"
}
```

### Example 3: JavaScript/TypeScript

```javascript
const submitLead = async (leadData) => {
  try {
    const response = await fetch('https://api.yourdomain.com/lead/Submit', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-API-KEY': process.env.DASH_API_KEY // Store securely in environment variables
      },
      body: JSON.stringify({
        email: leadData.email,
        firstName: leadData.firstName,
        lastName: leadData.lastName,
        phone: leadData.phone,
        locationUuid: 'a1b2c3d4-e5f6-7890-abcd-ef1234567890', // Your location UUID
        source: 'Website',
        formName: 'Contact Form',
        notes: leadData.message,
        metadata: {
          customField1: leadData.customValue1,
          customField2: leadData.customValue2
        }
      })
    });

    const result = await response.json();

    if (result.success) {
      console.log('Lead submitted successfully:', result.leadUuid);
      return result;
    } else {
      console.error('Failed to submit lead:', result.errors);
      throw new Error(result.message);
    }
  } catch (error) {
    console.error('Error submitting lead:', error);
    throw error;
  }
};
```

### Example 4: Python

```python
import requests
import os
from datetime import datetime

def submit_lead(email, first_name=None, last_name=None, phone=None, notes=None, metadata=None):
    url = "https://api.yourdomain.com/lead/Submit"

    headers = {
        "Content-Type": "application/json",
        "X-API-KEY": os.environ.get("DASH_API_KEY")  # Store securely
    }

    payload = {
        "email": email,
        "firstName": first_name,
        "lastName": last_name,
        "phone": phone,
        "locationUuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",  # Your location UUID
        "source": "Website",
        "formName": "Contact Form",
        "notes": notes,
        "metadata": metadata or {}
    }

    # Remove None values
    payload = {k: v for k, v in payload.items() if v is not None}

    try:
        response = requests.post(url, json=payload, headers=headers)
        response.raise_for_status()

        result = response.json()
        print(f"Lead submitted successfully: {result['leadUuid']}")
        return result

    except requests.exceptions.HTTPError as e:
        print(f"HTTP Error: {e}")
        print(f"Response: {response.text}")
        raise
    except Exception as e:
        print(f"Error: {e}")
        raise

# Usage
submit_lead(
    email="customer@example.com",
    first_name="John",
    last_name="Doe",
    phone="+1-555-123-4567",
    notes="Interested in service quote",
    metadata={
        "propertyType": "Commercial",
        "squareFootage": 10000
    }
)
```

### Example 5: PHP

```php
<?php

function submitLead($email, $firstName = null, $lastName = null, $phone = null, $notes = null, $metadata = []) {
    $url = "https://api.yourdomain.com/lead/Submit";

    $data = [
        'email' => $email,
        'firstName' => $firstName,
        'lastName' => $lastName,
        'phone' => $phone,
        'locationUuid' => 'a1b2c3d4-e5f6-7890-abcd-ef1234567890', // Your location UUID
        'source' => 'Website',
        'formName' => 'Contact Form',
        'notes' => $notes,
        'metadata' => $metadata
    ];

    // Remove null values
    $data = array_filter($data, function($value) {
        return $value !== null;
    });

    $headers = [
        'Content-Type: application/json',
        'X-API-KEY: ' . getenv('DASH_API_KEY') // Store securely
    ];

    $ch = curl_init($url);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_POST, true);
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
    curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);

    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);

    if ($httpCode >= 200 && $httpCode < 300) {
        $result = json_decode($response, true);
        echo "Lead submitted successfully: " . $result['leadUuid'] . "\n";
        return $result;
    } else {
        echo "Error submitting lead: " . $response . "\n";
        throw new Exception("Failed to submit lead");
    }
}

// Usage
submitLead(
    'customer@example.com',
    'John',
    'Doe',
    '+1-555-123-4567',
    'Interested in service quote',
    [
        'propertyType' => 'Commercial',
        'squareFootage' => 10000
    ]
);
?>
```

---

## Rate Limiting

To ensure fair usage and prevent abuse, the API enforces the following rate limits:

- **5 requests per 60 seconds** per IP address
- Rate limiting applies to both successful and failed requests
- If rate limit is exceeded, the API returns HTTP status code `429 Too Many Requests`

**Best Practices:**
- Implement exponential backoff when rate limited
- Cache location UUIDs rather than making repeated lookups
- Batch process leads if possible
- Contact support if you need higher rate limits for your use case

---

## CAPTCHA Support (Optional)

If CAPTCHA verification is enabled on your account, you must include a CAPTCHA token with your request. The API supports:

- **Google reCAPTCHA** (v2 and v3)
- **hCaptcha**
- **Cloudflare Turnstile**

To include CAPTCHA verification:

```json
{
  "email": "john.doe@example.com",
  "locationUuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "captchaToken": "03AGdBq24PBCbwiDRXx...",
  "captchaType": "recaptcha"
}
```

Contact your Dash Marketing representative to determine if CAPTCHA is required for your integration.

---

## Finding Your Location UUID

Your location UUID is a unique identifier for each of your business locations. You can obtain this by:

1. **From Dash Marketing:** Your representative can provide a list of all location UUIDs
2. **From Dash Core:** Navigate to **Locations** in Dash Core, select a location, and find the UUID in the location details

**Important:** Each location has its own UUID. Leads must be submitted to the correct location UUID based on where the inquiry originated.

---

## Troubleshooting

### Common Errors

#### "Invalid or expired API Key"
- Verify your API key is correct and hasn't been revoked
- Check that the `X-API-KEY` header is properly formatted
- Ensure your API key hasn't expired (if expiration was set)

#### "Location not found"
- Verify the location UUID is correct
- Ensure the location UUID is associated with your organization
- Check that the location UUID is properly formatted (36 characters with hyphens)

#### "Email format is invalid"
- Ensure the email address follows standard email format (user@domain.com)
- Check for leading/trailing spaces

#### "Validation failed"
- Review the `errors` array in the response for specific validation issues
- Ensure all required fields (`email` and `locationUuid`) are present
- Check that field lengths don't exceed maximum allowed values

#### Rate Limit Exceeded
- Wait 60 seconds before retrying
- Implement exponential backoff in your code
- Contact support if you consistently hit rate limits

---

## Support

For technical support, questions, or to request higher rate limits:

- **Email:** support@dashmarketing.io
- **Documentation:** https://docs.dashmarketing.io

---

## Changelog

### Version 1.0 (December 2025)
- Initial release
- Support for basic lead submission
- API key authentication
- Rate limiting
- CAPTCHA support
- Custom metadata fields
