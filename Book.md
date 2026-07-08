# Book Management System - Full Documentation

## Overview

The Book Management System is the core module of the Library application. It provides comprehensive functionality for managing the book catalog, including viewing, creating, editing, and deleting books (admin-only operations), as well as advanced features for renting, purchasing, and tracking books.

---

## 1. Full Feature Specification

### 1.1 Book Catalog Management

#### 1.1.1 View Books
- **Public access**: All users (authenticated and unauthenticated) can browse the book catalog
- **Pagination**: Books are displayed in paginated lists
- **Search**: Search books by title, author, category
- **Filtering**: Filter by availability, price range, category, author
- **Sorting**: Sort by title, price, rental price, publication year, popularity
- **Book Details**: View detailed information about a specific book including:
  - Title, description, ISBN
  - Author(s) with roles (main author, co-author)
  - Category/Categories
  - Price (purchase price)
  - Rental price
  - Deposit amount
  - Stock count (physical copies available)
  - Published year
  - Cover image URL
  - Availability status (available/unavailable for rent/purchase)
  - Active/Inactive status

#### 1.1.2 Create Books (Admin Only)
- Admin can create new books with all metadata
- Required fields: title, ISBN, price, rentalPrice, stockCount
- Optional fields: description, publishedYear, coverUrl, depositAmount
- Authors and categories can be linked during creation
- ISBN uniqueness validation
- PDF file upload for digital reading/downloading

#### 1.1.3 Edit Books (Admin Only)
- Full update: Replace all book fields
- Partial update: Update specific fields only
- Price update: Update purchase price, rental price, deposit amount separately
- Stock update: Adjust physical copy count
- Availability toggle: Mark book as available/unavailable for rent/purchase
- PDF replacement: Update the digital copy of the book

#### 1.1.4 Delete Books (Admin Only)
- **Soft delete**: Mark book as inactive (isActive = false), book remains in database
- **Hard delete**: Permanently remove book from database (with confirmation)
- Cannot delete books with active rentals/purchases
- Cannot delete books with pending transactions

### 1.2 Book Rental System

#### 1.2.1 Rent a Book
- User can rent a book for a specified period (default: 30 days)
- Rental period is configurable per book
- During rental period, user has digital access to read the book on the website
- After rental period expires, digital access is revoked
- Rental requires payment selection:
  - **Cash payment**: User receives a receipt via email with a barcode
  - **Card payment**: Payment is processed immediately

#### 1.2.2 Cash Payment Flow
1. User selects "Pay with Cash" option
2. System generates a rental receipt with:
   - Unique receipt number
   - Barcode (QR code or standard barcode)
   - Book details
   - Rental period (start date, end date)
   - Rental price
   - Deposit amount (if applicable)
3. Receipt is sent to user's email
4. User visits the library physically
5. Admin scans the barcode to confirm the rental
6. Upon confirmation:
   - Stock count is decremented by 1
   - Rental is activated
   - User gains digital access to the book

#### 1.2.3 Card Payment Flow
1. User selects "Pay with Card" option
2. System processes payment through integrated payment gateway
3. Upon successful payment:
   - Stock count is decremented by 1
   - Rental is activated immediately
   - User gains digital access to the book
   - Transaction record is created

#### 1.2.4 Stock Management
- Each book has a `stockCount` field representing physical copies
- When a book is rented: `stockCount = stockCount - 1`
- When a book is returned: `stockCount = stockCount + 1`
- If `stockCount = 0`:
  - Book is marked as unavailable for rent
  - Status displays "Unavailable"
  - Users cannot initiate new rentals

#### 1.2.5 Availability Status
- **Available**: Book can be rented/purchased (stockCount > 0)
- **Unavailable**: Book cannot be rented/purchased (stockCount = 0 or manually set)
- Availability can be manually toggled by admin
- Automatic availability calculation based on stock count

### 1.3 Book Purchase System

