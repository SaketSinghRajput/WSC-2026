# REST API vs HTTP API - Choosing the Right Type

## REST API Overview

REST API is the full-featured API type in AWS API Gateway, providing comprehensive functionality for complex API requirements. It supports extensive customization, request validation, caching, and integration with AWS Web Application Firewall (WAF).

**Key Characteristics**:
- **Fully Featured**: Request validation, request/response mapping, caching, authorization options
- **Customizable**: Extensive configuration options for advanced use cases
- **Established Standard**: REST principles widely understood by developers
- **Production Ready**: Enterprise-grade features for mission-critical APIs

**When REST API is Preferred**:
- Complex request/response transformation requirements
- Need for built-in request schema validation
- Response caching with configurable TTL
- Multiple authorization mechanisms (IAM, Cognito, Lambda authorizers, API keys)
- WAF integration for security
- Usage plans and API key-based access control
- Detailed API documentation and models

## HTTP API Overview

HTTP API is a lightweight, high-performance API type designed for simple, cost-effective APIs. It's a newer offering that addresses the needs of modern serverless architectures with lower latency and reduced complexity.

**Key Characteristics**:
- **Lightweight**: Simplified configuration, fewer options to manage
- **Lower Latency**: 15-20ms typical vs 50-100ms for REST API
- **Lower Cost**: $1 per million requests vs $3.50 for REST API
- **Modern**: Built for serverless, microservices, and modern architectures
- **Simplified Authorization**: Focus on JWT and Lambda authorizers

**When HTTP API is Preferred**:
- Simple CRUD operations with minimal customization
- Cost optimization is priority (70% cheaper than REST API)
- Latency-sensitive applications
- Modern JWT-based authorization
- Building microservices with minimal overhead
- Rapid prototyping and MVP development

## Detailed Comparison Table

| Feature | REST API | HTTP API |
|---------|----------|----------|
| **Pricing** | $3.50 per million requests | $1.00 per million requests |
| **Latency** | 50-100ms typical | 15-20ms typical |
| **Monthly Free Tier** | 1 million requests | 1 million requests |
| **Setup Complexity** | Moderate (resources, methods, stages) | Simple (routes, integrations) |
| **Authorization Types** | IAM, API Keys, Cognito, Lambda authorizers | IAM, JWT, Lambda authorizers |
| **Request Validation** | Built-in schema validation | Not available |
| **Response Caching** | Yes, configurable per method | Limited |
| **Cache TTL Control** | Yes, 0 seconds to 1 hour | No |
| **API Keys** | Yes, built-in support | No |
| **Usage Plans** | Yes, create usage plans for API keys | No |
| **WAF Integration** | Yes, attach AWS WAF to REST API | No |
| **Throttling** | Configurable per method | Global per API |
| **Request Mapping** | Yes, full control over transformation | Limited |
| **Response Mapping** | Yes, full control over transformation | Limited |
| **CORS Support** | Yes, built-in CORS configuration | Yes, built-in CORS |
| **Custom Authorizers** | Yes, Lambda authorizers supported | Yes, Lambda authorizers supported |
| **Payload Limits** | 10 MB | 10 MB |
| **API Gateway Metrics** | Detailed CloudWatch metrics | Basic CloudWatch metrics |
| **X-Ray Tracing** | Yes | Yes |
| **Custom Domains** | Yes | Yes |
| **Stage Variables** | Yes, for environment-specific settings | Limited support |
| **Deployment Model** | Manual deployment to stages | Automatic deployment |
| **AWS CLI Support** | Full apigateway commands | Limited apigatewayv2 commands |
| **Integration Types** | Proxy, custom, HTTP, AWS services, mock | Proxy, HTTP (primarily Lambda proxy) |

## Cost Comparison Analysis

### Scenario 1: Low-Traffic Development API
**100,000 API calls/month**

**REST API**:
- Calls: 100,000 (within Free Tier)
- Cost: $0.00/month

**HTTP API**:
- Calls: 100,000 (within Free Tier)
- Cost: $0.00/month

**Winner**: Tie (both free), but HTTP API has lower latency

### Scenario 2: Medium-Traffic Production API
**5,000,000 API calls/month (5M - 1M free = 4M paid)**

**REST API**:
- Paid requests: 4,000,000
- Cost: 4,000,000 × ($3.50 / 1,000,000) = $14.00/month

