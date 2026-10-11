# M2N Integration Architecture & Repository Audit
**Authoritative System Integration Document**
**Date:** October 2026

---

## 1. Executive Summary

This architecture document presents the comprehensive audit and technical integration plan for synchronizing the **M2N Website** (comprising the User Portal, Admin Dashboard, and Reception Dashboard) with the **React Native (Expo) Mobile Application** (`M2N/`).

The integration enforces a single shared data model where:
- The website's existing backend servers and PostgreSQL database (hosted on Supabase) are the **authoritative source of truth**.
- Both website and mobile platforms read and write the exact same underlying database records and canonical UUIDs.
- All three roles (**Admin**, **Receptionist**, and **User**) operate with identical permissions, business rules, and state across web and mobile.
- Image uploads exclusively reuse the existing **Cloudinary** integration (`m2n-hotels` directory).
- Identity card collection and uploads exclusively reuse the existing **Google Drive** backend workflow (`Root -> Hotel -> Year -> Month -> Date -> [bookingId].[ext]`), storing the returned Drive webView link in `Booking.validIdUrl`.
- All rich mobile-only frontend screens, luxury animations, and customer features are fully preserved without reduction.

---

## 2. Current Architecture Inventory

### 2.1 Website Backend Architecture (Authoritative)

The website is composed of three dedicated microservices running Node.js / Express with TypeScript and Prisma ORM:

```text
┌────────────────────────────────────────────────────────────────────────┐
│                        SUPABASE POSTGRESQL DB                          │
│               Host: aws-0-ap-northeast-2.pooler.supabase.com           │
│                    Project Ref: cpeuatsblwxwgykhixos                   │
└───────────────────▲────────────────────▲────────────────────▲──────────┘
                    │                    │                    │
          Prisma ORM│          Prisma ORM│          Prisma ORM│
                    │                    │                    │
┌───────────────────┴────┐   ┌───────────┴────────┐   ┌───────┴──────────┐
│     USER BACKEND       │   │   ADMIN BACKEND    │   │RECEPTION BACKEND │
│      Port: 4000        │   │     Port: 5000     │   │    Port: 6001    │
│  (user/backend/src)    │   │(admindashboard/...)│   │(receptiondash...)│
├────────────────────────┤   ├────────────────────┤   ├──────────────────┤
│ • /api/auth/register   │   │ • /api/auth/login  │   │ • /api/auth/login│
│ • /api/auth/login      │   │ • /api/auth/2fa/*  │   │ • /api/bookings  │
│ • /api/auth/google     │   │ • /api/hotels (CRUD│   │ • /api/bookings/ │
│ • /api/bookings/create │   │   via Cloudinary)  │   │   :id/checkin    │
│ • /api/bookings/verify │   │ • /api/categories  │   │   (Google Drive) │
│ • /api/bookings/my-bkgs│   │ • /api/public/*    │   │ • /api/bookings/ │
│ • /api/bookings/cancel │   │ • /api/admins      │   │   walkin         │
│                        │   │ • /api/reception   │   │   (Google Drive) │
└────────────────────────┘   └────────────────────┘   └──────────────────┘
```

1. **User Backend (`user/backend`)**:
   - **Port:** `4000` (Dev: `http://localhost:4000/api`, Prod: `https://m2nhotels.com/api`)
   - **Responsibilities:** Customer authentication, registration with email OTP, Google OAuth token verification, room availability verification, Razorpay payment creation & verification, customer booking retrieval (`/api/bookings/my-bookings`), and booking cancellation requests.
   - **Authentication:** JWT signed with `JWT_SECRET`, token payload `{ userId: User.id }`.

2. **Admin Backend (`admindashboard/admin/backend`)**:
   - **Port:** `5000` (Dev: `http://localhost:5000/api`, Prod: `https://admin.m2nhotels.com/api`)
   - **Responsibilities:** Superadmin and Hotel Admin authentication with TOTP 2FA (`otplib`), hotel catalog management with Cloudinary image upload (`multer-storage-cloudinary`), room categories management with Cloudinary multi-image uploads, global room category templates, staff management (creating and updating Receptionists and Admins), and booking oversight.
   - **Authentication:** JWT signed with `JWT_SECRET`, token payload `{ id: Admin.id, role: Admin.role }`.
   - **Public Endpoints:** `/api/public/hotels`, `/api/public/hotels/:id`, `/api/public/hotels/:id/categories`, `/api/public/categories`.

