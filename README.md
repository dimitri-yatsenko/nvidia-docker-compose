# `nvidia-docker-compose`
`nvidia-docker-compose` is a simple python script that wraps [`docker-compose`](https://docs.docker.com/compose/) to allow `docker-compose` to work with GPU enabled Docker containers as made available with [`nvidia-docker`](https://github.com/NVIDIA/nvidia-docker)!

## Dependencies
`nvidia-docker-compose` requires following dependencies to be installed on the system:
* Docker engine
* `nvidia-docker`
* `docker-compose`

also it depends on `PyYAML` Python package which would be installed automatically during the installation step described below.

## Installing
To install the script, simply run:

```bash
$ pip install nvidia-docker-compose
```

If you are using system Python, it may be necessary to run the above command with `sudo` upfront.

## Using `nvidia-docker-compose`
The `nvidia-docker-compose` is a drop-in replacement for the `docker-compose`. Simply run as you would run `docker-compose`:
```bash
$ nvidia-docker-compose ...
```
Depending on how your system is configured, you may need to run the script with `sudo` (i.e. if you usually need `sudo` to run `docker`, you will need `sudo`).

Running `nvidia-docker-compose` generates a new YAML config file `nvidia-docker-compose.yml` locally. It is safe to delete this file in-between usages and I recommend you add this to your `.gitignore` file if you are going to use `nvidia-docker-compose` withint a Git repository. Once generated, you can also use the `nvidia-docker-compose.yml` directly to launch GPU enabled containers directly with the standard `docker-compose`. You can do so as:
```bash
$ docker-compose -f nvidia-docker-compose.yml ...
```

## Running flexibly on multi-GPU setup

When working on multi-GPU setup, you would often want to run separate container for each GPU or at least limit the visibility of GPUs to only specific Docker containers. If you are not afraid to dig in, you would discover that you can control visibility of GPUs to each container by selectively including `/dev/nvidia*` under the `devices` section (i.e. `/dev/nvidia0` for the first GPU, and so on) . However, doing this manually would mean that you will have to interfere with the function of `nvidia-docker` and `nvidia-docker-compose`, and previously there was no natural way to specify which service in the `docker-compose.yml` should be run with which GPUs. This is further complicated by the fact that different machine would have different numbers of GPUs, and thus keeping a service with `/dev/nvidia4` under `devices` section on a 2 GPU machine could cause an error.

### Specifying GPU target
*New from version 0.4.0*
`nvidia-docker-compose` now allows you to specify which GPU a specific service should be run with by including `/dev/nvidia*` under the `devices` heading. As in the following

```yaml
version: "2"
services
  process1:
    image: nvidia/cuda
    devices:
      - /dev/nvidia0
  process2:
    image: nvidia/cuda
    devices:
      - /dev/nvidia1
      - /dev/nvidia2
```

The service `process1` will now only see the first GPU (`/dev/nvidia0`) while the service `process2` will see second and third GPU (`/dev/nvidia0` and `/dev/nvidia1`). If you don't specify any `/dev/nvidia*` under devices section, the service will automatically see all available GPUs as have been the case previously.

Although this feature will allow you to finely control which service sees which GPU(s), it is still rather inflexible as will require you to adjust the `docker-compose.yml` per computer device. This is precisely where the Jinja2 templating can help you! 

### Using [Jinja2](http://jinja.pocoo.org/) in `docker-compose.yml` file 
*New from version 0.4.0*

To support the relatively common use case of wanting to launch as many compute containers (with the same configuration) as the number of GPUs avaialble on the target machine, `nvidia-docker-compose` now supports use of [Jinja2](http://jinja.pocoo.org/) in the `docker-compose.yml.jinja` file. Combined with the ability to specify GPU targeting, you can now write `docker-compose` config that adapts flexibility to the GPU availability. For an example if you prepare `docker-compose.yml.jinja` (note the `.jinja` suffix):

```yaml
version: "2"
services:
  {% for i in range(N_GPU) %}
  notebook{{i}}:
    image: eywalker/tensorflow:cuda
    ports:
      - "300{{i}}:8888"
    devices:
      - /dev/nvidia{{i}}
    volumes:
      - ./notebooks:/notebooks
  {% endfor %}
```

and run `nvidia-docker-compose`, it will automatically pick up the Jinja template, process it and expand it to the following `docker-compose.yml`:

```yaml
version: "2"
services:
  notebook0:
    image: eywalker/tensorflow:cuda
    ports:
      - "3000:8888"
    devices:
      - /dev/nvidia0
    volumes:
      - ./notebooks:/notebooks
  notebook1:
    image: eywalker/tensorflow:cuda
    ports:
      - "3001:8888"
    devices:
      - /dev/nvidia1
    volumes:
      - ./notebooks:/notebooks
  notebook2:
    image: eywalker/tensorflow:cuda
    ports:
      - "3002:8888"
    devices:
      - /dev/nvidia2
    volumes:
      - ./notebooks:/notebooks
```
on a 3 GPU machine. The Jinja variable `N_GPU` automatically reflects the available number of the GPUs on the system. This `docker-compose.yml` is then processed by `nvidia-docker-compose` just like any other config file to launch GPU enabled containers.

## How it works
`nvidia-docker-compose` is a simple Python script that performs two actions:
* parse `docker-compose` config file (defaults to `docker-compose.yml`) and creates a new config YAML `nvidia-docker-compose.yml` with configurations necessary to run GPU enabled containers. Configuration parameters are read from `nvidia-docker-plugins`.
* run `docker-compose` with the newly generated config file `nvidia-docker-compose.yml`

