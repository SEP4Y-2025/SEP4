# Use the official Eclipse Mosquitto image as the base
FROM eclipse-mosquitto:latest

# Copy the local configuration directory into the container's config directory.
# Ensure that the "mosquitto/config" folder exists relative to the Dockerfile.
COPY ./config/ /mosquitto/config/

# Expose the MQTT port (1883)
EXPOSE 1883

# The base image already includes an entrypoint so we can just inherit it.
