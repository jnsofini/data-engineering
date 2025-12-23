Open code spaces in local vscode

Check the following
- `python -v`
- ls
-  PS1="> " -  for shortening paths
- echo 'PS1="> "' > ~/.bashrc   - makes it to work at every restart

`docker run -it python:3.13.11-slim` with start a python slim docker container. the entry point in this container is the python interpreter. We can start a different program example `bash` via `docker run -it --entrypoint=bash python:3.13.11-slim`

With multiple docker images now stored, we can check via
`docker ps -a`. And to remove one do `docker rm name-of-container` and to remove all of them first check `docker ps -aq` and then remove them ```docker remove `docker ps -aq```.

We can execute files sitting in the directory from inside the the container. This is achieved by mounting them as volumns. Lets create file with a python program `/test/scripts.py`. To use the full path we cah use `(pwd)/test`.

Create files inside test via
```bash
touch file1.txt file2.txt file3.txt
echo "Hello from host" > file1.txt
```

`docker run -it -v $(pwd)/test:/app/test --entrypoint=bash python:3.13.11-slim` to mapp the `(pwd)/test` in the local computer to `/app/test` in the docker.

Now if we create a data frame we will see that pandas is non installed in our codespaces. So we need to install pandas. Now this brings the question of isolation. What if we want a specific version of pandas for this project. And if we also have packages that are tied to this project. Simply installing them install in the system. We create a venv.

We want to have a different python for the project as well so we now have three pythons
- system 3.12
- container python 3.13.11
- venv python 3.13

We do this with uv via `pip install uv` in the system. To set env navigate to `/workspaces/data-engineering/pipeline` run `uv init --python=3.13` Next `uv add pandas pyarrow`.
```
uv run python pipeline.py 12
```

Lets build a bare python file
```
`
FROM python:3.13.11-slim
RUN pip install pandas pyarrow
WORKDIR /code
COPY pipeline.py .
`
docker build -t test:pandas .
docker run -it --entrypoint=bash --rm  test:pandas
```

Update file to

```
FROM python:3.13.11-slim
RUN pip install pandas pyarrow
WORKDIR /code
COPY pipeline.py .

ENTRYPOINT ["python", "pipeline.py"]
```

To use uv in the docker container update the image to
```
# Start with slim Python 3.13 image
FROM python:3.13.10-slim

# Copy uv binary from official uv image (multi-stage build pattern)
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/

# Set working directory
WORKDIR /app

# Add virtual environment to PATH so we can use installed packages
ENV PATH="/app/.venv/bin:$PATH"

# Copy dependency files first (better layer caching)
COPY "pyproject.toml" "uv.lock" ".python-version" ./
# Install dependencies from lock file (ensures reproducible builds)
RUN uv sync --locked

# Copy application code
COPY pipeline.py pipeline.py

# Set entry point
ENTRYPOINT ["python", "pipeline.py"]
```