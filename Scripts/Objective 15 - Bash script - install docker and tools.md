```
cd $home
sudo apt install -y docker-compose-v2 unzip python3-aiohttp nano
wget https://hhc24-elfstack.holidayhackchallenge.com/download_file/elf-stack-siem-with-logs.zip
unzip elf-stack-siem-with-logs.zip
cd elf-stack-siem
# Remove the syslog_sender.py because it is broken
echo "print('syslog_sender disabled')" > syslog-sender/assets/syslog_sender.py
echo "Removing old docker containers"
sudo docker rm -v -f $(sudo docker ps -qa)
echo "Running one-time Docker setup"
sudo docker compose up setup
echo "Starting Docker containers"
echo "Container startup typically takes about 5 minutes"
sudo docker compose up > /dev/null 2>&1 &

```