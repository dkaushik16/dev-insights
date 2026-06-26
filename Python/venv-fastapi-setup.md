# Python Virtual Environments (venv) and FastAPI Setup Guide

## A Node.js Developer's Perspective

---

# Table of Contents

1. Introduction
2. Understanding Python Execution
3. What is a Python Interpreter?
4. Why Virtual Environments Exist
5. What is `venv`?
6. Node.js vs Python Comparison
7. Creating a New Python Project
8. Creating and Activating a Virtual Environment
9. Installing Packages
10. Understanding the Project Structure
11. VS Code Interpreter Configuration
12. Running FastAPI Applications
13. Common Errors and Solutions
14. Understanding `__pycache__`
15. Git and Virtual Environments
16. Working Across Different IDEs
17. Typical Developer Workflow
18. Key Takeaways

---

# 1. Introduction

When coming from a Node.js background, Python's environment management can initially feel confusing.

The main concepts to understand are:

* Python Interpreter
* Virtual Environments (`venv`)
* Package Installation
* IDE Interpreter Selection

Once understood, the workflow becomes straightforward and predictable.

---

# 2. Understanding Python Execution

Python code does not execute directly.

It is executed by a Python interpreter.

Example:

```bash
python main.py
```

Here:

```text
python
```

is the interpreter.

```text
main.py
```

is your application.

---

# 3. What is a Python Interpreter?

A Python interpreter is the program responsible for:

* Reading Python code
* Executing Python code
* Loading libraries
* Managing imports

Example:

```text
C:\Users\Lenovo\AppData\Local\Programs\Python\Python311\python.exe
```

This is a Python interpreter installation.

When you run:

```bash
python app.py
```

you are actually running:

```text
python.exe
```

which executes:

```text
app.py
```

---

# 4. Why Virtual Environments Exist

Suppose you have:

Project A:

```text
FastAPI 0.138
```

Project B:

```text
FastAPI 0.100
```

If both projects install packages globally:

```text
Global Python
```

the versions may conflict.

Python solves this through virtual environments.

---

# 5. What is `venv`?

A virtual environment is an isolated Python environment.

It contains:

* A Python interpreter
* A package manager (pip)
* Project-specific packages

Example:

```text
project/
│
├── venv/
│   ├── Scripts/
│   │   └── python.exe
│   │
│   └── Lib/
│       └── site-packages/
│
└── src/
```

The virtual environment is completely independent from the global Python installation.

---

# 6. Node.js vs Python Comparison

| Node.js              | Python            |
| -------------------- | ----------------- |
| node                 | python            |
| npm                  | pip               |
| package.json         | requirements.txt  |
| node_modules         | venv              |
| npm install          | pip install       |
| npm run dev          | python -m uvicorn |
| package dependencies | site-packages     |

---

# 7. Creating a New Python Project

Create a folder:

```bash
mkdir my-project
cd my-project
```

---

# 8. Creating and Activating a Virtual Environment

## Create Environment

```bash
python -m venv venv
```

This creates:

```text
venv/
```

---

## Activate in Command Prompt

```cmd
venv\Scripts\activate.bat
```

---

## Activate in PowerShell

```powershell
.\venv\Scripts\Activate.ps1
```

If execution policies block it:

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

---

## Activate in Git Bash

```bash
source venv/Scripts/activate
```

---

## Successful Activation

You'll see:

```text
(venv)
```

before the terminal prompt.

Example:

```text
(venv) D:\project>
```

---

# 9. Installing Packages

Example:

```bash
pip install fastapi uvicorn
```

Installed packages go inside:

```text
venv/Lib/site-packages
```

not into global Python.

---

# 10. Understanding the Project Structure

Example FastAPI Project:

```text
backend/
│
├── src/
│   └── main.py
│
├── venv/
│
├── requirements.txt
│
└── .gitignore
```

---

# 11. VS Code Interpreter Configuration

This is the step that caused the original issue.

VS Code must know which Python interpreter to use.

Open:

```text
Ctrl + Shift + P
```

Search:

```text
Python: Select Interpreter
```

Select:

```text
project/venv/Scripts/python.exe
```

Example:

```text
D:\z-repo-sense\backend\venv\Scripts\python.exe
```

---

## Why This Matters

Without selecting the interpreter:

VS Code may use:

```text
C:\Users\Lenovo\AppData\Local\Programs\Python\Python311\python.exe
```

instead.

That interpreter may not contain:

```text
fastapi
uvicorn
pydantic
```

which leads to:

```python
ModuleNotFoundError
```

or

```text
Cannot find module "fastapi"
```

errors.

---

# 12. Running FastAPI Applications

Example:

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def root():
    return {"message": "Hello FastAPI"}
```

File:

```text
src/main.py
```

Run:

```bash
python -m uvicorn src.main:app --reload
```

---

## Explanation

```text
src.main
```

means:

```text
src/main.py
```

---

```text
app
```

means:

```python
app = FastAPI()
```

---

## Open in Browser

API:

```text
http://127.0.0.1:8000/
```

Swagger Docs:

```text
http://127.0.0.1:8000/docs
```

ReDoc:

```text
http://127.0.0.1:8000/redoc
```

---

# 13. Common Errors and Solutions

## Error

```text
No module named fastapi
```

### Cause

Wrong interpreter selected.

### Fix

Select the project's virtual environment interpreter.

---

## Error

```text
No module named uvicorn
```

### Cause

Uvicorn not installed in the active environment.

### Fix

```bash
pip install uvicorn
```

---

## Error

```text
Could not import module "main"
```

### Cause

Wrong module path.

Example:

Wrong:

```bash
uvicorn main:app
```

Correct:

```bash
uvicorn src.main:app
```

---

# 14. Understanding `__pycache__`

Python automatically compiles modules into bytecode.

Example:

```text
__pycache__/
```

contains:

```text
module.cpython-311.pyc
```

files.

Purpose:

* Faster imports
* Faster startup

Safe to delete:

```text
Yes
```

Python recreates it automatically.

---

# 15. Git and Virtual Environments

Do NOT commit:

```text
venv/
__pycache__/
```

Example `.gitignore`:

```gitignore
venv/
__pycache__/
*.pyc
```

---

# 16. Working Across Different IDEs

Examples:

* VS Code
* Cursor
* Windsurf
* PyCharm

When opening the project:

Select:

```text
venv/Scripts/python.exe
```

once.

Most IDEs remember the choice.

---

# 17. Typical Developer Workflow

## New Project

```bash
mkdir project
cd project
```

---

Create venv:

```bash
python -m venv venv
```

---

Activate:

```bash
source venv/Scripts/activate
```

---

Install dependencies:

```bash
pip install fastapi uvicorn
```

---

Save dependencies:

```bash
pip freeze > requirements.txt
```

---

Run:

```bash
python -m uvicorn src.main:app --reload
```

---

# 18. Key Takeaways

1. Python applications run through a Python interpreter.

2. A virtual environment (`venv`) creates an isolated Python environment.

3. Packages installed inside `venv` are not available to global Python.

4. VS Code must use the project's virtual environment interpreter.

5. Most import-related issues come from using the wrong interpreter.

6. `venv` is conceptually similar to `node_modules`, but it also includes its own Python interpreter.

7. Never commit `venv` or `__pycache__` to Git.

8. For FastAPI projects:

```bash
python -m uvicorn src.main:app --reload
```

is the most common development command.

9. Swagger documentation is automatically available at:

```text
http://127.0.0.1:8000/docs
```

10. The most important rule:

> Always activate the virtual environment and ensure your IDE is using the project's interpreter.
