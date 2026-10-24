# Imagen mínima para probar container-image.yml en el CI de este repo.
FROM busybox:1.37
ARG MENSAJE=sin-argumento
RUN echo "$MENSAJE" > /mensaje
CMD ["cat", "/mensaje"]
