FROM --platform=${BUILDPLATFORM} rust:slim as builder
ARG HTTPS_CRYPTO=rustls
# predefined variables supported by BuildKit
ARG TARGETARCH
ARG BUILDARCH

ENV DEBIAN_FRONTEND noninteractive

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    curl \
    gpg \
    gnupg-agent \
    git

RUN if [ "${TARGETARCH}" = "amd64" ]; then curl -fsSL https://download.01.org/intel-sgx/sgx_repo/ubuntu/intel-sgx-deb.key | \
    gpg --dearmor --output /usr/share/keyrings/intel-sgx.gpg && \
    echo 'deb [arch=amd64 signed-by=/usr/share/keyrings/intel-sgx.gpg] https://download.01.org/intel-sgx/sgx_repo/ubuntu focal main' | \
    tee /etc/apt/sources.list.d/intel-sgx.list; fi && \
    apt-get update && \
    apt-get install -y --no-install-recommends \
    libclang-dev \
    libprotobuf-dev \
    libssl-dev \
    make \
    perl \
    pkg-config \
    protobuf-compiler \
    wget \
    clang \
    cmake \
    libtss2-dev && \
    if [ "${TARGETARCH}" = "amd64" ]; then apt-get install -y --no-install-recommends \
    libsgx-dcap-quote-verify-dev \
    libtdx-attest-dev; fi

RUN wget https://go.dev/dl/go1.20.1.linux-${BUILDARCH}.tar.gz && \
    tar -C /usr/local -xzf go1.20.1.linux-${BUILDARCH}.tar.gz
ENV PATH="/usr/local/go/bin:${PATH}"

# To support cross-compilation
RUN if [  "${BUILDARCH}" != "${TARGETARCH}" ]; then apt-get install -y --no-install-recommends \
    gcc-"${TARGETARCH}"-linux-gnu \
    libc6-dev-"${TARGETARCH}"-cross && \
    rustup target add "${TARGETARCH}-unknown-linux-gnu" && \
    go env -w CC="${TARGETARCH}"-linux-gnu-gcc; fi

# Build and Install KBS
WORKDIR /usr/src/kbs
COPY . .

RUN https_crypto_feature="${HTTPS_CRYPTO}" && \
    if [ "${TARGETARCH}" = "s390x" ]; then https_crypto_feature="openssl"; fi && \
    if [ "${BUILDARCH}" != "${TARGETARCH}" ]; then \
        RUSTFLAGS=" -C linker=${TARGETARCH}-linux-gnu-gcc" CGO_ENABLED=1 GOARCH="${TARGETARCH}" \
        cargo install --locked --path kbs/src/kbs --no-default-features --features coco-as-builtin,resource,opa,${https_crypto_feature} \
        --target "${TARGETARCH}-unknown-linux-gnu"; \
    else \
        cargo install --locked --path kbs/src/kbs --no-default-features --features coco-as-builtin,resource,opa,${https_crypto_feature}; \
    fi

FROM --platform=${TARGETPLATFORM} ubuntu:22.04
# predefined variables supported by BuildKit
ARG TARGETARCH

RUN apt-get update && \
    apt-get install -y \
    clang \
    curl \
    gnupg-agent

# Install TDX Runtime Dependencies
RUN if [ "${TARGETARCH}" = "amd64" ]; then curl -fsSL https://download.01.org/intel-sgx/sgx_repo/ubuntu/intel-sgx-deb.key | \
    gpg --dearmor --output /usr/share/keyrings/intel-sgx.gpg; fi
RUN if [ "${TARGETARCH}" = "amd64" ]; then echo 'deb [arch=amd64 signed-by=/usr/share/keyrings/intel-sgx.gpg] https://download.01.org/intel-sgx/sgx_repo/ubuntu jammy main' | tee /etc/apt/sources.list.d/intel-sgx.list; fi
RUN apt-get update
RUN if [ "${TARGETARCH}" = "amd64" ]; then apt-get install -y --no-install-recommends \
    libsgx-dcap-default-qpl \
    libsgx-dcap-quote-verify \
    tpm2-tools; fi

# Intel PCCS URL Configurations
# If you want the AS in KBS to connect to your customized PCCS for Intel TDX/SGX evidence verification,
# please modify this parameter.
# Default using localhost PCCS (Run in Host which the container land on).
ENV INTEL_PCCS_URL "https://localhost:8081/sgx/certification/v4/"
ENV INTEL_PCCS_USE_SECURE_CERT false

# Setup Intel PCCS URL
RUN if [ "${TARGETARCH}" = "amd64" ]; then sed -i "s|\"pccs_url\":.*$|\"pccs_url\":$INTEL_PCCS_URL,|" /etc/sgx_default_qcnl.conf; \
    sed -i "s/\"use_secure_cert\":.*$/\"use_secure_cert\":$INTEL_PCCS_USE_SECURE_CERT,/" /etc/sgx_default_qcnl.conf; fi

COPY --from=builder /usr/local/cargo/bin/kbs /usr/local/bin/kbs
