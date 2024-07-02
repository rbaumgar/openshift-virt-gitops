FROM quay.io/centos/centos:stream9 AS taskBuilder

RUN dnf install -y tar gzip && dnf clean all

ENV GO_VERSION="1.21.7"
RUN curl -L https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz | tar -C /usr/local -xzf -
ENV PATH=$PATH:/usr/local/go/bin

COPY . /src
WORKDIR /src
RUN mkdir /out

RUN task_names=("copy-template" "create-vm" "execute-in-vm" "generate-ssh-keys" "modify-data-object" "modify-vm-template" "wait-for-vmi-status"); \
    echo "compiling disk-virt-customize"; \
    CGO_ENABLED=0 GOOS=linux go build -mod=vendor -o /out/disk-virt-customize ./cmd/disk-virt-customize || exit 1; 

FROM quay.io/centos/centos:stream9
ENV USER_UID=1001 \
    USER_NAME=tekton-tasks \
    HOME=/home/${USER_NAME}

RUN dnf install -y xorriso openssh-clients && dnf clean all

# install task binaries
COPY --from=taskBuilder /out/. /usr/local/bin/
COPY build/bin /usr/local/bin

USER 0
RUN  /usr/local/bin/user_setup

ENTRYPOINT ["/usr/local/bin/entrypoint"]
CMD ["--help"]

USER ${USER_UID}