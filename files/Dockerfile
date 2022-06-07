FROM debian:bullseye-slim

RUN apt update && apt install -y hugo git

# Create new page and install PaperMod theme
RUN cd /root && hugo new site blog && cd blog && git clone https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod

# Copy Hugo config
RUN rm -rf config.toml
COPY config.yaml ./

# Run Hugo Server
CMD ["hugo", "server", "--bind=0.0.0.0"]
