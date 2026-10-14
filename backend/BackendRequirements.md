# PhonePark BD — Backend Requirements (Lite)

**Stack:** NestJS + MongoDB (Mongoose) + JWT · **Frontend:** Next.js 16 / React 19 (already built on mock data)

## 1. Goals
- Serve every public page: phone catalog, phone detail, compare, finder, search, rankings.
- Serve the role-based dashboard (user, moderator, admin) with real data and server-side permission checks.
- Enforce roles on the server; the frontend role guard is only a UX convenience.

## 2. Roles and Permissions
| Dashboard tab | User | Moderator | Admin |
|---|---|---|---|
| Overview, Profile | Yes | Yes | Yes |
| My wishlist, My price alerts, My reviews | Yes | - | - |
| Users (ban / unban) | - | Yes | Yes |
| Visitors, Analytics, Search insights | - | Yes | Yes |
| Products (phones) | - | Yes | Yes |
| Reviews (moderation) | - | Yes | Yes |
| Price updates | - | Yes | Yes |
| Brands and categories | - | Yes | Yes |
| Moderators (add / suspend / remove) | - | - | Yes |
| Site settings, Audit log | - | - | Yes |

## 3. Authentication and Account
| Method | Path | Access | Description |
|---|---|---|---|
| POST | `/auth/register` | Public | name, email, password, acceptTerms. Role `user`. |
| POST | `/auth/login` | Public | email, password, rememberMe. Rejects banned/suspended. Rate limited. |
| POST | `/auth/refresh` | Cookie | Rotates refresh token, issues new access token. |
| POST | `/auth/logout` | Auth | Invalidates refresh token, clears cookie. |
| POST | `/auth/forgot-password` | Public | Sends time-limited reset link; always returns 200. |
| POST | `/auth/reset-password` | Public | token, newPassword; invalidates all refresh tokens. |
| GET | `/auth/me` | Auth | id, name, email, phone, avatarUrl, role, status. |
| PATCH | `/users/me` | Auth | Update name, email (unique), phone. |
| PATCH | `/users/me/password` | Auth | currentPassword, newPassword (min 8). |
| POST | `/users/me/avatar` | Auth | Image only, max 5 MB. Returns avatarUrl. |
| DELETE | `/users/me/avatar` | Auth | Remove photo. |
| DELETE | `/users/me` | Auth | Soft delete (see below). |

**Security**
- Password min 8 chars, hashed, never returned.
- Access token ~15 min; refresh ~7 days (30 with "remember me"); refresh tokens stored hashed and rotated on use.
- httpOnly + Secure + SameSite cookie; CORS limited to frontend origin(s).
- Rate limit login/register/forgot-password (`@nestjs/throttler`, e.g. 5 per minute per IP + email).
- First admin is created by seed script only.

**Delete account:** soft delete (status `deleted`, personal fields anonymised), remove wishlist and alerts, keep reviews anonymised as "Deleted user". The last admin cannot delete themselves.

### 4.1 Phones (catalog)
| Method | Path | Access | Description |
|---|---|---|---|
| GET | `/phones` | Public | Filters: q, brand, minPrice, maxPrice, usage, features[], isOfficial, marketStatus, ram, storage; sort (price, overallScore, views, newest); published only. |
| GET | `/phones/:slug` | Public | Full detail (specs, variants, scores, pros/cons, aiSummary). Increments views (debounced per visitor). |
| GET | `/phones/trending` | Public | Top phones by recent views. |
| GET | `/phones/rankings` | Public | `category`, `maxPrice`, `minPrice`, `brand`; sorted by overallScore. |
| GET | `/phones/:slug/price-history` | Public | Price points over time. |
| POST | `/phones` | Mod+ | Create (default `draft`). |
| PATCH | `/phones/:id` | Mod+ | Update; price change writes history and runs alert check. |
| PATCH | `/phones/:id/status` | Mod+ | Publish / unpublish (audited). |
| DELETE | `/phones/:id` | Mod+ | Soft delete. |
| GET | `/dashboard/phones` | Mod+ | Products table: name, brand, price, status, author, updated. |

**Requirements**
- `slug` unique and URL-safe; changing it must not break compare links (keep slug history or redirect).
- `overallScore` and the five sub-scores (performance, camera, battery, display, value) are 0-10 with one decimal.
- A phone has a brand reference and one or more variants (label, price, isOfficial).
- Indexes: slug (unique), brand, price, overallScore, text index on name/brand/model.

### 4.2 Brands and categories
| Method | Path | Access | Description |
|---|---|---|---|
| GET | `/brands` | Public | Visible brands with phoneCount. |
| GET | `/brands/:slug/phones` | Public | Phones of one brand. |
| GET | `/dashboard/brands` | Mod+ | All brands incl. hidden. |
| POST | `/brands` | Mod+ | Create. |
| PATCH | `/brands/:id` | Mod+ | Rename; toggle visible/hidden. |
| DELETE | `/brands/:id` | Mod+ | Blocked while phones exist (hide instead). |

