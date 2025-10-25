# MVP Specification: Escrow Platform for Service Businesses

## Project Overview

A simplified MVP of the secure escrow platform combining payment protection, scheduling, and automated bookkeeping for service-based businesses. Built with Next.js, Prisma, and SQLite with mock payment functionality to validate core workflows.

---

## Core MVP Features

### 1. Escrow System (Simplified)

#### Features
- **Mock Wallet System**: Simulate payment processing without real money transactions
- **Payment Hold on Booking**: Funds automatically held in escrow when customer books a service
- **Manual Payment Release**: Service providers can release payments after service completion
- **Manual Refund**: Admins and service providers can issue refundws when needed
- **Transaction Status Tracking**: 
  - `PENDING` - Payment initiated but not yet held
  - `HELD` - Funds in escrow, service not completed
  - `RELEASED` - Payment released to service provider
  - `REFUNDED` - Payment returned to customer

#### Implementation Details
- Each user has a mock wallet balance (starting with test funds)
- Transactions create audit trail entries
- Balance updates are atomic (within database transactions)
- Simple ledger system tracking debits/credits

---

### 2. Scheduling System

#### Features
- **Service Offerings Management**
  - Service providers create services with:
    - Name and description
    - Price
    - Duration (in minutes)
    - Category/type
  - Edit and archive services

- **Availability Calendar**
  - Service providers set available time slots
  - Day-based availability with start/end times
  - Simple blocking/availability toggle
  - Prevent double-booking automatically

- **Booking Management**
  - Real-time availability checking
  - Booking status workflow:
    - `PENDING` - Customer requested, awaiting confirmation
    - `CONFIRMED` - Provider accepted
    - `IN_PROGRESS` - Service is happening
    - `COMPLETED` - Service finished
    - `CANCELLED` - Booking cancelled by either party
  
- **Views**
  - Service provider: Calendar view of bookings, upcoming appointments
  - Customer: Browse services, view booking history
  - Both: Booking details page

#### Implementation Details
- Time slots stored in 30-minute increments
- Availability check queries database for conflicting bookings
- Timezone handling using UTC, display in local time
- Simple date picker for booking selection

---

### 3. Basic Bookkeeping

#### Features
- **Automatic Transaction Recording**
  - Every booking creates a transaction record
  - Payment holds logged automatically
  - Releases and refunds tracked with timestamps
  - Categories assigned based on service type

- **Financial Dashboard**
  - **Overview Cards**:
    - Total Earnings (sum of released payments)
    - Pending Amount (currently in escrow)
    - Total Transactions (count)
    - Available Balance (wallet balance)
  
  - **Transaction History**:
    - Chronological list of all transactions
    - Filter by status, date range, service type
    - Search by customer name or transaction ID
  
  - **Revenue Analytics** (Basic):
    - Revenue by service type (pie/bar chart)
    - Daily/weekly/monthly earnings view
    - Date range selector
    - Export to CSV

- **Receipt Generation**
  - Simple receipt view for each transaction
  - Customer and provider details
  - Service information
  - Payment breakdown
  - Transaction ID and timestamps

#### Implementation Details
- Transaction records immutable (append-only)
- Dashboard queries aggregate from transactions table
- Charts use simple counting/summing queries
- CSV export generates from filtered transaction queries

---

## User Roles & Permissions

### Customer Role

**Capabilities:**
- Register and login (email/password)
- Browse all service providers and their services
- View service details and pricing
- Book services with available providers
- Make mock payments from wallet
- View own booking history and status
- View transaction receipts
- Cancel bookings (with potential refund)
- Update profile information

**Dashboard Sections:**
- Browse Services
- My Bookings
- My Transactions
- Wallet/Balance
- Profile Settings

---

### Service Provider Role

**Capabilities:**
- Register and login (email/password)
- Create service provider profile (business name, description, category)
- Create and manage service offerings
- Set availability schedule (daily time slots)
- View incoming booking requests
- Confirm or decline bookings
- Mark services as in-progress/completed
- Release payments to own account after service completion
- View bookkeeping dashboard with financial analytics
- View transaction history filtered by own services
- Issue refunds (returns money to customer wallet)
- Update business profile

