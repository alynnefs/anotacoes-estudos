FROM debian

LABEL app="Giropops"
ENV ALYNNE="COMYE2N"

RUN apt-get update && apt-get install -y stress && apt-get clean

CMD stress --cpu 1 --vm-bytes 64M --vm 1
