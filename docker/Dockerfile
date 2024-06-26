FROM rust:slim as builder

WORKDIR /usr/src/kbs
COPY . .

ENV DEBIAN_FRONTEND noninteractive

# Install Build Dependencies
RUN apt-get update && apt-get install -y apt-utils
RUN apt-get install -y \
    clang \
    cmake \
    curl \
    gnupg \
    libclang-dev \
    libprotobuf-dev \
    libssl-dev \
    make \
    pkg-config \
    protobuf-compiler \
    wget
RUN wget https://go.dev/dl/go1.20.1.linux-amd64.tar.gz
RUN tar -C /usr/local -xzf go1.20.1.linux-amd64.tar.gz
ENV PATH="/usr/local/go/bin:${PATH}"

# Install TPM Build Dependencies
RUN apt-get install -y --no-install-recommends libtss2-dev

# Install TDX Build Dependencies
RUN curl -fsSL https://download.01.org/intel-sgx/sgx_repo/ubuntu/intel-sgx-deb.key | apt-key add -
RUN echo 'deb [arch=amd64] https://download.01.org/intel-sgx/sgx_repo/ubuntu focal main' | tee /etc/apt/sources.list.d/intel-sgx.list
RUN apt-get update
RUN apt-get install -y --no-install-recommends \
    libsgx-dcap-quote-verify-dev \
    libtdx-attest-dev

# Build and Install KBS
ARG KBS_FEATURES=coco-as-builtin,rustls,resource,opa
RUN cargo install --path src/kbs --no-default-features --features ${KBS_FEATURES}

FROM debian:stable-slim

RUN apt-get update && apt-get install -y apt-utils
RUN apt-get install -y \
    clang \
    curl \
    gnupg

# Install TDX Runtime Dependencies
RUN curl -fsSL https://download.01.org/intel-sgx/sgx_repo/ubuntu/intel-sgx-deb.key | apt-key add -
RUN echo 'deb [arch=amd64] https://download.01.org/intel-sgx/sgx_repo/ubuntu focal main' | tee /etc/apt/sources.list.d/intel-sgx.list
RUN apt-get update
RUN apt-get install -y --no-install-recommends \
    libsgx-dcap-default-qpl \
    libsgx-dcap-quote-verify \
    libtdx-attest \
    tpm2-tools

# Intel PCCS URL Configurations
# If you want the AS in KBS to connect to your customized PCCS for Intel TDX/SGX evidence verification,
# please modify this parameter.
# Default using localhost PCCS (Run in Host which the container land on).
ENV INTEL_PCCS_URL "https://localhost:8081/sgx/certification/v4/"
ENV INTEL_PCCS_USE_SECURE_CERT false

# Setup Intel PCCS URL
RUN sed -i "s|\"pccs_url\":.*$|\"pccs_url\":$INTEL_PCCS_URL,|" /etc/sgx_default_qcnl.conf; \
    sed -i "s/\"use_secure_cert\":.*$/\"use_secure_cert\":$INTEL_PCCS_USE_SECURE_CERT,/" /etc/sgx_default_qcnl.conf

COPY --from=builder /usr/local/cargo/bin/kbs /usr/local/bin/kbs
