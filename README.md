# Snappy update feed

The signed data Snappy reads to judge whether a PC's BIOS, drivers and
Windows build are current.

Everything under `feed/` is generated and pushed by CI, so a pull request
changing it is pointless - nothing stops it merging, but the next scheduled
run overwrites it. Fix the scrapers instead; they live in the private
repository that publishes here.

This README is the exception: the mirror step copies only the `feed/`
artifacts, so documentation here is maintained by hand and survives every run.

- [Terms of use](#terms-of-use)
- [Files](#files)
- [Verifying a copy](#verifying-a-copy)
- [Feed format](#feed-format)
- [Freshness](#freshness)
- [Where the data comes from](#where-the-data-comes-from)

## Terms of use

**This data is public so that the app can read it, not so that it can be
reused.** The individual facts — a BIOS version, a driver date — belong to
their vendors and are not claimed. The compilation is: the scraping,
normalisation and continuous verification of several thousand entries across a
dozen sources, none of which offers a stable public API for this.

Redistributing it, or building it into another application, product, dataset
or model, needs permission. Re-deriving the same facts from the vendors' own
pages is expressly not restricted. Full terms in [LICENSE.md](LICENSE.md), and
`updates.json` carries a `license` field saying the same thing, so the terms
travel with any copy of the data.

## Files

| file | what it holds |
| --- | --- |
| `feed/updates.json` | the BIOS, driver and Windows versions the app compares against |
| `feed/updates.sig` | ECDSA P-256 signature over `updates.json`, base64 |
| `feed/checked.json` | when the producer last ran, and the per-section check times as of that run |
| `feed/checked.sig` | ECDSA P-256 signature over `checked.json`, base64 |
| `feed/public-key.txt` | the signing key's public half, SubjectPublicKeyInfo, base64 |
| `feed/app.json` | the newest published Snappy version |

`checked.json` is signed with the same key and must be verified the same way.
It is the one file here that can make stale data look freshly checked, which
is exactly the claim it exists to make - so it is never trusted unsigned. It
is also written best-effort: a copy of this feed without it is valid, and a
consumer that cannot get it should fall back to `updates.json`'s own stamps.

`app.json` is **not** signed. It carries no version data the app acts on -
only the newest released version number, which the app holds to the shape a
version actually has rather than rendering whatever it receives.

## Verifying a copy

The app carries its own copy of the public key and refuses any feed whose
signature does not verify against it, so a tampered copy of this data cannot
be fed to it. The same check by hand, in PowerShell 7 (Windows PowerShell's
ECDsaCng cannot import a SubjectPublicKeyInfo):

```powershell
$base = 'https://raw.githubusercontent.com/MattNorthOfficial/snappy-update-feed-public/main/feed'
Invoke-WebRequest -UseBasicParsing "$base/updates.json" -OutFile updates.json
$signature = (Invoke-WebRequest -UseBasicParsing "$base/updates.sig").Content.Trim()
$publicKey = (Invoke-WebRequest -UseBasicParsing "$base/public-key.txt").Content.Trim()

$ecdsa = [System.Security.Cryptography.ECDsa]::Create()
$read = 0
$ecdsa.ImportSubjectPublicKeyInfo([Convert]::FromBase64String($publicKey), [ref] $read)
$ecdsa.VerifyData(
    [System.IO.File]::ReadAllBytes("$PWD/updates.json"),
    [Convert]::FromBase64String($signature),
    [System.Security.Cryptography.HashAlgorithmName]::SHA256)
```

`True` means the file is exactly what was signed. The signature covers the
bytes of `updates.json`, so fetch it as bytes rather than as text - a copy
that has been through newline translation will not verify.

The signature is IEEE P1363 format (a fixed-width `r || s` pair), not DER.
Verifiers that default to DER, OpenSSL among them, need telling.

The public key's SHA-256 fingerprint, for comparison against a copy obtained
some other way:

```
5748dbf69e5a3fda65628b30aef1ea28972532285a296ccf491b0d6d39767f9d
```

The app ships this key beside its signed executable rather than trusting
whatever sits beside the feed, so a key change requires an app release, and a
fetched key alone can never move a machine's trust root.

## Feed format

```jsonc
{
  "schemaVersion": 1,
  "license": "Compilation (c) 2026 Matt North. ...",
  "updated": "2026-08-05T13:06:09Z",
  "freshness":   { "amd.windows": "...", "motherboards.gigabyte": "..." },
  "communitySources": { "...": { "repository": "...", "commit": "..." } },

  "amd":           { "windows": { "current": {...}, "rdna1-2": {...}, "polaris-vega": {...} },
                     "chipset": { "revision": "...", "date": "...", "components": {...} } },
  "nvidia":        { "gameReady": "610.88", "url": "...", "download": "..." },
  "intel":         { "chipset": {...}, "arc": {...}, "xe": {...},
                     "rst20": {...}, "rst21": {...}, "chipsetInf": {...} },
  "windowsBuilds": { "25H2": { "build": "...", "date": "...", "kb": "...",
                               "eosHome": "...", "eosEnterprise": "..." } },
  "windows10":     { "22H2": { "eosHome": "...", "eosEnterprise": "..." } },
  "motherboards":  { "B650 AORUS ELITE": { "bios": "F41", "date": "...",
                                           "url": "...", "vendor": "gigabyte" } },
  "motherboardConflicts": { "...": { "msi": {...}, "gigabyte": {...} } },
  "dell":          { "0A5C": { "bios": "1.15.0", "date": "...", "url": "..." } }
}
```

- `license` states the terms above inside the data itself, so a copy taken
  without the repository still carries them. Consumers ignore it.
- `schemaVersion` changes only for a consumer-facing breaking change.
  Additive fields stay within the current version, so a consumer that ignores
  unknown keys will not be broken by a new section appearing.
- `amd.windows` splits into the mainline release (`current`) and the
  maintenance branches AMD keeps for older GPU generations (`rdna1-2`,
  `polaris-vega`). Each carries `adrenalin`, the marketing version AMD's site
  shows, and `driverStore`, the version WMI and Device Manager report - which
  is what lets a client match an installed driver to the right branch.
- `intel` holds the chipset INF utility and the two graphics-driver branches
  Intel has maintained since its 2025 split: `arc` (Arc cards and Core Ultra
  iGPUs) and `xe` (the legacy package for 11th-14th gen Iris Xe / UHD).
- `windowsBuilds` maps each Windows 11 version to its latest *required*
  build. Patch Tuesday (B) and out-of-band releases count; optional D-week
  previews do not, so a fully patched machine is never flagged as outdated.
- `motherboards` is keyed by the clean marketing name, without the
  "(MS-7E51)"-style suffix WMI appends, and holds the latest non-beta BIOS
  with a link to the vendor's own page. Where several board revisions share
  one marketing name the entry is marked `revisionAmbiguous`; consumers
  should withhold a verdict rather than send every revision to one file.
- `motherboardConflicts` records models whose name is claimed by more than
  one vendor, with each vendor's answer kept separate.
- `dell` is keyed by the 4-hex system id every Dell and Alienware reports as
  its SKU, from Dell's own update catalog.
- `download`, where present, is the vendor's own installer URL, so a client
  can offer the file straight from the vendor's CDN.
- `communitySources` records the immutable commits behind the Intel INF and
  NVIDIA product-ID maps. Nothing here follows a moving upstream branch.

## Freshness

`updated` is when the file was written. `freshness` is the more useful field:
it stamps each section as that section's own check finishes, so a section that
could not be refreshed keeps its previous timestamp instead of inheriting a
misleadingly recent one.

Neither answers "did anyone look recently", though, and that is what
`checked.json` is for. `updates.json` is only rewritten when a value actually
moves, so on a quiet week its own stamps sit still - which is indistinguishable
from a producer that has died. `checked.json` is written on every run, whether
or not anything changed, so it separates the two:

```json
{
  "checked": "2026-08-29T21:11:00Z",
  "freshness": { "amd.windows": "2026-08-29T21:10:26Z", "...": "..." }
}
```

A producer that has stopped running keeps serving well-formed, correctly
signed, increasingly wrong answers, and `checked` is the only field that can
notice. Read it alongside a live feed only: beside a bundled or cached
snapshot, a freshly fetched check time would describe a producer run whose
data that copy does not actually have.

That distinction matters because **a section whose source cannot be reached
keeps its last known values rather than disappearing**. Consumers should read
`freshness` to tell a newly written section from one carried forward, and
treat a section's age as a property of that section rather than of the file.

Driver, graphics and Windows sections refresh several times a day. The
motherboard sections are swept on a slower cycle, so a board entry is
routinely a day or two old by design; BIOS releases are rare enough that this
costs nothing in practice.

## Where the data comes from

Vendor sources swept on a schedule: AMD's and NVIDIA's driver releases, Intel
driver branches, Microsoft's Windows release information, Dell's update
catalog, and the BIOS listings of the four desktop board vendors. Values are
taken from each vendor's own published pages or catalogs, with one exception:
`intel.chipsetInf` republishes a community-maintained table of Intel chipset
INF versions, pinned to the commit `communitySources` names. Nothing else is
community-submitted, and nothing is inferred where a vendor is ambiguous.

The scrapers that produce this data, and the signing key, live elsewhere. Only
their output lands here.
