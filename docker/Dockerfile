# OVMS Build Environment
# Based on ESP-IDF v3.3.4 for compatibility with OVMS firmware
FROM espressif/idf:v3.3.4

# Install additional dependencies
RUN apt-get update && apt-get install -y \
    git \
    wget \
    libncurses-dev \
    flex \
    bison \
    gperf \
    python3 \
    python3-pip \
    python3-setuptools \
    cmake \
    ninja-build \
    ccache \
    libffi-dev \
    libssl-dev \
    dfu-util \
    libusb-1.0-0 \
    && rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /project

# Default command
CMD ["/bin/bash"]
