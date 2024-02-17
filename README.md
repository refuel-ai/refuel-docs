# refuel-docs

## Local Testing

```bash
# clone the repo
git clone https://github.com/refuel-ai/refuel-docs.git
# change the directory to refuel-docs
cd refuel-docs
# get the submodules
git submodule update --init --recursive 
# install the dependencies
pip install mkdocstrings mkdocstrings-python mkdocs-jupyter mkdocs-table-reader-plugin
# start the server
mkdocs serve
```
