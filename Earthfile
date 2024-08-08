VERSION 0.6
FROM mcr.microsoft.com/vscode/devcontainers/base:0-bionic
ARG DEVCONTAINER_IMAGE_NAME_DEFAULT=ghcr.io/haxefoundation/haxe_devcontainer

ARG USERNAME=vscode
ARG USER_UID=1000
ARG USER_GID=$USER_UID

ARG WORKDIR=/workspace
RUN mkdir -m 777 "$WORKDIR"
WORKDIR "$WORKDIR"

ARG --required TARGETARCH

devcontainer-library-scripts:
    RUN curl -fsSLO https://raw.githubusercontent.com/microsoft/vscode-dev-containers/main/script-library/common-debian.sh
    RUN curl -fsSLO https://raw.githubusercontent.com/microsoft/vscode-dev-containers/main/script-library/docker-debian.sh
    SAVE ARTIFACT --keep-ts *.sh AS LOCAL .devcontainer/library-scripts/

devcontainer:
    # Avoid warnings by switching to noninteractive
    ENV DEBIAN_FRONTEND=noninteractive

    ARG INSTALL_ZSH="false"
    ARG UPGRADE_PACKAGES="true"
    ARG ENABLE_NONROOT_DOCKER="true"
    ARG USE_MOBY="false"
    COPY .devcontainer/library-scripts/common-debian.sh .devcontainer/library-scripts/docker-debian.sh /tmp/library-scripts/
    RUN apt-get update \
        && /bin/bash /tmp/library-scripts/common-debian.sh "${INSTALL_ZSH}" "${USERNAME}" "${USER_UID}" "${USER_GID}" "${UPGRADE_PACKAGES}" "true" "true" \
        && /bin/bash /tmp/library-scripts/docker-debian.sh "${ENABLE_NONROOT_DOCKER}" "/var/run/docker-host.sock" "/var/run/docker.sock" "${USERNAME}" "${USE_MOBY}" \
        # Clean up
        && apt-get autoremove -y && apt-get clean -y && rm -rf /var/lib/apt/lists/* /tmp/library-scripts/

    # Setting the ENTRYPOINT to docker-init.sh will configure non-root access
    # to the Docker socket. The script will also execute CMD as needed.
    ENTRYPOINT [ "/usr/local/share/docker-init.sh" ]
    CMD [ "sleep", "infinity" ]

    # Configure apt and install packages
    RUN apt-get update \
        && apt-get install -qqy --no-install-recommends apt-utils dialog 2>&1 \
        && apt-get install -qqy --no-install-recommends \
            iproute2 \
            procps \
            sudo \
            bash-completion \
            build-essential \
            curl \
            wget \
            software-properties-common \
            direnv \
            tzdata \
            # install docker engine for using `WITH DOCKER`
            docker-ce \
        # install node
        && curl -sL https://deb.nodesource.com/setup_16.x | bash - \
        && apt-get install -qqy --no-install-recommends nodejs=16.* \
        # install ocaml and other haxe compiler deps
        && add-apt-repository ppa:avsm/ppa \
        && add-apt-repository ppa:haxe/ocaml \
        && apt-get install -qqy --no-install-recommends \
            ocaml-nox \
            camlp5 \
            opam \
            libpcre2-dev \
            zlib1g-dev \
            libgtk2.0-dev \
            libmbedtls-dev \
            ninja-build \
            libstring-shellquote-perl \
            libipc-system-simple-perl \
        #
        # Clean up
        && apt-get autoremove -y \
        && apt-get clean -y \
        && rm -rf /var/lib/apt/lists/*

    # Switch back to dialog for any ad-hoc use of apt-get
    ENV DEBIAN_FRONTEND=

    DO +INSTALL_NEKO

    COPY +earthly/earthly /usr/local/bin/
    RUN earthly bootstrap --no-buildkit --with-autocomplete

    USER $USERNAME

    # Do not show git branch in bash prompt because it's slow
    # https://github.com/microsoft/vscode-dev-containers/issues/1196#issuecomment-988388658
    RUN git config --global codespaces-theme.hide-status 1

    # Install OCaml libraries
    COPY haxe.opam .
    RUN opam init --disable-sandboxing
    RUN opam switch create 4.08.1
    RUN eval $(opam env)
    RUN opam env
    RUN opam install . --yes --deps-only --no-depexts
    RUN opam list
    RUN ocamlopt -v

    USER root

    ARG IMAGE_NAME="$DEVCONTAINER_IMAGE_NAME_DEFAULT"
    ARG IMAGE_TAG="development"
    ARG IMAGE_CACHE="$IMAGE_NAME:$IMAGE_TAG"
    SAVE IMAGE --cache-from="$IMAGE_CACHE" --push "$IMAGE_NAME:$IMAGE_TAG"

devcontainer-multiarch-amd64:
    ARG IMAGE_NAME="$DEVCONTAINER_IMAGE_NAME_DEFAULT"
    ARG IMAGE_TAG="development"
    FROM --platform=linux/amd64 +devcontainer --IMAGE_NAME="$IMAGE_NAME" --IMAGE_TAG="$IMAGE_TAG-amd64"
    SAVE IMAGE --push "$IMAGE_NAME:$IMAGE_TAG"

devcontainer-multiarch-arm64:
    ARG IMAGE_NAME="$DEVCONTAINER_IMAGE_NAME_DEFAULT"
    ARG IMAGE_TAG="development"
    FROM --platform=linux/arm64 +devcontainer --IMAGE_NAME="$IMAGE_NAME" --IMAGE_TAG="$IMAGE_TAG-arm64"
    SAVE IMAGE --push "$IMAGE_NAME:$IMAGE_TAG"

devcontainer-multiarch:
    BUILD +devcontainer-multiarch-amd64
    BUILD +devcontainer-multiarch-arm64

# Usage:
# COPY +earthly/earthly /usr/local/bin/
# RUN earthly bootstrap --no-buildkit --with-autocomplete
earthly:
    ARG --required TARGETARCH
    RUN curl -fsSL https://github.com/earthly/earthly/releases/download/v0.6.13/earthly-linux-${TARGETARCH} -o /usr/local/bin/earthly \
        && chmod +x /usr/local/bin/earthly
    SAVE ARTIFACT /usr/local/bin/earthly

INSTALL_PACKAGES:
    COMMAND
    ARG PACKAGES
    RUN apt-get update -qqy && \
        apt-get install -qqy --no-install-recommends $PACKAGES && \
        apt-get autoremove -y && apt-get clean -y && rm -rf /var/lib/apt/lists/*

INSTALL_NEKO:
    COMMAND
    ARG NEKOPATH=/neko
    COPY +neko/* "$NEKOPATH/"
    ARG PREFIX=/usr/local
    RUN bash -c "ln -s \"$NEKOPATH\"/{neko,nekoc,nekoml,nekotools} \"$PREFIX/bin/\""
    RUN bash -c "ln -s \"$NEKOPATH\"/libneko.* \"$PREFIX/lib/\""
    RUN bash -c "ln -s \"$NEKOPATH\"/*.h \"$PREFIX/include/\""
    RUN mkdir -p "$PREFIX/lib/neko/"
    RUN bash -c "ln -s \"$NEKOPATH\"/*.ndll \"$PREFIX/lib/neko/\""
    RUN ldconfig

INSTALL_HAXE:
    COMMAND
    ARG PREFIX=/usr/local
    COPY +build/haxe "$PREFIX/bin/"
    COPY std "$PREFIX/share/haxe/std"

try-neko:
    DO +INSTALL_NEKO
    RUN neko -version
    RUN nekotools

try-haxe:
    DO +INSTALL_NEKO
    DO +INSTALL_HAXE
    RUN haxe --version

neko:
    RUN set -ex && \
        case "$TARGETARCH" in \
            amd64) PLATFORM=linux64;; \
            arm64) PLATFORM=linux-arm64;; \
            *) exit 1;; \
        esac && \
        curl -fsSL https://build.haxe.org/builds/neko/$PLATFORM/neko_latest.tar.gz -o neko_latest.tar.gz && \
        tar -xf neko_latest.tar.gz && \
        mv `echo neko-*-*` /tmp/neko-unpacked
    SAVE ARTIFACT /tmp/neko-unpacked/*
    SAVE IMAGE --cache-hint

build:
    FROM +devcontainer

    USER $USERNAME

    # Build Haxe
    COPY --dir extra libs plugins src* std dune* Makefile* .

    # the Makefile calls git to get commit sha
    COPY .git .git
    ARG SET_SAFE_DIRECTORY="false"
    IF [ "$SET_SAFE_DIRECTORY" = "true" ]
        RUN git config --global --add safe.directory "$WORKDIR"
    END

    ARG ADD_REVISION
    ENV ADD_REVISION=$ADD_REVISION
    RUN opam config exec -- make -s -j`nproc` STATICLINK=1 haxe && ldd -v ./haxe
    RUN make -s package_unix && ls -l out

    ARG TARGETPLATFORM
    SAVE ARTIFACT --keep-ts ./out/* AS LOCAL out/$TARGETPLATFORM/
    SAVE ARTIFACT --keep-ts ./haxe AS LOCAL out/$TARGETPLATFORM/
    SAVE IMAGE --cache-hint

build-multiarch:
    ARG ADD_REVISION
    BUILD --platform=linux/amd64 --platform=linux/arm64 +build --ADD_REVISION=$ADD_REVISION

github-actions:
    DO +INSTALL_NEKO
    DO +INSTALL_HAXE
    RUN mkdir -p "$WORKDIR"/.github/workflows
    COPY extra/github-actions extra/github-actions
    WORKDIR extra/github-actions
    RUN haxe build.hxml
    SAVE ARTIFACT --keep-ts "$WORKDIR"/.github/workflows AS LOCAL .github/workflows

ghcr-login:
    LOCALLY
    RUN echo "$GITHUB_CR_PAT" | docker login ghcr.io -u "$GITHUB_USERNAME" --password-stdin
