# Overview

The reference-implementation project provides a version of the job manager
based on Symfony along with some basic command-line tools for simulating
a composer and a renderer.

This document discusses how to implement the [Protocol](protocol.md) in Symfony.

# Database

Several entities should be stored in a database using Doctrine.

 * `PreviewBatch`
    * `PRIMARY KEY(user, batch)`
    * `user VARCHAR(63) NOT NULL` - The user who owns this batch. (ex: `savethedolphins`)
    * `batch VARCHAR(63) NOT NULL` - A unique name for this batch. (ex: `newsletter-draft-123`)
    * `message TEXT NOT NULL`  - A JSON-encoded object.
    * `createTime TIMESTAMP` - The time at which the `PreviewBatch` was initially created.
 * `PreviewTask`
    * `PRIMARY KEY(id)`
    * `id UNSIGNED INT AUTOINCREMENT` - A unique ID for this task.
    * `user VARCHAR(63) NOT NULL` - The user who owns this task.
    * `batch VARCHAR(63) NOT NULL` - The batch which produced this task. (FK: `PreviewBatch.name`)
    * `renderer` VARCHAR(63) NOT NULL` - The renderer for this batch.
    * `options TEXT` - A JSON-encoded set of options to pass through to the renderer. (Ex: `{'window-width':'800','window-height':'600'}`)
    * `createTime TIMESTAMP` - The time at which the `PreviewTask` was initially created.
    * `claimTime TIMESTAMP` - The time at which the `PreviewTask` was last claimed.
    * `finishTime TIMESTAMP` - The time at which the `PreviewTask` completed.
    * `attempts INT` - The number of times we have attempted to render this task.
    * `errorMessage TEXT` - 
 * `Renderer`
    * `PRIMARY KEY(renderer)`
    * `renderer VARCHAR(63) NOT NULL` - A unique symbolic name. (ex: `winxp-thunderlook-9.1`)
    * `title VARCHAR(255) NOT NULL` - A displayable title. (ex: `Thunderlook 9.1 (Windows XP)`)
    * `os VARCHAR(63)` - A symbolic name of a platform. (ex: `linux`, `windows`, `darwin`)
    * `osVersion VARCHAR(63)` - The version of the platform.
    * `app VARCHAR(63)` - A symbolic name of the email application (ex: `thunderlook`, `gmail`)
    * `appVersion VARCHAR(63)` - The version of the email application.
    * `icons TEXT` - A JSON-encoded object. (ex: `{'16x16': 'http://example.com/thunderlook-16x16.png'}`)
    * `options TEXT` - A JSON-encoded list of options supported by this renderer. (ex: `['window-width','window-height']`)
    * `lastSeen TIMESTAMP` - The last time we last had communication with this renderer.
 * `User`
    * `PRIMARY KEY(user)`
    * `user VARCHAR(63) NOT NULL` - The username (ex: `alice` or `thunderlook`)
    * `password VARCHAR(63) NOT NULL` - A hashed or digested password.
    * `is_active TINYINT(1) UNSIGNED NOT NULL` - A boolean indicating whether the account is currently active.
    * `roles TEXT` - A JSON-encoded array. (ex: `['renderer']`)

# Files

When processing email messages and their screenshots, we may encounter some large data files.  These files are not stored in the database. 
Instead, they are files in the folder `web/files/{user}/{batch}`.

The use of a naming convention makes it easier to administer files (e.g.  shard users across different filesystems; delete old batches and
old users); however, this also makes it easier for a bot to scan files (by guessing common usernames and batchnames).  To mitigate this,
file names should also be influenced by keys which are harder to guess (such as task-id, render options, or creation-time).

For example, given a `PreviewTask`, the image file name could be constructed as:

```
assert $user matches /^[a-zA-Z0-9][a-zA-Z0-9_-\.]*$/
assert $batch matches /^[a-zA-Z0-9][a-zA-Z0-9_-\.]*$/
$hash = md5(serialize([$id, $user, $batch, $renderer, $options, $createTime]));
$imageFile = "web/files/" . $user . "/" . $batch . "/" . $hash . ".png"
```

# Security

The `User`, `Role`, and `Password` concepts are provided by the [Symfony Security](http://symfony.com/doc/current/book/security.html)
layer.  Users are defined in the database.  For our purposes, a `user` may be a person, a bot, or an entire organization.  Users have a simple name, a password,
an activation flag, and a list of roles.

*Note*: We do not formally model organizations or domains with multiple people who collaborate on newsletters, but this is an important
use-case, and the specification includes functionality to support this use-case.  The organization would have one user-account and one
data-set; however, it may share limited authentication tokens so that each person gets limited access to the user-account.

Users may have one of two roles:

 * `ROLE_COMPOSE`: When granted this role, one has permission to:
    * Create new `PreviewBatch` records (as long as the the owner matches the user)
    * Read `PreviewTask` records (as long as the owner matches the user)
    * Create new session tokens
 * `ROLE_RENDER`: When granted this role, one has permission to:
    * Read and update `PreviewTask` records
    * Read and update `Renderer` records

# Parameters

Some additional parameters (`app/config/config.yml`) should influence the
behavior of the job manager:

 * `batch_ttl`: The amount of time (seconds) to wait before deleting an old `PreviewBatch`.
    (Default: `2` weeks or `14*24*60*60` seconds)
 * `render_ttl`: The amount of time (seconds) to wait for rendering a `PreviewTask`.
    After `render_ttl` is passed, the render attempt is considered failed.
    (Default: `5` minutes or `5*60` seconds)
 * `render_attempts`: The max number of times to try rendering `PreviewTask`. (Default: `2`)
 * `render_agent_ttl`: If a renderer stops polling, we should assume that it has gone offline.
   This is how long we should wait before assuming it's gone offline.
 * `max_token_ttl`: When generating a bearer token, this is the longest permission TTL.
   (Default: `1` week or `7*24*60*60` seconds)

# Commands

The CLI commands make it easier to write, use, and test the system.  They provide a way to ensure that the interfaces are complete
and help with onboarding new developers and administrators.

### 1. renderer:poll

The first CLI tool, `renderer:poll`, makes it easier to write a new renderer.  A renderer must poll an HTTP resource, then perform some
custom rendering logic, and then post another HTTP resource.  This tool lets you write a quick-and-dirty renderer without boilerplate
code for HTTP. To use it, create a dummy rendering script like:

```php
#!/usr/bin/env php
<?php
$job = json_decode(file_get_contents('php://stdin'),1);
echo json_encode(array(
  'id' => $job['PreviewTask']['id'],
  'image' => base64_encode(file_get_contents('/tmp/rick-astley.png')),
));
```

then call it:

```
./app/console renderer:poll --url 'http://user:pass@localhost:9000/' --name 'dummy' --cmd 'examples/dummy-render.php'
```

This command should basically follow the steps from "Example: Renderer prepares a preview" in [protocol.md](protocol.md), .i.e.

 * Register the dummy renderer with `PUT /Renderers/dummy`. (Use the option `--name dummy` to seed fields like `renderer`, `title,` `app`.)
 * Poll `POST /PreviewTask/claim`.
 * Whenever there's a new job, execute `examples/dummy-render.php` and pipe in the job data. Take the output and `POST` it back to `/PreviewTask/submit`. (Be careful to check for erroneous output and report it back.)


### 2. batch:create

The second CLI tool, `batch:create`, is a simple email composer -- it submits a `PreviewBatch`, waits for the response, and downloads the
images.  This is useful for manually testing a renderer.

```bash
./app/console batch:create --subject 'Hello world' --text "Hello world" --render thunderlook,iphone --url 'http://user:pass@localhost:9000/' --out '/tmp/rendered/'
```

This command should basically follow the steps from "Examples: Composer requests a preview" in [protocol.md](protocol.md), .i.e.

 * Generate a `{batch}` name like `prevem-cli-{uniqid}`
 * `PUT /PreviewBatch/{user}/{batch}`
 * while not done ; do echo "Processing..." ; sleep 1 ; GET /PreviewBatch/{user}/{batch}/tasks
 * Dump files to `/tmp/rendered`

### 3. batch:prune

Over time, we may accumulate a lot of previews, and we may want to delete old ones. For example, to delete anything older than two weeks:

```bash
./app/console: batch:prune '14 days ago'
```

This evaluates `strotime('14 days ago')` and finds any `PreviewBatch` / `PreviewTask` which is older -- then deletes
any DB records or files produced by it.

## 4. user:create

Whenever we wish to register a new user account (for a composer or a renderer), we'll need to add a record to the database.
When setting up a new installation, one might typically create two users:

```bash
## Create a composer
## INSERT INTO User (user,password,roles) VALUES ('alice',MD5('s3rc3t'),'[\'compose\']')
./app/console user:create alice --pass=s3cr3t --role=compose