**Dashboard Sections:**
- Dashboard (overview stats)
- My Services (CRUD)
- My Schedule (calendar + availability)
- Bookings (incoming and active)
- Financials (bookkeeping dashboard)
- Transactions
- Profile Settings

---

### Admin Role

**Capabilities:**
- View all users in the system
- View all bookings across providers
- View all transactions system-wide
- Manually release payments (override)
- Manually refund payments
- Basic user management (view, disable accounts)
- System-wide statistics dashboard
- Resolve payment issues

**Dashboard Sections:**
- System Overview (stats)
- Users Management
- All Bookings
- All Transactions
- Payment Actions (release/refund interface)

---

## Technical Implementation

### Tech Stack

- **Frontend/Backend**: Next.js 14 (App Router)
- **Database ORM**: Prisma
- **Database**: SQLite (file-based for MVP)
- **Authentication**: NextAuth.js v5 (Auth.js)
- **Styling**: Tailwind CSS + shadcn/ui components
- **Forms**: React Hook Form + Zod validation
- **Charts**: Recharts or Chart.js
- **Date Handling**: date-fns or Day.js

### Project Structure

```
finmatt/
├── prisma/
│   ├── schema.prisma
│   └── seed.ts
├── src/
│   ├── app/
│   │   ├── (auth)/
│   │   │   ├── login/
│   │   │   └── register/
│   │   ├── (customer)/
│   │   │   ├── browse/
│   │   │   ├── bookings/
│   │   │   └── transactions/
│   │   ├── (provider)/
│   │   │   ├── dashboard/
│   │   │   ├── services/
│   │   │   ├── schedule/
│   │   │   ├── bookings/
│   │   │   └── financials/
│   │   ├── (admin)/
│   │   │   ├── dashboard/
│   │   │   ├── users/
│   │   │   └── transactions/
│   │   ├── api/
│   │   │   └── auth/
│   │   └── layout.tsx
│   ├── components/
│   │   ├── ui/ (shadcn components)
│   │   ├── forms/
│   │   ├── dashboard/
│   │   └── layout/
│   ├── lib/
│   │   ├── prisma.ts
│   │   ├── auth.ts
│   │   └── utils.ts
│   └── types/
├── public/
├── package.json
└── tsconfig.json
```

### Database Schema (Prisma)

```prisma
// Core Models

model User {
  id            String    @id @default(cuid())
  email         String    @unique
  name          String
  password      String    // Hashed
  role          UserRole  @default(CUSTOMER)
  walletBalance Float     @default(1000.0) // Mock wallet
  
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  
  // Relations
  serviceProvider ServiceProvider?
  bookingsAsCustomer Booking[] @relation("CustomerBookings")
  transactions  Transaction[]
}

enum UserRole {
  CUSTOMER
  SERVICE_PROVIDER
  ADMIN
}

model ServiceProvider {
  id              String   @id @default(cuid())
  userId          String   @unique
  user            User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  businessName    String
  description     String?
  category        String?
  profileImage    String?
  
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
  
  // Relations
  services        Service[]
  availability    Availability[]
  bookings        Booking[] @relation("ProviderBookings")
}

model Service {
  id              String   @id @default(cuid())
  providerId      String
  provider        ServiceProvider @relation(fields: [providerId], references: [id], onDelete: Cascade)
  
  name            String
  description     String
  price           Float
  duration        Int      // Duration in minutes
  category        String?
  isActive        Boolean  @default(true)
  
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
  
  // Relations
  bookings        Booking[]
}

model Availability {
  id              String   @id @default(cuid())
  providerId      String
  provider        ServiceProvider @relation(fields: [providerId], references: [id], onDelete: Cascade)
  
  dayOfWeek       Int      // 0-6 (Sunday to Saturday)
  startTime       String   // "09:00"
  endTime         String   // "17:00"
  isAvailable     Boolean  @default(true)
  
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
  
  @@unique([providerId, dayOfWeek])
}

model Booking {
  id              String        @id @default(cuid())
  customerId      String
  customer        User          @relation("CustomerBookings", fields: [customerId], references: [id])
  providerId      String
  provider        ServiceProvider @relation("ProviderBookings", fields: [providerId], references: [id])
  serviceId       String
  service         Service       @relation(fields: [serviceId], references: [id])
  
  scheduledDate   DateTime
  scheduledTime   String        // "14:00"
  status          BookingStatus @default(PENDING)
  totalAmount     Float
  
  notes           String?
  
  createdAt       DateTime      @default(now())
  updatedAt       DateTime      @updatedAt
  
  // Relations
  transaction     Transaction?
}

enum BookingStatus {
  PENDING
  CONFIRMED
  IN_PROGRESS
  COMPLETED
  CANCELLED
}

model Transaction {
  id              String            @id @default(cuid())
  bookingId       String            @unique
  booking         Booking           @relation(fields: [bookingId], references: [id])
  customerId      String
  customer        User              @relation(fields: [customerId], references: [id])
  
  amount          Float
  status          TransactionStatus @default(PENDING)
  category        String?           // Service category
  
  paymentHeldAt   DateTime?
  paymentReleasedAt DateTime?
  refundedAt      DateTime?
  
  notes           String?
  
  createdAt       DateTime          @default(now())
  updatedAt       DateTime          @updatedAt
}

enum TransactionStatus {
  PENDING
  HELD
  RELEASED
  REFUNDED
}
```

