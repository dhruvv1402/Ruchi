# Ruchi on Render's free web service (512 MB RAM, no persistent disk).
#
# Differences from the root Dockerfile, each for the host rather than the engine:
#
#   * No torch, no CUDA. The lockfile carries PyTorch and fifteen NVIDIA packages for
#     sentence-transformers, several gigabytes the server never imports: dense search is off by
#     default and the encoders are imported lazily. The lockfile is exported, those lines are dropped,
#     and the rest is installed with --no-deps so nothing pulls them back.
#   * The corpus is baked into the image, fetched from the public Hugging Face dataset, because a
#     free service has no persistent disk. Its layer sits above the code, so a code change does not
#     download 1.2 GB again.
#   * Served open, with no ORDERORDER_API_TOKEN: a public demo of public case law. Started through
#     uvicorn rather than `orderorder serve`, which rightly refuses an off-loopback binding without a
#     token everywhere else.
#   * --proxy-headers, so the rate limiter counts each visitor rather than Render's proxy.
#   * $PORT, which Render sets.

FROM python:3.12-slim

COPY --from=ghcr.io/astral-sh/uv:0.9.7 /uv /usr/local/bin/uv

ENV UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy \
    UV_PYTHON_DOWNLOADS=never

RUN apt-get update \
    && apt-get install -y --no-install-recommends libgomp1 curl ca-certificates \
    && rm -rf /var/lib/apt/lists/* \
    && useradd --create-home --uid 10001 ruchi \
    && mkdir -p /data \
    && chown ruchi:ruchi /data

ARG CORPUS_URL=https://huggingface.co/datasets/dhruvv1402/ruchi-corpus/resolve/main/orderorder.sqlite3
RUN curl -fsSL --retry 5 --retry-delay 5 "$CORPUS_URL" -o /data/orderorder.sqlite3 \
    && chown ruchi:ruchi /data/orderorder.sqlite3 \
    && ls -la /data

WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv export --frozen --no-dev --no-hashes --no-emit-project -o /tmp/all.txt \
    && grep -viE '^(torch|triton|nvidia-|cuda-)' /tmp/all.txt > /tmp/requirements.txt \
    && uv venv /app/.venv \
    && uv pip install --python /app/.venv/bin/python --no-deps -r /tmp/requirements.txt \
    && rm -rf /root/.cache /tmp/*.txt

COPY src ./src

ENV PATH="/app/.venv/bin:$PATH" \
    PYTHONPATH=/app/src \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    ORDERORDER_DATA_DIR=/data \
    HF_HOME=/data/hf \
    PORT=10000

USER ruchi
EXPOSE 10000

CMD ["sh", "-c", "exec uvicorn orderorder.web.api:app --host 0.0.0.0 --port ${PORT} --proxy-headers --forwarded-allow-ips '*' --log-level warning"]
