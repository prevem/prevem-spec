# Prevem: Open source preview manager for emails (v2)

Prevem is a batch manager for email previews based on a RESTful CRUD API.

 * *Composer* - A mail user-agent (MUA) in which a user composes a mailing.
   Generally, the composer submits a `PreviewBatch` and polls to track its progress.
   (Example: CiviCRM or SimpleNews would be sensible composers.)
 * *Renderer* - A mail user-agent (MUA) which acts as a backend service.  A
   renderer prepares a screenshot of how an email would appear.  Generally,
   a Renderer polls for a pending `PreviewTask` record, prepares a
   screenshot, and then marks it as completed.
   (Example: Microsoft Outlook, Mozilla Thunderbird, or Gmail would be sensible renderers.)
 * *Prevem* - A batched job manager which relays data between composers
   and renderers.

The batch manager relies on a few conceptual entities:

 * `PreviewBatch`: When a composer submits a request to render an email in 3 different ways (e.g. Outlook + iPhone + Gmail), it creates a batch. 
 * `PreviewTask`: For each batch, prevem autocreates a list of tasks. The renderers poll for pending tasks; reserve them; and then post back the results.
 * `Renderer`: When a user in the `Composer` browses a menu of available renderers, this includes details such as a name, version, platform, and icon.
   The `Renderer` entity defines 
 * `EmailMessage`: When a composer creates a PreviewBatch, it must define the content of the email to preview.

Most of these entities are mapped to RESTful end-points, i.e.:

 * `GET /Renderers`: Return a list of active renderers.
 * `PUT /Renderer/{rendername}`: Create or update a renderer.
 * `PUT /PreviewBatch/{username}/{batch}`: Create a new batch.
 * `GET /PreviewBatch/{username}/{batch}`: Read the definition of a batch and all its tasks. 
 * `GET /PreviewBatch/{username}/{batch}/tasks`: Read the status of the batch and all its tasks.
 * `POST /PreviewTask/claim`: Fetch the next task for a renderer.
 * `POST /PreviewTask/submit`: Submit the results after rendering an `EmailMessage`.
 * `POST /User/login`: Accepts a username and password; produces an authentication token.

# 1. Examples

In these examples, we consider the HTTP traffic in some use-cases.  The examples involve a user Alice who is composing
a newsletter.  She connects to prevem instance (`https://alice:secret@example.org/prevem`) to request a preview.  We
also have a renderer for the email application Thunderlook (which connects to the same instance using different
credentials, `https://thunderlook:secret@example.org/prevem`).

### Composer requests a preview

Alice begins by creating a new PreviewBatch. She assigns a name to her batch, `newsletter-1234`.

```
PUT https://example.org/prevem/PreviewBatch/alice/newsletter-1234
Authorization: Basic {base64("alice:secret")}
Content-Type: text/javascript

{
  "message": {
    "from": "\"Alice Alexander\" <alice.alexander@examplemail.org>",
    "subject": "Hello world",
    "body_html": "<html><body>Hello world</body></html>",
    "body_text": "Hello world"
  },
  "tasks": [
    {"renderer":"gmail"},
    {"renderer":"winxp-thunderlook-9.1", "options": {"window-height": "600"}}
  ]
}
```

which responds

```
HTTP/1.1 200 OK
Content-Type: text/javascript

{}
```

She may then follow-up by requesting the status of the tasks:

```
GET https://example.org/prevem/PreviewBatch/alice/newsletter-1234/tasks
Authorization: Basic {base64("alice:secret")}
```

which responds

```
HTTP/1.1 200 OK
Content-Type: text/javascript

{
  "tasks": [
    {"id":"12", "renderer":"gmail", ...},
    {"id":"13", "renderer":"winxp-thunderlook-9.1", ...}
  ]
}
```

Each task record includes the fields:

 * `id`, `user`, `batch`, `renderer`, `attempts`, and `errorMessage`. These are exactly the same as the DB entity.
 * `createTime`, `claimTime`, `finishTime` are also passed through. It appears as either NULL or seconds-since-epoch.
 * `options` is passed through, but it expressed natively as a JSON object. It is *not* double-encoded.
 * `status` is a string -- `pending`, `rendering`, `finished`, or `failed`. 
     * Suggested: this value can be computed using the timestamps, TTL policies, and `errorMessage`.
          * if `finishTime` is set and `errorMessage` is empty, then `finished`
          * if `finishTime` is set and `errorMessage` is defined and `attempts` exceeds `render_attempts`, then `failed`
          * if `claimTime` is set and it's not passed `render_ttl`, then `rendering`
          * otherwise, `pending`
 * `imageUrl` is an optional string which references the screenshot.

