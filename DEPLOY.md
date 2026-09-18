# Deploying maven.mackery.com

The Worker serves a read-only Maven repository from an R2 bucket. Mods don't
commit artifacts here — their release workflows upload straight to R2.

## Prerequisites

- A Cloudflare account with `mackery.com` added as a zone.
- R2 enabled on the account (it already is — `antileak-builds` uses it).

## 1. Install + log in

```bash
npm install
npx wrangler login
```

## 2. Create the bucket

Must exist **before** the first deploy, or the `MAVEN` binding fails to resolve.

```bash
npx wrangler r2 bucket create maven
```

## 3. Deploy + attach the subdomain

```bash
npx wrangler deploy
```

`wrangler.toml` already declares `routes = [{ pattern = "maven.mackery.com",
custom_domain = true }]`, so this creates the DNS record and TLS cert for you —
no dashboard step. Verify:

```bash
curl -I https://maven.mackery.com/
```

## 4. Seed the artifacts already released

The bucket starts empty. Publish the Biomancy versions that predate this setup:

```bash
for V in 2.9.8.1 2.9.8.2; do
  D="com/github/elenterius/Biomancy/$V"
  mkdir -p "$D"
  gh release download "v$V" --repo Mackery6969/Biomancy \
    --pattern "biomancy-neoforge-*-$V.jar" --output "$D/Biomancy-$V.jar"
  cat > "$D/Biomancy-$V.pom" <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.github.elenterius</groupId>
  <artifactId>Biomancy</artifactId>
  <version>$V</version>
  <packaging>jar</packaging>
</project>
EOF
  npx wrangler r2 object put "maven/$D/Biomancy-$V.jar" --file "$D/Biomancy-$V.jar" --remote
  npx wrangler r2 object put "maven/$D/Biomancy-$V.pom" --file "$D/Biomancy-$V.pom" --remote
done

cat > maven-metadata.xml <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<metadata>
  <groupId>com.github.elenterius</groupId>
  <artifactId>Biomancy</artifactId>
  <versioning>
    <latest>2.9.8.2</latest>
    <release>2.9.8.2</release>
    <versions>
      <version>2.9.8.1</version>
      <version>2.9.8.2</version>
    </versions>
  </versioning>
</metadata>
EOF
npx wrangler r2 object put maven/com/github/elenterius/Biomancy/maven-metadata.xml \
  --file maven-metadata.xml --content-type application/xml --remote
```

Check it resolves:

```bash
curl -sI https://maven.mackery.com/com/github/elenterius/Biomancy/2.9.8.1/Biomancy-2.9.8.1.jar
```

## 5. API token for CI

The token used by `auth.mackery.com` writes R2 through a *binding* at runtime,
which needs no API permission — uploading from CI does. Either extend that token
or create a new one with:

| scope | permission |
| --- | --- |
| Account → Workers Scripts | Edit |
| Account → Workers R2 Storage | **Edit** |
| Account → Account Settings | Read |
| Zone (`mackery.com`) → Workers Routes | Edit |

Grab the account id with `npx wrangler whoami`.

## 6. Repository secrets

```bash
gh secret set CLOUDFLARE_API_TOKEN   --repo Mackery6969/Maven

for R in Mackery6969/Biomancy Mackery6969/Bio-Factory; do
  gh secret set CLOUDFLARE_API_TOKEN   --repo "$R"
  gh secret set CLOUDFLARE_ACCOUNT_ID  --repo "$R"
done
```

The Maven repo only deploys the Worker, so it needs the token alone. Publishing
projects call `publish.yml`, which needs both.

## Adding another mod

Copy `.github/workflows/maven.yml` from either publishing repo verbatim — it
needs no per-project configuration. The project must apply `maven-publish` and
stage into `repo/`:

```gradle
publishing {
    publications { register('mavenJava', MavenPublication) { from components.java } }
    repositories { maven { url = layout.projectDirectory.dir("repo").asFile.toURI() } }
}
```

Coordinates come from the staged tree, so `group`, `version` and
`rootProject.name` are the only things that decide them.
