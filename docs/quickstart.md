This sketches the intended experience of a developer who sets up a copy of
`prevem` for local development.

# Install prevem

If you use `civibuild`, download the code and setup the database with one command, e.g.

```
civibuild create prevem --url http://prevem.l
```

Alternatively, you can install manually. Create an empty MySQL database. Download the code and create a config file.

```
git clone https://github.com/civicrm/prevem
cd prevem
composer install
cp app/config/parameters.yml{.dist,}
vi app/config/parameters.yml
./app/console doctrine:schema:create
```

Then setup HTTP access to the `web/` folder (per Symfony instructions). For
example, you might setup `http://prevem.l`.

# Configure users

 * For dev/test, the `security.yml` should just hard-code some example users (e.g. `demo:demo` and `dummy:dummy`).
 * For prod, you should override `security.yml` and implement real user accounts.

# Start a renderer

For testing purposes, it helps to have a local render. Launch a renderer
with the command:

```bash
./app/console renderer:poll --url 'http://dummy:dummy@prevem.l/' --name dummy --cmd examples/dummy-render.php
```

(This command will continue running indefinitely; when you're ready to stop the renderer, hit `Ctrl-C`.)

# Send a test

To ensure that job manager and renderer and working as expected, send a test
batch:

```bash
./app/console batch:create --subject 'Hello world' --text "Hello world" --render dummy --url 'http://demo:demo@prevem.l/' --out '/tmp/rendered/'

ls -l /tmp/rendered/*.png
```

# Configure CiviCRM

Once `prevem` is setup, you can connect your local CiviCRM site by running:

```
cd ~/buildkit/build/dmaster
drush cvapi setting.create prevem_url='http://demo:demo@prevem.l/'
```