3. **Reception Backend (`receptiondashboard/reception/backend`)**:
   - **Port:** `6001` (Dev: `http://localhost:6001/api`, Prod: `https://reception.m2nhotels.com/api`)
   - **Responsibilities:** Hotel receptionist authentication, fetching bookings scoped to the receptionist's assigned `hotelId`, available room inventory inspection, check-in processing (`/api/bookings/:id/checkin`) with Google Drive ID document upload and room status change, walk-in booking creation (`/api/bookings/walkin`) with Google Drive upload and 5-digit booking ID generation, and cancellation request processing.
   - **Authentication:** JWT signed with `JWT_SECRET`, token payload `{ id: Receptionist.id, role: 'receptionist' }`.

---

### 2.2 Database Entities & Relationships (Prisma / PostgreSQL)

All three backends connect to the same PostgreSQL database schema defined in `prisma/schema.prisma`:

| Model | Primary Key | Foreign Keys / Relations | Key Attributes & Enums | Storage / File References |
|---|---|---|---|---|
| `User` | `id` (UUID) | `bookings: Booking[]` | `name`, `email` (unique), `password` (bcrypt), `phone`, `nationality`, `isVerified` (Boolean) | None |
| `Admin` | `id` (UUID) | `hotels: Hotel[]` | `email` (unique), `password` (bcrypt), `role` ("admin" \| "superadmin"), `permissions` (`String[]`), `twoFactorEnabled` | None |
| `Receptionist` | `id` (UUID) | `hotelId` -> `Hotel.id` (1-to-1) | `name`, `email` (unique), `password` (bcrypt), `role` ("receptionist"), `permissions` (`String[]`) | None |
| `Hotel` | `id` (String) | `addedBy` -> `Admin.id`, `receptionist: Receptionist?`, `categories: RoomCategory[]`, `bookings: Booking[]` | `name`, `city`, `state`, `type`, `tagline`, `address`, `lat`, `lng`, `description`, `isArchived` | `images: String[]` (Cloudinary URLs in `m2n-hotels`) |
| `GlobalRoomCategory` | `id` (String) | None | `name`, `amenities: String[]`, `isArchived` | None |
| `RoomCategory` | `id` (String) | `hotelId` -> `Hotel.id`, `bookings: Booking[]` | `name`, `numberOfRooms`, `roomSize`, `numberOfBeds`, `isAC`, `view`, `capacity`, `price`, `features: String[]`, `rooms: Json` (`[{ id, roomNumber, status: 'Available'\|'CheckIn'\|'Occupied' }]`) | `images: String[]`, `galleryImages: String[]` (Cloudinary URLs) |
| `Booking` | `id` (UUID) | `userId` -> `User.id`, `hotelId` -> `Hotel.id`, `roomCategoryId` -> `RoomCategory.id` | `bookingId` (unique 5-digit string e.g. "00001"), `checkIn`, `checkInTime`, `checkOut`, `checkOutTime`, `quantity`, `adults`, `children`, `totalAmount`, `status` ("pending" \| "confirmed" \| "cancelled" \| "failed"), `assignedRoomNumber`, `cancellationRequested` | `validIdUrl: String?` (Google Drive `webViewLink`) |

---

### 2.3 Cloud Storage Integrations

#### 1. Cloudinary Integration (Image Assets)
- **Configuration:** `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET`.
- **Target Folder:** `m2n-hotels`.
- **Allowed Formats:** `jpg`, `jpeg`, `png`, `webp`.
- **Storage Strategy:** Handled server-side by Multer (`multer-storage-cloudinary`). The uploaded file's Cloudinary CDN URL (`file.path`) is stored in the database array fields (`Hotel.images`, `RoomCategory.images`, `RoomCategory.galleryImages`).
- **Mobile Rule:** The mobile app must upload images through the existing backend endpoints (`multipart/form-data`) or request signed upload authorizations. **ImageKit is forbidden and removed.**

