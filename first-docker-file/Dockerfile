############################################################
# Original version (kept for reference)
#
# FROM ubuntu:latest
#
# WORKDIR /app
#
# COPY . /app
#
# RUN apt-get update && apt-get install -y python3 python3-pip
#
# ENV NAME World
#
# CMD ["python3", "app.py"]
############################################################

FROM python:3.11-slim

WORKDIR /app

# Copy only what's needed
COPY app.py /app/

# Optional: smaller, cleaner runtime
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    NAME=World

CMD ["python", "app.py"]