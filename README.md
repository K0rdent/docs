# K0rdent Documentation

The home of the consolidated documentation for K0rdent sponsored by Mirantis.
K0rdent may be referred to as Project 2A or Project 0x2A in the documentation
due to legacy reasons.

[K0rdent Docs](https://k0rdent.github.io/docs)

This project utilises Mkdocs with the Material theme and Mermaid for
diagrams. Currently the docs are published using github actions on github pages
from the branch gh-pages.

Development is tracked under [K0rdent](https://github.com/orgs/Mirantis/projects/8) on github.

The related K0rdent repositories can be found as follows:
 * [HMC](https://github.com/k0rdent/kcm)

## Project layout

    mkdocs.yml    # The configuration file.
    docs/
        index.md  # The documentation homepage.
        stylesheets  # CSS stylesheets to control look and feel
        assets  # Images and other served material
        ...       # Other markdown pages, images and other files.

## Setting up MKdocs and dependancies

1. Setup python Virtual Environment

    ```bash
    python3 -m venv ./mkdocs
    source ./mkdocs/bin/activate
    ```

2. Install MkDocs

    ```bash
    pip install mkdocs
    ```

3. Install plugins

    ```bash
    pip install mkdocs-mermaid2-plugin
    pip install mkdocs-material
    pip install markdown-callouts
    ```

## Run MKdocs for dev

* Start the live-reloading docs server.

    ```bash
    mkdocs serve
    ```

For full documentation visit [mkdocs.org](https://www.mkdocs.org).

## MKdocs Commands

* `mkdocs new [dir-name]` - Create a new project.
* `mkdocs build` - Build the documentation site.
* `mkdocs -h` - Print help message and exit.

# Documentation Standards

By default, we follow the [Kubernetes documentation style guide](https://kubernetes.io/docs/contribute/style/style-guide/). 

## Header Capitalization

All header text should be capitalized.

## Referencing Kubernetes nested resources

Please use the dot notation.  So in the following:

```
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
```

To refer to the `name` field, please use `.metadata.name` and not `name`.
