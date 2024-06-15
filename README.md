# Web service workshop

This workshop aims students of EPITA (promotion 2025) of the SIGL specialisation in the context of the course of SOCRA.

In this workshop, you will learn how to:

- create a web service using [NodeJS](https://nodejs.org/en/) and [Express library](https://expressjs.com/)
- deploy a web service using Docker and your Scaleway instance
- consume a web service from client (in this case your reactive frontend from last workshop)

> Note: for this workshop, you will NOT use any databases.
> This will be the subject of another workshop.

## Step 0: Setup your web-service from provided template

From your group's repository:

- create a `web-service` folder (same level as `frontend`)
- copy the template code inside `web-service` folder [EPITA-SIGL2025-SOCRA/web-service-template](https://github.com/EPITA-SIGL2025-SOCRA/web-service-template)

> Warning: make sure you also copy both `.gitignore` and `.nvmrc` hidden files

You should have the following structure in your project repository:

```sh
web-service/
├── .gitignore # contains files you don't want to push
├── .nvmrc  # make sure you run `nvm use` to match node 19
├── Dockerfile # your dockerfile for the web-service
├── package.json  # where all node dependencies are set
├── package-lock.json # freezes exact dependencies
└── src
    ├── data.json # some tractor data in JSON format
    ├── database.mjs # helper function to query the data JSON to get tractors around a GPS position
    ├── distance.mjs # contains code to compute distance between 2 GPS position
    └── server.mjs # your web service code
frontend/ # ... same as previous workshop

```

Follow the [web-service-template README.md](https://github.com/EPITA-SIGL2025-SOCRA/web-service-template) to make sure your application template runs correctly.

## Step 1: **Challenge** Create the tractor service

**Objective**: exposes a web service that returns tractors around an input GPS position (`latitude` and `longitude` as floating numbers)

More precisely, if a you want to query **tractors** around the position with `latitude=48.8583145` and `longitude=2.292334` on a `50km` radius, you will:

- send an http `GET` request
- at the endpoint `http://localhost:3000/v1/tractors?latitude=48.8583145&longitude=2.292334&radius=50`

> Note: to visualize the lat long GPS coordinate, you can navigate to https://maps.google.com and paste
> `48.8583145 2.292334` in the search field
> ![eiffel-tower](docs/eiffel-tower-gps-maps.png)

- Your turn to play! You just have to implement the `v1/tractor` code inside `web-service/src/server.mjs`
- Make use of the `getTractorsAround` (inside the `web-service/src/database.mjs` file) function created for you
  - `getTractorsAround` will look for tractors in the `web-service/src/data.json` file and return the nearest tractor
    around the given GPS postion.
  - code should be commented to help you out

## Step 2: Deploy your web service

**Objective**: have your web service deployed at [https://api.groupXX.socra-sigl.fr](https://api.groupXX.socra-sigl.fr) (groupXX replaced by your groupe number) and integrated to your CD pipeline

### Create a new github workflow file for your web service

Now that you have a runnning docker setup for your node web service, add a new job to your current CD workflow.

From `.github/workflows/web-service-cd.yml`, making sure to replace `YOUR_ORG` by your organization/user's name on github (we consider that your project name is `sotracteur`):

```yaml
name: Web service CD

on:
  push:
    branches: ["main"]
    # See. https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#patterns-to-match-file-paths
    paths: ["web-service/**"]

  workflow_dispatch:

jobs:
  build-and-deploy-web-service:
    if: inputs.image == ''
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: build web-service docker image
        working-directory: web-service
        run: |
          docker build -t ghcr.io/YOUR_ORG/sotracteur/sotracteur-web-service:${{ github.sha }} .
          docker login ghcr.io -u ${{ secrets.DOCKER_USER }} -p ${{ secrets.DOCKER_PASSWORD }}
          docker push ghcr.io/YOUR_ORG/sotracteur/sotracteur-web-service:${{ github.sha }}
      - name: deploy on scaleway via SSH
        uses: appleboy/ssh-action@v0.1.9
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            docker login ghcr.io -u ${{ secrets.DOCKER_USER }} -p ${{ secrets.DOCKER_PASSWORD }}
            docker stop sotracteur-web-service 2> /dev/null && docker rm sotracteur-web-service 2> /dev/null
            docker run -d --network web \
              --name sotracteur-web-service \
              --label "traefik.http.routers.sotracteur-web-service.rule=Host(\`api.groupXX.socra-sigl.fr\`)" \
              --label "traefik.http.routers.sotracteur-web-service.tls=true" \
              --label "traefik.http.routers.sotracteur-web-service.tls.certresolver=letsencrypt" \
              --label "traefik.enable=true" \
              --label "traefik.docker.network=web" \
              ghcr.io/YOUR_ORG/sotracteur/sotracteur-web-service:${{ github.sha }}
```

It's very similar to your frontend build job, expect:

- `working-directory` is `web-service`
- `image-name` and `container-name` are `sotracteur-web-service`
- `label` in traefik is `api.groupXX.socra-sigl.fr`
- `paths: [ "web-service/**" ]` is there to make sur that the web-service is deployed **only if some web-service files have changed**.

You can also add a new `paths: ["frontend/**"]` in your other github workflow from previous workshop, to trigger frontend deployment **only when frontend file have changes**.

Commit/push your changes, and you should trigger the web-service CD workflow.

After few minutes, you should be able to access your web API on [https://api.groupXX.socra-sigl.fr](https://api.groupXX.socra-sigl.fr)

## Step 3: Integrate web service to Sotracteur's frontend

**Objective**: Your frontend fetch data served by your new web service.

### Setup for different domain names (localhost vs api.groupXX.socra-sigl.fr)

**Problem**: Frontend need to query `localhost:3000` when running on your local machine and `api.groupXX.socra-sigl.fr` when running on production.

To solve this issue, you will create a new `public/api-info.json` file in your **frontend** code:

```js
# inside frontend/public
{
  "domain": "http://localhost:3000"
}
```

- and inside your frontend's deployment github workflow file, you will add a new step **before** building the frontend image (replace `XX` by your group number):

```yml
- name: Set correct api domain in frontend image
working-directory: frontend
run: |
    echo '{ "domain": "https://api.groupXX.socra-sigl.fr" }' > public/api-info.json
```

This way, your app have the following config:

- in dev mode (e.g. on `localhost:5173`): the `domain` value inside `frontend/public/api-info.json` will be `http://localhost:3000`
- in production mode (e.g. on `https://groupXX.socra-sigl.fr`): the `domain` value inside `frontend/public/api-info.json` will be `https://api.groupXX.socra-sigl.fr`

### Helper method to query your web-service

**Objective**: write a helper method to query sotracteur's web service, no matter which environments we are on (dev or production).

- Add a new `frontend/src/api.js` file with :

```js
function makeHeaders() {
  return {
    "Content-Type": "application/json",
  };
}

async function makeUrl(path) {
  const slashlessPath = path.startsWith("/") ? path.slice(1) : path;
  const domainResponse = await fetch("/api-info.json");
  const { domain } = await domainResponse.json();
  const domainSlash = domain.endsWith("/") ? domain : `${domain}/`;
  return `${domainSlash}${slashlessPath}`;
}

export async function httpGET(path) {
  const url = await makeUrl(path);
  const headers = makeHeaders();
  const response = await fetch(url, { headers });
  const data = await response.json();
  return data;
}
```

The function `httpGET` will make a `GET` request to the `path` given as argument. This function will
make sure the correct web service URL will be chosen depending on which env user is currently on (dev or production).
It will do so by reading **at runtime** the `/api-info.json` file exposed by the frontend.

> Note: this part is using asynchronous code in JavaScript. If you are not familiar with it,
> there are really good guides on MDN:
>
> - [Introducing asynchronous JavaScript](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Asynchronous/Introducing)
> - [how to use Promises and async/await](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Asynchronous/Promises)

### Getting tractors with React.useEffect

[React.useEffect](https://react.dev/reference/react/useEffect) is a nice API from React that can trigger
some code when some component's properties has changed.

We will use this in our code to call our web service when browsing tractors.

Adapt your frontend's react component that rendered tractors (named `Tractors` in the example below) by adding the following `React.useEffect` code:

```jsx
// Make sure to import httpGET method
// e.g. import { httpGET } from "./api"
// if your file is at the same level of the src/api.js one

function TractorsNear() {
  const position = { latitude: 48.8583145, longitude: 2.292334 };
  const [tractors, setTractors] = React.useState([]);
  const [loading, setLoading] = React.useState(true);

  // Define an asynchronous function that calls the web-service
  // using the httpGET method from previous step
  async function searchTractorNear({ latitude, longitude }, radius) {
    try {
      const path = `/v1/tractors?latitude=${latitude}&longitude=${longitude}&radius=${radius}`;
      const tractorsFromApi = await httpGET(path);
      setTractors(tractorsFromApi);
    } catch (error) {
      console.error("error fetching web service: ", error);
      setTractors([]);
    } finally {
      setLoading(false);
    }
  }

  // the function as first parameter will be triggered only once.
  // Because we give an empty array of dependencies as second parameter.
  React.useEffect(() => {
    // search from the hardcoded position we set
    // in a 50 km radius
    searchTractorNear(position, 50);
  }, []);

  return loading ? (
    <span>chargement...</span>
  ) : (
    tractors.map((tractor) => (
      <div>
        {tractor.raison_social} disponible à {tractor.distance}km
      </div>
    ))
  );
}
```

Of course, you can do better than just rendering text with the `raison_social` with `distance`.
You can re-use your card component from before and adapt it with the tractor data fields returned
by the web-service.

That would be all for this workshop, you should display some tractors provided by your web-service, congrats!
