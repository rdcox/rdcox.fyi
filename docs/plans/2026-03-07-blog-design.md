# rdcox.fyi Blog Design

## Goal

A simple, cheap personal blog for sharing thoughts and showcasing work, hosted on AWS.

## Architecture

Hugo static site hosted on S3, served via CloudFront CDN, with custom domain rdcox.fyi via Route 53. Content lives as markdown files in a GitHub repo. Deployment via GitHub Actions — push to main triggers build and deploy.

```
Write markdown → git push → GitHub Actions → hugo build → aws s3 sync → CloudFront invalidation
```

## Tech Stack

- **Static site generator:** Hugo
- **Theme:** PaperMod (clean, minimal, dark mode, search)
- **Hosting:** AWS S3 + CloudFront
- **DNS/SSL:** Route 53 (already owned) + ACM certificate (free, us-east-1)
- **CI/CD:** GitHub Actions
- **Content format:** Markdown

## Content Structure

```
content/
  posts/          # Blog posts (short, medium, long)
  projects/       # Portfolio/showcase pieces
```

Media (images, etc.) stored in `static/`. Large media (videos) embedded from YouTube/Vimeo.

## AWS Infrastructure

- **S3 bucket:** `rdcox.fyi` — static website hosting, public access for CloudFront
- **CloudFront distribution:** HTTPS, caching, S3 origin
- **ACM certificate:** Free SSL for `rdcox.fyi` (us-east-1)
- **Route 53:** A record aliased to CloudFront distribution

**Estimated cost:** ~$0.50-1/month

## Deployment

GitHub Actions workflow on push to `main`:

1. Check out repo
2. Install Hugo
3. `hugo build`
4. `aws s3 sync public/ s3://rdcox.fyi`
5. Invalidate CloudFront cache

AWS credentials stored as GitHub Actions secrets. Dedicated IAM user with minimal permissions (S3 write + CloudFront invalidation).
