
**Docker Compose** is used to run multiple containers as a single service.

>For example, suppose you had an application which required NGNIX and MySQL, you could create one file which would start both the containers as a service without the need to start each one separately


```bash
docker compose COMMAND
```




> [!note]
> The `-f` flag is optional. If you don’t provide this flag on the command line, Compose traverses the working directory and its parent directories looking for a `compose.yaml` or `docker-compose.yaml` file.

### Use `-f` to specify the name and path of one or more Compose files

- Specifying a path to a single Compose file
	Use the `-f` flag to specify a path to a Compose file that is not located in the current directory. Suppose you have a `compose.yaml` file in a directory called `sandbox/rails`.

```bash
docker compose -f ~/sandbox/rails/compose.yaml
```

- Specifying multiple Compose files
	Docker compose combines multiple configuration files into a single configuration in the order files are supplied. 

```bash
docker compose -f compose.base.yml -f compose.override.yml
```

When you use multiple Compose files, all paths in the files are relative to the first configuration file specified with `-f`. This is required because override files need not be valid Compose files. Override files can contain small fragments of configuration.

You can use the `--project-directory` option to override the base path.

### Understand the extends configuration

Docker Compose’s `extends` keyword enables the sharing of common configurations among different files. 

Keep in mind that `volumes_from` and `depends_on` are never shared between services using `extends`. These exceptions exist to avoid implicit dependencies; you always define `volumes_from` locally. This ensures dependencies between services are clearly visible when reading the current file. Defining these locally also ensures that changes to the referenced file don’t break anything.

==docker-compose.yml==

```yaml
services:
  web:
    extends:
      file: common-services.yml
      service: webapp
    environment:
      - DEBUG=1
    cpu_shares: 5
    
```

==common-services.yml==

```yaml
services:
  webapp:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - "/data"
```

We can extend a service locally in the compose file.

```yaml
services:
  web:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - "/data"
    environment:
      - DEBUG=1
    cpu_shares: 5
    
  important_web:
    extends: web
    cpu_shares: 10
```



### Adding and overriding configuration

If a configuration option of same service is defined in both Compose files, the new value _replaces_ or _extends_ the previous value.

- For single-value options like `image`, `command` or `mem_limit`, the new value replaces the old value.
- For the **multi-value options** `ports`, `expose`, `external_links`, `dns`, `dns_search`, and `tmpfs`, Compose concatenates both sets of values.
- In the case of `environment`, `labels`, `volumes`, and `devices`, Compose “merges” entries together with locally-defined values taking precedence.

==original service:==

```yaml
services:
  myservice:
    command: python app.py
    expose:
      - "3000"
    environment:
	  - FOO=original
	  - BAR=original
	volumes:
	  - ./original:/foo
	  - ./original:/bar
```

==local service:==

```yaml
services:
  myservice:
    command: python otherapp.py
    expose:
      - "4000"
      - "5000"
    environment:
      - BAR=local
      - BAZ=local
    volumes:
      - ./local:/bar
      - ./local:/baz
```

==result:==

```yaml
services:
  myservice:
    command: python otherapp.py
    expose:
      - "3000"
      - "4000"
      - "5000"
    environment:
	  - FOO=original
	  - BAR=local
      - BAZ=local
	volumes:
	  - ./original:/foo
	  - ./local:/bar
      - ./local:/baz
```