#### 2. Google Drive Integration (Guest Government Identity Proofs)
- **Configuration:** Google Service Account (`GOOGLE_SERVICE_ACCOUNT_EMAIL`, `GOOGLE_PRIVATE_KEY`, `GOOGLE_DRIVE_ROOT_FOLDER_ID`).
- **Target Folder Hierarchy:**
  `[GOOGLE_DRIVE_ROOT_FOLDER_ID] -> [Hotel Name] -> [Year] -> [Month] -> [Date]`
- **Naming Pattern:** `[bookingId].[extension]` (e.g. `00001.png`, `00002.jpg`).
- **Permissions:** Set to viewable link (`role: 'reader'`, `type: 'anyone'`).
- **Database Mapping:** The resulting `webViewLink` is saved directly to `Booking.validIdUrl`.
- **Mobile Rule:** When a receptionist takes a live photo or selects an ID document in the mobile app, the file is submitted via `multipart/form-data` to `/api/bookings/:id/checkin` or `/api/bookings/walkin`. The backend processes the upload to Google Drive and sets `validIdUrl`.

---

## 3. Current Mobile Architecture (`M2N/`)

The mobile application is built with React Native (Expo SDK 57), featuring high-end luxury aesthetics.

### 3.1 Screen Inventory & Feature Matrix

| Screen / Feature | Website Portal Equivalent | Overlap Status | Mobile Integration Action |
|---|---|---|---|
| `HomeScreen` | `user/frontend/src/pages/Home.tsx` | Overlapping | Fetch real hotels & lowest rates from `/api/public/hotels`, fallback to `siteData.js` offline |
| `HotelsScreen` | `user/frontend/src/pages/Hotels.tsx` | Overlapping | Fetch real hotels from `/api/public/hotels`, live search & filter |
| `HotelDetailScreen`| `user/frontend/src/pages/HotelDetail.tsx`| Overlapping | Fetch hotel by ID and room categories from `/api/public/hotels/:id/categories` |
| `RoomsScreen` | `user/frontend/src/pages/Rooms.tsx` | Overlapping | Fetch room categories from `/api/public/categories` |
| `RoomDetailScreen` | Room modal / detail | Overlapping | Fetch category data, live availability checking via `/api/bookings/check-availability` |
| `BookingScreen` | `user/frontend/src/pages/Booking.tsx` | Overlapping | Check real-time room availability, calculate total amount, submit booking |
| `PaymentScreen` | `user/frontend/src/components/BookingWidget` | Overlapping | Create confirmed booking in backend, handle payment, add to `tripsStore` |
| `MyTripsScreen` | `user/frontend/src/pages/MyBookings.tsx`| Overlapping | Sync with `/api/bookings/my-bookings`, cancellation request via `/api/bookings/:id/request-cancel` |
| `LoginScreen` | `user/frontend/src/pages/Login.tsx` | Overlapping | Authenticate against `/api/auth/login` and `/api/auth/register` (stored in `User` table) |
| `AdminScreen` | `admindashboard/admin/frontend` | Overlapping | Connect all 14 tabs to `admin/backend` (port 5000): live hotels, categories, bookings, staff |
| `AdminLoginScreen` | `admindashboard/admin/frontend/Login` | Overlapping | Connect to `/api/auth/login` (port 5000) with Superadmin and TOTP 2FA verification |
| `ReceptionScreen` | `receptiondashboard/reception/frontend` | Overlapping | Connect to `reception/backend` (port 6001): live hotel bookings, check-in with ID upload, walk-in |
| `WishlistScreen` | None (Mobile Exclusive) | Mobile-Only | **Preserve completely** using local `wishlistStore` |
| `GiftCardScreen` | None (Mobile Exclusive) | Mobile-Only | **Preserve completely** using local voucher generator and store |
| `ExperiencesScreen`| None (Static on Web) | Mobile-Only | **Preserve completely** with rich interactive itineraries and booking modal |
| `RestaurantScreen`| None (Static on Web) | Mobile-Only | **Preserve completely** with dining reservations & menus |
| `WeddingsScreen` | None (Static on Web) | Mobile-Only | **Preserve completely** with venue inquiry form |
| `GalleryScreen` | None (Static on Web) | Mobile-Only | **Preserve completely** with filtered media tabs |
| `JournalScreen` | None (Static on Web) | Mobile-Only | **Preserve completely** with articles and stories |
| `SpaWellnessScreen`| None (Static on Web) | Mobile-Only | **Preserve completely** with spa treatment booking |
| `OffersScreen` | None (Static on Web) | Mobile-Only | **Preserve completely** with promo voucher cards |
| `AboutScreen` | `user/frontend/src/pages/About.tsx` | Overlapping | Informational screen preserved |
| `ContactScreen` | `user/frontend/src/pages/Contact.tsx` | Overlapping | Informational screen preserved |
| `AiAssistantModal`| None (Mobile Exclusive) | Mobile-Only | **Preserve completely** (interactive concierge AI modal) |

