FROM cgr.dev/chainguard/python:latest-dev AS builder
WORKDIR /app

RUN python -m venv venv
ENV PATH="/app/venv/bin:$PATH"

COPY requirements.txt .
RUN pip install --no-cache -r requirements.txt

FROM cgr.dev/chainguard/python:latest
WORKDIR /app

COPY --from=builder /app/venv /app/venv
ENV PATH="/app/venv/bin:$PATH"

COPY app.py .
COPY templates/ templates/
COPY static/ static/

EXPOSE 5000
ENTRYPOINT ["/app/venv/bin/python"]
CMD ["-m", "flask", "run", "--host=0.0.0.0"]