#### 1.3.1 Purchase a Book
- User can purchase a book for permanent ownership
- Upon purchase:
  - User gains permanent digital access to read the book on the website
  - User can download the PDF version of the book
  - Transaction record is created
  - Stock count is decremented by 1 (for physical copy purchase)
- Payment options:
  - **Cash payment**: Receipt with barcode sent via email, admin confirms in person
  - **Card payment**: Immediate processing

#### 1.3.2 Digital Library
- Purchased books appear in user's "My Books" section
- User can read purchased books online at any time
- User can download PDF of purchased books
- Access is permanent and does not expire

### 1.4 Rental Return System

#### 1.4.1 Return Process
- User returns the physical book to the library
- Admin confirms the return in the admin panel
- Upon confirmation:
  - Stock count is incremented by 1
  - Rental record is marked as returned
  - User's digital access to the book is revoked
  - Deposit is refunded (if applicable)

#### 1.4.2 Cash Payment Returns
- Only admin can confirm returns for cash-paid rentals
- Admin scans the original receipt barcode or looks up the rental
- Admin verifies the physical book condition
- Admin confirms return in the system

#### 1.4.3 Card Payment Returns
- Admin confirms returns for card-paid rentals
- Upon admin confirmation of return:
  - System calculates any outstanding fines based on days overdue
  - Fine is automatically charged to the user's saved card in a single transaction
  - If card charge fails:
    - User is notified via email
    - Fine status becomes PENDING
    - User must pay fine manually (cash or updated card)
    - User cannot rent new books until fine is paid
  - Remaining deposit is refunded to the user's card after fine deduction

### 1.5 Overdue and Fine System

#### 1.5.1 Overdue Detection
- System checks daily for overdue rentals
- A rental is overdue when: `currentDate > rentalEndDate`

#### 1.5.2 Reminder Notifications
- Starting from the day after the rental end date:
  - User receives a daily email notification reminding them to return the book
  - Notification includes:
    - Book title
    - Original due date
    - Number of days overdue
    - Current fine amount (if any)
    - Instructions for returning

#### 1.5.3 Fine Calculation Algorithm
- **Grace period**: 5 days after due date (no fine, only reminders)
- **Fine starts**: Day 6 after due date
- **Fine calculation formula**:

```
fine = baseFine * (daysOverdue - gracePeriod) * multiplier

Where:
  baseFine = rentalPrice * 0.1 (10% of rental price per day)
  gracePeriod = 5 days
  daysOverdue = currentDate - rentalEndDate
  multiplier = 1.0 for days 6-10
  multiplier = 1.5 for days 11-20
  multiplier = 2.0 for days 21+
```

- **Example**:
  - Rental price: $10.00
  - Days overdue: 15
  - Fine calculation:
    - Days 6-10 (5 days): 5 * ($10 * 0.1) * 1.0 = $5.00
    - Days 11-15 (5 days): 5 * ($10 * 0.1) * 1.5 = $7.50
    - Total fine: $12.50

- **Maximum fine**: Capped at 2x the rental price (to prevent excessive fines)

#### 1.5.4 Fine Collection - Cash Payment
- Fine is calculated and displayed to the user
- User must pay the fine in cash when returning the book
- Admin confirms fine payment upon return
- User cannot rent new books until fine is paid

#### 1.5.5 Fine Collection - Card Payment
- Fine is NOT charged automatically on a daily basis
- Fine is calculated and accumulated during the overdue period
- Fine is charged ONLY when admin confirms the book return:
  - Admin clicks "Confirm Return" in admin panel
  - System calculates total fine based on days overdue
  - System attempts to charge the full fine amount to the user's saved card
  - Transaction is recorded in the system
- If card charge fails due to insufficient funds or other card issues:
  - **Automatic fallback to cash payment**: Fine payment method is switched from CARD to CASH
  - Fine status becomes PENDING_CASH
  - User is notified via email about:
    - The failed card charge (with reason)
    - The automatic switch to cash payment
    - Available payment options
  - User can pay the fine in two ways:
    1. **Pay in cash at the library**: Visit the library, pay the fine in cash, admin confirms in system
    2. **Update payment method in profile**: Log into account, add/update a card with sufficient funds, trigger payment from profile
  - User cannot rent new books until fine is paid