### 4.3 Compare
| Method | Path | Access | Description |
|---|---|---|---|
| GET | `/compare?slugs=a,b,c` | Public | 2-4 published slugs. Returns `phones[]`, `categoryWinners[]`, `recommendation`. |
| GET | `/compare/popular` | Public | Popular pairs shown on `/compare` (editorial list first). |

### 4.4 AI Phone Finder
| Method | Path | Access | Description |
|---|---|---|---|
| GET | `/finder/options` | Public | Budget ranges, usage options, brands, features. |
| POST | `/finder/match` | Public | `{ budgetId, usage, brands[], features[] }` → phones with `matchPercent`, sorted desc, threshold applied. |

### 4.5 Search
| Method | Path | Access | Description |
|---|---|---|---|
| GET | `/search?q=` | Public | Natural-language-ish search ("gaming phone under 30000"). |
| GET | `/search/suggestions?q=` | Public | Autocomplete, top 5-8. |

Every search is logged (query, resultCount, timestamp, visitorId); zero-result searches feed Search insights.

### 4.6 Reviews and ratings
| Method | Path | Access | Description |
|---|---|---|---|
| GET | `/phones/:slug/reviews` | Public | Approved reviews + summary (avg, count, 1-5 distribution). |
| POST | `/phones/:slug/reviews` | Auth | rating 1-5, body. Created `pending`; one review per user per phone. |
| GET | `/reviews/mine` | Auth | Own reviews, any status. |
| PATCH | `/reviews/:id` | Auth | Edit own review; returns to `pending`. |
| DELETE | `/reviews/:id` | Auth | Delete own review. |
| GET | `/dashboard/reviews` | Mod+ | Moderation queue, filter by status. |
| PATCH | `/reviews/:id/moderate` | Mod+ | `{ status: approved \| rejected, reason? }`. Audited. |

- `ratingAverage` / `ratingCount` recomputed on every status change or delete.
- Banned users' reviews are hidden via a derived flag (`hiddenByBan`), not deleted; visible again on unban.

### 4.7 Wishlist
| Method | Path | Access | Description |
|---|---|---|---|
| GET | `/wishlist` | Auth | Own wishlist with current price and change since added. |
| PUT | `/wishlist/:slug` | Auth | Add (idempotent); stores `priceAtAdd`. |
| DELETE | `/wishlist/:slug` | Auth | Remove. |
| POST | `/wishlist/merge` | Auth | `{ slugs[] }` merges the anonymous localStorage list on login. |

### 4.8 Price alerts
| Method | Path | Access | Description |
|---|---|---|---|
| GET | `/price-alerts` | Auth | Own alerts: phone, targetPrice, currentPrice, status. |
| POST | `/price-alerts` | Auth | phoneSlug, targetPrice. Limit e.g. 20 active per user. |
| DELETE | `/price-alerts/:id` | Auth | Delete own alert. |
| GET | `/price-alerts/recent-drops` | Public | Recent price drops for the CTA widget. |

### 4.9 Price history and price updates (moderator workflow)
| Method | Path | Access | Description |
|---|---|---|---|
| GET | `/dashboard/price-updates` | Mod+ | Stale phones: name, brand, currentPrice, lastUpdatedAt, daysStale. |
| PATCH | `/phones/:id/price` | Mod+ | New price (and optional variant prices); writes history, updates `priceUpdatedAt`, runs alert check. |
| POST | `/phones/:id/price/verify` | Mod+ | "Mark price as updated" without changing value. Audited. |

A price is stale after N days (configurable, default 30).

### 4.10 User management
| Method | Path | Access | Description |
|---|---|---|---|
| GET | `/dashboard/users` | Mod+ | Search by name/email; filter All / Active / Banned. |
| PATCH | `/users/:id/ban` | Mod+ | Ban with reason; moderators may only ban role `user`. Audited. |
| PATCH | `/users/:id/unban` | Mod+ | Unban. |

### 4.11 Moderator management
| Method | Path | Access | Description |
|---|---|---|---|
| GET | `/dashboard/moderators` | Admin | name, email, assignedSince, status. |
| POST | `/moderators` | Admin | Promote an existing user by email (or invite). |
| PATCH | `/moderators/:id/suspend` | Admin | Toggle suspended/active. |
| DELETE | `/moderators/:id` | Admin | Demote to role `user`. |

### 4.12 Analytics, visitors and search insights
| Method | Path | Access | Description |
|---|---|---|---|
| POST | `/analytics/visit` | Public | path, referrer, anonymous visitorId, device. Lightweight, rate limited. |
| GET | `/dashboard/overview` | Auth | Role-aware stats (user: own counts; mod/admin: site totals + recent activity). |
| GET | `/dashboard/analytics` | Mod+ | Visitors, page views, top pages/phones/brands over time. |
| GET | `/dashboard/visitors` | Mod+ | Trend, unique vs returning, device split, referrers. |
| GET | `/dashboard/search-insights` | Mod+ | Top searches and zero-result searches. |
| DELETE | `/dashboard/search-insights/zeroresults/:id` | Mod+ | Clear a zero-result search log. |

