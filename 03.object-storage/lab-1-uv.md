---
duration: 1h
category:
  - name: LAB
components:
  - name: PYTHON
  - name: UV
platforms:
  - name: LINUX
resources:
  - title: uv (official homepage)
    url: https://docs.astral.sh/uv/
  - title: Installing uv (official documentation)
    url: https://docs.astral.sh/uv/getting-started/installation/
  - title: Faker (official homepage)
    url: https://pypi.org/project/Faker/
  - title: argparse (official homepage)
    url: https://docs.python.org/3/library/argparse.html
revisions:
  - date: 2026-09-13
    comment: Initial page
    author: david@adaltas.com
tags:
  - name: TUTORIAL
---

# Lab: project creation with uv and data generation

## Objectives

- Using UV
- Generate random datasets

## Vscode initialisation

In your Onyxia portal, go to the "My Services" page and create a new `vscode-pyspark` service.

When started, login to Vscode, choose your favorite theme betwen dark and light and complete the bootstrap process by clicking `Mark Done`.

## Terminal activation

In Vscode, press `F1` to print the "Show all commands" prompt. Start filtering the commands by entering `terminal` and select `Create new Terminal (with profile)`.

## Git project

An initial work directory is created in `/home/onyxia/work`.

```bash
pwd
#> /home/onyxia/work
```

Initialize and configure Git with your preferences.

```bash
git init
cat <<INI >.gitignore
.*
!.gitignore
INI
git config --global init.defaultBranch main
```

Initialize the Git project with your credentials. Your username and email are already set. If necessary, update them accordingly.

```bash
git config --global user.name
#> gollum
git config --global user.email
#> gollum@adaltas.com
```

## UV project creation

UV is already installed inside the the `Vscode-pyspark` service.

```bash
command -v uv
```

The project is created using `init` command.

```bash
uv init
#> Initialized project `work`
```

The following files and directories are generated:

- `.python-version`
  Simple text file that specifies which Python version a project is using.
- `.git` and `.gitignore`
  Git related directory and files.
- `pyproject.toml`
  Description of the project and its dependencies.
- `README.md`  
  Description and presentation of the project, empty on initialisation.
- `src/`
  A directory includes source code as a module directory and an `__init__.py` file.

Additionnal files are generated but on first dependency install or `uv run`.

```bash
uv run
#> ...
#> .venv
#> uv.lock
#> ...
```

- `.venv`
  Directory containing the virtual environment.
- `uv.lock`
  Lock file containing all the dependencies and their associated version.

`pyproject.toml` includes several sections:

- [project]: basic information about the project.
- [project.scripts]: command-line shortcuts
- [build-system]: specify how to build/package the project

```bash
cat pyproject.toml
#> version = "0.1.0"
#> description = "Add your description here"
#> readme = "README.md"
#> authors = [
#>     { name = "david", email = "david@adaltas.com" }
#> ]
#> requires-python = ">=3.13"
#> dependencies = []
#>
#> [project.scripts]
#> work = "work:main"
#>
#> [build-system]
#> requires = ["uv_build>=0.12.5,<0.13.0"]
#> build-backend = "uv_build"
```

## Initial git commit

```bash
git add \
  .gitignore \
  pyproject.toml \
  README.md \
  src \
  uv.lock
git commit -m "feat: initial project layout"
```

Using your GitHub account or your prefered Git provider, create a new repository, eg `https://github.com/<owner>/ece-2026-bigdata.git`.

```bash
owner="<username>"
git remote add origin "https://github.com/$owner/ece-2026-bigdata.git"
git push -u origin main
```

The `-u` argument links your local branch to a remote branch, allowing to use the shortcut `git push` or `git pull` without specifying the remote name and branch name in the future.

The `GitHub` Vscode extension opens a popup requesting permissions to connect to your GitHub account. The authorization flow generates a code and redirect the user to a GitHub page where the code must be pasted.

## Readme creation

`uv init` creates an empty `README.md` file. It is updated with an introduction and a usage section.

