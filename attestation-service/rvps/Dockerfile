# Copyright (c) 2023 by Alibaba.
# Licensed under the Apache License, Version 2.0, see LICENSE for details.
# SPDX-License-Identifier: Apache-2.0

FROM --platform=${BUILDPLATFORM} rust:latest as builder
# predefined variables supported by BuildKit
ARG TARGETARCH
ARG BUILDARCH

WORKDIR /usr/src/rvps

COPY . .

RUN apt-get update && apt-get install protobuf-compiler -y

# To support cross-compilation
RUN if [  "${BUILDARCH}" != "${TARGETARCH}" ]; then apt-get install -y --no-install-recommends \
    gcc-"${TARGETARCH}"-linux-gnu \
    libc6-dev-"${TARGETARCH}"-cross && \
    rustup target add "${TARGETARCH}-unknown-linux-gnu"; fi

RUN if [ "${BUILDARCH}" != "${TARGETARCH}" ]; then \
        RUSTFLAGS=" -C linker=${TARGETARCH}-linux-gnu-gcc" \
        cargo install --bin rvps --path attestation-service/rvps \
        --target "${TARGETARCH}-unknown-linux-gnu"; \
    else \
        cargo install --bin rvps --path attestation-service/rvps; \
    fi

FROM --platform=${TARGETPLATFORM} debian

LABEL org.opencontainers.image.source="https://github.com/confidential-containers/trustee/attestation-service"

COPY --from=builder /usr/local/cargo/bin/rvps /usr/local/bin/rvps

CMD ["rvps"]

VOLUME /opt/confidential-containers/attestation-service/reference_values/

EXPOSE 50003