---

## 4. Critical Findings & Discrepancies

### 4.1 Supabase Project Discrepancy (Mandatory Audit Requirement 12)

During the repository audit, two separate Supabase configurations were discovered:

1. **Website Authoritative Database Configuration:**
   - **Connection URL:** `postgresql://postgres.cpeuatsblwxwgykhixos:***@aws-0-ap-northeast-2.pooler.supabase.com:6543/postgres?pgbouncer=true`
   - **Authoritative Supabase Project Reference:** `cpeuatsblwxwgykhixos`
   - **Database Host:** `aws-0-ap-northeast-2.pooler.supabase.com`
   - **Contents:** Contains the live `User` (5 users), `Admin` (4 admins), `Receptionist` (2 receptionists), `Hotel` (9 hotels), `RoomCategory` (16 categories), and `Booking` (3 bookings) tables.

2. **Mobile Environment Configuration:**
   - **File:** `M2N/.env` and `M2N/.env.local`
   - **Current Value:** `EXPO_PUBLIC_SUPABASE_URL=https://pjtixijqdhpdofzvxyit.supabase.co`
   - **Analysis:** `pjtixijqdhpdofzvxyit` was created as an isolated mobile Supabase instance that does not contain the website's live hotels, bookings, or receptionist assignments.
   - **Correction Plan:** The mobile API service layer and environment configuration must target the website's authoritative backend endpoints (which query `cpeuatsblwxwgykhixos` directly). For Supabase client functionality, point to project `cpeuatsblwxwgykhixos`.

### 4.2 Missing Mobile Store Modules

`AdminScreen.js`, `ReceptionScreen.js`, and `LoginScreen.js` import the following modules from `M2N/src/data/`:
- `backendStore.js`
- `adminAuthStore.js`
- `authStore.js`
- `wishlistStore.js`
- `searchStore.js`
- `recentSearchesStore.js`

In `M2N/src/data/`, only `siteData.js` and `tripsStore.js` were present on disk. The other store modules were missing, causing runtime import errors in Metro/Expo. 

**Resolution:** Implement resilient, reactive store modules in `M2N/src/data/` that:
1. Connect directly to the website's Express APIs (`user/backend`, `admin/backend`, `reception/backend`).
2. Maintain subscription pub-sub mechanisms (`subscribe()`, `notify()`) required by existing components.
3. Fallback gracefully to offline `siteData.js` defaults if network connectivity is lost.

### 4.3 ImageKit vs. Cloudinary Discrepancy

`M2N/src/services/imagekitService.js` used ImageKit with client-side keys. 
**Resolution:** Deprecate and replace ImageKit calls with the website's Cloudinary-backed API endpoints.

### 4.4 Google Drive Upload Endpoint

