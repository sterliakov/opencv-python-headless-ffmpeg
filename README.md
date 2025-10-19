This repository hosts a build of [opencv-python-headless](https://github.com/opencv/opencv-python)
linked against a fixed version of [FFmpeg](https://www.ffmpeg.org).

It was created with docker in mind, with the intended use along the lines of

```dockerfile
FROM python:3.11 AS base

FROM base AS fetch-ffmpeg
ARG DEBIAN_FRONTEND=noninteractive
ADD https://github.com/sterliakov/opencv-python-headless-ffmpeg/releases/latest/download/ffmpeg.tar.xz /ffmpeg.tar.xz
RUN --mount=type=cache,target=/var/lib/apt/lists/ \
	apt-get -qq -o=Dpkg::Use-Pty=0 update \
    && apt-get -qq -o=Dpkg::Use-Pty=0 install -y --no-install-recommends \
        xz-utils unzip \
    && cd /tmp \
    && mkdir ffmpeg \
    && tar xf /ffmpeg.tar.xz -C ffmpeg --strip-components 1

# Assuming you have `uv add`ed the wheel from the release
# uv add 'https://github.com/sterliakov/opencv-python-headless-ffmpeg/releases/latest/download/opencv_python_headless-4.12.0.88-cp311-cp311-linux_x86_64.whl'
FROM base AS install-deps
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/uv
COPY uv.lock pyproject.toml ./
RUN --mount=type=cache,target=/root/.cache \
    uv sync --locked --no-install-project

FROM base AS deploy
COPY --from=install-deps /.venv /.venv
COPY --from=fetch-ffmpeg --chmod=555 /tmp/ffmpeg/bin /usr/bin
COPY --from=fetch-ffmpeg --chmod=444 /tmp/ffmpeg/lib /usr/lib
```

(the whole Dockerfile above is an untested extract from a real system using these build artifacts)

Right now builds are dispatched manually for fixed ffmpeg and opencv versions, but I plan
to expand the available sources later.

The build script is available under a permissive MIT license, but it does not
cover the generated build artifacts or the used libraries. The contained
FFmpeg version is GPL-licensed, opencv-python-headless itself is also
MIT-licensed.
