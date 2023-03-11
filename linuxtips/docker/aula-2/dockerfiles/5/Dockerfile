FROM golang AS buildando

WORKDIR /app
ADD . /app
RUN go build -o meugo

# o primeiro FROM é referente ao código até aqui

FROM alpine
WORKDIR /giropops
COPY --from=buildando /app/meugo /giropops/

# vai copiar o meugo da primeira camada para o giropops da segunda camada da imagem 

ENTRYPOINT ./meugo
