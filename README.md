# Documentation for Neurokraken

This is the source for building the Documentation for Neurokraken, a flexible, open-source, python-based behavior platform for Neuroscience 

You can find the actual documentation at https://Neurokraken.github.io/Neurokraken-book

## running a jupter book

- guide and documentation: https://jupyterbook.org/en/stable/start/your-first-book.html
- to install jupyter book in your current python environment (with mermaid support): `pip install "jupyter-book<2.0.0"` and `pip install sphinxcontrib-mermaid`
- to build this book, from the repositories root folder: `jupyter-book build --all .`

## Create github pages

The workflow to create github pages for the documentation can be found at https://jupyterbook.org/v1/start/publish.html

The curent published pages can be updated with the following command after having built the current version:

`ghp-import -n -p -f _build/html`