```bash
cat <<MD >README.md
# DBT lab

This project generates a random dataset consisting of users and orders. Scripts are written in Python and the project uses [uv](https://docs.astral.sh/uv/).

## Usage

\`\`\`bash
uv run dataset_users.py -h
#> usage: User generator [-h] [-c COUNT] [-o {csv,json,jsonline}]

#> options:
#>   -h, --help            show this help message and exit
#>   -c, --count COUNT     Number of users to generate.
#>   -o, --output {csv,json,jsonline}
#>                         Output format.
\`\`\`

MD
```

The readme is commited.

```bash
git add README.md
git commit -m "docs: project introduction and usages"
git push
```

## Readme best practices

A README file in the project directory is crucial for introducing the project, documenting its purpose, usage, and other essential information to help users understand the basics.

Several good points to have are mentioned on the [TLDP website](https://tldp.org/HOWTO/Software-Release-Practice-HOWTO/distpractice.html#readme):

1. A brief description of the project.
2. A pointer to the project website (if it has one)
3. Notes on the developer's build environment and potential portability problems.
4. An architecture introduction describing important files and subdirectories (usually [`ARCHITECTURE.md`](https://matklad.github.io//2021/02/06/ARCHITECTURE.md.html)).
5. Either build/installation instructions or a pointer to a file containing same (usually `INSTALL`).
6. Either a maintainers/credits list or a pointer to a file containing same (usually `CREDITS`).
7. Either recent project news or a pointer to a file containing same (usually `NEWS`).

## Serialization library script

First, a utility script used to serialize a dataset into CSV, JSON and JSON line format is created. The `serialize` function accept 3 format: `csv`, `json`, and `jsonline`. `jsonline` is a format where each line contains a JSON document.

```bash
cat <<PY >src/serialize.py
import csv
import json

class CsvBuilder:
    def __init__(self):
        self.data = []

    def write(self, row):
        self.data.append(row)

def serialize(dataset, output):
    if output == "":
        return dataset
    elif output == "json":
        dataset = json.dumps(dataset, default=str)
        print(dataset)
    elif output == "jsonline":
        for record in dataset:
            print(json.dumps(record, default=str))
    elif output == "csv":
        builder = CsvBuilder()
        writer = csv.writer(builder)
        writer.writerow(dataset[0].keys())
        for record in dataset:
            writer.writerow(record.values())
        print("".join(builder.data))
    else:
        raise ValueError("Unsupported output format.")
PY
```

## Project dependencies

The `src/serialize.py` script uses 2 native Python dependencies, `csv` and `json`.

The `dataset_users.py` script creates a dataset of users. It uses [Faker](https://faker.readthedocs.io/en/master/) to randomly generates the users dataset, [argparse](https://docs.python.org/3/library/argparse.html) to parse CLI arguments and serialize which is a custom script to convert an array into a CSV or a JSON format.

Dependencies in a project can be managed through `uv add` and `uv remove`. The dependency information is recorded in `pyproject.toml` instead of being scattered across scripts.

```bash
uv add faker argparse
# or
uv add "faker>=40.37.0" "argparse>=1.4.0"
# Verify
cat pyproject.toml
#> [project]
#> name = "dbt-lab"
#> version = "0.1.0"
#> description = "Add your description here"
#> readme = "README.md"
#> requires-python = ">=3.14"
#> dependencies = [
#>     "argparse>=1.4.0",
#>     "faker>=40.37.0",
#> ]
```

### Script to generate users

The `users_generate` function creates a default of 50 users serialized as JSON.

```bash
cat <<PY >>src/dataset_users.py
import argparse

from faker import Faker
from serialize import serialize


fake = Faker()
# Generate the same data set
Faker.seed(42)


def users_generate(count=50, output=""):
    users = []
    for i in range(count):
        # User generation
        user = {"uuid": fake.uuid4(), **fake.simple_profile()}
        users.append(user)
    serialize(users, output)
    return users


def main():
    parser = argparse.ArgumentParser(prog="User generator")
    parser.add_argument(
        "-c", "--count", help="Number of users to generate.", type=int, default=50
    )
    parser.add_argument(
        "-o",
        "--output",
        help="Output format.",
        default="json",
        choices=["csv", "json", "jsonline"],
    )
    args = parser.parse_args()
    users_generate(args.count, args.output)


if __name__ == "__main__":
    main()
PY
```

## Script execution

The script is executed with `uv run`.

For example, to print the command usage.

```bash
uv run src/dataset_users.py -h
#> usage: User generator [-h] [-c COUNT] [-o {csv,json,jsonline}]

#> options:
#>   -h, --help            show this help message and exit
#>   -c, --count COUNT     Number of users to generate.
#>   -o, --output {csv,json,jsonline}
#>                         Output format.
```

## Order dataset generation

The `dataset_orders.py` script creates a dataset of order records linked to users via `user_uuid`. Each order includes a fake product from a predefined list, quantity, and timestamp. Orders are distributed across an hourly timeline starting from January 1, 2020.

The `generate_orders` function creates a default of 100 orders serialized as json.

```bash
cat <<PY >src/dataset_orders.py
import argparse
import datetime

from faker import Faker
from faker.providers import DynamicProvider

from dataset_users import users_generate
from serialize import serialize

fake = Faker()
# Generate the same data set
Faker.seed(42)
products = DynamicProvider(
    provider_name="producs",
    elements=["bread", "brioche", "cookie", "croissant", "donuts", "dring"],
)
fake.add_provider(products)


def generate_orders(count=100, count_users=50, output=""):
    orders = []
    for user in users_generate(count_users):
        date_start = datetime.datetime(2020, 1, 1, 0, 0, 0, tzinfo=datetime.UTC).timestamp()
        date_end = date_start + 60 * 60
        for i in range(count):
            # User generation
            order = {
                "uuid": fake.uuid4(),
                "user_uuid": user["uuid"],
                "date": fake.date_time_between(
                    datetime.datetime.fromtimestamp(date_start, tz=datetime.UTC),
                    datetime.datetime.fromtimestamp(date_end, tz=datetime.UTC),
                ),
                "quantity": fake.pyint(min_value=1, max_value=5),
                "product": fake.producs(),
            }
            orders.append(order)
            # Generation period increment of 1 hour
            date_start = date_end
            date_end = date_start + 60 * 60
    serialize(orders, output)


def main():
    parser = argparse.ArgumentParser(prog="Orders generator")
    parser.add_argument(
        "-C",
        "--count_min",
        help="Minimum number of orders to generate per user.",
        type=int,
        default=0,
    )
    parser.add_argument(
        "-c",
        "--count_max",
        help="Maximum number of orders to generate per user.",
        type=int,
        default=100,
    )
    parser.add_argument(
        "-d",
        "--data_from",
        help="Date from",
    )
    parser.add_argument(
        "-o",
        "--output",
        help="Output format.",
        default="json",
        choices=["csv", "json", "jsonline"],
    )
    parser.add_argument(
        "-u", "--count_users", help="Number of users to generate.", type=int, default=50
    )
    args = parser.parse_args()
    generate_orders(args.count_max, args.count_users, args.output)


if __name__ == "__main__":
    main()

PY
```

## Execute the script

```bash
uv run src/dataset_orders.py -c 2 -o json | jq
#> [
#>   {
#>     "uuid": "bdd640fb-0667-4ad1-9c80-317fa3b1799d",
#>     "username": "garzaanthony",
#>     "name": "Charles Garcia",
#>     "sex": "M",
#>     "address": "908 Jennifer Squares\nRobinsonshire, KY 01352",
#>     "mail": "helenpeterson@gmail.com",
#>     "birthdate": "1935-08-15"
#>   },
#>   ...
```

## Commit

Changes are committed to Git.

```bash
git add \
  pyproject.toml \
  src/work uv.toml \
  src
git commit -m "feat: scrit generation"
git push
```
