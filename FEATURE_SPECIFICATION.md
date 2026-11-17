# CM Pay Test Shop — Feature Specification & Handoff

This document details the functional specifications, architectural requirements, and integration patterns built for the **CM Pay Test Shop**. It serves as the reference guide for developers building and deploying the next-generation version within the organization's hosting platform.

---

## 1. Purpose & Core Objective

A lightweight, zero-dependency payment gateway testing workbench built to demonstrate, validate, and QA card payment flows against the **CM.com Online Payments Platform** (both **Acceptance** and **Production** environments) without handling raw PAN or PCI-scoped data on the application server.

---

## 2. Core Payment Integration Flows

### 2.1 Hosted Checkout Flow (`POST /checkout`)
* **Authentication**: Requests an OAuth 2.0 bearer token using client credentials (`POST /v1/authorization/oauth2/token`).
* **Transaction Creation**: Dispatches `POST /v1/paymentmethods/checkout/v1/transactions` with:
  * `amount`: Transaction amount represented as an integer in cents.
  * `currency`: Transaction currency (`EUR`).
  * `merchantOrderReference`: Alphanumeric reference string (1–35 characters).
  * `description`: Line item description.
  * `paymentMethods`: Enabled payment method array (e.g. `["creditcard"]`).
  * `returnUrl`: Redirect destination following customer completion.
  * `webhooks`: Event notifications configuration.
* **Redirection**: Redirects shopper's browser to CM.com's hosted checkout page (`action.redirect.url`).
* **PCI Scope**: Cardholder data never touches the merchant application server.

### 2.2 Direct Hosted Card / Iframe Flow (`/direct`, `POST /direct-checkout`)
* **Initialization**: Dispatches `POST /v1/paymentmethods/creditcard/v2/transactions/initialize` to retrieve an `action.redirect.url`.
* **Embedded Experience**: Renders the CM.com hosted card entry form within an embedded `iframe` on the host site, providing a native checkout feel while keeping the application out of PCI scope.
* **Frame Breakout**: Implements automated parent-window breakout upon callback redirection to `/return`.

### 2.3 Product Catalog & Dynamic Pricing
* **Test Coin**: Fixed price of `€0.02` (the minimum amount accepted by card acquiring rails).
* **Custom Amount**: User-defined amount (`€0.02` – `€1,000.00`) to test amount-specific card rules, high-ticket approvals, and decline limits.
* **Validation**: Strict server-side integer cents parsing and bounds enforcement.

### 2.4 Custom Merchant Reference Testing (`/merchant`)
* Supports custom alphanumeric reference generation and manual entry (1–35 characters) to verify downstream reconciliation across acquiring channels and cardholder statements.

---

## 3. Dual-Environment Architecture (Acceptance & Production)

* **Global Environment Toggle**: Instant runtime switching between **Acceptance** (`https://gateway.acceptance.cmpayments.nl`) and **Production** (`https://api.pay.cm.com`).
* **Visual Safeguards**: Distinct thematic accent coloring per environment to eliminate the risk of accidental production charges:
  * **Production**: Vibrant Orange (`#ff5f1f`)
  * **Acceptance**: Cobalt Blue (`#2f7dff` / `#2563eb`)
* **State Isolation**: Orders retain the environment in which they were created. Status queries always resolve against the originating environment's endpoint regardless of the currently active toggle.

---

## 4. Verification, Webhooks & Order Management

### 4.1 Zero-Trust Status Verification (`/return`)
* The return page does not rely on URL query parameters to determine payment success.
* Re-fetches the authoritative transaction record directly from CM.com API (`GET /transactions/{id}`).
* Maps CM.com states (`SUCCESS`, `PENDING`, `AUTHORIZED`, `CANCELLED`, `EXPIRED`, `FAILURE`) to application order states.

### 4.2 Asynchronous Webhooks (`/webhook`)
* Automatically registered when running under a public HTTPS host URL.
* Ingests `FINALSTATUS` and `STATUS_CHANGE` webhooks.
* Uses the transaction ID payload to trigger an authoritative GET request back to CM.com before persisting state updates.

### 4.3 Orders Dashboard (`/orders`)
* Live table logging all test orders with:
  * Order ID & Merchant Reference
  * Timestamp
  * Amount & Currency
  * Payment Status badge
  * Environment badge (`PROD` vs `ACC`)
  * Transaction ID linked to CM.com
  * Manual status re-synchronization triggers

---

## 5. In-App QA & Developer Productivity Tools

### 5.1 Slide-out QA Test Card Drawer
Embedded slide-out drawer providing pre-configured test card presets for CM.com 3-D Secure v2 test flows:
* **Card Schemes**:
  * Visa & VPay (Prefix `4111 1111 1111`)
  * Mastercard (Prefix `5555 5555 5555`)
  * American Express (Prefix `3411 1111 111`)
  * Bancontact & Maestro (Prefix `6703 9999 8800`)
* **Simulated Test Scenarios**:
  * Standard (3DS Challenge Required)
  * Frictionless Flow (No Auth Required — Code 51)
  * Soft-Declines (1st Attempt — Code 52, Persistent — Code 53)
  * 3DSv2 Not Enrolled / Reason Codes (Code 82 for Mastercard, Code 13 for Visa/Amex)
  * Authentication Errors & Issuer Rejections (Codes 56, 57, 59)
  * 3RI Decoupled Authentication (Success / Failure with simulated 5-second delays)
* **Productivity Utilities**: One-click clipboard copy buttons for PAN, Expiry (`12/30`), CVV (`123` / `1234`), Cardholder name, or full credential string (`PAN | EXP | CVV`).

---

## 6. Configuration & Credentials Management (`/config`)

* **In-App Management UI**: Clean configuration cards to manage credentials per environment (Client ID, Client Secret, Base URL, Enabled Payment Methods).
* **Connection Health Check**: Interactive "Test Connection" trigger verifying OAuth token exchange against CM.com before saving.
* **Precedence Hierarchy**: Configuration stored in persistent state takes precedence over fallback environment variables (`CM_PAY_*` and `CM_PAY_ACCEPTANCE_*`). Client secrets are masked in the UI.

---

## 7. Implementation Guidelines for Organization Hosting Platform

1. **Service Architecture**:
   * Implement as a stateless containerized microservice (Node.js/TypeScript, Go, or Python).
   * Follow 12-factor configuration patterns using organization secret managers (e.g. HashiCorp Vault, AWS Secrets Manager, or Kubernetes Secrets).
2. **Ingress & Networking**:
   * Must be deployed behind a public HTTPS ingress / reverse proxy (e.g. Nginx, Traefik, Cloudflare) with valid SSL certificates to receive CM.com webhook calls and process return redirects.
3. **Identity & Access Control**:
   * Secure all endpoints (specifically `/config` and `/orders`) behind the organization's Single Sign-On (SSO) / Identity-Aware Proxy (e.g. Okta, Azure AD, Cloudflare Access, or Google IAP).
4. **Persistence Layer**:
   * Replace the local flat-file store (`orders.json`) with an enterprise-managed persistence tier (e.g. PostgreSQL, Redis, or DynamoDB).
5. **Observability & Logging**:
   * Export structured JSON logs and metrics (Prometheus/OpenTelemetry) for OAuth latency, checkout initiation failures, and webhook delivery status.
