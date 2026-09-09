FROM ubuntu:22.04

RUN apt-get update && apt-get install -y --no-install-recommends \
      python3=3.10.6-1~22.04.1 \
      python3-pip=22.0.2+dfsg-1ubuntu0.7 \
      python3-dev=3.10.6-1~22.04.1 \
      build-essential=12.9ubuntu3 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /usr/src/app

COPY requirements.txt .
RUN pip3 install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python3", "app.py"]
