# aeraki.net

The [aeraki.net](https://aeraki.net) website, built using [Hugo](https://gohugo.io) and hosted on [Netlify](https://www.netlify.com/).

## Build

To build and serve the site, you'll need the latest [LTS release][] of **Node**.
Like Netlify, we use **[nvm][]**, the Node Version Manager, to install and
manage Node versions:

```console
$ nvm install --lts
```

### Setup

 1. Clone this repo.
 2. From a terminal window, change to the cloned repo directory.
 3. Get NPM packages and git submodules, including the the [Docsy](https://www.docsy.dev/) theme:
    ```console
    $ npm install
    ```

### Build or serve the site

To locally serve the site at [localhost:1313 ](http://localhost:1313), run the following command:

```console
$ npm run serve
```

To build and check links, run these commands:

```console
$ npm run build
$ npm run check-links
```

You can also locally serve using [Docker](https://docker.com):

```console
$ make docker-serve
```

## Contributing

We welcome issues and PRs! For details, see [Contributing](https://github.com/aeraki-mesh/aeraki/blob/master/CONTRIBUTING.md).

If you submit a PR, Netlify will automatically create a [deploy preview](https://docs.netlify.com/site-deploys/deploy-previews/) so
that you can view your changes. Once merged, Netlify automcatically deploys to
the production site [aeraki.net](https://aeraki.net).
