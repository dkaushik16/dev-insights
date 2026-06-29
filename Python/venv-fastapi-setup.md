# FastAPI Project Setup Fundamentals

> A concise guide to understanding Python virtual environments, FastAPI setup, package management, and the roles of FastAPI, Uvicorn, and Pydantic.

---

# Table of Contents

1. Why Virtual Environments?
2. Project Setup
3. Understanding `venv`
4. Selecting the Local Interpreter
5. Installing Dependencies
6. Role of Each Library
7. Running the Application
8. Understanding `requirements.txt`
9. Why `pip freeze`?
10. Node.js vs Python Comparison
11. Typical Project Structure
12. Complete Workflow

---

# 1. Why Virtual Environments?

Imagine your computer has one global Python installation.

If every project installs packages into that global Python:

* Project A may require FastAPI 0.116
* Project B may require FastAPI 0.118
* Updating one project could break another.

To solve this, Python provides **Virtual Environments (venv).**

A virtual environment creates an isolated Python installation for a single project.

Each project can have:

* Its own Python interpreter
* Its own pip
* Its own installed packages

without affecting any other project.

---

# 2. Project Setup

Create a project

```bash
mkdir fastapi-demo
cd fastapi-demo
```

Create a virtual environment

```bash
python -m venv venv
```

Activate it

### Windows

```bash
venv\Scripts\activate
```

### Linux / macOS

```bash
source venv/bin/activate
```

---

# 3. Understanding `venv`

A virtual environment is much more than just installed libraries.

Example structure

```text
venv/
│
├── Scripts/ (Windows)
│   ├── python.exe
│   ├── pip.exe
│   └── activate
│
├── Lib/
│   └── site-packages/
│       ├── fastapi/
│       ├── uvicorn/
│       ├── pydantic/
│       └── ...
```

It contains:

* Python Interpreter
* pip
* Installed Packages
* Activation Scripts

---

## site-packages

This folder contains all installed Python libraries.

Example

```text
site-packages/
    fastapi/
    uvicorn/
    pydantic/
```

This is roughly equivalent to:

```text
node_modules/
```

---

# 4. Selecting the Local Interpreter

After creating a virtual environment, VS Code (or another IDE) may still use the **global Python interpreter**.

If that happens:

```python
import fastapi
```

may show

```text
ModuleNotFoundError
```

even though FastAPI is installed inside the virtual environment.

### Why?

Because the IDE is looking here:

```text
Global Python
```

instead of

```text
Project → venv → python.exe
```

Always select the Python interpreter from your project's `venv`.

Example

```
Python: Select Interpreter

✔ venv/Scripts/python.exe
```

Once selected:

* IntelliSense works
* Imports work
* Installed packages are detected
* Running the project uses the correct environment

---

# 5. Installing Dependencies

Install FastAPI

```bash
pip install fastapi
```

Install Uvicorn

```bash
pip install "uvicorn[standard]"
```

or together

```bash
pip install fastapi "uvicorn[standard]"
```

---

# 6. Role of Each Library

## FastAPI

FastAPI is the web framework.

Equivalent in Node.js:

```
Express
```

Responsibilities

* Routing
* Request handling
* Dependency Injection
* Swagger Documentation
* API Development

---

## Pydantic

FastAPI automatically installs Pydantic because it depends on it.

Usually you do NOT install it manually.

Pydantic validates incoming data.

Example

```python
from pydantic import BaseModel

class User(BaseModel):
    name: str
    age: int
```

If client sends

```json
{
    "name":"Tushar",
    "age":"22"
}
```

Pydantic converts

```
"22"
```

into

```
22
```

If client sends

```json
{
    "name":"Tushar",
    "age":"abc"
}
```

FastAPI automatically returns an error.

Your route function never executes.

### Why is this powerful?

Without Pydantic (Express)

```javascript
if(!name)
...

if(typeof age !== "number")
...
```

You manually validate everything.

With Pydantic

```python
class User(BaseModel):
    name:str
    age:int
```

Validation is automatic.

---

### Mongoose vs Pydantic

Mongoose

