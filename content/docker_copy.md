---
layout: page
title: A mistake I made copying files in Dockerfile
permalink: /tools/docker/copy
---

[Copying](https://docs.docker.com/engine/reference/builder/#copy) files is an essential step for many Docker builds. In my case, I wanted to copy a whole bunch of subdirectories and exclude stuff like local secrets and builds.

I thought I could exclude files and directories easily to with `.dockerignore`. But reality was different.[^1] When I started up a shell in my image and `ls`'d, I was horrified. I found my local secrets in the image. I also noticed that my application behaved oddly.

I identified two issues.[^2] The first issue is secret exposure risk is higher because the secret is just sitting in the image. The second issue is inconsistent image builds due to unintended dependencies on local build files.

I was bummed that `.dockerignore` didn't work, but I needed to find another solution to fix the above issues. The `COPY` command doesn't have a way to copy with exclusion. One solution would be to manually specify the exact directories and files with `COPY`. This solution works but I didn't want to litter my `dockerfile` with dozens of `COPY` commands.

I thought about ways to get my initial idea with `.dockerignore` to work, since I didn't like the thought of having to update the `COPY` commands every time the directory structure changed. Then I learned that [`tar`](https://linux.die.net/man/1/tar) could easily exclude files and directories with the `--exclude-from=FILE` flag.

By combining `tar` and `docker build`, I can achieve my desired outcome.

Here is the build script:

```python
import re
import subprocess
from datetime import datetime, UTC
from pathlib import Path

script_directory = Path(__file__).resolve().parent
root_directory = script_directory.parents[1]
dockerfile_path = script_directory / 'Dockerfile'
dockerignore_path = script_directory / '.dockerignore'
src_directory = root_directory / '<src directory>'
target_path = root_directory / '.build' / 'src.tar.gz'

repo = '<container registry>'
name = '<image name>'
image_name = f'{repo}/{name}'
image_tag = datetime.now(UTC).strftime('%Y%m%d-%H%M%S')

pattern = re.compile(r'^([A-Za-z]):/')

def wsl_path(p: Path) -> str:
    drive = p.drive[0].lower()
    wp = re.sub(pattern, rf'/mnt/{drive}/', p.as_posix())
    return wp

zip_cmd = [
    'bash', '-c',
    ' '.join([
        'tar',
        f'--exclude-from={wsl_path(dockerignore_path)}',
        f'--directory={wsl_path(src_directory)}',
        f'--transform="s|^|app/|"',
        '-czf',
        wsl_path(target_path),
        '.',
    ]),
]

build_cmd = [
    'docker',
    'build',
    '--file', str(dockerfile_path),
    '--tag', f'{image_name}:{image_tag}',
    '--no-cache',
    '.',
]

tag_cmd = [
    'docker',
    'tag',
    f'{image_name}:{image_tag}',
    f'{image_name}:latest',
]

def build():
    target_path.parent.mkdir(parents=True, exist_ok=True)
    subprocess.run(zip_cmd, cwd=root_directory, check=True)
    subprocess.run(build_cmd, cwd=root_directory, check=True)
    subprocess.run(tag_cmd, cwd=root_directory, check=True)

build()
```

And two lines for `Dockerfile`:

```
COPY <src directory>/.build/src.tar.gz /src.tar.gz
RUN tar -xzf /src.tar.gz
```

[^1]: It's non-obvious but the interaction where all files are copied is hinted at in the ["Build with PATH"](https://docs.docker.com/engine/reference/commandline/build/#build-with-path) section in the Docker documentation.

[^2]: The issues are more relevant for local builds and less for build automation since build automation tends to start with a clean repo.