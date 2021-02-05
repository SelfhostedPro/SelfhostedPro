---
title: "Yacht Guide"
date: 2021-02-04T19:31:08-08:00
draft: false
---
# Intro
It's been a while since I've added any new content here and what better content could I do than write about how to use an app I wrote?

I'm going to be going over setting up yacht, some of the different options you have when setting it up, and a general idea of the thoughts behind some of the different pages.

# Installation
This guide is going to assume you're using the standard installation instructions from docker to get docker installed on your system. Once you've done that run the following command to setup docker.

```bash
docker run -d -p 8000:8000 -v /var/run/docker.sock:/var/run/docker.sock -v ~/yacht:/config selfhostedpro/yacht
```

This will launch Yacht and forward port 8000 on your host to it. It also mounts the docker socket so Yacht can control docker on your host. You can replace `~/yacht` in the command above with wherever you want to have the yacht configuration files mounted on your host.

We deviate from the official instructions here so that we can easily add and modify more complex docker-compose projects. 

There are some additional options you can add with `-E` in [this table](https://github.com/selfhostedpro/yacht#supported-environment-variables). Some additional options are `THEME` where you can select from `RED`, `OMV`, `DigitalOcean`, and `Default`. You can also change the logo by setting the `LOGO` environment variable to a URL that's hosting your image.

If you're using something like Authelia, you should disable authentication within yacht by setting `DISABLE_AUTH` to `True`.

Now that we've got the initial installation complete lets move into setting up the app.

# Setup

Yacht should be accessible at `https://ip.of.server:8000`. Login with the default credentials `admin@yacht.local` and `pass`. Click on your username in the top right and go to the user settings to change your username and password.

Once you're done with that, click on `Server Settings` on the left sidebar. Then go to the `Template Variables` tab. Here you can set the variables that are used in templates and add new ones.

Once those are set, go to the templates tab on the left and add the default template. Once you're ready to go, go back to `Server Settings` and export your configuration so if you need to re-install yacht for any reason you can automatically populate your settings and templates.

