# Notification Service

A Go-based microservice for sending notifications across multiple channels (email, SMS, push notifications) with template support and delivery tracking.

## Overview

The Notification Service is a backend service that:
- Manages notification templates
- Sends notifications through multiple channels
- Tracks delivery status
- Handles retries for failed deliveries
- Provides a clean API for notification requests

## Features

- **Multi-Channel Support**: Email, SMS, and push notifications
- **Template Management**: Reusable notification templates with variable substitution
- **Delivery Tracking**: Monitor notification delivery status
- **Retry Logic**: Automatic retries for failed deliveries
- **Queue-Based Processing**: Asynchronous notification processing
- **Rate Limiting**: Prevent notification spam

## Project Structure

```
Notification Service/
├── main.go           # Application entry point
├── go.mod           # Module definition
└── README.md        # This file
```

## Getting Started

### Prerequisites

- Go 1.19 or higher

### Installation

```bash
git clone https://github.com/ankit-songara/notification-service.git
cd "notification-service"
```

### Running

```bash
go run main.go
```

## API Endpoints

### POST /send

Send a notification to a user.

**Request Body:**
```json
{
  "user_id": "user_123",
  "template_id": "welcome_email",
  "channels": ["email", "push"],
  "variables": {
    "name": "John",
    "activation_link": "https://example.com/activate"
  }
}
```

**Response:**
```json
{
  "notification_id": "notif_abc123",
  "status": "QUEUED",
  "channels": [
    {
      "channel": "email",
      "status": "PENDING"
    },
    {
      "channel": "push",
      "status": "PENDING"
    }
  ]
}
```

### GET /notifications/{id}

Get notification status and delivery information.

**Response:**
```json
{
  "notification_id": "notif_abc123",
  "user_id": "user_123",
  "status": "DELIVERED",
  "created_at": "2024-01-15T10:30:00Z",
  "channels": [
    {
      "channel": "email",
      "status": "DELIVERED",
      "delivered_at": "2024-01-15T10:31:15Z"
    },
    {
      "channel": "push",
      "status": "FAILED",
      "error": "Device token invalid"
    }
  ]
}
```

## Configuration

Set environment variables:

```bash
# Email configuration
EMAIL_PROVIDER=smtp
EMAIL_SMTP_HOST=smtp.gmail.com
EMAIL_SMTP_PORT=587
EMAIL_USERNAME=your-email@gmail.com
EMAIL_PASSWORD=your-password

# SMS configuration
SMS_PROVIDER=twilio
SMS_API_KEY=your-twilio-key
SMS_API_SECRET=your-twilio-secret

# Push notification configuration
PUSH_PROVIDER=firebase
PUSH_API_KEY=your-firebase-key

# Service configuration
PORT=8080
MAX_RETRIES=3
RETRY_DELAY_MS=1000
```

## Design Patterns

### 1. **Strategy Pattern**
Different notification channels implement a common interface:
```go
type NotificationChannel interface {
    Send(ctx context.Context, req *SendRequest) error
}
```

### 2. **Factory Pattern**
Create channel instances based on configuration:
```go
func NewChannel(provider string) NotificationChannel {
    // Return appropriate channel implementation
}
```

### 3. **Builder Pattern**
Fluent API for building notification requests:
```go
notification := NewNotification().
    WithUserID("user_123").
    WithTemplate("welcome_email").
    AddChannel("email").
    AddChannel("push").
    WithVariable("name", "John").
    Build()
```

### 4. **Observer Pattern**
Notify listeners when notification status changes:
```go
notification.OnStatusChange(func(status string) {
    // Log status change, update UI, etc
})
```

## Notification Types

### Email Notifications
- Transactional emails (password reset, verification)
- Marketing emails (promotions, newsletters)
- Digest emails (weekly reports)

### SMS Notifications
- OTP (One-Time Password)
- Account alerts
- Delivery confirmations

### Push Notifications
- App notifications
- Reminders
- Real-time alerts

## Template Variables

Use template variables for dynamic content:

```
Template: "welcome_email"
Content: "Hello {{name}}, welcome to {{app_name}}!"

Request:
{
  "variables": {
    "name": "John",
    "app_name": "MyApp"
  }
}

Result: "Hello John, welcome to MyApp!"
```

## Error Handling

The service uses typed errors:

