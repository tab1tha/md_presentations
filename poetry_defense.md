---
marp: true
---
**Poetry by Tambe**


---

**1. Handling installation from private repos**

with pip
```bash
pip install --index-url url-of-private-repo library-name
```
requirements.txt file if dependencies are frozen:
```
library-a==0.1.0
library-b==1.1.0
```
```
pip install -r requirements.txt
```
raises error *"No matching distribution found"*

---
**Poetry enables conditional installation**
<br/>

```
[[tool.poetry.source]]
name = "name-of-private-repo"
url = "url-of-private-repo"
secondary = true
```

Poetry will try to find the required package version in PyPI first then check the private repository if not found.

---
**2. Isolated dev dependencies**
No need to have multiple requirements.txt files, poetry's got you.
```
[tool.poetry.dependencies]
python = "^3.9"
pandas = "< 2.0.0"
openpyxl = "^3.0.9"

[tool.poetry.dev-dependencies]
black = "^23.3.0"
isort = "^5.12.0"

[build-system]
requires = ["poetry-core>=1.0.0"]
build-backend = "poetry.core.masonry.api"
```

---
**3. Publishing a package**
```python
import setuptools

with open("README.md", "r") as fh:
    long_description = fh.read()

setuptools.setup(
    name="quicksample",                     # This is the name of the package
    version="0.0.1",                        # The initial release version
    author="Aveek Das",                     # Full name of the author
    description="Quicksample Test Package for SQLShack Demo",
    long_description=long_description,      # Long description read from the the readme file
    long_description_content_type="text/markdown",
    packages=setuptools.find_packages(),    # List of all python modules to be installed
    classifiers=[
        "Programming Language :: Python :: 3",
        "License :: OSI Approved :: MIT License",
        "Operating System :: OS Independent",
    ],                                      # Information to filter the project on PyPi website
    python_requires='>=3.6',                # Minimum version requirement of the package
    py_modules=["quicksample"],             # Name of the python package
    package_dir={'':'quicksample/src'},     # Directory of the source code of the package
    install_requires=[]                     # Install other dependencies if any
)
```
--- 
and then
```
python -m pip install –-user –-upgrade setuptools wheel
```
and then
```
python setup.py sdist bdist_wheel
```
and then
```
python -m pip install — user — upgrade twine
```
and then
```
python -m twine upload dist/*
```

---
**Publishing with Poetry** 

Add PyPI token as environment variable
```
poetry config pypi-token.pypi <TOKEN>
```
then
```
poetry publish
```
that's all.

---
If you are looking for a single tool to manage 
- isolated environments, 
- dependency installation and resolution
- separation of dev and production dependencies
- packaging and installation

Poetry exists !
<br/>

*credits: [Yu Xuan Lee](https://blogs.sap.com/2022/05/08/why-you-should-use-poetry-instead-of-pip-or-conda-for-python-projects/), [Aveek Das](https://towardsdatascience.com/how-to-publish-a-python-package-to-pypi-7be9dd5d6dcd)*

---
**github.com/tab1tha**
LinkedIn: Tambe Tabitha Achere
Twitter: @TambeAchere
