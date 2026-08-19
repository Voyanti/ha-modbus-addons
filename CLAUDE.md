# Working in this repository

## These repositories are public

This repo and every add-on submodule under it are published on GitHub under the
Voyanti org. Anything committed here is public, and so is the commit history:
rewriting it later does not un-publish it, because unreachable objects stay
reachable by SHA and forks keep their own copies.

Note also that commits here may be pushed automatically. Do not assume a commit
is private until someone runs `git push` — treat `git commit` as publication.

## Never commit customer or site information

Specifically, none of the following belong in any file, commit message, or
branch name:

- Customer, client, site, farm or municipality names.
- Device serial numbers, plant IDs, MAC addresses, logger IDs.
- Hostnames, VPN or Tailscale addresses, and internal dashboard URLs.
- Credentials of any kind: tokens, JWTs (`eyJ...`), passwords, private keys.
- Tariffs, prices, demand charges, contract terms, or savings figures.
- Personal names and email addresses of colleagues or customer staff.

Measured data is the subtle case. Register behaviour, response curves and
protocol findings are exactly what this repo exists to record, and they are fine
on their own. What makes them a disclosure is attribution — a site name, a date
plus a location, or a serial number in the same document turns vendor
characterisation into a named client's operating data and demand-charge
exposure. Write findings as properties of the hardware, not of an installation:
"confirmed on an HPS in the field", never "confirmed at <site>, August 2026".

Placeholder values in shipped config must be obviously fake (`12345678`), never
a real unit copied from a working deployment.

## Before committing

Check the diff for the above, and check the commit message too — it is as public
as the code and is the easier of the two to write carelessly.
