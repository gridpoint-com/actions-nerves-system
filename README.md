# actions-nerves-system

A collection of github actions for building and deploying Nerves systems.

## Example

The following example shows how to build a system and deploy.

```yaml
on: [push]

env:
  OTP_VERSION: 26.2.1
  ELIXIR_VERSION: 1.16.2-otp-26
  NERVES_BOOTSTRAP_VERSION: 1.12.1

permissions:
  id-token: write
  contents: read

jobs:
  get-br-dependencies:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4
      - uses: gridpoint-com/actions-nerves-system@v1
      - name: Get Buildroot Dependencies
        uses: ./.actions/get-br-dependencies
        with:
          otp-version: ${{ env.OTP_VERSION }}
          elixir-version: ${{ env.ELIXIR_VERSION }}
          nerves-bootstrap-version: ${{ env.NERVES_BOOTSTRAP_VERSION }}
          push-to-download-site: true
          download-site-url: ${{ vars.PUBLIC_S3_SITE }}
          download-site-bucket-uri: ${{ vars.S3_BUCKET }}
          aws-role: ${{ secrets.AWS_ROLE }}
          aws-region: ${{ vars.AWS_REGION }}
  build-system:
    needs: [get-br-dependencies]
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4
      - uses: gridpoint-com/actions-nerves-system@v1
      - name: Build nerves_system
        uses: ./.actions/build-system
        with:
          otp-version: ${{ env.OTP_VERSION }}
          elixir-version: ${{ env.ELIXIR_VERSION }}
          nerves-bootstrap-version: ${{ env.NERVES_BOOTSTRAP_VERSION }}
  deploy-system:
    needs: [build-system]
    if: github.ref_type == 'tag'
    runs-on: ubuntu-22.04
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
      - uses: gridpoint-com/actions-nerves-system@v1
      - name: Deploy nerves_system
        uses: ./.actions/deploy-system
        with:
          github-token: ${{ secrets.ARTIFACT_GITHUB_TOKEN }}
```

## Setup for use

To get access to the actions provided by this package, you must first add a uses
of this package to your steps. After this, the actions provided will be available
to be used within the job.

```yaml
    steps:
      - uses: actions/checkout@v4
      - uses: gridpoint-com/actions-nerves-system@v1
```

### get-br-dependencies

```yaml
      - uses: gridpoint-com/actions-nerves-system@v1
      - name: Get Buildroot Dependencies
        uses: ./.actions/get-br-dependencies
        with:
          # OPT, Elixir and nerves_bootstrap versions to use.
          # These are required
          otp-version: ${{ env.OTP_VERSION }}
          elixir-version: ${{ env.ELIXIR_VERSION }}
          nerves-bootstrap-version: ${{ env.NERVES_BOOTSTRAP_VERSION }}

          # Validate hex package.
          # Default: true
          hex-validate: true

          # Cache the packages downloaded by Buildroot (`~/.nerves/dl`)
          # Default: true
          save-dl-cache: true

          # Push Buildroot packages to an aws s3 bucket
          push-to-download-site: false

          # Public URL for downloading packages uploaded to the aws s3 bucket
          # Required only if push-to-download-site is true
          download-site-url: 'http://dl.nerves-project.org'

          # aws s3 bucket uri for uploading buildroot packages
          # Required only if push-to-download-site is true
          download-site-bucket-uri: 's3://your-s3-bucket/uri/'

          # aws role and region with write access to download-site-bucket-uri
          # Required only if push-to-download-site is true
          aws-role: ${{ secrets.AWS_ROLE }}
          aws-region: ${{ vars.AWS_REGION }}
```

#### Download cache

By default the packages downloaded by Buildroot (`~/.nerves/dl`), are cached
for use in other steps. If you would like to disable saving this cache, set
`save-dl-cache: false` in the `build-tools/get-br-dependencies` job for your
Nerves system's CI workflow.

#### Package download sites

System builds download software packages from many sites on the Internet. In
order to mitigate download failures from sites going offline, package URLs
changing, slow downloads, etc, Buildroot supports primary and backup download
sites. The Buildroot project maintains the default backup download site.
Nerves-specific packages are not on the Buildroot site. Keeping an archive of
all downloaded packages is important to ensure that your project can be built
in the future.