---

## Out of Scope for MVP

The following features are explicitly **NOT** included in the MVP and are deferred to future iterations:

### Payments
- Real payment gateway integration (Stripe, PayPal, M-Pesa)
- Bank account connections
- Actual money transfers
- Payment verification with financial institutions
- Multi-currency support

### Automation
- Automated payment release based on milestones
- Scheduled payment releases
- Automated booking reminders (email/SMS)
- Automated confirmations
- Auto-cancellation for unpaid bookings

### Dispute Resolution
- Complex dispute workflow
- Evidence submission system
- Mediation process
- Third-party arbitration
- Dispute escalation paths
- Insurance claims

### Advanced Bookkeeping
- Tax document generation (1099, invoices)
- Integration with QuickBooks/Xero
- Advanced financial forecasting
- Profit/loss statements
- Tax calculation and filing
- Expense tracking
- Receipt photo uploads
- Bank reconciliation

### Notifications
- Email notifications
- SMS notifications
- Push notifications
- In-app notification center
- Reminder system

### Advanced Features
- Mobile applications (iOS/Android)
- Calendar sync (Google Calendar, Outlook)
- Video conferencing integration
- Multi-signature transaction approval
- API for third-party integrations
- Webhooks
- Advanced analytics with ML
- Industry-specific templates
- Multi-location support for providers
- Team/staff management for providers
- Review and rating system
- Chat/messaging between users
- File attachments and document storage

### Scalability Features
- Multi-tenancy
- International expansion features
- Regional banking partnerships
- Automated compliance checks
- Advanced security features (2FA, biometrics)

---

## User Workflows

### Workflow 1: Service Provider Onboarding & Service Creation

**Steps:**
1. User registers with email/password, selects "Service Provider" role
2. System creates user account with mock wallet balance (e.g., $1000)
3. User completes service provider profile:
   - Business name
   - Description
   - Category (e.g., "Beauty & Wellness", "Consulting", "Home Services")
   - Profile image (optional)
4. User creates first service offering:
   - Service name (e.g., "60-min Deep Tissue Massage")
   - Description
   - Price (e.g., $80)
   - Duration (e.g., 60 minutes)
   - Category
5. User sets weekly availability:
   - Select days of week
   - Set start and end times for each day
   - Mark as available/unavailable
6. Provider dashboard now shows:
   - Profile completion status
   - Services listed
   - Availability calendar
   - Empty bookings list (ready to receive bookings)

**Success Criteria:**
- Provider can log in and see their dashboard
- Services appear in browse/search for customers
- Availability slots are bookable

---

### Workflow 2: Customer Booking & Payment Flow

**Steps:**
1. Customer registers/logs in with email/password
2. Customer browses available service providers and services:
   - Filter by category, price range
   - Search by service name or provider
3. Customer selects a service, views details:
   - Service description, price, duration
   - Provider information
   - Reviews/ratings (future feature)
4. Customer clicks "Book Now"
5. System shows availability calendar:
   - Displays available dates based on provider's schedule
   - Shows available time slots for selected date
   - Grays out already booked slots
6. Customer selects date and time slot
7. Customer reviews booking summary:
   - Service details
   - Date and time
   - Total price
   - Current wallet balance
