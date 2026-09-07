# RSS reader Telegram bot Docker deployment

This repository provides an example of how you can set up my
[RSS reader Telegram bot](https://github.com/Electronic-Mango/rss-reader-telegram-bot)
via [Docker Compose](https://docs.docker.com/compose/).

`docker-compose.yml` will configure 4 containers and connect them in the same bridged network:
 - [MongoDB](https://www.mongodb.com/)
 - [RSS-Bridge](https://github.com/RSS-Bridge/rss-bridge)
 - [RSSHub](https://github.com/DIYgod/RSSHub)
 - bot itself

It will also configure bot's environment variables to correctly connect it with MongoDB,
and add configuration files which can be used for additional configuration of all containers.

Technically only the MongoDB and bot containers are necessary,
if you don't plan of using RSS-Bridge or RSSHub.
However, this serves as a nice example of how to deploy more complicated solution.

## Configuration

### MongoDB
The DB files will be stored in a mounted volume in `./mongo-db/data` subdirectory.
This means, that DB data is stored on the host, rather than on the container.

No additional configuration is required.

### RSS-Bridge
RSS-Bridge container can be started with `rss-bridge` Compose profile.
Without it the container won't be started.

Configuration files are stored in a mounted volume in `./rss-bridge/config` subdirectory.

Currently there's only a basic `config.ini.php`, which configures RSS-Bridge to output its errors
as HTTP codes, rather than send it in the feed itself.

You can add any additional configuration you need, custom bridges, or whitelist.
You can use the official
[RSS-Bridge documentation for Docker](https://rss-bridge.github.io/rss-bridge/For_Hosts/Docker_Installation.html)
for reference.

Docker Compose will load any custom environment variables from `./rss-bridge/rss-bridge.env` file.
The file itself is added to `.gitignore` to not track any credentials, or secrets.
For example, setting Reddit credentials for an OAuth2 access:

```bash
RSSBRIDGE_RedditBridge_app_id='<app id>'
RSSBRIDGE_RedditBridge_app_secret='<app secret>'
```


### RSSHub
RSSHub container can be started with `rss-hub` Compose profile.
Without it the container won't be started.

There are no configuration changes by default,
however you can add your own parameters by modifying `./rss-hub/.env` file.
This file stores all environment variables loaded into the RSSHub container.


### RSS reader Telegram bot
The bot itself requires some configuration to work correctly:
 - `./telegram-bot/config/custom_settings.yml` should be updated with Telegram bot token
 - `./telegram-bot/config/custom_feeds.yml` should be updated with your feed links

When using one of the self-hosting solutions keep in mind that you should use service name
instead of IP address. In this case it will be either `rss-bridge` or `rss-hub`.
Port still has to be provided.

You can use the `./telegram-bot/config/custom_settings.yml` for any additional configuration of the bot.
`docker-compose.yml` adds this file to the container as a mounted volume and sets its path via `CUSTOM_SETTINGS_PATH` environment variable.
You can check out the [RSS reader Telegram bot](https://github.com/Electronic-Mango/rss-reader-telegram-bot)
repository for more details.
This way you can change bot's configuration without the need of modifying the source code
and rebuilding bot's Docker image.

Whole `./telegram-bot/config/` subdirectory is mounted as a container volume,
so there's no need to rebuild the image if you change any files there (like the feeds YAML).

By default bot will also put there its internal logs, so you can access them without
getting into the container.

DB host parameter is already set to MongoDB container (as service name `mongo-db`),
so no additional configuration is required.

When Docker Compose is run with `--build` flag a new Docker image will be built,
using Dockerfile in git submodule containing bot's source code.

Docker Compose will load any custom environment variables from `./telegram-bot/bot.env` file.
The file itself is added to `.gitignore` to not track any credentials, or secrets.
For example, to configure bot token and allowed usernames without modifying tracked YAML file:

```bash
TELEGRAM_TOKEN='bot token'
TELEGRAM_ALLOWED_USERNAMES='comma separated,list of,allowed usernames'
```


### Docker Compose profiles
In order to start either self-hosted RSS container you have to run Compose with their profiles:
* `RSS-Bridge` starts with profile `rss-bridge`
* `RSS-Bridge` starts with profile `rss-hub`

You can start Compose with both profiles if you want to run both solutions.
Keep in mind that you also need to specify profiles when stopping containers via Compose.

More details about using Compose profiles are available in
[Using profiles with Compose](https://docs.docker.com/compose/profiles/) Docker documentation page.


## RSS reader Telegram bot source code

The source code for the bot itself is stored as a git submodule in this repository.

After cloing this repository you should update it to the correct version,
otherwise you won't have access to bot source code and won't be able to build the image:
```
git submodule update --init
```

Additionally after every update to this repository you should also update the submodule,
since it might not happen automatically:
```
git submodule update
```

When Docker Compose is run with `--build` flag a new Docker image will be built,
using Dockerfile in this submodule, so it is important the submodule is up to date.

Both this and the bot source code repositories on GitHub uses workflows to ensure,
that source code here is always up to date.


## Run the bot

You can run the bot and all necessary containers in just a few steps:

 1. Clone this repository
 1. Update the bot source code submodule with `git submodule update --init`
 1. Add your Telegram bot token to `./telegram-bot/config/custom_settings.yml` file
 1. Optionally add allowed usernames to `./telegram-bot/config/custom_settings.yml` file
 1. Add you feeds to `./telegram-bot/config/custom_feeds.yml`
 1. Optionally add any configuration to either `./rss-bridge/config` or `./rss-hub/.env` if you need it
 1. Run `docker compose up -d --build`

If you want to make any changes to configuration you can stop all containers and start them again:
```
docker compose down
# make your changes
docker compose up -d --build
```

If you haven't made any changes to the bot source code you can skip the `--build` flag.
Keep in mind, however, that without it any updates to the source code won't be present in
the container, since old version of the image will be used.
