FROM alpine:latest
# Install isync (which contains the mbsync executable) and CA certificates for SSL
RUN apk add --no-cache isync ca-certificates
# Create the virtual user (ID 1000) inside the container
RUN adduser -D -u 1000 -h /home/app appuser
WORKDIR /home/app
USER appuser
# Default command (can be overridden)
CMD ["mbsync", "-a"]
