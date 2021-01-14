# motivation
A simple template for deploying containerized panel apps behind a reverse proxy.

## Note: if you prefer a stripped-down version to work from, check out [simple-doginx](simple-doginx).
This version is simplified in a few ways:
1. No `git submodules`
2. No system packages

That might be right version for you if you're into it.
But this version has some key details for "real world" situations. Notably, submodules and system packages.

# acknowledgements

This is inspired by Marc's [template](https://github.com/MarcSkovMadsen/awesome-analytics-apps-template) 
and also draws heavily on [this post from testdriven.io](https://testdriven.io/blog/dockerizing-flask-with-postgres-gunicorn-and-nginx/).

The parts about Docker machine are informed by [this Real Python post](https://realpython.com/dockerizing-flask-with-compose-and-machine-from-localhost-to-the-cloud/).

# setup

Here's what you'll need:

1. Docker (v19.03.12)
2. Docker Compose (v1.23.2) 
3. Docker Machine (v0.16.1) 
4. VirtualBox

Make your first machine (I'll spare you the output, but there's a fair bit of it!):

    $ docker-machine create -d virtualbox dev

### Notice that `-d` flag? That's the `--driver` which is currently set to `virtualbox`. When we deploy to VirtualBox later, we'll change the driver.

Now check out the machine you just made:

    $ docker-machine ls

    NAME   ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER      ERRORS
    dev    -        virtualbox   Running   tcp://192.168.99.103:2376           v19.03.12   

Set it to `ACTIVE`:

    $ eval "$(docker-machine env dev)"

Now when you list your machines, you'll see an asterix:

    $ docker-machine ls

    NAME   ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER      ERRORS
    dev    *        virtualbox   Running   tcp://192.168.99.103:2376           v19.03.12   

To deactivate the machine, run:

    $ eval "$(docker-machine env --unset)"

That was a lot of prep! Now lets get to the (other) fun part.

### Here's something to remember: `docker-compose build` will build an image ontop of whichever machine is currently `ACTIVE`.

# Jumping into Panel

## Your app structure

Your panel app should have an `app.py` in its root directory which contains a [servable panel app] which is itself named `servable`. 
For example, in my `app.py` I build up a [panel.template] called `react`, and then, at the end of `app.py`, assign `servable = react.servable()`. Perhaps your servable is not a template? It might be as simple as a single [panel.row], in which case at the end of your you would simply assign:

    my_row = pn.Row(<some models>)
    servable = my_row.servable()

The app root directory should also contain a `requirements.txt` for dependencies.

## Changing the submodule

Now click "Use Template" and `git clone` the resulting repo to your local machine.

To swap in your panel app directory, first make sure you are in the root directory.
Then delete the example repo and deinit it as a submodule with: 

    $ rm -r service/web/terrain-corrector
    $ rm .gitmodules
    $ git deinit submodules

Then define a USER_NAME and REPO_NAME as shell variables:

    $ USER_NAME=<your_github_username>
    $ REPO_NAME=<your_panel_app_repo_name>

And run:
    
    $ git submodule init github.com/$USER_NAME/$REPO_NAME.git services/web/$REPO_NAME

## Checking the submodule

Before getting into Docker let's do bit of gut-checking and configuration on the app itself.

You cab now verify that your app works changing to the `services/web` directory by running:

    (env)$ panel serve $REPONAME/app.py --show

## Updating `config.toml`

Kill the server and update `services/web/settings/config.toml` accordingly:

    REPO_NAME =
    SITE_NAME =
    NUM_PROCS =
    WEBSOCKETS =

Now, from the web directory you can run:

    (env)$ python manage.py
    
You should see your app in the same manner as before. Again, kill the server.

Now you can `conda deactivate` (or equivalent) your virtual environment.

Great! Now we need your panel app's requirements in the `services/web` directory.
Assuming you have a `requirements.txt` in your panel app repo, the easiest way to do this is:

    $ cp $REPO_NAME/requirements.txt requirements.txt

(Yes, this does create an *ounce* of redundancy, because we now have the same file in two places,
but I think it's for the best.)

# The Docker Journey

## at the `services/web/` level

Now let's edit `services/web/Dockerfile`. This is pretty important and worth looking at line-by-line.
Here I'm adding some additional comments beyond those 

    # pull official base image
    FROM python:3.7.9-slim-buster

    # set work directory
    WORKDIR /usr/src/app

    # set environment variables
    ENV PYTHONDONTWRITEBYTECODE 1
    ENV PYTHONUNBUFFERED 1

    # install system dependencies
    RUN apt-get update && apt-get -y install gcc
    RUN apt-get -y install --reinstall build-essential
    RUN apt-get -y install ffmpeg
    RUN apt-get -y install zip

    # install pip dependencies
    RUN pip install --upgrade pip
    COPY ./requirements.txt /usr/src/app/requirements.txt
    RUN pip install -r requirements.txt

    # copy project
    COPY . /usr/src/app/

## at the root directory level

Now lets go back to the project root directory. (If you haven't changed the repo name, that'll be `doginx-panel-template/`).

And take a look at `docker-compose.dev.yml`:

    version: '3.7'

    services:
    web:
        build: ./services/web
        command: python manage.py
        volumes:
        - /usr/src/app/
        ports:
        - 5006:5006
        env_file: 
        - ./.env.dev

And finally the `.env.dev` file:

    DOCKER_ADDRESS='192.168.99.103'

This value should be updated to reflect the IP address of the docker-machine named `dev`.
You can find this by running `docker-machine ls` as before, or with `docker-machine inspect dev`.

### A note about environment variables vs. config.toml configuration: why IP is an ENV variable.

### Here's a very important note about volumes: mounted vs. true volumes.

## Building the container

Now, ensuring you are still in the project root directory (e.g. `doginx-panel-template/`), you can build your docker image.

Because we are using these uniquely-titled YAML file, we need to specify a filename in the command.
As such, it's probably easiest to alias this to a shorter name:

    $ alias dev-compose='docker-compose -f docker-compose.dev.yml'

Now we can build and run our development container with:

    $ dev-compose up -d --build

We should now be be able to see our panel app by navigating to `192.168.99.103:5006/app` in a web browser.
If don't see what you're expecting, you can check for errors with:

    $ dev-compose logs --follow

If you need to make any changes, you can bring down the container with:

    $ dev-compose down

Then edit the project and re-build iteratively until you are satisfied.

# Moving to DigitalOcean


