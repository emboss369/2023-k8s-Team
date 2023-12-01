FROM python:3.10-slim

WORKDIR /app

RUN apt-get update
RUN apt-get install -y git
RUN git clone https://github.com/aws/aws-iot-device-sdk-python-v2.git
RUN python3 -m pip install --user ./aws-iot-device-sdk-python-v2

COPY SimulatedTemperatureSensor.py ./aws-iot-device-sdk-python-v2/samples/SimulatedTemperatureSensor.py

WORKDIR /app/aws-iot-device-sdk-python-v2/samples/

ENV THING_NAME="k3s_iot_client"
ENV MQTT_TOPIC="clients/k3s_iot_client/hello/world"
ENV CA_FILE="/app/certs/AmazonRootCA1.pem"
ENV PEM_CRT="/app/certs/device.pem.crt"
ENV PEM_KEY="/app/certs/private.pem.key"
ENV REGION="ap-southeast-2"
ENV VERBOS="Warn"
ENV MAX_PUB_OPS="36000"

CMD [ "sh", "-c", "python3", "-u", "SimulatedTemperatureSensor.py", "--thing_name", "$THING_NAME", "--topic", "$MQTT_TOPIC", "--ca_file", "$CA_FILE", "--cert", "$PEM_CRT", "--key", "$PEM_KEY", "--region", "$REGION", "--verbosity", "${VERBOS}", "--max_pub_ops", "$MAX_PUB_OPS" ]