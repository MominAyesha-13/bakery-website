# Bakery Website - SKILL.md

This skill file documents the bakery website architecture, tech stack, and domain-specific patterns for AI-assisted development.

## Project Overview

A modern bakery e-commerce platform with real-time order management, inventory tracking, and customer engagement features.

## Tech Stack

### Frontend
- **Framework**: Next.js 14+ (React)
- **Styling**: Tailwind CSS
- **State Management**: TanStack Query (React Query) + Zustand
- **UI Components**: shadcn/ui
- **Forms**: React Hook Form + Zod validation
- **Build**: Webpack (via Next.js)
- **Package Manager**: npm or pnpm

**Frontend Structure**:
```
frontend/
├── src/
│   ├── app/              # Next.js app router
│   ├── components/       # Reusable UI components
│   ├── pages/           # API routes for Next.js
│   ├── hooks/           # Custom React hooks
│   ├── lib/             # Utilities (API client, helpers)
│   ├── services/        # API integration layer
│   ├── store/           # Zustand stores
│   └── types/           # TypeScript interfaces
├── public/              # Static assets
└── next.config.js
```

### Backend
- **Language**: Java 17+
- **Framework**: Spring Boot 3.x
- **Build Tool**: Maven
- **Database**: PostgreSQL (primary), Redis (caching/sessions)
- **API**: RESTful with OpenAPI/Swagger docs
- **Authentication**: JWT tokens + refresh token rotation
- **Async Processing**: Spring async + message queues (RabbitMQ)
- **Testing**: JUnit 5, Mockito, TestContainers

**Backend Structure**:
```
backend/
├── src/
│   ├── main/java/com/bakery/
│   │   ├── api/              # REST controllers
│   │   ├── domain/           # Business logic (services, repositories)
│   │   ├── dto/              # Data Transfer Objects
│   │   ├── entity/           # JPA entities
│   │   ├── exception/        # Custom exceptions
│   │   ├── config/           # Spring configuration
│   │   ├── security/         # JWT, auth filters
│   │   ├── util/             # Utilities
│   │   └── BakeryApplication.java
│   └── test/java/
├── resources/
│   ├── application.properties
│   ├── application-dev.properties
│   ├── application-prod.properties
│   └── db/migration/         # Flyway migrations
└── pom.xml
```

## Core Domains

### 1. **Menu Management**
- **Entities**: MenuItem, Category, Ingredients, NutritionInfo
- **Features**: Product catalog, pricing, availability scheduling, dietary info
- **Endpoints**: `GET /api/menu`, `POST /api/admin/menu-items`, etc.

### 2. **Orders**
- **Entities**: Order, OrderItem, OrderStatus, DeliveryAddress
- **Statuses**: PENDING → CONFIRMED → PREPARING → READY → DELIVERED/PICKUP
- **Features**: Order tracking, real-time notifications, custom orders, scheduling
- **Endpoints**: `POST /api/orders`, `GET /api/orders/{id}`, `PUT /api/orders/{id}/status`

### 3. **Inventory Management**
- **Entities**: Ingredient, Stock, StockAdjustment, Supplier
- **Features**: Low-stock alerts, automated reordering, expiry tracking
- **Background Job**: Daily inventory sync, expiry notifications

### 4. **Customer Management**
- **Entities**: Customer, Address, Preferences, Loyalty
- **Features**: Registration, profiles, order history, wishlist, reviews
- **Endpoints**: `POST /api/auth/register`, `GET /api/customers/me`, etc.

### 5. **Payment Processing**
- **Integrations**: Stripe (credit cards), Razorpay (Indian payments), PayPal
- **Flow**: Order → Payment Intent → Webhook Confirmation
- **Security**: PCI compliance via tokenization, no card data stored

### 6. **Notifications**
- **Channels**: Email, SMS, Push notifications, In-app
- **Triggers**: Order confirmation, status updates, promotions, reminders
- **Service**: Async queue-based (RabbitMQ + Spring async)

## Development Patterns

### Java Backend Conventions
- **Layering**: Controller → Service → Repository → Entity
- **Dependency Injection**: Spring @Autowired or constructor injection (preferred)
- **Exceptions**: Custom `BakeryException` hierarchy with HTTP status mapping
- **DTOs**: Separate request/response DTOs, use MapStruct for entity ↔ DTO conversion
- **Validation**: JSR-380 (@Valid, @NotNull, custom @Validator)
- **Logging**: SLF4J with Logback, context IDs for request tracing
- **Testing**: Service layer tested with @SpringBootTest, mocked repositories

