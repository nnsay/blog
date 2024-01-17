---
title: Source Code Defense
date: 2024-01-17 17:26:33
tags:
  - 研究
excerpt: How to do the source code defence?
---

# 1. Topic

defende the source code in commercial distribution.

- [How do you protect Python source code?](https://wiki.python.org/moin/Asking%20for%20Help/How%20do%20you%20protect%20Python%20source%20code%3F)
  - Use Compiled Bytecode
  - Executable Creators (or Installers)
  - Software as a Service
  - Python Source Code Obfuscators
- Which language?
  - nodejs
  - python

# 2. Python

- [sourcedefender(commercial)](https://pypi.org/project/sourcedefender/): SOURCEdefender can protect your plaintext Python source code with AES 256-bit Encryption. There is no impact on the performance of your running application as the decryption process takes place during the import of your module or when loading your script on the command-line. Encrypted code won't run any slower once loaded from a .pye file compared to loading from a .py or .pyc file
- [pyarmor(commercial)](https://pypi.org/project/pyarmor/): A tool used to obfuscate python scripts, bind obfuscated scripts to fixed machine or expire obfuscated scripts.
- [PyObfuscator](https://github.com/mauricelambert/PyObfuscator/): This module obfuscates python code.
- [pyconcrete](https://github.com/Falldog/pyconcrete): Protect your python script, encrypt it as .pye and decrypt when import it
- [py_complie](https://docs.python.org/3/library/py_compile.html): The [`py_compile`](https://docs.python.org/3/library/py_compile.html#module-py_compile) module provides a function to generate a byte-code file from a source file, and another function used when the module source file is invoked as a script.
- [pyinstaller](https://pyinstaller.org/en/stable/): PyInstaller bundles a Python application and all its dependencies into a single package. The user can run the packaged app without installing a Python interpreter or any modules. PyInstaller supports Python 3.8 and newer, and correctly bundles many major Python packages such as numpy, matplotlib, PyQt, wxPython, and others.

# 3. NodeJs

- [javascript-obfuscator](https://github.com/javascript-obfuscator/javascript-obfuscator): JavaScript Obfuscator is a powerful free obfuscator for JavaScript, containing a variety of features which provide protection for your source code.
- [uglifyjs](https://github.com/mishoo/UglifyJS): UglifyJS is a JavaScript parser, minifier, compressor and beautifier toolkit.
- [pkg](https://github.com/vercel/pkg): Package your Node.js project into an executable
- [bytenode](https://github.com/bytenode/bytenode): A minimalist bytecode compiler for Node.js

# 4. Test

## 4.1 python

Skip the commercial tool test because they need to register or license, so just include OSS. All test code can be find at [sourcedefender-tutorial](https://github.com/nnsay/sourcedefender-tutorial).

Setup the virtual python environment before testing:

```bash
python3.11 -m venv .venv
source .venv/bin/activate
```

- pyconcrete(★★★★)

  ```bash
  # install, NOTE: https://github.com/Falldog/pyconcrete/issues/94
  PYCONCRETE_PASSPHRASE=sourcedefender CFLAGS="-Wno-implicit-function-declaration" pip install pyconcrete # https://github.com/Falldog/pyconcrete/

  # compile, find more info by run: pyconcrete-admin.py --help
  pyconcrete-admin.py compile --source main.py --pye
  pyconcrete-admin.py compile --source main.py --pyc

  # run
  pyconcrete main.pye
  python main.pyc
  ```

  > ⚠️ PYCONCRETE_PASSPHRASE is required when install pyconcrete

- PyObfuscator(★★)

  ```bash
  # obfuscate, find more info by: PyObfuscator -h
  PyObfuscator main.py --output main-obfuscated.py
  # obfuscate and generate deobfuscate.json
  PyObfuscator main.py --output main-obfuscated.py --deobfuscate

  # run
  python main-obfuscated.py
  ```

  `main-obfuscated.py`:

  ![image-20240116173002242](https://img.picgo.net/2024/01/16/20240116172935c8eb692221454f64.png)

  `deobfuscate.json`:

  ![image-20240116173123459](https://img.picgo.net/2024/01/16/image-20240116173123459ab6db12a0d8d3fb2.png)

  - py_complie(★)

    ```bash
    # compile
    python -m py_compile main.py # or
    python -m compileall main.py

    # run
    python ./__pycache__/main.cpython-311.pyc
    ```

  - pyinstaller(★★★★★)

    ```bash
    # compile
    pyinstaller --onefile main.py

    # run
    ./dist/main
    ```

  **Summary**:

  - `pyconcrete` is safer than `PyObfuscator` because pyconcrete result is binary file, but its dependences are not bundled
  - `PyObfuscator` defendes the source code with obfuscation, while `pyconcrete` utilizes compile technology
  - `pyinstaller` binary result size is big because its dependences are bundled

## 4.2 nodejs

- uglify.js(★)

  ```bash
  npm install uglify-js -g

  # obfuscate file and compare the output with source file
  uglifyjs --compress --mangle -- dist/libs/infra-config/src/lib/api-config.js
  ```

- javascript-obfuscator(★★★)

  ```bash
  npm install javascript-obfuscator -g

  # obfuscate file
  javascript-obfuscator dist/apps/demo-service/main.js

  # run obfuscated nest application and test the apis
  node dist/apps/demo-service/main-obfuscated.js
  ```

  > main.js is the output result for a nest application building with webpack

- pkg

  ```bash
  npm install pkg -g

  # compile
  pkg --target node18-linux-x64 main.js

  # run
  ./main-linux # error
  ```

  > ⚠️ the `pkg` has been archived.
  >
  > ❌ [comiple failure](https://github.com/vercel/pkg/issues/641) on darwin(m2) and linux(x64) with node@18
  >
  > ❌ execute error when run the linux binary

- bytenode(★★★★)

  ```bash
  npm install -g bytenode

  # compile
  ENV_NAME=dev bytenode -c dist/apps/demo-service/main.js

  # run
  ENV_NAME=dev bytenode dist/apps/demo-service/main.jsc
  ```

**Summary**:

- `javascript-obfuscator` obfuscate file is harder to read than `uglifyjs`, but the log also is obfuscated if your class name and function name in it
- `uglifyjs` is not a obfuscator
- `bytenode` compile the binary result which is better than the obfucated file

# 5. Is python compilation safe?

There many tool to decompile:

- [pycdc](https://github.com/zrax/pycdc): C++ python bytecode disassembler and decompiler
- [others](https://medium.com/@bolexzy/decompiling-a-compiled-python-pyc-file-crackme4-edad72784c7e)

```bash
# install
git clone https://github.com/zrax/pycdc
cd pycdc
cmake .
make
make check
cp pycdc ~/.local/bin

# decompile
pycdc __pycache__/main.cpython-311.pyc

```

The decompile result:

```python
# Source Generated with Decompyle++
# File: main.cpython-311.pyc (Python 3.11)

import requests

def get_github_status():
Unsupported opcode: PUSH_EXC_INFO
    api_url = 'https://www.githubstatus.com/api/v2/status.json'
    response = requests.get(api_url)
    if response.status_code == 200:
        status_data = response.json()
        status = status_data.get('status')
        description = status_data.get('body', { }).get('markdown')
        print(f'''GitHub Status: {status}''')
        print(f'''Status Description: {description}''')
        return None
    None(f'''Failed to retrieve GitHub status. Status code: {response.status_code}''')
    return None
# WARNING: Decompyle incomplete

if __name__ == '__main__':
    get_github_status()
    return None
```

**Summary**:

The decompile result is very close to real code, so the `python compile` is not 100% safe.
