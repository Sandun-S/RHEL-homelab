docker stack deploy -c ai-iot-stack.yml ai_stack
docker stack deploy -c portainer-stack.yml portainer
docker stack deploy -c homepage-stack.yml homepage
docker stack deploy -c npm-stack.yml edge
docker stack deploy -c media-stack.yml media

docker network create --driver overlay --attachable hub_net