**Example Service Pattern**:
```java
@Service
public class OrderService {
  private final OrderRepository orderRepository;
  private final MenuItemRepository menuItemRepository;
  
  public OrderService(OrderRepository orderRepository, MenuItemRepository menuItemRepository) {
    this.orderRepository = orderRepository;
    this.menuItemRepository = menuItemRepository;
  }
  
  public OrderResponse createOrder(CreateOrderRequest request) {
    // 1. Validate items exist
    // 2. Calculate total
    // 3. Create order entity
    // 4. Save and publish event
    // 5. Return DTO
  }
}
```

### API Response Format
All endpoints return a consistent envelope:
```json
{
  "success": true,
  "data": { /* payload */ },
  "error": null,
  "timestamp": "2026-05-09T23:35:00Z"
}
```

Error responses:
```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "ORDER_NOT_FOUND",
    "message": "Order with ID 123 not found",
    "details": {}
  },
  "timestamp": "2026-05-09T23:35:00Z"
}
```

### Frontend-Backend Integration
- **API Client**: Axios + interceptors for auth token injection
- **Query Keys**: Structured for invalidation (`['orders', orderId]`, `['menu', 'items']`)
- **Error Handling**: Centralized error boundary, user-friendly error messages
- **Optimistic Updates**: For menu favorites, cart operations
- **Real-time Updates**: WebSocket connection for order status, live chat

### Database Migrations
- **Tool**: Flyway
- **Location**: `src/main/resources/db/migration/`
- **Naming**: `V1__Initial_schema.sql`, `V2__Add_loyalty_program.sql`
- **Constraints**: Foreign keys, indexes, check constraints enforced at DB level

## Key Features

### Customer Workflows
1. **Browse Menu** → Filter by category/dietary → Add to cart
2. **Customize Orders** → Select variants (size, toppings) → Schedule pickup/delivery
3. **Track Order** → Real-time status + notifications → Rate/review
4. **Loyalty** → Earn points on purchases → Redeem for discounts

### Admin Workflows
1. **Menu Management** → Create items, set pricing, manage availability
2. **Order Dashboard** → View queue, update statuses, manage cancellations
3. **Inventory** → Track stock, set reorder points, manage suppliers
4. **Reports** → Sales analytics, popular items, revenue trends

## Security Considerations

- **Authentication**: JWT (access token 15 min, refresh token 7 days)
- **Authorization**: Role-based (CUSTOMER, ADMIN, MANAGER)
- **HTTPS**: Enforced in production
- **CORS**: Configured for frontend origin only
- **CSRF**: Token-based for form submissions (Spring Security)
- **Input Validation**: Server-side validation (never trust client)
- **Rate Limiting**: API endpoint rate limiting (10 requests/sec per IP)
- **Secrets Management**: Environment variables, no secrets in git

## Testing Strategy

### Backend (Java)
- **Unit Tests**: Service logic, utilities (80%+ coverage)
- **Integration Tests**: Repository + database (TestContainers + PostgreSQL)
- **E2E Tests**: API endpoints via MockMvc or REST Assured

### Frontend (Next.js)
- **Unit Tests**: Components, hooks (Jest + React Testing Library)
- **E2E Tests**: User flows (Playwright or Cypress)

## Deployment

- **Frontend**: Vercel (auto-deploy from main branch)
- **Backend**: Docker + Kubernetes or traditional Java app server (Tomcat)
- **Database**: Managed PostgreSQL (AWS RDS, Heroku, etc.)
- **Cache**: Redis instance
- **CI/CD**: GitHub Actions (lint, test, build, deploy)

## API Documentation

Swagger UI available at `/swagger-ui.html` (development) or `/api-docs` (production).
Generate OpenAPI specs via `mvn clean install` (spring-openapi integration).

## Common Commands

```bash
# Frontend
npm run dev          # Start Next.js dev server
npm run build        # Production build
npm run test         # Run Jest tests
npm run lint         # ESLint

# Backend
mvn spring-boot:run  # Start Spring Boot dev server
mvn clean package    # Build JAR
mvn test             # Run JUnit tests
mvn spotless:check   # Check code formatting (Spotless)
mvn spotless:apply   # Auto-format code
```

## Extending This Skill

When adding new features:
1. **Define entities** in the domain layer (JPA)
2. **Create DTOs** for API contracts
3. **Implement service logic** with validation
4. **Expose REST endpoints** in controllers
5. **Add frontend pages/components** and integrate with API
6. **Write tests** for new logic
7. **Document** API changes in Swagger/OpenAPI

---

**Last Updated**: 2026-05-09  
**Version**: 1.0