* Defines Database Schema
* Validates before saving to MongoDB

Pydantic

* Defines API Request/Response Schema
* Validates before your route executes

Think of Pydantic as

```
Zod / Joi + TypeScript types
```

built into FastAPI.

---

## Uvicorn

FastAPI creates your API.

Uvicorn runs it.

Node analogy

```
node app.js
```

Python

```bash
uvicorn main:app --reload
```

Meaning

```
main.py

↓

app = FastAPI()
```

`--reload`

Automatically restarts the server whenever files change.

---

# 7. Running the Application

```bash
uvicorn main:app --reload
```

Open

```
http://127.0.0.1:8000
```

Swagger Docs

```
http://127.0.0.1:8000/docs
```

---

# 8. Understanding `requirements.txt`

Python's traditional dependency file.

Example

```text
fastapi==0.117.0
uvicorn==0.36.0
pydantic==2.11.0
```

Another developer installs everything using

```bash
pip install -r requirements.txt
```

---

# 9. Why `pip freeze`?

`pip freeze`

Lists every package installed in the current virtual environment.

Example

```bash
pip freeze
```

Output

```text
fastapi==...
uvicorn==...
starlette==...
pydantic==...
```

Save it

```bash
pip freeze > requirements.txt
```

This records all dependencies so another machine can recreate the same environment.

---

## What if we don't create `requirements.txt`?

Your application works on your computer because packages already exist inside your virtual environment.

Someone else clones your project.

They run

```bash
python main.py
```

Result

```text
ModuleNotFoundError
```

because Python has no record of which packages are required.

Unlike npm, pip does NOT automatically create a dependency file.

---

# 10. Node.js vs Python Comparison

| Node.js             | Python                                  |
| ------------------- | --------------------------------------- |
| Express             | FastAPI                                 |
| node                | Python Interpreter                      |
| node_modules        | site-packages                           |
| package.json        | requirements.txt (traditional)          |
| package-lock.json   | Pinned versions inside requirements.txt |
| npm install         | pip install -r requirements.txt         |
| npm install express | pip install fastapi                     |
| node app.js         | uvicorn main:app                        |

---

## Important Difference

Node

```
npm install express
```

automatically updates

```
package.json
```

Python

```
pip install fastapi
```

does **not** create or update any dependency file.

You must explicitly generate

```bash
pip freeze > requirements.txt
```

---

# 11. Typical Project Structure

```text
fastapi-demo/
│
├── app/
│   └── main.py
│
├── venv/
│
├── requirements.txt
│
├── .gitignore
│
└── README.md
```

Do NOT commit

```
venv/
```

Instead commit

```
requirements.txt
```

Anyone can recreate the environment using

```bash
pip install -r requirements.txt
```

---

# 12. Complete Workflow

```text
Create Project
      │
      ▼
Create Virtual Environment
      │
      ▼
Activate Virtual Environment
      │
      ▼
Select Local Python Interpreter in IDE
      │
      ▼
Install FastAPI + Uvicorn
      │
      ▼
Develop API
      │
      ▼
Pydantic validates incoming requests
      │
      ▼
Run using Uvicorn
      │
      ▼
Freeze dependencies
      │
      ▼
Generate requirements.txt
      │
      ▼
Share project
      │
      ▼
Others recreate the environment with

pip install -r requirements.txt
```

---

# Key Takeaways

* A **virtual environment (venv)** isolates your project's Python interpreter, pip, and installed packages.
* **site-packages** is the Python equivalent of **node_modules**.
* Always configure your IDE to use the project's **local interpreter** (`venv`) instead of the global Python installation.
* **FastAPI** is the web framework.
* **Pydantic** validates and converts request/response data before your route executes.
* **Uvicorn** is the ASGI server that runs your FastAPI application.
* Traditional `pip` does **not** track dependencies automatically.
* Use `pip freeze > requirements.txt` to capture project dependencies.
* Share `requirements.txt`, **not** the `venv` folder.
* Modern tools like **uv** and **Poetry** automatically manage dependencies using `pyproject.toml`, similar to how `package.json` works in Node.js.
