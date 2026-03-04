# ManghiPay

---

## System Overview

**1. Can you walk us through the overall architecture of ManghiPay and how the components interact with each other?**

ManghiPay follows the ASP.NET Core MVC pattern. The browser sends requests to Controllers, which call DatabaseHelper for data access, then return Razor Views to the client. Xendit handles payment processing through a webhook callback that updates payment status. SignalR is used for real-time updates on the admin dashboard. All data is persisted in SQL Server.

---

**2. Why did you choose ASP.NET Core MVC over other frameworks like Laravel or Django?**

ASP.NET Core MVC is well-suited for building structured, server-rendered web applications in C#. It provides built-in session management, routing, and Razor view engine out of the box. Since the system is deployed in a Windows/SQL Server environment, .NET integrates naturally. Laravel and Django are also valid choices but would require a different language stack.

---

**3. Why did you use raw ADO.NET instead of an ORM like Entity Framework?**

Raw ADO.NET gives full control over SQL queries, which is important for performance-sensitive operations like payment processing. Entity Framework adds abstraction overhead and can generate inefficient queries. Since the database schema is relatively simple and well-defined, raw ADO.NET was a practical and lightweight choice. The trade-off is more verbose code, but it is easier to debug and optimize.

---

## Database & Data Design

**4. Why does `PaymentItems` store `FeeTitle` and `FeePrice` directly instead of just relying on the `Fees` table?**

This is called a snapshot pattern. At the time of payment, the fee name and price are copied directly into `PaymentItems`. This means if an admin edits or deletes a fee later, the historical payment record remains accurate. If we only stored `FeeId`, editing the fee would silently change all past transaction records, which is incorrect behavior for a financial system.

---

**5. What happens to a student's payment history when a fee is deleted?**

When a fee is deleted, `DeleteFeeAsync` sets `FeeId = NULL` on all related `PaymentItems` rows instead of deleting them. Since `FeeTitle` and `FeePrice` are stored directly on `PaymentItems`, the history is fully preserved and still visible to students. Only the foreign key reference is removed.

---

**6. How does your transaction ID generation handle race conditions if two students pay at the exact same time?**

Currently it uses `MAX(TransactionId)` to get the next sequence number, which could produce duplicate IDs if two payments are created simultaneously. This is a known limitation. A proper fix would be to use a SQL `SEQUENCE` object or a database-generated identity column for the sequence number, wrapped in a transaction with appropriate isolation level.

---

**7. Why is the school year stored in a separate `SystemSettings` table instead of just a config file?**

Storing it in the database allows the admin to update the school year at runtime through the Settings page without needing to redeploy the application or modify config files. It also gets cached in `IMemoryCache` for performance, so repeated reads don't hit the database every time.

---

## Authentication & Security

**8. How does your session-based authentication differ from JWT? What are the trade-offs?**

Session-based authentication stores user state on the server. The browser holds only a session cookie that references server-side data. JWT is stateless — the token itself contains user claims and is verified without server storage. Sessions are simpler for server-rendered web apps but don't scale as easily across multiple servers. JWT is better suited for APIs and mobile clients. For a school web system like ManghiPay, sessions are the appropriate choice.

---

**9. What is BCrypt and why is it better than storing plain text passwords?**

BCrypt is a password hashing algorithm designed to be slow and computationally expensive, making brute-force attacks impractical. Plain text passwords are immediately exposed if the database is compromised. BCrypt generates a salted hash so even identical passwords produce different hashes, preventing rainbow table attacks. The work factor can also be increased over time to keep up with faster hardware.

---

**10. Your system has a legacy plain text password fallback — isn't that a security risk?**

Yes, it is a temporary measure to avoid breaking existing admin accounts that were created before BCrypt was implemented. The fallback checks if the stored password starts with `$2`, which is the BCrypt hash prefix. If it does not, it falls back to plain text comparison. The recommended next step is to migrate all existing plain text passwords to BCrypt hashes and remove the fallback entirely.

---

**11. What does your rate limiter store in memory? What happens to it when the server restarts?**

The rate limiter uses a `ConcurrentDictionary` stored in application memory, keyed by IP address and email. It tracks attempt count and the window start time. When the server restarts, all rate limit data is lost and counters reset to zero. For a production system, a distributed cache like Redis would be used to persist rate limit state across restarts and multiple server instances.

---

**12. You mentioned CSRF protection is not configured — how would you add it and why is it important?**

