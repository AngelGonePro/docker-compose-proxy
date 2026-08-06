```
mkdir softether-vpn && \
curl -L -o /tmp/vpn.zip https://raw.githubusercontent.com/AngelGonePro/StratumVPN/refs/heads/main/Full_VPN_Config.zip && \
python3 -c "import zipfile; zipfile.ZipFile('/tmp/vpn.zip').extractall('softether-vpn')" && \
rm /tmp/vpn.zip
```