**HTTP API**:
- Paid requests: 4,000,000
- Cost: 4,000,000 × ($1.00 / 1,000,000) = $4.00/month

**Savings**: $10.00/month (71% cheaper with HTTP API)

### Scenario 3: High-Traffic SaaS Application
**100,000,000 API calls/month**

**REST API**:
- Paid requests: 99,000,000
- Cost: 99,000,000 × ($3.50 / 1,000,000) = $346.50/month

**HTTP API**:
- Paid requests: 99,000,000
- Cost: 99,000,000 × ($1.00 / 1,000,000) = $99.00/month

**Savings**: $247.50/month (71% cheaper with HTTP API)

**Key Insight**: HTTP API is consistently 70% cheaper per request than REST API at scale.

## Performance Comparison

### Latency Benchmarks

| Operation | REST API | HTTP API | Difference |
|-----------|----------|----------|-----------|
| Invoke Lambda | 50-70ms | 15-25ms | 40-50ms faster |
| Simple response | 5-10ms | <5ms | Near instant |
| Complex mapping | 30-50ms | N/A | REST adds overhead |
| Cold start included | 100-2000ms | 100-2000ms | Lambda dominates |

**Analysis**: While HTTP API is 2-3x faster, cold starts from Lambda (100-2000ms) dominate total latency. HTTP API advantage becomes significant only when calling warm Lambda functions.

## Feature Matrix - Detailed Analysis

### Authorization Support

**REST API Options**:
- AWS IAM (for AWS principal authentication)
- Amazon Cognito (for user pools and identity providers)
- Lambda authorizers (custom authorization logic)
- API keys (for simple access control)

**HTTP API Options**:
- AWS IAM (for AWS principal authentication)
- Native JWT authorization (built-in, no Lambda needed)
- Lambda authorizers (custom authorization logic)

**Comparison**: REST API supports more authorization types. HTTP API's native JWT is convenient but less flexible.

### Request Validation

**REST API**: Built-in JSON schema validation
```json
{
  "type": "object",
  "required": ["name", "email"],
  "properties": {
    "name": {"type": "string"},
    "email": {"type": "string", "format": "email"}
  }
}
```

**HTTP API**: No built-in validation; must validate in Lambda

**Impact**: REST API can reject invalid requests without invoking Lambda (saves cost), while HTTP API must invoke Lambda for every request (higher costs).

### Caching Capabilities

**REST API**:
- Cache responses at API Gateway level
- TTL: 0 seconds to 1 hour
- Cache invalidation: Manual or time-based
- Reduces backend invocations

**HTTP API**:
- Limited caching support
- Primarily relies on Lambda caching or external cache

**Impact**: REST API better for high-read, low-write workloads (news sites, catalogs).

## Decision Framework

### Choose REST API If:
1. Need request validation to reduce Lambda invocations
2. Require response caching for frequently-accessed data
3. Using multiple authorization mechanisms
4. Need WAF integration for security
5. Building complex enterprise APIs with many integrations
6. Cost is less important than feature richness
7. Team familiar with REST API concepts

**Example Scenario**: E-commerce product catalog API with high read volume, caching requirements, and request validation.

### Choose HTTP API If:
1. Cost optimization is priority (70% savings)
2. Building simple CRUD APIs
3. Latency-sensitive applications
4. Using JWT-based authorization
5. Building microservices with many small APIs
6. Rapid prototyping and MVP development
7. Want simplified configuration and management

**Example Scenario**: Internal microservices backend, cost-sensitive startup MVP, high-frequency API calls.

## WorldSkills Recommendation

**For Competition**: Use **HTTP API**

**Reasoning**:
1. **Cost Advantage**: Save 70% on API calls ($1 vs $3.50 per million)
2. **Speed**: HTTP API deploys faster with simpler configuration
3. **Sufficient Features**: Most competition scenarios need only CRUD operations and basic validation
4. **Simplicity**: Fewer configuration options = less to debug during competition
5. **Judging Criteria**: Judges typically focus on functionality, not which API type used

**Exception**: Use REST API only if competition scenario specifically requires:
- Request schema validation
- Response caching
- Multiple complex authorization mechanisms
- WAF integration

## Feature-by-Feature Breakdown

### CORS Configuration

**REST API**:
```
1. Select resource
2. Click "Actions" → "Enable CORS and replace existing CORS headers"
3. Configure allowed methods, headers, credentials
4. Deploy API
```