The Nerves project maintains a package download site. This action will use it
by setting `BR2_PRIMARY_SITE` to it. Buildroot will attempt to download
packages from there first. If they don't exist, the normal download URLs will
be tried and if those fail, Buildroot's backup download site will be tried.

This action can push new packages to the download site for future builds. Only
AWS S3 is supported. To use this feature, you will need to set the following
permissions before the `jobs:` in your workflow:
```yaml
permissions:
  id-token: write
  contents: read
```

And you will need to set the following in the `./.actions/get-br-dependencies` action:
```yaml
          push-to-download-site: true

          # Public URL for downloading packages uploaded to the aws s3 bucket
          download-site-url: 'http://dl.nerves-project.org'

          # aws s3 bucket uri for uploading buildroot packages
          download-site-bucket-uri: 's3://your-s3-bucket/uri/'

          # aws role and region with write access to download-site-bucket-uri
          aws-role: ${{ secrets.AWS_ROLE }}
          aws-region: ${{ vars.AWS_REGION }}
```

- Create an AWS S3 bucket.
- Set the bucket policy to allow read-only access for anonymous users.

```json
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Sid":"PublicRead",
      "Effect":"Allow",
      "Principal": "*",
      "Action":["s3:GetObject","s3:GetObjectVersion"],
      "Resource":["arn:aws:s3:::<bucket-name>/*"]
    }
  ]
}
```

- In the AWS dashboard, create an AWS role that CI will use to access the S3 bucket.

- If you would like your Nerves system to pull packages from the download site
  when building outside of CI, set the Buildroot primary site in your Nerves
  system `nerves_defconfig` file to the S3 bucket URL:
  `BR2_PRIMARY_SITE="https://<bucket-name>.s3.amazonaws.com"`

- For more details, reference the following documents:
  * [use iam roles to connect github actions to actions in aws](https://aws.amazon.com/blogs/security/use-iam-roles-to-connect-github-actions-to-actions-in-aws/)
  * [configuring openid connect in amazone web services](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)

### build-system

```yaml
      - uses: gridpoint-com/actions-nerves-system@v1
      - name: Build nerves_system
        uses: ./.actions/build-system
        with:
          # OPT, Elixir and nerves_bootstrap versions to use.
          # These are required
          otp-version: ${{ env.OTP_VERSION }}
          elixir-version: ${{ env.ELIXIR_VERSION }}
          nerves-bootstrap-version: ${{ env.NERVES_BOOTSTRAP_VERSION }}

          # Validate hex package.
          # Default: true
          hex-validate: true

          # Enable ccache to speed up build time
          # Enabled by default
          enable-ccache: true

          # Initial ccache settings to apply.
          # Default: '--max-size=1G'
          ccache-initial-setup: '--max-size=1G'
```

### deploy-system

The `./.actions/deploy-system` action should only be run after a job running the `./.actions/build-system` action and only on a pushed tag.

```yaml
  deploy-system:
    needs: [build-system]
    if: github.ref_type == 'tag'
    runs-on: ubuntu-22.04
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
      - uses: gridpoint-com/actions-nerves-system@v1
      - name: Deploy nerves_system
        uses: ./.actions/deploy-system
```

### Creating a release

To create a release for your custom Nerves system:
  - Update the system's `VERSION` file with the version number for the release,
    *without* a leading `v` (`1.0.0`).
  - Update `CHANGELOG.md`, ensuring to create a header with the version number
    that *includes* the leading `v` (`## v1.0.0`).
  - Commit these changes.
  - Add a tag to this commit named after the version that *includes* the leading
    `v` (`v1.0.0`).
  - Push the commit and tag to the git remote (GitHub).
  - The `deploy-system` job will run when the tag is pushed. It will create a
    `v<VERSION>` draft release with the `CHANGELOG.md` notes and compiled Nerves
    system artifact.
  - Review the draft release in GitHub, change the name, and publish the
    release.