| Error | Status | Description |
|-------|--------|-------------|
| `INVALID_REQUEST` | 400 | Missing required fields |
| `TEMPLATE_NOT_FOUND` | 404 | Template doesn't exist |
| `USER_NOT_FOUND` | 404 | User not found |
| `CHANNEL_ERROR` | 503 | Notification channel unavailable |
| `DELIVERY_FAILED` | 500 | Failed to deliver notification |

## Testing

### Unit Tests

```go
func TestEmailChannel_Send(t *testing.T) {
    channel := NewEmailChannel(&EmailConfig{})
    req := &SendRequest{
        To: "test@example.com",
        Subject: "Test",
        Body: "Test body",
    }
    
    err := channel.Send(context.Background(), req)
    if err != nil {
        t.Fatalf("expected no error, got %v", err)
    }
}
```

### Integration Tests

```go
func TestNotificationService_SendMultiChannel(t *testing.T) {
    svc := setupTestService(t)
    
    req := &NotificationRequest{
        UserID: "user_123",
        TemplateID: "welcome",
        Channels: []string{"email", "sms", "push"},
    }
    
    resp, err := svc.Send(context.Background(), req)
    if err != nil {
        t.Fatalf("send failed: %v", err)
    }
    
    if resp.Status != "QUEUED" {
        t.Errorf("expected QUEUED, got %s", resp.Status)
    }
}
```

## Performance Considerations

- **Async Processing**: Notifications are queued and processed asynchronously
- **Connection Pooling**: Reuse connections to email/SMS providers
- **Batch Operations**: Group notifications to same provider for efficiency
- **Caching**: Cache templates and user preferences

## Monitoring and Observability

### Metrics to Track

```
notification_requests_total{channel="email", status="delivered"}
notification_delivery_latency_seconds{channel="sms"}
notification_failures_total{channel="push", error="invalid_token"}
queue_depth{channel="email"}
```

### Logging

All notifications are logged with:
- Request ID for tracing
- Channel information
- Delivery status
- Error details (if any)

## Retry Strategy

Failed notifications are retried with exponential backoff:

| Attempt | Delay | Total Delay |
|---------|-------|------------|
| 1st | 1s | 1s |
| 2nd | 2s | 3s |
| 3rd | 4s | 7s |
| 4th | 8s | 15s |

After max retries, notification is marked as permanently failed and requires manual intervention.

## Security

### Best Practices

1. **Authentication**: All endpoints require valid API key
2. **Encryption**: Sensitive data encrypted in transit and at rest
3. **Rate Limiting**: Prevent notification spam
4. **Audit Logging**: Track all notification sends
5. **Webhook Signature Verification**: Verify callback authenticity

### Data Privacy

- PII data is encrypted
- Logs are redacted (no phone numbers, email addresses in logs)
- Notifications are soft-deleted (not hard-deleted)
- Retention policy: 90 days by default

## Deployment

### Docker

```dockerfile
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o notification-service main.go

FROM alpine:3.18
RUN apk add --no-cache ca-certificates
COPY --from=builder /app/notification-service /usr/local/bin/
EXPOSE 8080
CMD ["notification-service"]
```

### Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notification-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: notification-service
  template:
    metadata:
      labels:
        app: notification-service
    spec:
      containers:
      - name: notification-service
        image: notification-service:latest
        ports:
        - containerPort: 8080
        env:
        - name: EMAIL_PROVIDER
          value: "smtp"
```

## Future Enhancements

- [ ] Support for more channels (Slack, Teams, Discord)
- [ ] Batch notification API
- [ ] Scheduling API (send at specific time)
- [ ] A/B testing for templates
- [ ] Preference management (do-not-disturb hours)
- [ ] Analytics dashboard
- [ ] Template versioning
- [ ] Notification whitelist/blacklist

## Troubleshooting

### Notifications Not Being Sent

1. Check service is running: `curl http://localhost:8080/health`
2. Verify configuration: Check environment variables
3. Check logs: Look for error messages
4. Verify provider credentials: Test with provider directly

### High Latency

1. Check queue depth: Monitor queue size
2. Increase worker threads: Scale horizontally
3. Optimize template rendering: Profile code
4. Check provider limits: May be rate-limited

### Provider Errors

Refer to specific provider documentation:
- [Twilio SMS Docs](https://www.twilio.com/docs)
- [SendGrid Email Docs](https://docs.sendgrid.com)
- [Firebase Push Docs](https://firebase.google.com/docs)

## License

MIT License

## Author

Ankit Songara

---

**Note:** This is a template for a notification service. Implement the actual business logic based on your specific requirements.