`M2N/src/services/googleDriveService.js` attempted to call `POST /bookings/upload-aadhaar`, which did not exist on `reception/backend`. 
**Resolution:** Use the existing, production-tested `reception/backend` routes:
- `POST /api/bookings/:id/checkin` (Multipart form-data: `validId`, fields: `roomId`, `categoryId`)
- `POST /api/bookings/walkin` (Multipart form-data: `validId`, fields: `hotelId`, `roomCategoryId`, `checkIn`, `checkOut`, guest details)

Both endpoints invoke `uploadToDrive()` in `reception/backend/src/utils/googleDrive.ts` and populate `Booking.validIdUrl`.

---

## 5. Mobile API Layer Architecture

To keep the mobile application modular and strictly aligned with the website backends, a unified API service client layer is established in `M2N/src/services/api/`:

```text
M2N/src/services/api/
├── apiClient.js          <-- Central Axios/Fetch instance, base URLs, token interceptors
├── authService.js        <-- User login/register, Admin login (2FA), Receptionist login
├── hotelService.js       <-- Public hotels, hotel details, room categories
├── bookingService.js     <-- Availability checks, create booking, my bookings, cancel
├── adminService.js       <-- Admin dashboard metrics, hotels CRUD, categories, bookings
├── receptionService.js   <-- Reception bookings, check-in with ID, walk-in with ID
└── storageService.js     <-- Multi-platform token & session persistence
```

### Base URL Resolution

```javascript
// Automatically handles Web (localhost) vs Android Emulator (10.0.2.2) vs Physical Device (LAN IP / Production)
const getApiBase = (port) => {
  if (process.env[`EXPO_PUBLIC_API_URL_${port}`]) {
    return process.env[`EXPO_PUBLIC_API_URL_${port}`];
  }
  if (Platform.OS === 'android') {
    return `http://10.0.2.2:${port}/api`;
  }
  return `http://localhost:${port}/api`;
};

export const API_ENDPOINTS = {
  USER: getApiBase(4000),      // user/backend
  ADMIN: getApiBase(5000),     // admin/backend
  RECEPTION: getApiBase(6001), // reception/backend
};
```

---

## 6. Implementation Sequence & Milestones

1. **Phase 1: Discovery & Audit (Complete)**
   - Audit all 4 codebases (`user`, `admin`, `reception`, `M2N`).
   - Document database models, APIs, and credentials.
   - Establish `docs/integration-architecture.md`.

2. **Phase 2: Mobile API Layer & Store Foundation**
   - Implement `M2N/src/services/api/` modules.
   - Reconstruct and enhance `authStore.js`, `adminAuthStore.js`, `backendStore.js`, `wishlistStore.js`, `searchStore.js`.
   - Ensure full backwards-compatibility with `AdminScreen.js`, `ReceptionScreen.js`, `LoginScreen.js`, etc.

3. **Phase 3: Customer Portal Integration**
   - Connect `LoginScreen.js` to `/api/auth/login` and `/api/auth/register`.
   - Connect `HomeScreen.js`, `HotelsScreen.js`, `RoomsScreen.js` to `/api/public/*`.
   - Connect `BookingScreen.js` and `PaymentScreen.js` to `/api/bookings/create`.
   - Connect `MyTripsScreen.js` to `/api/bookings/my-bookings`.

4. **Phase 4: Admin & Receptionist Dashboard Integration**
   - Connect `AdminLoginScreen.js` to `/api/auth/login` (Port 5000) with 2FA handling.
   - Connect `AdminScreen.js` tabs to live backend CRUD endpoints.
   - Connect `ReceptionScreen.js` to `reception/backend` (Port 6001), binding receptionist `hotelId`.
   - Wire Check-in and Walk-in ID uploads to the existing Google Drive workflow.

5. **Phase 5: Cloud Storage & Image Cleanup**
   - Ensure hotel and room image uploads route to Cloudinary via backend Multer.
   - Replace direct ImageKit usage with Cloudinary URL normalization and backend upload.

6. **Phase 6: Verification & Cross-Platform Testing**
   - Test User signup/login and shared booking creation.
   - Test Receptionist check-in with Google Drive ID document link verification.
   - Test Admin hotel catalog updates and reflection in mobile app.
   - Ensure all mobile-only luxury features remain operational without errors.
