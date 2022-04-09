---
title: "Optimize Docker Size Image with Python Environment"
date: 2022-04-07T00:28:02+01:00
draft: false
author: "Mangabo Kolawole"
featured_image: "dockersize.png"
---

Building Docker image with Python can be well quite heavy. 

For a multistage build for example, instead of building wheels at each, you can specify a path for the python environment once it's initialized at the first stage of the build.

```docker
ENV PATH="/opt/venv/bin:$PATH"
```
Make sure you have created the virtual environment tho.👀

```docker
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
```

Here's an example with two steps:

```docker
# first stage
FROM python:3.10-slim as builder

WORKDIR /app

ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

RUN pip install virtualenv

RUN virtualenv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt .
RUN pip install -r requirements.txt

# another stage
FROM python:3.10-slim

COPY --from=builder /opt/venv /opt/venv

WORKDIR /app

ENV PATH="/opt/venv/bin:$PATH"
```
## Summary

In conclusion, here are the steps again 🚀: 
- Create the virtual environment in the builder image
- Copy the virtual environment to the final image
