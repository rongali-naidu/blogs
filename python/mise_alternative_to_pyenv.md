### `mise` – An Alternative to `pyenv` and `nvm`

I usually work with **Python for data pipelines** and **Node.js for TypeScript-based AWS CDK** development. So my daily setup often includes:

- `pyenv` for managing different **Python** versions  
- `pip` for Python package management  
- `nvm` for switching between **Node.js** versions  
- `npm` for installing Node packages  

This combo has served me well. But today, I came across something new: **[mise](https://mise.jdx.dev/dev-tools/)** — a unified version manager that can replace both `pyenv` and `nvm`. 

---

### `mise` as Replacement for `pyenv`

Just like [`pyenv`](https://github.com/pyenv/pyenv), `mise` can manage multiple Python versions with ease. You can install and switch Python versions, and it works seamlessly with `pip`, the default package manager bundled with Python.

More details here 👉 [mise + Python](https://mise.jdx.dev/lang/python.html)

```bash
mise install python@3.12
mise use python@3.12
python --version
pip --version
```

---

### `mise` as a Replacement for `nvm`

Similarly, [`mise`](https://mise.jdx.dev/lang/node.html) supports managing Node.js versions—just like `nvm`. And since **Node ships with `npm`**, you're all set once you've installed your desired Node version with `mise`.

```bash
mise install node@20
mise use node@20
node --version
npm --version
```

---

### Final Thoughts

Giving it a shot and exploring what flexibility i will be missing compared to `pyenv` and `nvm`

