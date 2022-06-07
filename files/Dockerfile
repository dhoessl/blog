FROM debian:bullseye-slim

RUN apt update && apt install -y hugo git

# Create new page and install PaperMod theme
WORKDIR /root
RUN hugo new site blog
WORKDIR /root/blog
RUN git clone https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod

# Copy Hugo config
RUN rm config.toml
COPY config.yaml ./

# Run Hugo Server
CMD ["hugo", "server", "--bind=0.0.0.0", "--baseURL=https://blog.dhoessl.de", "--appendPort=false"]
