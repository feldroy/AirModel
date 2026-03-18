# AirModel

![PyPI version](https://img.shields.io/pypi/v/AirModel.svg)

The ORM made for Air web framework, usable by non-Air projects too. Async work with Pydantic models and Air forms

* GitHub: https://github.com/feldroy/AirModel/
* PyPI package: https://pypi.org/project/AirModel/
* Created by: **[Audrey M. Roy Greenfeld](https://audrey.feldroy.com/)** | GitHub [@audreyfeldroy](https://github.com/audreyfeldroy) | PyPI [@audreyr](https://pypi.org/user/audreyr/)
* Free software: MIT License

## Features

* TODO

## Documentation

Documentation is built with [Zensical](https://zensical.org/) and deployed to GitHub Pages.

* **Live site:** https://feldroy.github.io/airmodel/
* **Preview locally:** `just docs-serve` (serves at http://localhost:8000)
* **Build:** `just docs-build`

API documentation is auto-generated from docstrings using [mkdocstrings](https://mkdocstrings.github.io/).

Docs deploy automatically on push to `main` via GitHub Actions. To enable this, go to your repo's Settings > Pages and set the source to **GitHub Actions**.

## Development

To set up for local development:

```bash
# Clone your fork
git clone git@github.com:your_username/AirModel.git
cd AirModel

# Install in editable mode with live updates
uv tool install --editable .
```

This installs the CLI globally but with live updates - any changes you make to the source code are immediately available when you run `airmodel`.

Run tests:

```bash
uv run pytest
```

Run quality checks (format, lint, type check, test):

```bash
just qa
```

## Author

AirModel was created in 2026 by Audrey M. Roy Greenfeld.

Built with [Cookiecutter](https://github.com/cookiecutter/cookiecutter) and the [audreyfeldroy/cookiecutter-pypackage](https://github.com/audreyfeldroy/cookiecutter-pypackage) project template.
