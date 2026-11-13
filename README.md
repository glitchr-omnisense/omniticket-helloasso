# omniticket/helloasso

[omnisense/omniticket](https://github.com/glitchr-omnisense/omniticket)'s HelloAsso, under any PHP
host: an association's event forms with their tiers, the registrations of an event, and a ticket's
QR code checked among them. Reading only.

## Install and a first call

```sh
composer require omniticket/helloasso
```

No account for this first call: HelloAsso's answers (its access token, then the association's
event forms) are recorded (`Omniticket\Testing\RecordedHttpClient` answers in place of the API), so
nothing leaves the process. A real call needs the association's HelloAsso account: the client id
and secret of its back office's API section (the sandbox has its own, at api.helloasso-sandbox.com).

```php
<?php

require __DIR__.'/vendor/autoload.php';

use Omniticket\HelloAsso\HelloAssoFactory;
use Omniticket\Testing\RecordedHttpClient;

// HelloAsso's answers, recorded: nothing sent. HttpClient::create() (symfony/http-client) for a real call.
$http = RecordedHttpClient::routes([
    'POST /oauth2/token' => '{"access_token": "recorded", "token_type": "bearer", "expires_in": 1800}',
    'GET /forms' => '{"data": [{"title": "Bal de printemps", "formSlug": "bal-de-printemps", "formType": "Event", "state": "Public", "currency": "EUR", "startDate": "2036-04-18T20:00:00+02:00", "endDate": "2036-04-19T01:00:00+02:00", "url": "https://www.helloasso.com/associations/example-association/evenements/bal-de-printemps", "organizationSlug": "example-association"}], "pagination": {"totalPages": 1, "pageIndex": 1}}',
]);
$helloasso = (new HelloAssoFactory($http))->create(['client_id' => 'recorded', 'client_secret' => 'recorded', 'organization' => 'example-association']);

foreach ($helloasso->events() as $event) {
    echo $event->title, ' ', $event->start?->format('d/m/Y H:i'), ' ', $event->url, "\n";   // Bal de printemps 18/04/2036 20:00 https://www.helloasso.com/...
}
```

With your credentials, through the family's registry:

```php
use Omniticket\HelloAsso\HelloAssoFactory;
use Omniticket\Registry;
use Symfony\Component\HttpClient\HttpClient;

$registry = new Registry([new HelloAssoFactory(HttpClient::create())], [
    'asso' => ['factory' => 'helloasso', 'options' => ['client_id' => '...', 'client_secret' => '...', 'organization' => 'my-association']],
]);
```

Options: `client_id`, `client_secret` (the association's back office, API section),
`organization` (its slug), `url` (`https://api.helloasso.com`; `https://api.helloasso-sandbox.com`
for the sandbox).

## The API, as read on 2026-10-24

HelloAsso's API v5 (<https://dev.helloasso.com>): a bearer token from `POST /oauth2/token`
(client credentials, 30 minutes, kept until then), `GET /v5/organizations/{slug}/forms?formTypes=Event`
(an event's id is its form slug; `widgetFullUrl` is its embeddable module),
`.../forms/Event/{formSlug}/public` (tiers: prices in cents, a price of 0 with `minAmount` is the
buyer's to choose), `.../forms/Event/{formSlug}/items` (one item per ticket: `qrCode`, `state`,
payments), pages by continuation token. Limits: none stated in what was read - keep calls
moderate and cache the catalogue.

What it cannot say: the items have no "changed since" filter (`updatedSince` throws
`NotSupportedException`; `Capabilities::$incremental` is false), and no scan state - a check is
valid, void or unknown, never "already used". `check()` reads the event's items and looks the code
up among their QR codes.

**Reuse**: HelloAsso publishes a PHP SDK, `helloasso/helloasso-php` (generated from its OpenAPI,
maintained: last update 2026-09-16). It is not wrapped: it is GPL-3.0-or-later (this family is
MIT), it brings its own HTTP stack (Guzzle, ext-curl) where the family injects the application's
client, and its namespace is the generator's generic `OpenAPI\Client`, which collides with any
other generated client. Its model documentation is what the tests' answers follow.

## Usage policy

Declared as data in `policy.json` (omnisense/contracts' schema), read on 2026-11-10 at
[limitation-api](https://dev.helloasso.com/docs/limitation-api): the authentication API - the
access token, the operation `token` - 10 calls every 10 seconds, 20 every 10 minutes, 50 an hour;
the token is kept 30 minutes per provider. No figure for the other endpoints, and the general terms
answered 403: their rights unknown. Through omniticket's registry every call is checked against it,
the token requests by their own limits, a 429 holds the next calls, and a site may tighten it,
never loosen it.

## Verified, and not

- Verified (the core's harness, PHP 8.4): the token (asked once, kept), events and their status,
  one event whole (tiers, free choice, free), registrations (valid, refunded through a payment,
  cancelled), checks, pages by continuation token, a refused client - against answers composed from
  the field names of HelloAsso's OpenAPI (as its SDK documents its models), not recorded; every
  class loading with the package's requirements alone (`BareTest`).
- Not verified live: no HelloAsso account or sandbox was used.

## Licence

MIT. An unofficial client, affiliated with HelloAsso in no way.