## Creaet a renderer
## INSERT INTO User (user,password,roles) VALUES ('thunderlook',MD5('s3rc3t'),'[\'renderer\']')
./app/console user:create thunderlook --pass=s3cr3t --role=renderer

## Grant multiple orles
## INSERT INTO User (user,password,roles) VALUES ('omniscient',MD5('s3rc3t'),'[\'renderer\',\'compose\']')
./app/console user:create omniscient --pass=s3cr3t --role=renderer,compose
```

## 5. user:update

To change permissions or a change password, we'll need to update an account:

```bash
## Revoke roles
## UPDATE User SET role = '[]' WHERE user 'alice'
./app/console user:create alice --role=''

## Change password
## UPDATE User SET password = md5('n3ws3cr3t') WHERE user = 'alice'
./app/console user:create alice --pass=n3ws3cr3t
```

These commands will INSERT or UPDATE a record in the database.

# Projects

### 1. Create job manager

> Tip: Symfony has conventional ways to implement HTTP routes, controllers, and tests. Use them!

 * Create a skeletal Symfony application called "prevem"
     * Based on https://github.com/gimler/symfony-rest-edition (unless there's some disagreement/issue)
 * Implement the database model using Doctrine
 * Implement route: `GET /Renderers` (per [protocol.md](protocol.md))
     * Implement controller and a PHPUnit test.
 * Implement route: `PUT /Renderer/{rendername}` (per [protocol.md](protocol.md))
     * Implement controller and a PHPUnit test.
 * Implement route: `PUT /PreviewBatch/{username}/{batch}` (per [protocol.md](protocol.md))
     * Implement controller and a PHPUnit test.
 * Implement route: `GET /PreviewBatch/{username}/{batch}` (per [protocol.md](protocol.md))
     * Implement controller and a PHPUnit test.
 * Implement route: `GET /PreviewBatch/{username}/{batch}/tasks` (per [protocol.md](protocol.md))
     * Implement controller and a PHPUnit test.
 * Implement route: `POST /PreviewTask/claim` (per [protocol.md](protocol.md))
     * Implement controller and a PHPUnit test.
 * Implement route: `POST /PreviewTask/submit` (per [protocol.md](protocol.md))
     * Implement controller and a PHPUnit test.
 * Implement `Bearer` tokens (Route `POST /User/login` and security service) (per [protocol.md](protocol.md))
     * Implement controller and test for `POST /User/login` as well as plugin for Symfony Security layer to process `Bearer` tokens
 * Create an installation script in buildkit (`app/config/prevem`)
     * The script should download prevem+dependencies and setup the DB+vhost.  There's are examples of using `civibuild` to setup a
       Symfony app in `app/config/messages` and `app/config/docs`.
 
### 2. Create CLI tools

> Tip: Symfony has conventional ways to implement CLI commands. Use them!
> An example: [DocsPublishCommand.php](https://github.com/civicrm/civicrm-docs/blob/master/src/AppBundle/Command/DocsPublishCommand.php)

 * Implement command `batch:create` (per [refimpl.md](refimpl.md))
 * Implement command `renderer:poll` (per [refimpl.md](refimpl.md))
 * Implement command `batch:prune` (per [refimpl.md](refimpl.md))
 * Implement an integration test which runs `batch:create` and `renderer:poll`

### 3. Update CiviMail UI

(TODO: This section can be fleshed out more.)

 * Implement CiviMail composer
     * Update/finish [PR 6564](https://github.com/civicrm/civicrm-core/pull/6564). The dataflows should be pretty close, although you may need to tweak the REST calls.

### 4. Create real services

(TODO: This section can be fleshed out more.)

 * Implement delegated renderer that works with our preferred commercial service. Build on the `renderer:poll` command.
 * Maybe: Update/slim down [webmail-renderer](https://github.com/utkarshsharma/webmail-renderer). It can build on the `renderer:poll` command.

### 5. Deploy

(TODO: This section can be fleshed out more.)

 * Deploy an instance on `prevem` mycivi.org
 * Integrate with cxnapp
     * prevem:
         * Write a custom UserProvider. It should determine roles by checking membership status on `civicrm.org`
     * cxnapp:
         * Declare a new application `app:org.civicrm.prevem`
         * Whenever a new site registers, generate a username and password. Store these in a way that `UserProvider` will detect.
         * Automatically use CiviCRM API to set `prevem_url`
