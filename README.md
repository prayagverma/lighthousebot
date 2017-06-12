# Lighthouse CI

This repo contains the frontend and backend for the Lighthouse CI server.

## Auditing a Github Pull Request

Lighthouse can be setup as part of your CI on Travis. As new pull requests come in, the **Lighthouse CI tests the changes and reports back the new score**.

To audit pull requests, do the following:

### 1. Initial setup

First, add [lighthousebot](https://github.com/lighthousebot) as a collaborator on your repo. Lighthouse CI uses an OAuth token scoped to the `repo` permission in order to update the status of your PRs and post comments on the issue.

### 2. Deploy the PR changes

In `.travis.yml`, add an  `after_success` that deploys the PR changes to a staging server. Since this is different for every hosting environment, it's left to the reader to figure out the details on doing that.

```
after_success:
  - ./deploy.sh # TODO(you): deploy the PR changes to your staging server.
```

> **Tip:** if you're using Google App Engine, check out [`deploy_pr_gae.sh`](https://github.com/GoogleChrome/chromium-dashboard/blob/master/travis/deploy_pr_gae.sh), which shows how to install the GAE SDK and deploy PR changes programmatically.

**Why a staging server? Can I use localhost?**
Yes, but we recommend that you deploy the PR to a real staging server instead of running a local server on Travis. The reason is that a staging environment will be more accurate and reflect your production setup. As an example, Lighthouse performance numbers will be more realistic.

### 3. Call runLighthouse.js

> Get this script from: `yarn add --dev https://github.com/ebidel/lighthouse-ci#0.0.1`

The last step in `after_success` is to call [`runLighthouse.js`][runlighthouse-link]:

```
after_success:
  - ./deploy.sh # TODO(you): deploy the PR changes to your staging server.
  - node runLighthouse.js https://staging.example.com
```

When Lighthouse is done auditing the URL, the CI will post a comment to the pull
request containing the updated scores:

<img width="779" alt="Lighthouse Github comment" src="https://user-images.githubusercontent.com/238208/27057277-5282fcca-4f80-11e7-8bbe-73117f0768d0.png">

**Note**: turn that off by passing the `--no-comment` flag.

#### Failing a PR when it drops your Lighthouse score

Lighthouse CI can prevent PRs from being merged when the overall score falls below
a specified value (`--score=96`):

```
after_success:
  - ./deploy.sh # TODO(you): deploy the PR changes to your staging server.
  - node runLighthouse.js --score=96 https://staging.example.com
```

<img width="779" src="https://user-images.githubusercontent.com/238208/26909890-979b29fc-4bb8-11e7-989d-7206a9eb9c32.png">

**Options:**

```sh
$ node runLighthouse.js -h

Usage:
runLighthouse.js [--score=<score>] [--no-comment] [--runner=chrome,wpt] <url>

Options:
  --score      Minimum score for the pull request to be considered "passing".
               If omitted, merging the PR will be allowed no matter what the score. [Number]

  --no-comment Doesn't post a comment to the PR issue summarizing the Lighthouse results. [Boolean]

  --runner     Selects Lighthouse running on Chrome or WebPageTest. [--runner=chrome,wpt]

  --help       Prints help.

Examples:

  Runs Lighthouse and posts a summary of the results.
    runLighthouse.js https://example.com

  Fails the PR if the score drops below 93. Posts the summary comment.
    runLighthouse.js --score=93 https://example.com

  Runs Lighthouse on WebPageTest. Fails the PR if the score drops below 93.
    runLighthouse.js --score=93 --runner=wpt --no-comment https://example.com
```

## Running on WebPageTest instead of Chrome

By default, `runLighthouse.js` runs your PRs through Lighthouse hosted in the cloud. As an alternative, you can test on real devices using the WebPageTest integration:

```
node runLighthouse.js --score=96 --runner=wpt https://staging.example.com
```

At the end of testing, your PR will be updated with a link to the WebPageTest results containing the Lighthouse report!

## Source & Components

This repo contains several different pieces for the Lighthouse CI: a backend, frontend, and frontend UI.

### UI Frontend
> Quick way to try Lighthouse: https://lighthouse-ci.appspot.com/try

#### Relevant source:

- `frontend/public/` - UI for https://lighthouse-ci.appspot.com/try.

### CI server (frontend)
> Server that responds to requests from Travis.

REST endpoints:
- `https://lighthouse-ci.appspot.com/run_on_chrome`
- `https://lighthouse-ci.appspot.com/run_on_wpt`

#### Example

**Note:** this is what `runLighthouse.js` does for you.

```
POST https://lighthouse-ci.appspot.com/run_on_chrome
Content-Type: application/json

{
  testUrl: "https://staging.example.com",
  minPassScore: 96,
  addComment: true,
  repo: {
    owner: "<REPO_OWNER>",
    name: "<REPO_NAME>"
  },
  pr: {
    number: <PR_NUMBER>,
    sha: "<PR_SHA>"
  }
}
```

#### Relevant source:

- [`frontend/server.js`](https://github.com/ebidel/lighthouse-ci/blob/master/frontend/server.js) - server which accepts Github pull requests and updates the status of your PR.

### CI backend (builder)
> Server that runs Lighthouse against a URL, using Chrome.

REST endpoints:
- `https://lighthouse-ci.appspot.com/ci`

#### Example

**Note:** this is what `runLighthouse.js` does for you.

```sh
curl -X POST \
  -H "Content-Type: application/json" \
  --data '{"format": "json", "url": "https://staging.example.com"}' \
  https://builder-dot-lighthouse-ci.appspot.com/ci
```

#### Relevant source:

Contains example Dockerfiles for running Lighthouse using [Headless Chrome](https://developers.google.com/web/updates/2017/04/headless-chrome) and full Chrome. Both setups us [Google App Engine Flexible containers](https://cloud.google.com/appengine/docs/flexible/nodejs/) (Node).

- [`builder/Dockerfile.nonheadless`](https://github.com/ebidel/lighthouse-ci/blob/master/builder/Dockerfile.nonheadless) - Dockerfile for running full Chrome.
- [`builder/Dockerfile.headless`](https://github.com/ebidel/lighthouse-ci/blob/master/builder/Dockerfile.headless) - Dockerfile for running headless Chrome.
- `builder/server.js` - The `/ci` endpoint that runs Lighthouse.

## Running your own CI server

TODO

- Add a `CI_HOST` parameter to Travis settings.

## Development

Initial setup:

1. Ask an existing dev for the oauth2 token. If you need to regenerate one, see below.
- Create `frontend/.oauth_token` and copy in the token value.

Run the dev server:

    yarn start

This will start a web server and use the token in `.oauth_token`. The token is used to update PR status in Github.

Follow the steps in [Testing a Github PR](#testing-a-github-pr) for setting up
your repo.

Notes:

- If you want to make changes to the builder, you'll need [Docker](https://www.docker.com/) and the [GAE Node SDK](https://cloud.google.com/appengine/docs/flexible/nodejs/download).
- To make changes to the CI server, you'll probably want to run [ngrok](https://ngrok.com/) so you can test against a local server instead of deploying for each change.

##### Generating a new OAuth2 token

If you need to generate a new OAuth token:

1. Sign in to the `[lighthousebot](https://github.com/lighthousebot) Github account. The credentials are in the usual v password tool.
2. Visit personal access tokens: https://github.com/settings/tokens.
3. Regenerate the token. **Important**: this invalidates the existing token so other developers will need to be informed.
4. Update token in `frontend/.oauth_token`.

#### Deploy

Deploy the frontend:

    ./deploy.sh YYYY-MM-DD frontend

Deploy the CI builder backend:

    ./deploy.sh YYYY-MM-DD builder

## FAQ

##### Why not deployment events?

Github's [Deployment API](https://developer.github.com/v3/repos/deployments/) would
be ideal, but it has some downsides:

- Github Deployments happen __after__ a pull is merged. We want to support blocking PR
merges based on a LH score.
- We want to be able to audit changes as they're add to the PR. `pull_request`/`push` events are more appropriate for that.

##### Why not a Github Webhook?

The main downside of a Github webhook is that there's no way to include custom
data in the payload Github sends to the webhook handler. For example, how would
Ligthhouse know what url to test? With a webhook, the user also has to setup it
up and configure it properly.

Future work: Lighthouse CI could define a file that developer includes in their
repo. The CI endpoint could pull a `.lighthouse_ci` file that includes meta
data `{minLighthouseScore: 96, testUrl: 'https://staging.example.com'}`. However,
this requires work from the developer.

[runlighthouse-link]: https://github.com/GoogleChrome/chromium-dashboard/blob/master/travis/runLighthouse.js