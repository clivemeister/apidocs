<h1 id="admin-api">Admin API</h1>

The Admin API provides programmatic access to admin operations.
It mirrors the structure of the public API but allows admin users to operate on **any user's** goals, datapoints, charges, and account settings.

This is intended for internal tools and automation of support tasks.

### API Base URL

The base URL for all admin API requests is `https://www.beeminder.com/api/admin/`.

### Authentication

> Authenticate with your admin auth_token:

```shell
  curl https://www.beeminder.com/api/admin/users/alice/goals.json?auth_token=ADMIN_TOKEN
```

> Non-admin tokens get a 403 Forbidden:

```json
  { "errors": {
      "token": "bad_token",
      "message": "The user associated with this token is not an admin." }}
```

All Admin API endpoints require an `auth_token` (or OAuth `access_token`) belonging to an admin user.
Authentication works the same as the public API &mdash; append your token as a query or POST parameter, or use the `Authorization: Bearer` header.

Unlike the public API, there is no `me` shortcut.
You must always specify the target username explicitly.

<aside class="warning">
These endpoints are admin-only. Non-admin tokens will receive a 403 Forbidden response.
</aside>

[Back to top](#)

<h2 id="admin-getgoals">Get all goals for a user</h2>

```shell
  curl https://www.beeminder.com/api/admin/users/alice/goals.json?auth_token=ADMIN_TOKEN
```

```json
  [ { "slug": "weight",
      "title": "Weight Loss",
      "goal_type": "fatloser",
      "losedate": 1711584000,
      "pledge": 10,
      "rate": -0.1,
      "..." : "..." },
    { "slug": "pushups", "..." : "..." } ]
```

### HTTP Request

`GET /admin/users/`*u*`/goals.json`

Returns all active (non-archived, non-deleted) goals for user *u*, sorted by urgency.
Each goal object has the same shape as the public API's [Goal](#goal) response.

<h2 id="admin-getgoal">Get a single goal</h2>

```shell
  curl https://www.beeminder.com/api/admin/users/alice/goals/weight.json?auth_token=ADMIN_TOKEN
```

```json
  { "slug": "weight",
    "title": "Weight Loss",
    "goal_type": "fatloser",
    "goalval": 150,
    "rate": -0.1,
    "goaldate": 1743465600,
    "pledge": 10,
    "contract": { "amount": 10,
                  "stepdown_at": null,
                  "pending_amount": null,
                  "pending_at": null },
    "roadall": [[1609459200, 180, null], [1743465600, null, -0.1]],
    "..." : "..." }
```

### HTTP Request

`GET /admin/users/`*u*`/goals/`*g*`.json`

Returns the full goal object for goal *g* belonging to user *u*.
Same format as the public `GET /users/u/goals/g.json`.

### Parameters

* \[`datapoints`\] (boolean): If `true`, include the goal's datapoints in the response.
* \[`datapoints_count`\] (number): Limit the number of datapoints returned.
* \[`diff_since`\] (number): Unix timestamp. Only return datapoints updated since this time.

<h2 id="admin-getroad">Get road matrix</h2>

```shell
  curl https://www.beeminder.com/api/admin/users/alice/goals/weight/road.json?auth_token=ADMIN_TOKEN
```

```json
  { "slug": "weight",
    "roadall": [[1609459200, 180, null], [1743465600, null, -0.1]],
    "runits": "d" }
```

### HTTP Request

`GET /admin/users/`*u*`/goals/`*g*`/road.json`

Returns the road matrix (`roadall`) for goal *g*.
Each row is a triple `[date, value, rate]` where exactly one element is null (to be inferred from the other two and the previous row).
Dates are Unix timestamps.
The `runits` field indicates the rate units (`d` for daily, `w` for weekly, `m` for monthly, `y` for yearly).

### Returns

* `slug` (string): The goal's slug.
* `roadall` (array): The road matrix. See [Goal Resource](#goal) for details on the format.
* `runits` (string): Rate units.

<h2 id="admin-updateroad">Update road matrix</h2>

```shell
  curl -X PUT https://www.beeminder.com/api/admin/users/alice/goals/weight/road.json \
    -d auth_token=ADMIN_TOKEN \
    -d 'roadall=[[1609459200,180,null],[1743465600,null,-0.05]]'
```

```json
  { "slug": "weight",
    "title": "Weight Loss",
    "roadall": [[1609459200, 180, null], [1743465600, null, -0.05]],
    "..." : "..." }
```

### HTTP Request

`PUT /admin/users/`*u*`/goals/`*g*`/road.json`

Updates the road matrix for goal *g*.
Unlike the public API's goal update, this endpoint **skips the akrasia horizon check**, allowing admins to make any road change immediately.
This is essential for fixing broken roads or making emergency adjustments.

Basic structural validation is still performed (each row must be a 3-element array).

### Parameters

* `roadall` (string, required): JSON-encoded road matrix. Each row is `[date_or_null, value_or_null, rate_or_null]` with dates as Unix timestamps.

### Returns

The full updated [Goal](#goal) object.

<h2 id="admin-getpledge">Get pledge info</h2>

```shell
  curl https://www.beeminder.com/api/admin/users/alice/goals/weight/pledge.json?auth_token=ADMIN_TOKEN
```

```json
  { "slug": "weight",
    "amount": 10,
    "stepdown_at": null,
    "pending_amount": null,
    "pending_at": null,
    "pledge_cap": 90,
    "schedge": [0, 5, 10, 30, 90, 270, 810, 2430] }
```

### HTTP Request

`GET /admin/users/`*u*`/goals/`*g*`/pledge.json`

Returns the current pledge/contract information for goal *g*.

### Returns

* `slug` (string): The goal's slug.
* `amount` (number): Current pledge amount in dollars (e.g., `10` means $10).
* `stepdown_at` (number or null): Unix timestamp of when a scheduled stepdown will take effect, or null.
* `pending_amount` (number or null): Amount of a pending charge from a recent derailment, or null.
* `pending_at` (string or null): ISO 8601 timestamp of when the pending charge is scheduled to execute, or null.
* `pledge_cap` (number): The maximum pledge this goal can escalate to.
* `schedge` (array): The pledge schedule &mdash; the ordered list of possible pledge amounts for this goal.

<h2 id="admin-delaycharge">Delay a pending charge</h2>

```shell
  curl -X POST https://www.beeminder.com/api/admin/users/alice/goals/weight/delay_charge.json \
    -d auth_token=ADMIN_TOKEN
```

```json
  { "success": true,
    "slug": "weight",
    "delayed_until": "2026-03-30T14:00:00Z",
    "amount": 10 }
```

### HTTP Request

`POST /admin/users/`*u*`/goals/`*g*`/delay_charge.json`

Delays the pending derailment charge on goal *g* by 48 hours.
This is the API equivalent of the "+48 hours" button in the admin panel.
The charge is rescheduled to `pending_at + 48 hours`.

Returns a 422 error if there is no pending charge on the goal's contract.

### Returns

* `success` (boolean): `true` on success.
* `slug` (string): The goal's slug.
* `delayed_until` (string): ISO 8601 timestamp of the new scheduled charge time.
* `amount` (number): The amount of the delayed charge.

<h2 id="admin-cancelcharge">Cancel a pending charge</h2>

```shell
  curl -X POST https://www.beeminder.com/api/admin/users/alice/goals/weight/cancel_charge.json \
    -d auth_token=ADMIN_TOKEN
```

```json
  { "success": true,
    "slug": "weight" }
```

### HTTP Request

`POST /admin/users/`*u*`/goals/`*g*`/cancel_charge.json`

Cancels the pending derailment charge on goal *g*.
This clears both `pending_at` and `pending_amount` on the contract.
The user will not be charged.

Returns a 422 error if the goal has no contract or no pending charge.

[Back to top](#)

<h2 id="admin-getdatapoints">Get datapoints for a goal</h2>

```shell
  curl https://www.beeminder.com/api/admin/users/alice/goals/weight/datapoints.json?auth_token=ADMIN_TOKEN
```

```json
  [ { "id": "abc123def456",
      "timestamp": 1711497600,
      "daystamp": "20260327",
      "value": 172.5,
      "comment": "morning weigh-in",
      "updated_at": 1711497700,
      "requestid": null },
    { "id": "abc123def789", "..." : "..." } ]
```

### HTTP Request

`GET /admin/users/`*u*`/goals/`*g*`/datapoints.json`

Returns datapoints for goal *g* belonging to user *u*.
Each datapoint object has the same shape as the public API's [Datapoint](#datapoint) response.

### Parameters

* \[`sort`\] (string): Field to sort by, e.g., `daystamp` or `id`. Default: `id`. Results are in descending order.
* \[`count`\] (number): Maximum number of datapoints to return. Only used when not paginating.
* \[`page`\] (number): Page number (1-indexed) for pagination.
* \[`per`\] (number): Number of results per page. Default: 25.

<h2 id="admin-deletedatapoint">Delete a datapoint</h2>

```shell
  curl -X DELETE https://www.beeminder.com/api/admin/users/alice/goals/weight/datapoints/abc123def456.json \
    -d auth_token=ADMIN_TOKEN
```

```json
  { "id": "abc123def456",
    "timestamp": 1711497600,
    "daystamp": "20260327",
    "value": 172.5,
    "comment": "morning weigh-in" }
```

### HTTP Request

`DELETE /admin/users/`*u*`/goals/`*g*`/datapoints/`*id*`.json`

Deletes the specified datapoint and triggers a graph regeneration.
Returns the deleted [Datapoint](#datapoint) object.

[Back to top](#)

<h2 id="admin-getcharges">Get charge history for a user</h2>

```shell
  curl https://www.beeminder.com/api/admin/users/alice/charges.json?auth_token=ADMIN_TOKEN
```

```json
  { "charges": [
      { "id": "507f1f77bcf86cd799439011",
        "amount": 10,
        "note": "alice@example.com lost $10 on bmndr.com/alice/weight",
        "status": "succeeded",
        "username": "alice" },
      { "id": "507f1f77bcf86cd799439012",
        "amount": 5,
        "note": "alice@example.com lost $5 on bmndr.com/alice/pushups",
        "status": "pending",
        "username": "alice" } ],
    "total": 17,
    "page": 1,
    "per": 50 }
```

### HTTP Request

`GET /admin/users/`*u*`/charges.json`

Returns the user's internal charge records, newest first.
These track the amount, status, and description of each charge associated with a derailment or other payment event.

### Parameters

* \[`page`\] (number): Page number (1-indexed). Default: 1.
* \[`per`\] (number): Number of results per page. Default: 50.

### Charge Attributes

* `id` (string): The Beeminder charge ID.
* `amount` (number): Charge amount in dollars.
* `note` (string): Description of the charge (typically includes email, amount, and goal slug).
* `status` (string): One of `pending`, `succeeded`, `canceled`, `failed`, or `refunded`.
* `username` (string): The username of the charged user.

### Returns

* `charges` (array): Array of Charge objects.
* `total` (number): Total number of charges for this user.
* `page` (number): Current page number.
* `per` (number): Results per page.

[Back to top](#)

<h2 id="admin-incrementnonlegit">Increment non-legit count</h2>

```shell
  curl -X POST https://www.beeminder.com/api/admin/users/alice/increment_nonlegit.json \
    -d auth_token=ADMIN_TOKEN
```

```json
  { "success": true,
    "username": "alice",
    "nonlegits": 3,
    "nonlegit_ts": "2026-03-28T14:00:00Z",
    "monthly_nonlegits": { "2026-01": 1, "2026-03": 2 } }
```

### HTTP Request

`POST /admin/users/`*u*`/increment_nonlegit.json`

Increments the non-legit counter for user *u*.
This is used to track users who have derailments that appear non-legitimate.
The counter is tracked both as a lifetime total and broken down by month.

### Returns

* `success` (boolean): `true` on success.
* `username` (string): The user's username.
* `nonlegits` (number): The updated lifetime non-legit count.
* `nonlegit_ts` (string): ISO 8601 timestamp of this increment.
* `monthly_nonlegits` (object): Hash of `YYYY-MM` keys to monthly counts.