- Raw events in a TTL collection; dashboards read pre-aggregated daily rollups (scheduled job).
- No raw IP stored long-term (hash or truncate).
- The `/admin/dashboard` widgets ("phones requiring price updates", "zero-result searches") reuse 5.9 and 5.12 endpoints; no extra module.

### 4.13 Audit log
| Method | Path | Access | Description |
|---|---|---|---|
| GET | `/dashboard/audit-log` | Admin | Filter by actor, action, date range; paginated. |

Written automatically (no public write endpoint). Record at least: publish, unpublish, ban, unban, override, role_change, settings_change, review_moderated, content_deleted.

### 4.14 Site settings
| Method | Path | Access | Description |
|---|---|---|---|
| GET | `/settings/public` | Public | siteName, contactEmail, currency, maintenanceMode, adsEnabled, SEO title/description. |
| GET | `/dashboard/settings` | Admin | Full settings document. |
| PATCH | `/dashboard/settings` | Admin | Update. Audited as `settings_change`. |

Stored as a singleton (also holds `staleDays`).

### 4.15 File uploads
| Method | Path | Access | Description |
|---|---|---|---|
| POST | `/uploads/image` | Auth | jpeg/png/webp, max 5 MB. Only Mod+ may upload phone images. |

## 6. Data Model (MongoDB)
All collections use `timestamps: true`. Use ObjectId references; embed only data always read together.

**users:** name, email (unique, lowercase), phone, passwordHash (`select: false`), role (`user|moderator|admin`), status (`active|banned|suspended|deleted`), avatarUrl, moderatorSince, banReason, tokenVersion / refreshTokens (hashed).

**phones:** slug (unique), name, model, brand → brands, price, isOfficial, marketStatus (`available|upcoming|discontinued`), releaseDate, status (`draft|published`), author → users, overallScore, scores `{performance, camera, battery, display, value}`, specs (compact), keySpecs, fullSpecs (`[{title, category, items:[{label,value}]}]`), variants `[{label, price, isOfficial}]`, colors, images/imageUrl, pros, cons, aiSummary `{shortDescription, keyStrengths[], potentialLimitations[]}`, tags (gaming|camera|battery|office|social), features (5g, nfc, amoled, …), views, ratingAverage, ratingCount, priceUpdatedAt.

**brands:** name (unique), slug (unique), status (`visible|hidden`), logoUrl, phoneCount.

**reviews:** user + phone (unique together), rating 1-5, body (max 2000), status (`pending|approved|rejected`), moderatedBy, moderatedAt, reason, hiddenByBan.

**wishlists:** user, phone (unique together), priceAtAdd.

**priceAlerts:** user, phone, targetPrice, status (`waiting|triggered`), triggeredAt, notifiedAt.

**priceHistory:** phone, price, variantLabel, source (`manual|import`), recordedBy, recordedAt.

**searchLogs:** query, normalizedQuery, resultCount, visitorId, createdAt (index on normalizedQuery + resultCount).
**pageVisits:** path, phoneSlug?, visitorId, referrer, device, createdAt (TTL ~90 days) + daily rollup collection.
**auditLogs:** actor, actorRole, action, targetType, targetId/label, meta, createdAt (append-only).
**settings (singleton):** siteName, contactEmail, currency, maintenanceMode, adsEnabled, seoTitle, seoDescription, staleDays.

## 7. Business Rules
**7.1 Scores.** Sub-scores are set by moderators; `overallScore` is a weighted average (equal weights until the product owner decides), 0-10 with one decimal.

**7.2 Compare.** For Gaming (performance), Camera, Battery, Display, the highest sub-score wins (ties return all tied slugs). Recommendation = phone with most category wins; tie-break by `overallScore`; reason text lists won categories and the value score. Phones without scores are ignored.

**7.3 Finder matching (100 points).**

| Factor | Points | Rule |
|---|---|---|
| Budget | 35 | Full inside range; reduced linearly outside (tolerance = range width, or 15,000 for open-ended top range). No answer: 18. |
| Usage | 30 | 30 if phone tags include chosen usage, else 5. No answer: 13. |
| Brand | 10 | 10 if no brand chosen or brand matches, else 0. |
| Features | 15 | (matched / selected) × 15. None selected: 8. |
| Quality | 10 | overallScore / 10 × 10. |

Clamp to 100, round, drop results below 35, sort descending.

**7.4 Search parsing.** Lowercase; extract price ceiling (`30k`, `30,000`, `30000`); map usage keyword (gaming, camera, battery, office, study, social…) to a tag; remove filler words (under, below, best, good, top, taka, phone, for); match the rest against brand/model/name. Score = text relevance + usage match + within-budget bonus. Log query and result count.

**7.5 Price alert triggering.** On every price change, all `waiting` alerts for that phone with `targetPrice >= newPrice` become `triggered` (set `triggeredAt`), notify the user (email + in-app), at most once per alert.

**7.6 Ban / suspend / delete.**
- Ban: cannot log in; reviews hidden and excluded from aggregates; unban reverses it.
- Suspend moderator: loses dashboard access, account intact.
- Remove moderator: role reverts to `user`.
- Delete brand: blocked while phones exist.
- Delete account: see section 4.