### Renderer prepares a preview

The Thunderlook renderer begins by trying to claim a task:

```
PUT https://example.org/prevem/PreviewTask/claim
Authorization: Basic {base64("thunderlook:secret")}
Content-Type: text/javascript

{
  "renderer": "winxp-thunderlook-9.1"
}
```

and receives a response

```
HTTP/1.1 200 OK
Content-Type: text/javascript

{}
```

which is a bit disappointing. There's nothing to do! So Thunderlook waits a minute and
sends the same request again. This time, the response indicates some work to do:

```
HTTP/1.1 200 OK
Content-Type: text/javascript

{
  "PreviewBatch": {
    "user": "alice",
    "batch": "newsletter-1234",
    "message": {
      "from": "\"Alice Alexander\" <alice.alexander@examplemail.org>",
      "subject": "Hello world",
      "body_html": "<html><body>Hello world</body></html>",
      "body_text": "Hello world"
    }
  },
  "PreviewTask": {
    "id": "987",
    "user": "alice",
    "batch": "newsletter-1234",
    "renderer": "winxp-thunderlook-9.1",
    "options": {"window-height": "600"}
  }
}
```

Thunderlook does some processing to prepare a screenshot of the
message. If everything goes well, it produces the PNG file and
encodes it in Base64 (e.g.  `Im9AGeP+Ng==`) and submits this:

```
POST https://example.org/prevem/PreviewTask/submit
Authorization: Basic {base64("alice:secret")}
Content-type: text/javascript

{
  "id": "987",
  "image": "Im9AGeP+Ng=="
}
```

which responds

```
HTTP/1.1 200 OK
```

If Thunderlook encounteres an error 


### Authenticating via token

The previous examples use HTTP Basic authentication
If you're writing a client-side application which interacts with `prevem`, you may not want to trust the client with
Alice's main `secret`.  Instead, you can generate a temporary token (valid for 2 hours):

```
POST https://example.org/prevem/User/login?ttl=7200
Authorization: Basic {base64("alice:secret")}
Content-Type: text/javascript
```

which responds

```
HTTP/1.1 200 OK
Content-Type: text/javascript

{"token":"abcd1234abcd1234"}
```

In subsequent requests (for the next two hours), you can use this token as a bearer token:

```
PUT https://example.org/prevem/PreviewBatch/alice/newsletter-1234
Authorization: Bearer abcd1234abcd1234
...
```

Note: In this example, the bearer token is only 16 characters.  However, do not hard-code a limit of 16 characters.  If
you must a choose limit, use something higher -- like 255 characters.

# 2. Entities

### PreviewBatch

 * `user` (`string`, required) - The user who owns this batch. (ex: `savethedolphins`)
 * `batch` (`string`, required) - A unique name for this batch. (ex: `newsletter-draft-123`)
 * `message` (`object`, required) - An EmailMessage object.
 * `tasks` (`array`, optional) - A list of PreviewTask objects.

### PreviewTask

 * `id` (`scalar`, required) - A unique ID for this task.
 * `user` (`string`, required) - The user who owns this task.
 * `batch` (`string`, required) - The batch which produced this task. (FK: `PreviewBatch.name`)
 * `renderer` (`string`, required) - The renderer for this batch.
 * `options` (`object`, optional) - A JSON-encoded set of options to pass through to the renderer. (Ex: `{'window-width':'800','window-height':'600'}`)
 * `createTime` (`timestamp`, optional) - The time at which the `PreviewTask` was initially created.
 * `claimTime` (`timestamp`, optional) - The time at which the `PreviewTask` was last claimed.
 * `finishTime` (`timestamp`, optional) - The time at which the `PreviewTask` completed.
 * `attempts` (`int`, optional) - The number of times we have attempted to render this task.
 * `errorMessage` (`string`, optional) - An error message explaining why the render task failed.
 * `status` (`string`, required) - The current status of this task (`pending`, `rendering`, `finished`, or `failed`).
 * (Varies) `image` (`string`, base64) or `imageUrl` (`string`) - The image produced by this task. Note that certain endpoints may specifically require `image` or `imageUrl`.