- User can view fine history, status, and payment options in their profile

### 1.6 Admin Features for Books

#### 1.6.1 Price Management
- Set purchase price for each book
- Set rental price for each book
- Set deposit amount (optional, for high-value books)
- Update prices at any time

#### 1.6.2 Rental Statistics
- View total number of times a book has been rented
- View list of users who have rented a specific book
- View current active rentals for a book
- View rental history with dates and status

#### 1.6.3 User Management for Books
- View all users who have rented a specific book
- View user's rental history
- View user's current active rentals
- View user's overdue books and fines

#### 1.6.4 Return Confirmation
- Admin panel displays pending returns
- Admin can search for rentals by:
  - Receipt barcode
  - User ID/email
  - Book ID/title
- Admin confirms return with one click
- System handles stock count, access revocation, fine settlement

### 1.7 PDF Management

#### 1.7.1 PDF Upload
- Admin can upload PDF files for books
- PDF is stored securely (not publicly accessible)
- PDF is linked to the book record

#### 1.7.2 PDF Access
- **Rental**: User can read PDF online during rental period only
- **Purchase**: User can read PDF online and download it permanently
- PDF viewer is embedded in the website
- Download button is available only for purchased books

#### 1.7.3 PDF Security - Anti-Piracy System
- PDFs are not directly accessible via URL
- Access is validated against user's rental/purchase records
- **Anti-Piracy Watermarking**:
  - When a user downloads a PDF, the system generates a unique anti-piracy code
  - Code format: `LIB-{userId}-{bookId}-{timestamp}-{randomHash}`
  - The code is embedded into the PDF file in multiple locations:
    - Visible watermark on each page (user's email/name + code)
    - Invisible metadata embedded in PDF properties
    - Hidden text layer with the code (not visible when reading, but present in file)
  - The anti-piracy code is recorded in the database:
    - Table: `pdf_copies`
    - Fields: code, userId, bookId, downloadDate, pdfFileHash, status
  - If a pirated copy is found:
    - Admin can extract the code from the PDF
    - System looks up the code in the database
    - Identifies the user who downloaded that specific copy
    - Legal action can be taken against the user
  - Each download generates a new unique copy with a new code
  - PDF file hash is stored to verify integrity
  - User cannot download the same PDF twice without generating a new code

---

## 2. Currently Implemented Features

### 2.1 Book Catalog
| Feature | Status | Notes |
|---------|--------|-------|
| View all books (paginated) | ✅ Implemented | `GET /api/v1/books` |
| View all books (no pagination) | ✅ Implemented | `GET /api/v1/books/all` |
| View book by ID | ✅ Implemented | `GET /api/v1/books/{id}` with Redis caching |
| Search books by title | ✅ Implemented | `GET /api/v1/books/search?title=` |
| Create book (Admin) | ✅ Implemented | `POST /api/v1/books` |
| Full update book (Admin) | ✅ Implemented | `PUT /api/v1/books/{id}` |
| Partial update book (Admin) | ✅ Implemented | `PATCH /api/v1/books/{id}` |
| Update prices (Admin) | ✅ Implemented | `PATCH /api/v1/books/{id}/prices` |
| Soft delete book (Admin) | ✅ Implemented | `DELETE /api/v1/books/{id}` |
| Hard delete book (Admin) | ✅ Implemented | `DELETE /api/v1/books/hard-delete/{id}` |
| Book import from Open Library | ✅ Implemented | `POST /api/v1/books/import` (async via RabbitMQ) |
| Redis caching for books | ✅ Implemented | 10-minute TTL |

### 2.2 Book Entity Fields
| Field | Status | Notes |
|-------|--------|-------|
| id | ✅ | Primary key |
| title | ✅ | |
| description | ✅ | |
| isbn | ✅ | Unique constraint |
| price | ✅ | Purchase price |
| rentalPrice | ✅ | Rental price per period |
| depositAmount | ✅ | Deposit for rental |
| stockCount | ✅ | Physical copies count |
| publishedYear | ✅ | |
| coverUrl | ✅ | Cover image URL |
| isActive | ✅ | Soft delete flag |

### 2.3 Related Entities
| Entity | Status | Notes |
|--------|--------|-------|
| Author | ✅ | With bio |
| Category | ✅ | |
| BookAuthor | ✅ | With author role (MAIN_AUTHOR/CO_AUTHOR) and order |
| BookCategory | ✅ | Many-to-many relationship |

### 2.4 Admin Features
| Feature | Status | Notes |
|---------|--------|-------|
| Admin dashboard statistics | ✅ Implemented | `GET /admin/dashboard` |
| View all users | ✅ Implemented | `GET /admin/users` |
| View user details | ✅ Implemented | `GET /admin/users/{id}` |
| Change user roles | ✅ Implemented | `PUT /admin/users/{id}/roles` |
| Activate/deactivate user | ✅ Implemented | `PATCH /admin/users/{id}/status` |

### 2.5 Authentication & Authorization
| Feature | Status | Notes |
|---------|--------|-------|
| JWT authentication | ✅ Implemented | Access token (24h) + Refresh token (7d) |
| Role-based access control | ✅ Implemented | ROLE_ADMIN, ROLE_USER |
| Admin-only endpoints | ✅ Implemented | `@PreAuthorize` annotations |

### 2.6 Infrastructure
| Component | Status | Notes |
|-----------|--------|-------|
| PostgreSQL database | ✅ | Flyway migrations |
| Redis caching | ✅ | Spring Cache |
| RabbitMQ messaging | ✅ | Async operations |
| Email notifications | ✅ | Welcome email on registration |
| Docker compose | ✅ | All services containerized |

---

## 3. Planned Features (Not Yet Implemented)

### 3.1 Rental System
| Feature | Priority | Notes |
|---------|----------|-------|
| Rent a book endpoint | High | Create rental record, decrement stock |
| Rental entity/model | High | Store rental details, dates, status |
| Rental period management | High | Configurable rental duration |
| Digital access control | High | Grant/revoke book reading access |
| Rental history | Medium | View past rentals |

### 3.2 Purchase System
| Feature | Priority | Notes |
|---------|----------|-------|
| Purchase a book endpoint | High | Create purchase record, decrement stock |
| Purchase entity/model | High | Store purchase details |
| Permanent digital access | High | User's "My Books" section |
| PDF download for purchased books | High | Secure download link |
| Online PDF reader | Medium | Embedded viewer |

### 3.3 Payment System
| Feature | Priority | Notes |
|---------|----------|-------|
| Payment method management | High | Add/edit/delete payment methods |
| Cash payment flow | High | Receipt generation, barcode, email |
| Card payment flow | High | Payment gateway integration |
| Payment entity/model | High | Transaction records |
| Payment service layer | High | Process payments |
| Payment controller | High | API endpoints |

### 3.4 Stock & Availability
| Feature | Priority | Notes |
|---------|----------|-------|
| Automatic stock decrement | High | On rent/purchase |
| Automatic stock increment | High | On return |
| Availability status calculation | High | Based on stock count |
| Manual availability toggle | Medium | Admin override |
| "Unavailable" status display | Medium | API response field |

### 3.5 Return System
| Feature | Priority | Notes |
|---------|----------|-------|
| Return book endpoint (Admin) | High | Confirm return, increment stock |
| Return confirmation UI | Medium | Admin panel feature |
| Barcode scanning support | Medium | For cash payment receipts |
| Deposit refund processing | Medium | Automatic refund |

### 3.6 Overdue & Fine System
| Feature | Priority | Notes |
|---------|----------|-------|
| Overdue detection scheduled job | High | Daily check |
| Reminder notifications | High | Daily email for overdue books |
| Fine calculation algorithm | High | Progressive fine system |
| Fine entity/model | High | Store fine records |
| Fine charge on return confirmation | High | Single charge when admin confirms return |
| Fallback to cash on card failure | High | Auto-switch to cash if card charge fails |
| Fine payment (cash) | Medium | Admin confirmation at return |
| Fine payment via profile | Medium | User can update card and pay from profile |
| Fine history | Medium | View fine history |
| Account suspension for unpaid fines | Low | Until fine is paid |

### 3.7 Admin Book Statistics
| Feature | Priority | Notes |
|---------|----------|-------|
| Total rentals per book | Medium | Counter in book entity |
| Users who rented a book | Medium | List with dates |
| Current active rentals | Medium | Real-time count |
| Rental revenue statistics | Low | Financial reporting |

### 3.8 PDF Management
| Feature | Priority | Notes |
|---------|----------|-------|
| PDF upload (Admin) | High | Secure storage |
| PDF access control | High | Based on rental/purchase |
| PDF viewer | Medium | Web-based reader |
| PDF download | Medium | For purchased books only |
| Anti-piracy watermarking | High | Visible + invisible watermarks |
| Unique code generation | High | Per-download unique code |
| Code storage in database | High | Track who downloaded which copy |
| Pirated copy identification | Medium | Extract code to find source |

### 3.9 Receipt & Barcode System
| Feature | Priority | Notes |
|---------|----------|-------|
| Receipt generation | High | For cash payments |
| Barcode/QR code generation | High | Unique identifier |
| Email receipt delivery | High | Send to user's email |
| Barcode scanning (Admin) | Medium | Confirm rental/purchase |

### 3.10 Notification Enhancements
| Feature | Priority | Notes |
|---------|----------|-------|
| Rental confirmation email | High | Sent on successful rental |
| Purchase confirmation email | High | Sent on successful purchase |
| Overdue reminder emails | High | Daily until return |
| Fine notification email | High | When fine is applied |
| Return confirmation email | Medium | Sent on successful return |
| Receipt email with barcode | High | For cash payments |

---

## 4. API Endpoints (Planned)

### 4.1 Rental Endpoints
| Method | Endpoint | Access | Description |
|--------|----------|--------|-------------|
| POST | `/api/v1/books/{id}/rent` | User | Rent a book |
| GET | `/api/v1/rentals` | User | View user's rentals |
| GET | `/api/v1/rentals/{id}` | User | View rental details |
| GET | `/api/v1/rentals/active` | User | View active rentals |
| GET | `/api/v1/rentals/overdue` | User | View overdue rentals |

### 4.2 Purchase Endpoints
| Method | Endpoint | Access | Description |
|--------|----------|--------|-------------|
| POST | `/api/v1/books/{id}/purchase` | User | Purchase a book |
| GET | `/api/v1/purchases` | User | View user's purchases |
| GET | `/api/v1/purchases/{id}` | User | View purchase details |
| GET | `/api/v1/purchases/{id}/download` | User | Download purchased PDF |

### 4.3 Return Endpoints (Admin)
| Method | Endpoint | Access | Description |
|--------|----------|--------|-------------|
| POST | `/admin/rentals/{id}/return` | Admin | Confirm book return |
| POST | `/admin/rentals/scan-barcode` | Admin | Scan receipt barcode |

### 4.4 Payment Endpoints
| Method | Endpoint | Access | Description |
|--------|----------|--------|-------------|
| GET | `/api/v1/payment-methods` | User | View payment methods |
| POST | `/api/v1/payment-methods` | User | Add payment method |
| PUT | `/api/v1/payment-methods/{id}` | User | Update payment method |
| DELETE | `/api/v1/payment-methods/{id}` | User | Delete payment method |
| POST | `/api/v1/payments/process` | User | Process payment |

### 4.5 Admin Book Statistics Endpoints
| Method | Endpoint | Access | Description |
|--------|----------|--------|-------------|
| GET | `/admin/books/{id}/rentals` | Admin | View rentals for a book |
| GET | `/admin/books/{id}/rentals/count` | Admin | Total rentals count |
| GET | `/admin/books/{id}/rentals/users` | Admin | Users who rented |
| GET | `/admin/rentals/overdue` | Admin | All overdue rentals |
| POST | `/admin/fines/{id}/collect` | Admin | Collect fine manually (cash) |
| POST | `/admin/fines/{id}/confirm-cash-payment` | Admin | Confirm cash payment for failed card charge |

### 4.7 User Fine Payment Endpoints
| Method | Endpoint | Access | Description |
|--------|----------|--------|-------------|
| GET | `/api/v1/fines` | User | View user's fines |
| GET | `/api/v1/fines/{id}` | User | View fine details |
| POST | `/api/v1/fines/{id}/pay` | User | Pay fine with updated card from profile |
| PUT | `/api/v1/fines/{id}/payment-method` | User | Update payment method for pending fine |

### 4.6 PDF Endpoints
| Method | Endpoint | Access | Description |
|--------|----------|--------|-------------|
| POST | `/admin/books/{id}/pdf` | Admin | Upload PDF |
| GET | `/api/v1/books/{id}/read` | User | Read book online |
| GET | `/api/v1/books/{id}/pdf` | User | View PDF metadata |

---

## 5. Database Schema (Planned Additions)

### 5.1 New Entities

#### Rental
| Field | Type | Description |
|-------|------|-------------|
| id | UUID | Primary key |
| userId | UUID | Foreign key to User |
| bookId | UUID | Foreign key to Book |
| startDate | LocalDate | Rental start date |
| endDate | LocalDate | Rental end date |
| actualReturnDate | LocalDate | Actual return date (null if not returned) |
| paymentMethod | Enum | CASH/CARD |
| status | Enum | ACTIVE/RETURNED/OVERDUE/CANCELLED |
| rentalPrice | BigDecimal | Price paid for rental |
| depositAmount | BigDecimal | Deposit paid |
| receiptCode | String | Barcode/receipt code for cash payments |
| isReceiptConfirmed | Boolean | Whether admin confirmed receipt |
| fineAmount | BigDecimal | Total fine accrued |
| createdAt | LocalDateTime | Creation timestamp |
| updatedAt | LocalDateTime | Last update timestamp |

#### Purchase
| Field | Type | Description |
|-------|------|-------------|
| id | UUID | Primary key |
| userId | UUID | Foreign key to User |
| bookId | UUID | Foreign key to Book |
| purchaseDate | LocalDateTime | Date of purchase |
| paymentMethod | Enum | CASH/CARD |
| status | Enum | PENDING/COMPLETED/CANCELLED |
| purchasePrice | BigDecimal | Price paid |
| receiptCode | String | Barcode/receipt code for cash payments |
| isReceiptConfirmed | Boolean | Whether admin confirmed receipt |
| createdAt | LocalDateTime | Creation timestamp |

#### Fine
| Field | Type | Description |
|-------|------|-------------|
| id | UUID | Primary key |
| rentalId | UUID | Foreign key to Rental |
| userId | UUID | Foreign key to User |
| amount | BigDecimal | Fine amount |
| reason | String | Reason for fine |
| status | Enum | PENDING/PAID/WAIVED/PENDING_CASH |
| originalPaymentMethod | Enum | Original payment method (CASH/CARD) |
| currentPaymentMethod | Enum | Current payment method (may change after fallback) |
| chargedDate | LocalDate | Date fine was charged |
| fallbackReason | String | Reason for fallback to cash (e.g., INSUFFICIENT_FUNDS) |
| createdAt | LocalDateTime | Creation timestamp |
| updatedAt | LocalDateTime | Last update timestamp |

#### BookPdf
| Field | Type | Description |
|-------|------|-------------|
| id | UUID | Primary key |
| bookId | UUID | Foreign key to Book |
| fileUrl | String | Secure storage URL |
| fileSize | Long | File size in bytes |
| uploadedAt | LocalDateTime | Upload timestamp |
| uploadedBy | UUID | Admin who uploaded |

#### PdfCopy (Anti-Piracy Tracking)
| Field | Type | Description |
|-------|------|-------------|
| id | UUID | Primary key |
| code | String | Unique anti-piracy code (LIB-{userId}-{bookId}-{timestamp}-{hash}) |
| userId | UUID | Foreign key to User who downloaded |
| bookId | UUID | Foreign key to Book |
| downloadDate | LocalDateTime | When the PDF was downloaded |
| pdfFileHash | String | SHA-256 hash of the generated PDF |
| visibleWatermark | String | Visible watermark text applied |
| invisibleMetadata | String | Hidden metadata embedded |
| status | Enum | ACTIVE/REVOKED/FLAGGED |
| flaggedReason | String | Reason if flagged (e.g., found pirated) |

---

## 6. Fine Calculation Algorithm Details

### 6.1 Algorithm Specification

**Important**: Fines are NOT charged automatically on a daily basis. The fine is calculated during the overdue period and charged ONLY when the admin confirms the book return.

```java
public BigDecimal calculateFine(BigDecimal rentalPrice, int daysOverdue) {
    int gracePeriod = 5;
    
    if (daysOverdue <= gracePeriod) {
        return BigDecimal.ZERO;
    }
    
    BigDecimal baseFine = rentalPrice.multiply(new BigDecimal("0.1"));
    BigDecimal totalFine = BigDecimal.ZERO;
    
    int daysWithFine = daysOverdue - gracePeriod;
    
    for (int day = 1; day <= daysWithFine; day++) {
        BigDecimal multiplier;
        if (day <= 5) {
            multiplier = new BigDecimal("1.0");      // Days 6-10
        } else if (day <= 15) {
            multiplier = new BigDecimal("1.5");      // Days 11-20
        } else {
            multiplier = new BigDecimal("2.0");      // Days 21+
        }
        totalFine = totalFine.add(baseFine.multiply(multiplier));
    }
    
    // Cap at 2x rental price
    BigDecimal maxFine = rentalPrice.multiply(new BigDecimal("2.0"));
    return totalFine.min(maxFine);
}
```

### 6.2 Fine Charging Process

**For Cash Payments:**
1. User returns the book to the library
2. Admin confirms return in admin panel
3. System calculates total fine based on days overdue
4. Admin displays fine amount to user
5. User pays fine in cash
6. Admin marks fine as PAID in the system
7. Return is fully processed

**For Card Payments:**
1. User returns the book to the library
2. Admin confirms return in admin panel
3. System calculates total fine based on days overdue
4. System attempts to charge the full fine amount to user's saved card
5. If charge succeeds:
   - Fine is marked as PAID
   - Transaction is recorded
   - Remaining deposit (if any) is refunded
   - Return is fully processed
6. If charge fails due to insufficient funds:
   - Fine payment method is automatically switched from CARD to CASH
   - Fine status becomes PENDING_CASH
   - User is notified via email about the failed card charge and the switch to cash payment
   - User has two options to pay:
     a. **Pay in cash at the library**: User visits the library, pays the fine in cash, admin confirms payment in the system
     b. **Update payment method in profile**: User logs into their account, adds/updates a payment method with sufficient funds, and triggers payment from their profile
   - User cannot rent new books until fine is paid
7. If charge fails for other reasons (expired card, blocked card, etc.):
   - Same fallback process as insufficient funds
   - User is notified with the specific error reason

### 6.3 Fine Calculation Examples

| Rental Price | Days Overdue | Fine Amount | Notes |
|--------------|--------------|-------------|-------|
| $10.00 | 3 | $0.00 | Within grace period |
| $10.00 | 5 | $0.00 | Last day of grace period |
| $10.00 | 6 | $1.00 | First day of fine |
| $10.00 | 10 | $5.00 | 5 days at 1.0x |
| $10.00 | 15 | $12.50 | 5 days at 1.0x + 5 days at 1.5x |
| $10.00 | 20 | $20.00 | 5 days at 1.0x + 10 days at 1.5x |
| $10.00 | 25 | $20.00 | Capped at 2x rental price |

---

## 7. State Diagrams

### 7.1 Rental Lifecycle

```
[Created] → [Pending Confirmation] → [Active] → [Overdue] → [Returned]
                    ↓                      ↓
              [Cancelled]           [Fine Applied]
```

### 7.2 Purchase Lifecycle

```
[Created] → [Pending Payment] → [Completed] → [Access Granted]
                    ↓
              [Cancelled]
```

### 7.3 Book Availability

```
[Available] ←→ [Unavailable]
     ↓
[Out of Stock] (stockCount = 0)
```

---

## 8. Security Considerations

### 8.1 Access Control
- Book viewing: Public (no authentication required)
- Book creation/editing/deletion: Admin only
- Book renting/purchasing: Authenticated users only
- Return confirmation: Admin only
- PDF access: Based on active rental or purchase record
- PDF download: Purchased books only

### 8.2 Data Protection
- PDFs stored outside public web root
- Access tokens validated before PDF serving
- Receipt codes are unique and non-guessable
- Payment information encrypted at rest
- **Anti-Piracy Protection**:
  - Each downloaded PDF contains a unique identifier code
  - Code is embedded visibly (watermark) and invisibly (metadata)
  - All codes are stored in database linked to specific users
  - Pirated copies can be traced back to the original downloader
  - PDF file hashes stored to verify file integrity
- **Payment Fallback Security**:
  - When card charge fails, payment method switch is logged
  - User must authenticate to pay fine from profile
  - Admin must confirm cash payments in person

### 8.3 Audit Trail
- All admin actions logged
- Rental/purchase history immutable
- Fine calculations logged with timestamps
- Payment transactions recorded
- PDF download history with unique codes
- Anti-piracy code generation and verification logged

---

## 9. Integration Points

### 9.1 External Services
- **Payment Gateway**: For card payments (Stripe, PayPal, etc.)
- **Email Service**: For receipts, notifications, reminders
- **PDF Storage**: Secure cloud storage (S3, etc.)
- **Barcode Generator**: For receipt generation
- **PDF Watermarking Service**: For anti-piracy code embedding
- **Code Generator**: For unique anti-piracy code generation

### 9.2 Internal Services
- **Notification Service**: For email reminders and notifications
- **Cache Service**: For book data caching
- **Scheduled Jobs**: For overdue detection and fine calculation
- **Message Queue**: For async operations (receipt generation, notifications)

---

## 10. Testing Strategy

### 10.1 Unit Tests
- Fine calculation algorithm
- Fine charging on return confirmation
- Fallback to cash on card failure
- Stock count management
- Availability status calculation
- Rental period validation
- Anti-piracy code generation
- PDF watermarking logic
- Code uniqueness validation

### 10.2 Integration Tests
- Rental flow (create, activate, return)
- Purchase flow (create, confirm, access)
- Payment processing (mock gateway)
- Email notification delivery
- Fine charging on return confirmation (card and cash)
- Fallback to cash payment when card charge fails
- User paying fine from profile with updated card
- Admin confirming cash payment for failed card charge
- PDF download with anti-piracy code
- Anti-piracy code extraction and user identification

### 10.3 End-to-End Tests
- Complete rental lifecycle
- Complete purchase lifecycle
- Overdue detection and fine application
- Admin return confirmation with fine charging
- Card charge failure and fallback to cash payment
- User updating payment method and paying fine from profile
- Admin confirming cash payment after failed card charge
- PDF download and anti-piracy code verification
- Pirated copy identification workflow