CSRF (Cross-Site Request Forgery) tricks a logged-in user's browser into making unintended requests to your application. In ASP.NET Core, protection is added by registering `AddAntiforgery()` in `Program.cs`, adding `@Html.AntiForgeryToken()` to forms, and applying `[ValidateAntiForgeryToken]` to POST actions. It was not yet implemented in this version but would be a priority before production deployment.

---

## Payment Processing

**13. What is Xendit and how does it integrate with your system?**

Xendit is a payment gateway that handles online transactions. In ManghiPay, when a student submits a payment, the `PayController` creates a payment record with a `Pending` status, then calls the Xendit API to create an invoice. Xendit sends the student to a hosted payment page. After payment, Xendit sends a webhook callback to our system, which updates the payment status to `Paid` and triggers a SignalR notification.

---

**14. What happens if a student pays but the Xendit webhook never arrives — will the payment stay as pending forever?**

Yes, in the current implementation the payment would remain `Pending` indefinitely if the webhook fails. A proper fix is to implement a polling mechanism or a background job that periodically checks Xendit's API for the status of pending payments and updates them accordingly.

---

**15. How do you prevent a student from paying for the same fee twice?**

Currently the system does not explicitly block duplicate payments for the same fee. This is a known gap. The recommended fix is to check if a `Paid` payment already exists for the same `UserId` and `FeeId` before allowing a new payment to be created.

---

## OTP & Registration

**16. Where is the OTP stored and for how long is it valid?**

The OTP is generated and managed by the `OtpService`. It is stored temporarily — either in memory or a database table depending on the implementation — and has an expiry window. The exact duration depends on the `OtpService` configuration, typically 5 to 10 minutes.

---

**17. What happens if a student closes the browser after sending the OTP but before verifying it?**

The registration data is stored in the ASP.NET Core session. If the browser is closed and the session expires, the session data is lost and the student would need to start registration again. The OTP itself would also expire on its own after the validity window.

---

**18. Why did you choose email OTP over SMS verification?**

Email OTP is free to implement using an SMTP service, while SMS verification requires a paid provider like Twilio. For a school system with limited budget, email OTP is practical and sufficient. Students already need a valid email to register, so it serves as both verification and a communication channel.

---

## Report & Analytics

**19. Your per-item report filters by `FeeTitle` as a string — what happens if two fees have the same name?**

Both fees would appear together in the filtered results since the filter matches by title string. This could cause confusion. A better approach would be to filter by `FeeId` instead of `FeeTitle`, but since `FeeId` is set to NULL after deletion, the more robust solution would be to store a `FeeId` snapshot at payment time in a non-nullable column alongside the title.

---

**20. Can the report be exported to Excel or PDF? If not, how would you implement that?**

Not yet — currently only browser print is supported. To export to Excel, we could use the `ClosedXML` NuGet package to generate `.xlsx` files from the report data. For PDF export, `iTextSharp` or `DinkToPdf` could render the report to a downloadable PDF. Both would be added as new controller actions that return a `FileResult`.

---

**21. The revenue analytics are computed in C# after fetching all records — how would this scale with thousands of payments?**

It would not scale well. Fetching all payment records into memory just to sum them in C# is inefficient at large scale. The correct approach is to push the aggregation to SQL using `SUM()`, `GROUP BY`, and date filters directly in the query, so only the final totals are returned rather than all raw records.

---

## General

**22. What is SignalR and what specifically does it update in real time in your system?**

SignalR is a library that enables real-time bidirectional communication between the server and browser using WebSockets with fallbacks. In ManghiPay, it is used to push payment status updates to the admin dashboard in real time when a Xendit webhook is received, so the admin sees payments change from `Pending` to `Paid` without needing to refresh the page.

---

**23. If you were to improve this system further, what would be your top three priorities?**

1. **CSRF protection** — add antiforgery tokens to all POST forms to prevent cross-site request forgery attacks.
2. **Duplicate payment prevention** — check for existing paid records before allowing a new payment for the same fee.
3. **Export to Excel/PDF** — allow admins to download reports for record-keeping and submission to school administration.

---

**24. How would you handle a scenario where the school has multiple campuses with different admins?**

The current system has a single admin role. To support multiple campuses, a `CampusId` column would be added to `Users`, `Fees`, and `Payments` tables. Admin accounts would be scoped to their campus, so each admin only sees and manages data belonging to their campus. A super-admin role could be added for system-wide oversight across all campuses.

---