### Renderer

 * `renderer` (`string`, required) - A unique symbolic name. (ex: `winxp-thunderlook-9.1`)
 * `title` (`string`, required) - A displayable title. (ex: `Thunderlook 9.1 (Windows XP)`)
 * `os` (`string`, optional) - A symbolic name of a platform. (ex: `linux`, `windows`, `darwin`)
 * `osVersion` (`string`, optional) - The version of the platform.
 * `app` (`string`, optional) - A symbolic name of the email application (ex: `thunderlook`, `gmail`)
 * `appVersion` (`string`, optional) - The version of the email application.
 * `icons` (`object`, optional) - A list of icons, keyed by size (ex: `{'16x16': 'http://example.com/thunderlook-16x16.png'}`).
 * `options` (`array`, optional) - An open-ended list of rendering options supported by this renderer. (ex: `['window-width','window-height']`).
 * `lastSeen` (`timestamp`, optional) - The last time we last had communication with this renderer.

## EmailMessage

The email message may be specified in any of these three formats:

 * `from`, `subject`, `body_text`, `body_html` (`string`) - The content of an email message.

*FIXME: We have some discrepancies in how the different agents prefer to handle messages; some work well with RFC (2)822
(which is the canonical way to describe an email), but others require a more reductive model (from+subject+body).  The
DRYest approach is probably to have the job-manager handle conversions. We may want to enhance this part of the
spec to better support RFC (2)822 representations.*

# 3. Routes

The remainder of this specification examines the inputs and outputs expected for each RESTful endpoint.

In discussing each endpoint, we include some notes on how the Symfony reference-implementation may work.

### Baseline

The protocol is generally RESTful. All end-points should meet some common criteria:

 * Inputs and outputs are treated as JSON (`application/json`) unless otherwise specified. (Alternative encodings *may* be supported through use of
   HTTP headers `Content-Type` and `Accept`, but this is currently outside scope.)
 * All timestamps in the JSON payload are encoded as seconds-since-epoch (UTC).
 * All end-points require authentication. The client may submit credentials as *either* HTTP `Basic` (username/password) or `Bearer` (token).

### GET /Renderers

 * _Route_: `GET /Renderers`
 * _Purpose_: Return a list of active renderers.
 * _CORS_: Allow cross-origin access.
 * _HTTP Request Data_: None.
 * _HTTP Response Data_: HTTP status code. List of `Renderer` records (default: JSON-encoded).
 * _Reference Implementation Notes_
    * Require `ROLE_COMPOSE`.
    * `SELECT * FROM Renderers WHERE lastSeen >= NOW()-{render_agent_ttl}`

### PUT /Renderer/{rendername}

 * _Route_: `PUT /Renderer/{rendername}`
 * _Purpose_: Create or update a renderer.
 * _HTTP Request Data_: One `Renderer` record (default: JSON-encoded).
 * _HTTP Response Data_: HTTP status code.
 * _Reference Implementation Notes_
    * Require `ROLE_RENDER`.
    * `INSERT` or `UPDATE` a row in `Renderer`, matching on the `rendername`

### PUT /PreviewBatch/{username}/{batch}

 * _Route_: `PUT /PreviewBatch/{username}/{batch}`
 * _Purpose_:  Create a new batch.
 * _CORS_: Allow cross-origin access.
 * _HTTP Request Data_: A `PreviewBatch` (default: JSON-encoded).
 * _HTTP Response Data_: HTTP status code.
 * _Reference Implementation Notes_
     * Require `ROLE_COMPOSE`.
     * INSERT a row in `PreviewBatch` with the given `user`, `batch`, `message`. (Note: Duplicate is an error.)
     * For each item in `tasks`, INSERT a row in `PreviewTask`. Be sure to set `user`, `batch`, `renderer`, `options`, `createTime`.

### GET /PreviewBatch/{username}/{batch}

 * _Route_: `GET /PreviewBatch/{username}/{batch}`
 * _Purpose_: Read the definition of a batch and all its tasks. 
 * _CORS_: Allow cross-origin access.
 * _HTTP Request Data_: None.
 * _HTTP Response Data_: HTTP status code. A `PreviewBatch` (default: JSON-encoded). Do *not* include the full list of `tasks`.
 * _Reference Implementation Notes_
     * Require `ROLE_COMPOSE`.
     * Verify that `{username}` matches the user credentials
     * `SELECT * FROM PreviewBatch WHERE user = {username} AND batch = {batch}`

