FROM mcr.microsoft.com/devcontainers/python:2.0.2-3.13-trixie

ENV PYTHONUNBUFFERED=1

ARG USER_UID=1000
ARG USER_GID=$USER_UID
RUN if [ "$USER_GID" != "1000" ] || [ "$USER_UID" != "1000" ]; then groupmod --gid $USER_GID vscode && usermod --uid $USER_UID --gid $USER_GID vscode; fi

# Copy install script to bin:
COPY ./dev_install /bin

CMD ["sleep", "infinity"]
