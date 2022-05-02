FROM debian:bullseye

# Enable third-part sources
RUN apt update && apt install ca-certificates -y

# Add custom apt sources.list
ADD ./sources.list /etc/apt

# Update & Upgrade apt & Install Dependencies
RUN apt update && apt upgrade -y
RUN apt install -y \
        build-essential \
        curl

# Install Rust and mdbook
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y \
        && . $HOME/.cargo/env \
        && cargo install mdbook --locked

# Clean apt
RUN apt autopurge && apt autoclean