### GET /PreviewBatch/{username}/{batch}/tasks

 * _Route_: `GET /PreviewBatch/{username}/{batch}/tasks`
 * _Purpose_: Read the status of the batch and all its tasks.
 * _CORS_: Allow cross-origin access.
 * _HTTP Request Data_: None.
 * _HTTP Response Data_: HTTP status code. An array of `PreviewTask` records (default: JSON-encoded).
 * _Reference Implementation Notes_
     * Require `ROLE_COMPOSE`.
     * Verify that `{username}` matches the user credentials
     * `SELECT * FROM PreviewTask WHERE user = {username} AND batch = {batch}`
     * In each `PreviewTask`, add field `status`. (Defined later.)
     * In each `PreviewTask` with status `finished`, add field `imageUrl`. This is an absolute URL to file `web/files/{user}/{batch}/{md5(id . user . batch . renderer . options . createTime)}.png`

### POST /PreviewTask/claim

 * _Route_: `POST /PreviewTask/claim`
 * _Purpose_: Fetch the next task for a renderer.
 * _HTTP Request Data_: A `renderer` name.
 * _HTTP Response Data_: HTTP status code. Either an empty object (`{}`) or `PreviewBatch` and `PreviewTask`.
 * _Reference Implementation Notes_
     * Require `ROLE_RENDER`.
     * `SELECT * FROM PreviewTask FOR UPDATE
           WHERE renderer = {renderer}
           AND attempts < {render_attempts}
           AND (claimTime IS NULL OR claimTime < now()-{render_ttl})
           AND (createTime > now()-{batch_ttl})
           ORDER BY createTime ASC LIMIT 1`
     * If a `PreviewTask` is found, then update the `attempts` and `claimTime`.
     * If a `PreviewTask` is found, then also fetch the related `PreviewBatch`.

### POST /PreviewTask/submit

 * _Route_: `POST /PreviewTask/submit`
 * _Purpose_: Submit the results after rendering an `EmailMessage`.
 * _HTTP Request Data_: A JSON record with field `id` (string, required). Additionally, either `errorMessage` (string) or `image` (string, base64-encoded png).
 * _HTTP Response Data_: HTTP status code.
 * _Reference Implementation Notes_
     * Require `ROLE_RENDER`.
     * `SELECT * FROM PreviewTask WHERE id = {id}`. Expect 1 result.
     * If `image` is provided, then store it in a file named `web/files/{user}/{batch}/{md5(id . user . batch . renderer . options . createTime)}.png`
     * If `image` is provided, then clear any `PreviewTask.errorMessage`.
     * If `image` is provided, then set `PreviewTask.finishTime` to now.
     * If `errorMessage` is provided, then set the `PreviewTask.errorMessage`.
     * If `errorMessage` is provided and `PreviewTask.attempts < render_attempts`, then set `PreviewTask.claimTime` to empty.
     * If `errorMessage` is provided and `PreviewTask.attempts >= render_attempts`, then set `PreviewTask.finishTime` to now.

### POST /User/login

 * _Route_: `POST /User/login`
 * _Purpose_: Accepts a username and password; produces an authentication token.
 * _CORS_: Allow cross-origin access.
 * _HTTP Request Data_: An optional parameter `ttl=<seconds>`.
 * _HTTP Response Data_: HTTP status code. A token (default: JSON-encoded).
 * _Reference Implementation Notes_
     * No special roles are required. Any user may request.
     * Try to use JWT (JSON Web Tokens)
        * If you can't use PHP 5.5+, then let's talk to find an alternative.
        * Read https://jwt.io/introduction/
        * Read https://github.com/lcobucci/jwt . This is a library to generate and validate tokens.
        * HMAC SHA256 should be fine.
        * For a signing key, see `$container->getParameter('secret')`.
        * Our tokens should set `exp` (`setExpiration(time()+$ttl)`) and `name` (`set('name',$username)`)
        * Mostly the other fields seem like a waste of space for us right now (`iss`, `aud`, `iat`, `nbf`). Skip those.
    * To ensure that every endpoint accepts `Authenticate: Bearer` in lieu of `Authenticate: Basic`, read up
      Symfony Security and define a suitable plugin/service.