**HTTP API**:
```
1. Go to CORS configuration
2. Enter allowed origins, methods, headers
3. Deploy (automatic)
```

**Winner**: HTTP API (simpler configuration, automatic deployment)

### Lambda Proxy Integration

**REST API**:
```
1. Create resource and method
2. Choose integration type: "Lambda Proxy Integration"
3. Select Lambda function
4. Deploy API
```

**HTTP API**:
```
1. Create route
2. Choose integration: "Lambda"
3. Select Lambda function
4. Automatically deployed
```

**Winner**: HTTP API (no manual deployment step)

### Testing

**REST API**:
```
1. Open method
2. Click "Test" button
3. Enter test data
4. View response
```

**HTTP API**:
```
1. Use AWS CLI or curl (no built-in test console)
2. Or test from Lambda console directly
```

**Winner**: REST API (built-in testing interface)

## Migration Considerations

### Moving from REST to HTTP API

**Compatibility**:
- Code remains same (Lambda function unchanged)
- Response format must match HTTP API expectations
- Authorization may need adjustment

**Breaking Changes**:
- Lose request validation (move to Lambda)
- Lose response caching (implement separately)
- API Gateway metrics less detailed

**Advantages**:
- Immediate 70% cost reduction
- Improved latency
- Simplified configuration

### Moving from HTTP to REST API

**Advantages**:
- Add request validation (reduce Lambda calls)
- Enable response caching
- Access more authorization options

**Disadvantages**:
- Higher costs
- More complex configuration
- Slower deployment process

## Creating REST API (AWS Console)

1. **Navigate to API Gateway dashboard**
2. **Click "Create API"**
3. **Choose "REST API"**
4. **Configure API settings**:
   - API name: "UserManagementAPI"
   - Description: "API for managing users"
   - Endpoint type: "Regional" (or "Edge optimized" for global)
5. **Click "Create API"**
6. **Create resources and methods** (see get_method.md, post_method.md, delete_method.md)
7. **Deploy API** to stage (dev, prod)

## Creating HTTP API (AWS Console)

1. **Navigate to API Gateway dashboard**
2. **Click "Create API"**
3. **Choose "HTTP API"**
4. **Click "Build"**
5. **Configure integration**:
   - Integration target: "Lambda"
   - Lambda function: Select function
   - CORS: Configure if needed
6. **Click "Next"**
7. **Create routes** (automatically maps HTTP methods to paths)
8. **Review and create** (automatically deploys)

## AWS CLI Commands

### Create REST API

```bash
aws apigateway create-rest-api \
    --name UserManagementAPI \
    --description "REST API for user management"
```

### Create HTTP API

```bash
aws apigatewayv2 create-api \
    --name UserManagementAPI \
    --protocol-type HTTP \
    --description "HTTP API for user management"
```

### List REST APIs

```bash
aws apigateway get-rest-apis
```

### List HTTP APIs

```bash
aws apigatewayv2 get-apis
```

## Competition Decision Matrix

| Scenario | API Type | Reasoning |
|----------|----------|-----------|
| Simple CRUD API with Lambda | HTTP API | Cost savings, sufficient features |
| API with request validation required | REST API | Built-in validation saves Lambda calls |
| High-read, cached data (catalog) | REST API | Response caching improves performance |
| Microservices with many endpoints | HTTP API | Simple configuration × many APIs = big savings |
| Authentication service | REST API | More authorization options |
| Real-time API | HTTP API | Lower latency |
| Rapid prototyping | HTTP API | Faster setup |
| Enterprise production | REST API | More features and controls |

## Verification Checklist

Before choosing API type:
- [ ] Understand functional requirements (what operations needed)
- [ ] Estimate request volume per month
- [ ] Identify authorization requirements
- [ ] Check if caching would improve performance
- [ ] Consider validation complexity
- [ ] Calculate cost difference at expected volume
- [ ] Test both API types if time permits
- [ ] Confirm team familiarity with chosen type

## Next Steps

After deciding on API type:
- **If HTTP API**: Proceed to [get_method.md](get_method.md) for simple setup
- **If REST API**: Proceed to [get_method.md](get_method.md) for detailed configuration
- [python_lambda_integration.md](python_lambda_integration.md): Lambda code patterns for either API type
- [server_lab.md](server_lab.md): Complete lab using HTTP API for cost optimization
