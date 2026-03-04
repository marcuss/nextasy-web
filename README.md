# nextasy-web

Nextasy company website — [nextasy.co](https://nextasy.co)

## Architecture

```
nextasy.co (Route 53)
    │
    ├── nextasy.co          → CloudFront → S3 /index.html           (main branch)
    ├── www.nextasy.co      → CloudFront → S3 /index.html
    ├── v1-bento.nextasy.co → CloudFront → S3 /branches/v1-bento/
    ├── v2-cinematic.nextasy.co         → S3 /branches/v2-cinematic/
    ├── v3-minimal-bold.nextasy.co      → S3 /branches/v3-minimal-bold/
    └── {slug}.nextasy.co   → CloudFront → S3 /branches/{slug}/     (any branch)
```

**CloudFront Function** (`nextasy-branch-router`) reads the `Host` header on every request and rewrites the URI to point to the correct S3 folder. Deployed to `LIVE` stage.

## S3 Structure

```
s3://nextasy-co-website/
├── index.html              ← main branch (nextasy.co)
├── branches/
│   ├── v1-bento/
│   │   └── index.html
│   ├── v2-cinematic/
│   │   └── index.html
│   └── v3-minimal-bold/
│       └── index.html
```

## Preview Deployments

Every push to a non-`main` branch automatically:

1. **Sanitizes** the branch name → slug (strips `design/`, `feature/`, `preview/` prefixes; replaces `/` `_` with `-`)
2. **Syncs** files to `s3://nextasy-co-website/branches/{slug}/`
3. **Invalidates** CloudFront at `/branches/{slug}/*`
4. **Upserts** Route 53 CNAME `{slug}.nextasy.co → CloudFront`
5. **Comments** on the PR with both the custom URL and CloudFront fallback URL

### Branch naming → subdomain

| Branch | Subdomain | CloudFront path |
|--------|-----------|-----------------|
| `main` | `nextasy.co` | `/` |
| `design/v1-bento` | `v1-bento.nextasy.co` | `/branches/v1-bento/` |
| `feature/dark-mode` | `dark-mode.nextasy.co` | `/branches/dark-mode/` |

### URLs before DNS propagation

Custom domains (`v1-bento.nextasy.co`) require `nextasy.co` DNS to propagate globally.  
**CloudFront direct URLs work immediately:**

| Branch | CloudFront URL |
|--------|----------------|
| `main` | https://d3heh2lnt32ajw.cloudfront.net |
| `design/v1-bento` | https://d3heh2lnt32ajw.cloudfront.net/branches/v1-bento/ |
| `design/v2-cinematic` | https://d3heh2lnt32ajw.cloudfront.net/branches/v2-cinematic/ |
| `design/v3-minimal-bold` | https://d3heh2lnt32ajw.cloudfront.net/branches/v3-minimal-bold/ |

## Custom Domain with HTTPS (nextasy.co)

The wildcard ACM certificate (`*.nextasy.co` + `nextasy.co`) has been requested and DNS validation records are in Route 53. Once issued (~minutes to hours after DNS propagation), update the CloudFront distribution to:

1. Add aliases: `nextasy.co`, `www.nextasy.co`, `*.nextasy.co`
2. Set viewer certificate to the ACM cert ARN: `arn:aws:acm:us-east-1:092042970121:certificate/c5e8a312-39dd-4478-b592-d124ff61a38b`
3. Set `SSLSupportMethod: sni-only`, `MinimumProtocolVersion: TLSv1.2_2021`

> **Note:** CloudFront does not support wildcard in the `Aliases` field — individual subdomain aliases must be added as branches are created, or use a Lambda@Edge solution for fully dynamic routing. The CloudFront Function already handles the routing logic; only the alias + cert need updating.

## GitHub Secrets

| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | IAM user for S3/CF/R53 access |
| `AWS_SECRET_ACCESS_KEY` | IAM secret key |
| `AWS_REGION` | `us-east-1` |
| `S3_BUCKET` | `nextasy-co-website` |
| `CLOUDFRONT_DISTRIBUTION_ID` | `E640UP3DK37WP` |
| `ROUTE53_ZONE_ID` | `Z02633611RLKX976F44TP` |

## Infrastructure

Managed via Terraform in [nextasyapps-infra](https://github.com/marcuss/nextasyapps-infra) →  
`terraform/environments/dev/nextasy-web.tf`

State: `s3://nextasyapps-terraform-state-dev/nextasy/website/terraform.tfstate`

## Design Branches / PRs

| PR | Branch | Design |
|----|--------|--------|
| [#1](https://github.com/marcuss/nextasy-web/pull/1) | `design/v1-bento` | Bento Grid + Glassmorphism |
| [#2](https://github.com/marcuss/nextasy-web/pull/2) | `design/v2-cinematic` | Cinematic Scroll (Apple-style) |
| [#3](https://github.com/marcuss/nextasy-web/pull/3) | `design/v3-minimal-bold` | Minimal Bold + Typographic Energy |