8. Customer confirms booking and payment:
   - System checks wallet has sufficient balance
   - If yes: Creates booking with status `PENDING`
   - Deducts amount from customer wallet
   - Creates transaction with status `HELD`
   - Adds amount to escrow (tracked in transaction)
9. Provider receives notification (in dashboard) of new booking
10. Provider confirms booking → Status changes to `CONFIRMED`
11. Customer sees confirmed booking in "My Bookings"

**Success Criteria:**
- Customer can complete booking without errors
- Funds are held in escrow (transaction status = HELD)
- Both parties see the booking in their dashboards
- No double-booking occurs

---

### Workflow 3: Service Completion & Payment Release

**Steps:**
1. On service day, provider marks booking as `IN_PROGRESS` when customer arrives
2. After service completion, provider marks booking as `COMPLETED`
3. System shows "Release Payment" option to provider
4. Provider clicks "Release Payment":
   - System confirms action
   - Updates transaction status from `HELD` to `RELEASED`
   - Transfers amount from escrow to provider's wallet
   - Sets `paymentReleasedAt` timestamp
   - Updates provider's bookkeeping dashboard automatically
5. Provider's financial dashboard updates:
   - Total earnings increase
   - Pending amount (escrow) decreases
   - Transaction appears in history as "Released"
   - Revenue chart updates
6. Customer receives confirmation (shown in transaction history)
7. Customer can view receipt with transaction details

**Alternative Flow: Refund**
- If service was unsatisfactory or cancelled:
  - Provider (or Admin) clicks "Issue Refund"
  - System updates transaction status to `REFUNDED`
  - Returns amount to customer's wallet
  - Updates both parties' transaction histories
  - Booking status changes to `CANCELLED`

**Success Criteria:**
- Provider can release payment successfully
- Wallet balances update correctly
- Bookkeeping dashboard reflects the transaction
- Transaction history shows accurate status and timestamps
- Customer can view receipt

---

## Implementation Phases

### Phase 1: Project Setup & Authentication (Week 1)
**Tasks:**
- Initialize Next.js project with TypeScript
- Configure Tailwind CSS and shadcn/ui
- Set up Prisma with SQLite
- Create database schema
- Implement NextAuth.js authentication
- Create registration/login pages for all roles
- Build basic layout components (header, sidebar, navigation)
- Implement role-based routing and middleware

**Deliverables:**
- Users can register and login
- Role-based dashboards show different navigation
- Database schema is complete and seeded with test data

---

### Phase 2: Service Provider Features (Week 2)
**Tasks:**
- Create service provider profile setup flow
- Build service CRUD interface:
  - Create service form (name, description, price, duration, category)
  - Service list view with edit/delete actions
  - Service detail page
- Implement availability management:
  - Weekly schedule interface
  - Time slot selection
  - Save availability to database
- Provider dashboard overview:
  - Statistics cards (total services, bookings, earnings)
  - Recent bookings list
  - Quick actions

**Deliverables:**
- Service providers can create and manage services
- Availability can be set and edited
- Provider dashboard displays relevant information

---

### Phase 3: Customer Booking Flow (Week 2-3)
**Tasks:**
- Build service browsing interface:
  - Service listing with filters
  - Search functionality
  - Service detail pages
- Implement booking calendar:
  - Date picker showing available dates
  - Time slot selector based on availability
  - Real-time availability checking (no double-booking)
- Create booking form and confirmation:
  - Date/time selection
  - Booking summary
  - Payment confirmation from wallet
- Build customer bookings view:
  - List of all bookings with statuses
  - Booking detail page
  - Cancel booking functionality

**Deliverables:**
- Customers can browse and book services
- Booking flow is smooth and intuitive
- Availability is accurately reflected
- Double-booking is prevented

---

### Phase 4: Escrow & Transaction Management (Week 3)
**Tasks:**
- Implement mock wallet system:
  - Display wallet balance
  - Deduct on booking creation
  - Add on payment release/refund
- Create transaction processing logic:
  - Hold funds on booking (status: HELD)
  - Release payment functionality for providers
  - Refund functionality
  - Transaction status updates
- Build transaction views:
  - Transaction history for customers
  - Transaction history for providers
  - Transaction detail/receipt page
- Implement atomic balance updates (database transactions)

**Deliverables:**
- Funds are correctly held and released
- Wallet balances are accurate
- Transaction history is complete and auditable
- No race conditions in balance updates

