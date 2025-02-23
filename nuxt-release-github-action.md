# Nuxt 3 Deployment: Automating Environment-Specific Builds with GitHub Actions

## Introduction  

Hi All,  

Recently, I needed to prepare a deployment procedure for a Nuxt-based frontend. Based on my experience, I initially expected to create a single build that could be used across all environments — QA, UAT, and PROD. However, I discovered that I had to generate separate builds for each environment, each with its own `.env` file defining the Strapi URL and other parameters. This limitation arose due to the `@nuxt/image` library which does not support runtime configuration option for its plugins (we're using [Strapi](https://strapi.io) to store images).  

Different teams have different workflows—some prefer a single build deployed across all environments (*Build-Once, Deploy Everywhere*), while others go with separate builds per environment. Both approaches have their *pros* and *cons*.  

I prefer the first approach due to the following advantages:  

* Program behavior remains unchanged, ensuring consistency. If only configuration files or databases change, you can guarantee that something working in preproduction will work in production simply by ensuring those configurations match.  
* It's slightly more robust. You maintain only one "gold" build, reducing the risk of mistakenly deploying a dev version to QA, and so on.  

However, due to the limitation I mentioned above, I couldn't use this approach and had to venture into the unfamiliar waters of per-environment builds. Unfortunately, Nuxt’s documentation on this topic is unclear (see [Nuxt Deployment Guide](https://nuxt.com/docs/getting-started/deployment)), focusing more on exact deployment or [testing procedures](https://nuxt.com/docs/getting-started/testing) rather than workflow guidance for development teams.  

## Environment Setup  

To address this issue, I explored and documented a structured approach to handling environment-specific builds in Nuxt, aiming to bridge this gap in the official documentation.  

### Technology Stack  

- **Nuxt 3**  
- **Strapi** as the backend  
- **Node.js 18** for building and running  
- **GitHub** for storing source code and artifacts  

## Requirements  

- Two environments: **UAT** and **PROD**  
- A changelog for each release  
- The ability to initiate a release by assigning a tag to a commit  
- The `@nuxt/image` plugin requires a statically injected (`.env`) `STRAPI_URL`, which cannot be placed in `runtimeConfig` (See the [Nuxt Runtime Config Docs](https://nuxt.com/docs/getting-started/configuration#runtimeconfig-vs-appconfig))  

## Workflow  

This workflow automates the build and release process by generating environment-specific builds and creating a changelog for each release.  

### How It Works  

1. **Triggering the Workflow**: The build and release process is triggered when a tag matching `v*` is pushed. To deploy to UAT, for example, we assign a tag to the respective commit. The first tag (e.g., `v1.0.0`) requires a manual `CHANGELOG.md` generation via:  
   ```sh
   git log %YOUR_FIRST_COMMIT%..${{ env.TAG_NAME }} --pretty=format:"%h %s by %an"
   ```
   Please refer to the Git documentation for a [detailed explanation of the output format](https://git-scm.com/docs/pretty-formats).
2. **Repository Permissions**: The workflow requires `write` access to create releases and attach artifacts.  
3. **Changelog Generation**: The difference between the current and previous tags is used to generate a meaningful changelog.  
4. **Environment-Specific Configuration**: GitHub Action Secrets store the Strapi backend URL for each environment, ensuring secure and correct configurations.  
5. **Release Artifacts**: Each release includes two build artifacts, one for UAT and one for PROD, ensuring proper separation.  

---  

**GitHub Actions Workflow:**  

```yaml
on:
  push:
    tags:
      - 'v*'

permissions:
  contents: write
  
env:
  artifact_package_name: project61-nuxt

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18.x]
        environment: [UAT, PROD]
    steps:
      - uses: actions/checkout@v4
      - name: Save tag as environment variable
        run: echo "TAG_NAME=${GITHUB_REF#refs/tags/}" >> $GITHUB_ENV
      - name: Store artifact package name as environment variable
        run: echo ARTIFACT_NAME=${{ env.artifact_package_name }}-${{ env.TAG_NAME }}-${{ matrix.environment }}.tar.gz >> $GITHUB_ENV
      - name: Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
      - run: |
          yarn install --immutable --check-cache --non-interactive
      - name: Load Environment Variables
        run: |
          echo "STRAPI_URL=${{ secrets[format('{0}_STRAPI_URL', matrix.environment)] }}" > .env
      - name: Build Nuxt App
        run: |
          npx nuxt build
      - name: Pack build artifacts
        run: >
          tar zcvf ${{ env.ARTIFACT_NAME }} .output
      - name: Upload Artifact
        uses: actions/upload-artifact@v4
        with:
          name: ${{ env.ARTIFACT_NAME }}
          path: ${{ env.ARTIFACT_NAME }}
  
  release:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Download All Artifacts
        uses: actions/download-artifact@v4
        with:
          path: dist
          merge-multiple: true
      - name: Save tag as environment variable
        run: echo "TAG_NAME=${GITHUB_REF#refs/tags/}" >> $GITHUB_ENV
      - name: Store previous tag
        run: echo "PREV_TAG_NAME=$(git describe --abbrev=0 --tags $(git rev-list --tags --skip=1 --max-count=1))" >> $GITHUB_ENV
      - name: Create changelog
        run: |
          echo ${{ env.PREV_TAG_NAME }}..${{ env.TAG_NAME }}   
          git log ${{ env.PREV_TAG_NAME }}..${{ env.TAG_NAME }} --pretty=format:"%h %s by %an" > CHANGELOG.md
          ls -l dist/
      - name: Create Release
        run: gh release create ${{ github.ref_name }} dist/*.tar.gz --title "Release ${{ github.ref_name }}" --notes-file CHANGELOG.md
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---  

## Conclusion  

This workflow streamlines the deployment process by automating build creation, versioning, and changelog generation. Each environment gets its own dedicated build, ensuring proper separation and accurate testing. By using GitHub Action Secrets, environment-specific configurations remain secure, and every release includes artifacts for both UAT and PROD.  

Oh, and one final discovery—while generating changelogs, we realized that *some developers* might need a refresher on writing meaningful commit messages. Turns out, "fix stuff" and "update" aren't the most helpful descriptions when trying to track changes! 😅