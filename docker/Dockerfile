FROM debian:bullseye

# Enable third-part sources
RUN apt update && apt install ca-certificates -y

# Add custom apt sources.list
ADD ./sources.list /etc/apt

# Update & Upgrade apt
RUN apt update && apt upgrade -y

# Install Dependencies

# Clean apt
RUN apt autopurge && apt autoclean