---

### Phase 5: Bookkeeping Dashboard (Week 4)
**Tasks:**
- Create financial dashboard for providers:
  - Overview cards (total earnings, pending, balance)
  - Revenue by service type (chart)
  - Date range filtering
  - Daily/weekly/monthly views
- Build transaction analytics:
  - Aggregate queries for statistics
  - Revenue trends over time
  - Service performance metrics
- Implement CSV export:
  - Export filtered transactions
  - Include all relevant fields
- Create receipt generation:
  - Printable receipt format
  - Transaction details display

**Deliverables:**
- Providers see comprehensive financial overview
- Charts display accurate data
- CSV export works correctly
- Receipts are professional and complete

---

### Phase 6: Admin Panel (Week 4)
**Tasks:**
- Build admin dashboard:
  - System-wide statistics
  - Recent activity feed
- Create user management interface:
  - List all users with role indicators
  - User detail view
  - Disable/enable accounts
- Implement admin transaction management:
  - View all transactions
  - Manual release interface
  - Manual refund interface
  - Override capabilities
- Add admin booking oversight:
  - View all bookings
  - Booking details
  - Status management

**Deliverables:**
- Admins can view system-wide data
- Admins can manage payments manually
- User management is functional
- Admin actions are logged

---

## Success Metrics for MVP

**Technical Metrics:**
- All user workflows complete successfully
- No critical bugs in payment flows
- Database queries perform adequately (<500ms for most operations)
- Forms validate correctly with helpful error messages
- Role-based access control works correctly

**User Experience Metrics:**
- Users can complete registration in <2 minutes
- Booking flow takes <3 minutes from browsing to confirmation
- Payment release is one-click for providers
- Dashboards load in <2 seconds

**Data Integrity Metrics:**
- Zero instances of double-booking
- Zero instances of incorrect wallet balances
- All transactions have corresponding bookings
- Audit trail is complete for all payment operations

---

## Security Considerations (MVP)

**Authentication:**
- Passwords hashed with bcrypt
- Session-based authentication via NextAuth.js
- HTTP-only cookies for session tokens
- CSRF protection enabled

**Authorization:**
- Role-based access control on all routes
- API endpoints validate user roles
- Users can only access their own data (except admins)
- Sensitive operations (payment release/refund) require additional confirmation

**Data Protection:**
- Input validation on all forms (Zod schemas)
- SQL injection prevention via Prisma
- XSS prevention via React's built-in escaping
- Rate limiting on auth endpoints (future enhancement)

**SQLite Considerations:**
- File-based database suitable for MVP/demo
- Not recommended for production with concurrent users
- Migration to PostgreSQL/MySQL planned for production

---

## Future Enhancements (Post-MVP)

1. **Real Payment Integration**: Integrate Stripe/M-Pesa for actual transactions
2. **Notification System**: Email and SMS reminders for bookings
3. **Automated Workflows**: Auto-release payments based on triggers
4. **Mobile Apps**: Native iOS/Android applications
5. **Advanced Analytics**: ML-powered insights and forecasting
6. **Dispute System**: Structured dispute resolution workflow
7. **Reviews & Ratings**: Customer feedback system
8. **Chat System**: In-app messaging between parties
9. **Calendar Sync**: Integration with Google Calendar, Outlook
10. **Tax Features**: Automated tax document generation

---

## Development Timeline

**Total Duration:** 4 weeks (160 hours)

- **Week 1:** Phase 1 (Authentication & Setup)
- **Week 2:** Phase 2 & 3 (Provider Features & Booking Flow)
- **Week 3:** Phase 4 (Transactions & Escrow)
- **Week 4:** Phase 5 & 6 (Bookkeeping & Admin Panel)

**Team Size:** 1-2 developers

---

## Conclusion

This MVP specification provides a clear, actionable plan for building a functional prototype of the escrow platform. By focusing on core features with simplified implementations, we can validate the concept and gather user feedback before investing in complex features like real payment processing and advanced automation.

The MVP will demonstrate:
- ✅ The complete user journey from booking to payment
- ✅ Value proposition for all three user types
- ✅ Technical feasibility of the core features
- ✅ Foundation for future enhancements

Next steps: Begin Phase 1 implementation with project setup and authentication.

