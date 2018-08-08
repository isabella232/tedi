[![Build Status](https://travis-ci.org/elastic/tedi.svg?branch=master)](https://travis-ci.org/elastic/tedi)

# Tedi: A Template Engine for Docker Images
Tedi is a tool for building Docker images for your project. It adds a templating
layer to your Dockerfile and any other files you choose. It then renders
those templates to a build context, and arranges for Docker to build it.

## Usage

Tedi is a CLI tool written in Python 3.6. It can be installed with Pip or run
as a Docker container.

If you are interested in using Tedi but not in developing it, Docker is the
recommended approach.

### Running in Docker

``` shell
TEDI=docker.elastic.co/tedi/tedi:0.4
docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock -v $PWD:/mnt $TEDI build
```

* Because Tedi is itself a Docker client, we mount the Docker socket inside
  the container.
* We also mount the directory of whatever project we want to build on `/mnt`,
  where Tedi expects to find it. In particular, it expects to a find a `tedi.yml`
  there (see below).

#### Controlling verbosity

Verbose output, including the output from the Docker build, can be requested by
setting the `TEDI_DEBUG` environment variable to `true`:

``` shell
docker run -e TEDI_DEBUG=true [...] tedi build
```

#### Other commands
You can explore other commands via the built-in help:

``` shell
docker run --rm -it tedi --help
```

### Declaring `tedi.yml`

To build a Docker image for your project, create a file within your project at
`tedi/tedi.yml`.

Like this:

``` yaml
# Tedi can produce multiple images for a single project.
# Declare them under the "images" key.
images:
  elasticsearch-full:
    # Aliases will cause the image to have several names. Having both a short
    # local name and a fully qualified name in a registry is particularly handy.
    aliases:
      - elasticsearch
      - docker.elastic.co/elasticsearch/elasticsearch

    # Differences between images can be expressed with "facts". Facts are used
    # for variable expansion in template files, notably the Dockerfile template.
    facts:
      image_flavor: full

  elasticsearch-oss:
    facts:
      image_flavor: oss
    aliases:
      - docker.elastic.co/elasticsearch/elasticsearch-oss

# Global facts apply to all images.
facts:
  # Some facts have explicit support within Tedi. Setting the "elastic_version"
  # fact will arrange for image tags to be versioned correctly.
  elastic_version: 6.3.1

# Asset sets declare files that need be acquired (downloaded) before building
# the image. Declaring different asset sets is a good way to build different
# variants of your project without having to have a bunch of conditionals in
# your templates.
asset_sets:

  # The "default" asset set is special and will be used unless you specify
  # otherwise.
  default:
    # Global facts can be used in asset declarations.
    - filename: elasticsearch-{{ elastic_version }}.tar.gz
      source: https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-{{ elastic_version }}.tar.gz

  # Alternative asset sets can be used with the "--asset-set" CLI flag.
  remote_snapshot:
    - filename: elasticsearch-{{ elastic_version }}.tar.gz
      source: https://snapshots.elastic.co/downloads/elasticsearch/elasticsearch-{{ elastic_version }}-SNAPSHOT.tar.gz

```

### Build context files
Add any other files that are needed to complete your image to the `tedi`
directory. They will be provided as the Docker build context, via a staging
area in the hidden directory `.tedi`.

Both plain files and Jinja2 templates are supported.

The contents of `.tedi` are temporary and can be regenerated. You'll likely
want to add it to your `.gitignore` file.

#### Plain files
Any plain files, including arbitrarily deep directory structures will be copied
into the Docker build context.

#### Jinja2
Any files with a `.j2` extension will be rendered through the Jinja2 template
engine before being added to the Docker build context. Generally, you will want
to create (at least) a Dockerfile template at `tedi/Dockerfile.j2`.

The template engine will expand variables from the _facts_ defined in `tedi.yml`.

## Development

Tedi is written in Python 3.6 with tests in pytest. Some type annotations can be
found sprinkled around, for consumption by [mypy](http://mypy-lang.org/).

There's a [seperate README](./tedi/README.md) covering some of the design and
implementation details.

### Initial setup

``` shell
git clone git@github.com:elastic/tedi.git
cd tedi
virtualenv --python=python3.6 venv
. venv/bin/activate
pip install -e .
```

### Running the tests

``` shell
python setup.py test
```

#### Test Watcher

A small wrapper is provided to run a fast suite of tests continuously while
doing Test Driven Development. It uses pytest-watch with some options to skip
slow things like coverage reporting and mypy type checking.
``` shell
python setup.py testwatch
```
