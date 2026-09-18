# maven.mackery.com

A Maven repository for Mackery's Minecraft mods. Artifacts live in a Cloudflare R2
bucket; this Worker serves them read-only at `maven.mackery.com`.

## Using it

```gradle
repositories {
    maven {
        name = "Mackery"
        url = uri("https://maven.mackery.com")
        content {
            includeGroup("com.github.elenterius")
        }
    }
}
```

| artifact | coordinates |
| --- | --- |
| Biomancy (1.21.1 NeoForge port) | `com.github.elenterius:Biomancy:<version>` |

Available versions are listed in each artifact's `maven-metadata.xml`, e.g.
<https://maven.mackery.com/com/github/elenterius/Biomancy/maven-metadata.xml>.

## How it works

```
release published
   -> project CI runs `./gradlew publish` into a staging dir
   -> uploads the tree to the `maven` R2 bucket via wrangler
   -> rewrites maven-metadata.xml with the new version
```

Nothing is committed here when a mod is published — this repo holds only the Worker.
Pushing to `main` deploys it via `.github/workflows/deploy.yml`.

The Worker is deliberately small: `GET`/`HEAD` only, R2 key straight off the request
path (traversal rejected), correct content types, byte-range and `ETag` support, and
immutable caching for artifacts with a short TTL for metadata.

## Layout

```
com/github/elenterius/Biomancy/
├── maven-metadata.xml
└── <version>/
    ├── Biomancy-<version>.jar
    └── Biomancy-<version>.pom
```

## Local development

```sh
npm install
npx wrangler dev          # serves from a local R2 simulation
npx wrangler r2 object put maven/<key> --file <path> --local
```

## Setup

Requires an R2 bucket named `maven`, `maven.mackery.com` bound as a custom domain
for the Worker, and `CLOUDFLARE_API_TOKEN` as a repository secret here and in each
publishing project (plus `CLOUDFLARE_ACCOUNT_ID` there).
