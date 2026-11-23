# n8n Integration Guide for Google Maps Lead Extraction

## 1. Transform Node (Code Node)

Add this after your Webhook Trigger node:

```javascript
// Extract city from Google Maps URL
function extractCityFromURL(url) {
  try {
    // Pattern 1: /place/Location+Name/
    const placeMatch = url.match(/\/place\/([^/@]+)/);
    if (placeMatch) {
      const placeName = decodeURIComponent(placeMatch[1].replace(/\+/g, ' '));
      // Take first part before comma (usually city)
      return placeName.split(',')[0].trim();
    }
    
    // Pattern 2: ?q=keyword,City,State
    const queryMatch = url.match(/[?&]q=([^&]+)/);
    if (queryMatch) {
      const query = decodeURIComponent(queryMatch[1].replace(/\+/g, ' '));
      const parts = query.split(',');
      if (parts.length >= 2) {
        return parts[1].trim();
      }
    }
    
    // Pattern 3: /search/keyword+in+City/
    const searchMatch = url.match(/\/search\/[^\/]*\+in\+([^\/\+]+)/);
    if (searchMatch) {
      return decodeURIComponent(searchMatch[1].replace(/\+/g, ' '));
    }
    
    // Fallback: try to extract from coordinates (requires geocoding service)
    const coordMatch = url.match(/@([-\d.]+),([-\d.]+)/);
    if (coordMatch) {
      // You'll need to implement reverse geocoding here
      // For now, return a default
      return "Unknown City";
    }
    
    return "Unknown City";
  } catch (error) {
    console.error('Error extracting city:', error);
    return "Unknown City";
  }
}

// Transform the incoming webhook data
const input = $input.all()[0].json;

// Extract city from the Location URL
const city = extractCityFromURL(input["Location URL"]);

// Transform to expected format
const transformedData = {
  keyword: input["Keyword Search"],
  country: input["Country"],
  city: city,
  job_id: input["job_id"],
  company_segment: input["My company's segment"],
  location_url: input["Location URL"] // Keep original for reference
};

return [{
  json: transformedData
}];
```

## 2. Callback Node (HTTP Request)

Add this after your data processing is complete:

### Success Callback
```javascript
// HTTP Request Node settings:
// Method: POST
// URL: https://ycrstudbmkbxzgtbnclz.supabase.co/functions/v1/webhook-receiver
// Headers:
//   Content-Type: application/json
//   Authorization: Bearer YOUR_SUPABASE_ANON_KEY

// Body (JSON):
{
  "job_id": "{{ $('Transform Node').item.json.job_id }}",
  "status": "completed",
  "leads_data": "{{ $json.leads_data }}", // Your processed leads array
  "total_leads": "{{ $json.leads_data.length }}"
}
```

### Error Callback
```javascript
// For error handling, add another HTTP Request node:
{
  "job_id": "{{ $('Transform Node').item.json.job_id }}",
  "status": "failed",
  "error_message": "{{ $json.error_message }}",
  "leads_data": []
}
```

## 3. Environment Variables

Add to your `.env` file:

```bash
# n8n Webhook URL (replace with your actual n8n webhook URL)
VITE_N8N_WEBHOOK_URL=https://your-n8n-instance.com/webhook/maps-extraction

# Supabase (already configured)
VITE_SUPABASE_URL=https://ycrstudbmkbxzgtbnclz.supabase.co
VITE_SUPABASE_ANON_KEY=your_anon_key
```

## 4. Expected Data Flow

```
1. User Form Submission:
{
  "Location URL": "https://www.google.com/maps/place/Starbucks/@40.7589,-73.9851,17z",
  "Keyword Search": "coffee shop",
  "Country": "US", 
  "My company's segment": "Food service marketing",
  "job_id": "uuid-123-456"
}

2. After Transform Node:
{
  "keyword": "coffee shop",
  "country": "US",
  "city": "Starbucks", // Extracted from URL
  "job_id": "uuid-123-456",
  "company_segment": "Food service marketing",
  "location_url": "https://www.google.com/maps/place/Starbucks/@40.7589,-73.9851,17z"
}

3. Callback to Supabase:
{
  "job_id": "uuid-123-456",
  "status": "completed",
  "leads_data": [
    {
      "name": "Local Coffee Co",
      "address": "123 Main St",
      "phone": "+1234567890",
      "website": "localcoffee.com",
      "rating": 4.5,
      "reviews_count": 150,
      "sales_helper": {
        "opening_message": "Hi there! I noticed your great reviews...",
        "talking_points": ["Strong local presence", "Great customer satisfaction"]
      }
    }
  ],
  "total_leads": 1
}
```

## 5. Testing

Test the transformation with this curl command:

```bash
curl -X POST YOUR_N8N_WEBHOOK_URL \
  -H "Content-Type: application/json" \
  -d '{
    "Location URL": "https://www.google.com/maps/place/Coffee+Shops/@40.7589,-73.9851,17z",
    "Keyword Search": "coffee shop",
    "Country": "US",
    "My company's segment": "Food service",
    "job_id": "test-123"
  }'
```

## 6. Debugging

- Check n8n execution logs for transformation errors
- Verify webhook-receiver logs in Supabase Functions
- Test city extraction with different URL formats
- Monitor job status updates in `lead_extraction_jobs` table

## 7. Common URL Patterns to Test

```javascript
// Test these URL patterns:
const testUrls = [
  "https://www.google.com/maps/place/New+York,+NY/@40.7128,-74.0060,11z",
  "https://www.google.com/maps/search/coffee+shops/@40.7589,-73.9851,15z",
  "https://www.google.com/maps/@40.7589,-73.9851,17z/data=!3m1!4b1!4m2!3m1!1s0x0:0x0",
  "https://maps.google.com/?q=dentist,Chicago,IL",
  "https://www.google.com/maps/place/Starbucks/@40.7589,-73.9851,17z"
];
```

Each should extract a meaningful city name for the lead extraction process.