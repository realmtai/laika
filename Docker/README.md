
# ══════════════════════════════════════════════════════════════════════════════
# Go2 Robot Dev Environment (Fully Integrated Stack)
#
# Builds CycloneDDS from source, compiles unitree_sdk2, installs unitree_sdk2_python,
# and builds the custom rmw_cyclonedds workspace.

# Build Command:

docker build -f Dockerfile.go2 -t go2-bot .

# Run Command:

sudo docker run --rm -it \
-v "$(pwd)/config:/config" \
--network host \
go2-bot
