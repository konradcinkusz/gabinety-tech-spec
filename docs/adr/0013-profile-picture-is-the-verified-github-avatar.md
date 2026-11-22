# ADR-0013: The profile picture is the verified GitHub avatar, served by the web host

- Status: accepted
- Date: 2026-11-22

## Context

The list card and the profile show initials only; a marketplace that sells trust in people benefits from a face. A real upload needs object storage, size and type validation, moderation of images (a person can upload anything) and an erasure path (SPEC erasure job). None of that exists, and the owner's standing instruction is to decide without waiting.

A practitioner already proves control of a GitHub account (CONTRACTS 3.3, one-time code in the public bio) and `githubLogin` is on the card and the profile.

## Decision

The picture of an approved practitioner is the avatar of the GitHub account they verified. The browser never talks to GitHub: the web host serves `GET /avatar/{slug}`, which looks the practitioner up in Directory, validates the login (`[A-Za-z0-9-]{1,39}`), fetches `https://github.com/<login>.png?size=96` with a 3 s timeout, accepts only `image/png` or `image/jpeg` up to 200 KB, follows redirects only to `avatars.githubusercontent.com`, and caches the bytes in memory for 24 hours (failures for 10 minutes). The answer carries `Cache-Control: public, max-age=86400` and `X-Content-Type-Options: nosniff`. The CSP `img-src 'self' data:` does not change. Without a verified login, or on any failure, the response is `404` and the page shows the initials that sit under the image.

There is no upload, no image storage and no moderation of images in this version.

## Consequences

- The host makes an outbound request to GitHub per distinct login per day; GitHub learns nothing about the visitor (the request comes from the server).
- A practitioner changes the picture by changing it on GitHub; it follows within a day.
- Erasure needs no extra step: nothing is stored beyond the cache.
- A practitioner whose GitHub avatar is unsuitable is handled like any other unsuitable content: an administrator suspends the profile.

## Alternatives considered

- Upload to Fly volume or object storage: needs image moderation, storage, backups and an erasure path; rejected for this version, revisit when profiles have a reason to need more than the GitHub picture.
- Hot-linking `avatars.githubusercontent.com` from the page: loosens the CSP and sends every visitor's address to GitHub.
- Gravatar: needs the e-mail, which Directory does not and must not expose.
